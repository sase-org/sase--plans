---
tier: tale
title: Migrate /sase_questions to a gate shell behind the gate_shell_handoff flag
goal:
  Behind a new `gate_shell_handoff` beta flag, a `/sase_questions` round becomes a gate
  shell in the asking agent's family — the runner ends as `DONE` instead of blocking on
  `wait_for_gate`, each round's Q&A is durably persisted on its own gate shell, and the
  answer launches the follow-up agent with the full merged Q&A rebuilt from the family's
  settled question gate shells rather than from `LoopState` RAM.
size: medium
proposed_by: bbugyi200.athena.sase-ud.10
bead: sase-ud.10
---

- **PARENT:** [202608/gate_shells.md](gate_shells.md)
- **BEAD:**
  [sase-ud.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ud/sase-ud.10.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ud.10](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ud.10.md)
- **COMMITS:**
  - [05ce87f](https://github.com/sase-org/sase/commit/05ce87fbf3d0942372ccc3b74cec299f8374af39)
    — feat(gate-shell): migrate /sase_questions to a gate shell behind
    gate_shell_handoff

# Plan: Migrate /sase_questions to a gate shell

Phase `questions-migration` of epic `sase-ud` (bead `sase-ud.10`, plan
`plan:202608/gate_shells.md`). Read the epic plan's **§5 The `shell` block**, **§6 The
follow-up prompt**, **§7 `%auto` short-circuits**, **Feature flag**, and the
`questions-migration` phase section before starting; they are requirements, not
background.

Phases 1–9 already shipped the substrate this phase consumes: `sase.shells`,
`sase.gate_shell` (creation transaction, handoff, settlement, per-branch follow-up
policy, follow-up prompt composer, `gate.log`, CLI, fork classification) and the ACE
rendering. This phase is the first migration of an **existing user-visible** consumer,
so it is the phase that creates the epic's feature flag.

## Where the state actually lives today

`handle_questions_marker` (`src/sase/axe/run_agent_exec_questions.py`) adopts the
`.sase_questions_pending` marker, calls `handle_questions_flow`
(`src/sase/axe/run_agent_helpers_questions.py`) which creates a `question` gate and then
**blocks in `wait_for_gate`**, yielding and later reacquiring its runner slot around a
`pending_question.json` marker. On the answer it appends a `QARound` to
`LoopState.qa_rounds` (in RAM), merges every round with `merge_qa_for_prompt`, saves a
chat, hands off in-process through `continue_as_successor` with
`assemble_question_followup_prompt(state.question_base_prompt, state.qa_rounds)`, and
finally rewrites the SDD prompt archive's Q&A snapshot.

Two rounds work today only because the runner never dies between them. Under gate shells
it dies every round, so `qa_rounds` and `question_base_prompt` must become durable.

**The fix (from the epic plan): the family's gate shells _are_ the accumulator.** No new
store. Each question gate shell's own bundle already holds that round's questions
(`request.json` → `payload.questions`) and its answers (`response.json` →
`option_results[submit].result` plus `feedback`). All this phase adds is a durable
_chain link_ between consecutive rounds and a rebuild that walks it.

## Design decisions this plan fixes

1. **The runner creates the gate shell during marker adoption.** `sase questions` is
   unchanged: it still writes `.sase_questions_pending` and SIGTERMs the runner group.
   The runner has already killed its provider subprocess by the time it adopts the
   marker, so it does **not** write `.sase_gate_pending` and does **not** call
   `maybe_handoff_gate_from_agent`; it creates the gate shell and terminalizes itself,
   returning the existing `"gated"` loop outcome. `move_gate_shell_claim` retitles the
   workspace claim to `ace-gate`, which is what keeps the runner's own `ace-run` exit
   cleanup from releasing it — the same protection monitors already rely on.

2. **The `submit` branch prompt is declared at creation and rebuilt at settlement.**
   `resolve_gate_followup` returns `None` for an empty prompt, so the branch must carry
   a real prompt at creation time. Declare it as
   `base_prompt + "\n\n" + build_merged_qa_markdown(prior_rounds)` — correct except that
   it cannot contain the round being asked. At settlement, a kind-keyed hook replaces it
   with the same text rebuilt across **all** rounds including this one. If the rebuild
   ever fails, the declared prompt is a sensible fallback that is short exactly one
   round, never a sentinel and never empty.

3. **`build_merged_qa_markdown`, not `merge_qa_for_prompt`, inside the composed
   prompt.** `compose_gate_followup_prompt` already wraps its whole body in
   `wrap_disabled_region`, which escapes nested `%xprompts_enabled:` markers into
   `% xprompts_enabled:`. Emitting `merge_qa_for_prompt`'s own marker pair inside it
   would leave visible escaped markers. The `%auto` in-process path keeps
   `assemble_question_followup_prompt` (with the marker pair) exactly as today, because
   that prompt is not re-expanded.

4. **Chain metadata is Python-side `agent_meta.json` only — no `AgentMetaWire` change,
   no wire schema bump.** `gate_creator_claim_pid` and friends already establish this
   pattern: non-wire keys are written into `agent_meta.json` and read back with a direct
   `json.load`. The scan wire is the TUI's hot read path; question chain rebuild is a
   cold settle-time path and does not belong there. This keeps the phase entirely inside
   the Python repo — no `../sase-core` change is required.

5. **The answering surfaces need no new code path.** ACE
   (`_notification_question_modal.py`) and mobile (`_mobile_notification_actions.py`)
   both funnel through `execute_user_question_response`, so binding the gate-shell
   execution callbacks and settling the shell **inside that one function** covers every
   surface at once. That wiring keys on `isinstance(envelope.get("shell"), dict)`, not
   on the flag: the durable bundle decides, so an answer submitted from a process whose
   flag snapshot differs still settles the shell.

6. **The flag is read in exactly one place** — the top of `handle_questions_marker`. The
   Off branch is the current body verbatim; the On branch is a single new function. That
   is what makes `status-collapse` able to delete the Off branch mechanically.

## Work

### 1. Create the epic's feature flag

Run (do **not** hand-add a registry member, do **not** use `sase bead create`):

```bash
sase flag new gate_shell_handoff -k beta -z small \
  --when-enabled 'A /sase_questions or /sase_plan gate becomes a gate shell in the asking agent'"'"'s family: the asking agent ends as DONE, the gate shell owns the QUESTION/TALE status, and answering it launches the follow-up agent.' \
  --when-disabled 'The asking agent stays alive and blocks in wait_for_gate until the gate settles, then continues in-process as its own successor.' \
  --remove-when 'Both migrations have soaked and the plan and question flows have been exercised end to end on the enabled branch.'
```

Paste the printed registry entry into `src/sase/feature_flags/registry.py` (enum member
plus `FeatureFlagDefinition`, `bead=` set to the printed flag bead id), then run
`tools/sync_feature_flags_schema` so `src/sase/config/sase.schema.json` matches.
`tools/check_feature_flags` lints registry/bead agreement.

Add `src/sase/gate_shell/flag.py`, mirroring `src/sase/pager/flag.py`:

```python
def gate_shell_handoff_enabled() -> bool:
    """Return the process-local `gate_shell_handoff` flag decision."""
    return current_flags().enabled(FeatureFlag.gate_shell_handoff)
```

It lives under `gate_shell` (not `question_shell`) because `plan-migration`
(`sase-ud.11`) reuses the same flag.

### 2. Settle-time next-action hook — `src/sase/gate_shell/kind_next_action.py`

One small registry so a gate _kind_ can rebuild its follow-up's `## Your next action`
from durable state at settlement, without `sase.gate_shell` importing any kind's module:

```python
_KIND_NEXT_ACTIONS: dict[str, tuple[str, str]] = {
    "question": ("sase.question_shell.followup", "question_next_action"),
}

def resolve_shell_next_action(
    *, kind, artifacts_dir, meta, envelope, response, declared
) -> str: ...
```

Resolution is lazy (`importlib.import_module`) and defensive: an unregistered kind, an
import failure, an exception, or a falsy return all yield `declared`. Like
`followup_policy`'s module docstring says, this runs at settlement — it must never turn
a settlement into a crash. Log at `warning` with `exc_info=True` on failure.

Wire it in `src/sase/gate_shell/followup.py::_base_prompt_kwargs`: replace
`"next_action": policy.prompt` with the `resolve_shell_next_action(...)` call, passing
`meta.get("gate_kind")`, `artifacts_dir`, `meta`, `envelope`, `response`, and
`declared=policy.prompt`. This is inside `_base_prompt_kwargs`, so it applies to both
`launch_gate_followup_agent` and `build_suppressed_gate_followup_prompt`. Leave
`_apply_branch_policy`'s `meta["gate_next_action"]` as the _declared_ prompt — it is the
field `settle_shell_claim_and_followup` keys the launch decision off, and it must keep
meaning "the policy resolved a follow-up," not "here is the composed text."

### 3. `src/sase/question_shell/` — the question gate shell

New package with a lazy facade `__init__.py` in the style of
`sase/gate_shell/__init__.py`.

**`create.py`**

- `question_gate_shell_spec(questions, *, session_id, producer, action_data, auto, base_prompt, prior_rounds)`
  — reuses `create_user_question_gate`'s existing v3 request body (extract the dict
  construction in `src/sase/user_question_actions.py` into a shared
  `user_question_gate_spec(...)` helper so the two producers cannot drift), then layers
  the additive `shell` block:

  | field              | value                                                                                                        |
  | ------------------ | ------------------------------------------------------------------------------------------------------------ |
  | `suffix`           | omitted — `allocate_gate_suffix` picks `--gate`, `--gate-0`, …                                               |
  | `pending_status`   | `QUESTION`                                                                                                   |
  | `settled_status`   | `ANSWERED`                                                                                                   |
  | `accent`           | `#FFAF00`                                                                                                    |
  | `workspace`        | `inherit`                                                                                                    |
  | `next`             | `{"fork": "family", "output": ["results"], "prompt": null}`                                                  |
  | `branches.submit`  | `{"status": "ANSWERED", "accent": "#5FD7FF", "prompt": <declared>, "output": ["results"], "fork": "family"}` |
  | `branches.timeout` | `{"status": "QUESTION TIMED OUT", "accent": "#FFAF00", "prompt": null}`                                      |
  | `branches.stopped` | `{"status": "QUESTION CANCELLED", "accent": "#FFAF00", "prompt": null}`                                      |
  | `branches.failed`  | `{"status": "QUESTION FAILED", "accent": "#FF5F5F", "prompt": null}`                                         |

  `#FFAF00` and `#5FD7FF` are transcribed verbatim from the `QUESTION` and `ANSWERED`
  branches of `src/sase/ace/tui/widgets/_agent_list_render_agent_status.py` (epic plan
  §9 — pin them _before_ `status-collapse` deletes that ladder). Every status is ≤20
  chars. `gate_timeout_seconds` is left unset: `GateSpec.from_mapping` already defaults
  a shell gate to `GATE_SHELL_DEFAULT_TIMEOUT_SECONDS` (24h), and the reclaim chop
  enforces it.

- `create_question_gate_shell(...) -> GateShellCreation` — calls
  `sase.gate_shell.create_gate_shell(spec)`, then writes this round's chain metadata
  onto the new member's `agent_meta.json` with `update_meta_fields`:

  | key                           | value                                                                                                                        |
  | ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
  | `question_round_index`        | 1-based int                                                                                                                  |
  | `question_prev_artifacts_dir` | previous round's gate-shell artifacts dir, or absent                                                                         |
  | `question_base_prompt_path`   | path to `question_base_prompt.md`, written into the **round-1** shell's artifacts dir and inherited verbatim by later rounds |
  | `question_session_id`         | the gate's `request_id`                                                                                                      |
  | `question_sdd_spec_path`      | `LoopState.sdd_spec_path`, when set                                                                                          |

- `resolve_question_chain_parent(project_name, lane, creator_agent, *, hint)` — returns
  the previous round's gate-shell artifacts dir:
  1. `hint` (the in-process `LoopState` value, see §4) when it names a question gate
     shell in this lane — this is what covers the `%auto` case, where settlement runs
     under `creator_live=True`, records `gate_followup_outcome: "suppressed"`, and
     therefore never sets `gate_followup_agent`;
  2. otherwise the newest terminal gate shell in the lane with `gate_kind == "question"`
     whose `gate_followup_agent` equals the creating agent's own name — the durable
     cross-process link. `list_gate_shells(project=...)` takes no lane argument, so
     filter on `record.lane` and sort on `record.timestamp` in the caller;
  3. otherwise `None` (a new chain: a fresh phase agent, or a coder launched by a plan
     gate).

**`rounds.py`**

- `question_chain(head_artifacts_dir) -> tuple[str, ...]` — walks
  `question_prev_artifacts_dir` back to the root and returns artifacts dirs
  oldest-first. Bound the walk (e.g. 200 links) and stop on a repeated dir so a
  corrupted link can never loop forever.
- `question_rounds(head_artifacts_dir, *, head_response=None) -> list[QARound]` — for
  each dir: read `agent_meta.json` → `gate_bundle_path`, then `request.json` →
  `payload.questions` and `response.json` → `option_results[submit].result`; use
  `head_response` for the head instead of re-reading. Translate exactly as
  `_question_result` does today: `response["feedback"]`, when a non-empty string,
  overrides `global_note`. Build each round with `sase.main.qa_prompt.build_qa_round`.
  Skip a round whose bundle is unreadable or whose gate never produced a `submit` result
  (a timed-out or cancelled round contributes nothing), rather than failing the rebuild.
- `question_base_prompt(head_artifacts_dir) -> str` — read `question_base_prompt_path`
  from the head's meta; `""` when missing.

**`followup.py`**

- `question_next_action(*, artifacts_dir, meta, envelope, response, declared) -> str` —
  the hook registered in §2. Rebuilds
  `question_base_prompt(...) + "\n\n" + build_merged_qa_markdown(question_rounds(...))`,
  returns `declared` if the base prompt or the round list comes back empty, and — as a
  side effect, wrapped in `try`/`except` + `logger.warning` — updates the SDD prompt
  snapshot (below). Returning the rebuilt text must never depend on the snapshot
  succeeding.
- `update_question_sdd_prompt_snapshot(sdd_spec_path, merged_qa_text, *, workspace_dir, workspace_num, artifacts_dir)`
  — the body of `run_agent_exec_questions._update_sdd_prompt_snapshot_qa` lifted to
  explicit arguments (canonical `prompts/<n>/` path → `set_prompt_qa`; legacy
  plans-sidecar path → `set_prompt_qa` + `commit_sdd_store_files` for an out-of-tree
  store). The gate shell supplies `workspace_dir` / `workspace_num` from its own meta
  and `sdd_spec_path` from `question_sdd_spec_path`; the merged text is
  `merge_qa_for_prompt(rounds)` so the archive keeps the marker-wrapped form and the
  continuous numbering it has today.
  `run_agent_exec_questions._update_sdd_prompt_snapshot_qa` becomes a thin delegation so
  the Off branch is byte-for-byte unchanged.

### 4. `handle_questions_marker` — the flag branch

`src/sase/axe/run_agent_exec_questions.py`:

```python
def handle_questions_marker(q_data, ctx, state) -> str | None:
    if gate_shell_handoff_enabled():
        return _handle_questions_via_gate_shell(q_data, ctx, state)
    ...existing body, unchanged...
```

`_handle_questions_via_gate_shell`:

1. `normalize_handoff_interruption_state` + `finalize_handoff_artifacts_as_completed`
   (same as today, and the same as `handle_gate_marker`).
2. `reset_killed()` — the questions command's own SIGTERM must not read as a new kill.
3. Resolve the chain parent via
   `resolve_question_chain_parent(..., hint=state.question_gate_artifacts_dir)`; derive
   `base_prompt` from the parent when there is one, otherwise
   `state.question_base_prompt`; derive `prior_rounds` from the parent chain.
4. `create_question_gate_shell(...)` with `auto=is_auto_approve_active()` and the same
   `producer` / `action_data` payload `handle_questions_flow` builds today (agent, cl
   name, project file, timestamps, artifacts dir, session id).
5. `record_workflow_metadata(state.current_artifacts_dir, {...})` with
   `questions_submitted_at`, `question_request_path`, `question_response_path` (both
   from `creation.gate`), `question_session_id`, `patch_name`/`changespec_name` —
   preserving what `tests/test_axe_record_workflow_metadata_questions.py` asserts.
6. Set `state.question_gate_artifacts_dir = creation.record.artifacts_dir`.
7. **If `creation.should_handoff` is false** (the `%auto` short-circuit — the gate
   already settled synchronously inside creation, per epic plan §7): rebuild the rounds
   from the chain (which now includes this settled round), save the chat with
   `merge_qa_for_prompt(rounds)` as the response exactly as today, and
   `continue_as_successor(...)` with
   `assemble_question_followup_prompt(base_prompt, rounds)` — unchanged suffix, role,
   relationship and artifact-label arguments. `%auto` still costs exactly one agent.
   Return `None` (continue the loop).
8. **Otherwise**: delegate the creator's own bookkeeping to
   `handle_gate_marker(gate_data, ctx, state)` from
   `src/sase/axe/run_agent_exec_gate.py`, with `gate_data` synthesized from the creation
   record (`gate_id`, `member_artifacts_dir`, `member_agent_name`, `kind: "question"`).
   That reuses the promotion-aware starter-suffix logic, the handoff chat, and the meta
   updates already tested for gate shells, and returns `"gated"`. Do not duplicate it.
   It re-runs the two step-1 normalizations; both are structural and idempotent (a
   second call finds no non-terminal step), so the repeat is harmless.

Add `question_gate_artifacts_dir: str | None = None` to `LoopState`
(`src/sase/axe/run_agent_exec_types.py`) with a docstring saying it is an in-process
_hint_ for chain discovery whose durable fallback is `gate_followup_agent`.

On the On branch nothing writes `pending_question.json`, nothing yields or reacquires a
runner slot, and `wait_for_gate` is never called: the runner ends as `DONE` and the gate
shell carries `QUESTION`.

### 5. Settle the shell from every answering surface

`src/sase/user_question_actions.py::execute_user_question_response`, neutral
(non-legacy) branch only, mirroring
`launch_approval_actions._execute_neutral_launch_approval_response`:

- load the envelope (`load_and_verify_bundle`) once and set
  `shell_backed = isinstance(envelope.get("shell"), dict)`;
- when shell-backed, `find_gate_shell_by_gate_id(None, request_id)` and pass
  `bind_gate_shell_execution_callbacks(gate_shell.artifacts_dir).as_kwargs()` into
  `execute_gate_selection` so the submit command's output streams into `gate.log`;
- after a successful execution,
  `settle_gate_shell(gate_shell, gate_state="answered", reason="question answered")`
  **before** the `already_completed` check, exactly as the launch-approval path orders
  it;
- leave the legacy in-flight branch and every existing error translation untouched.

### 6. Skill template

Rewrite the **Handoff And Continuation** section of
`src/sase/xprompts/skills/sase_questions.md` to describe both branches: `sase questions`
still writes a durable handoff marker and sends `SIGTERM`; with `gate_shell_handoff`
enabled the runner creates a **question gate shell** in the agent's family, ends as
`DONE`, and the answer launches a follow-up agent carrying the merged Q&A; with the flag
disabled the runner blocks and continues in-process. Keep the "Do not poll question
request or response files" rule.

Keep the phrases `tests/main/test_init_skills_sources.py` already asserts
(`sase questions '<json>'`, `writes a durable handoff marker`, `sends \`SIGTERM\``, `Do
not poll question request or response
files`) and add assertions for the new gate-shell phrasing. Skill sources are generated: do **not** touch chezmoi `SKILL.md`files, and do not run`sase
skill init --force` here — deployment happens from a clean, landed tree.

## Tests

New `tests/question_shell/` package:

- `test_create.py` — the shell block: pinned statuses and accents, all four branch keys
  present and accepted by `GateShellSpec.from_mapping` against the compiled
  `("submit",)` branch list, the default 24h timeout, and the chain metadata written for
  round 1 (`question_round_index == 1`, no `question_prev_artifacts_dir`,
  `question_base_prompt.md` written and pointed at).
- `test_rounds_rebuild.py` — **the golden tests the phase brief requires.** Build a
  two-round and a three-round chain of real gate bundles + gate-shell members with
  `create_gate_shell_member` (the fixture pattern in
  `tests/gate_shell/test_settlement_followup.py`), answering each through
  `execute_gate_selection` and settling it, with **no live process carrying state
  between rounds** — that is the simulated runner death. Assert `question_rounds(head)`
  returns the rounds oldest-first and that `build_merged_qa_markdown` numbers them
  `Q1..Qn` continuously across the chain, that `question_base_prompt(head)` is the
  round-1 base prompt, and that the last non-empty `global_note` wins. Add cases for a
  broken link (missing `question_prev_artifacts_dir`) and an unanswered middle round
  (contributes nothing, later rounds still rebuild).
- `test_followup_prompt.py` — golden shape of the composed follow-up for an answered
  question gate: `#fork:<family>` prefix live, the whole body inside one disabled region
  with no nested escaped `% xprompts_enabled:` markers, `## Results` carrying the
  answers JSON, and `## Your next action` equal to base prompt + merged Q&A **including
  the round just answered**. Assert the fallback returns `declared` when the chain is
  unreadable.

Extend / add elsewhere:

- `tests/gate_shell/test_kind_next_action.py` — unregistered kind returns `declared`; a
  registered hook's return value is used; an importing/raising/empty-returning hook
  falls back to `declared` without raising.
- `tests/test_axe_run_agent_exec_questions_gate_shell.py` — **both flag states**, which
  the flag policy requires. Flag off: the existing blocking flow still runs and
  `create_gate_shell` is never called. Flag on, non-auto: a gate shell is created,
  `wait_for_gate` is never called, no `pending_question.json` is written, the loop
  outcome is `"gated"`, and `state.question_gate_artifacts_dir` is set. Flag on with
  `%auto`: `should_handoff` is false, `continue_as_successor` is called once with a
  prompt containing the merged Q&A, and no second agent is spawned.
- `tests/test_user_question_gates.py` — a shell-backed question gate answered through
  `execute_user_question_response` settles its gate shell (`gate_state == "answered"`,
  decision record and chat written) and streams the submit command into `gate.log`; a
  non-shell question gate answers exactly as it does today. Because ACE and mobile both
  call this function, assert that chokepoint explicitly (a grep-style assertion that
  `_notification_question_modal.py` and `_mobile_notification_actions.py` route through
  it is acceptable and cheap) rather than duplicating the matrix.
- `tests/main/test_init_skills_sources.py` — the new `sase_questions` phrases.

## Verification

- `just check` for the change (`just check-full` through `/sase_monitor` before landing;
  this phase touches the runner, the gate shell settlement path, and a feature-flag
  registry, so the broadening set is in play).
- `tools/check_feature_flags` must pass (registry ↔ flag bead agreement) and
  `src/sase/config/sase.schema.json` must be regenerated with
  `tools/sync_feature_flags_schema`.
- Exercise by hand with the flag on: ask one question and answer it (the asking agent
  goes `DONE`, a `⋔` gate shell shows `QUESTION` then `ANSWERED`, a follow-up agent
  starts with the merged Q&A); ask a second question from that follow-up and confirm the
  rebuilt prompt contains **both** rounds with continuous numbering; cancel a pending
  question gate with `sase gate cancel` and confirm the family ends at
  `QUESTION CANCELLED` with no successor. Then repeat the first case with the flag off
  and confirm today's behavior is unchanged.

## Out of scope

- `/sase_plan` (`sase-ud.11` — same flag, next phase), the `--q` suffix taxonomy
  (`sase-ud.12`), deleting the status ladder and the Off branch (`sase-ud.13`), and the
  decision record / memory updates (`sase-ud.14`).
- Any `../sase-core` change. The chain metadata is deliberately non-wire (decision 4).
- Re-pointing `_update_sdd_prompt_snapshot_qa`'s consumer at the gate shells so the
  merged-digest prompt could be dropped entirely — the epic plan explicitly defers that
  ("Revisit once that consumer is re-pointed at the gate shells").
