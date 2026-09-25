---
tier: tale
title: Let sase plan approve recover a failed coder
goal:
  Running `sase plan approve <plan>` on a plan that was already approved, but whose
  coder failed, was killed, or never launched, starts a replacement coder instead of
  refusing. A plan whose coder is still running or has finished is still refused, now
  with the real reason and a follow-up command that works.
size: medium
proposed_by: bbugyi200.athena.0sm
create_time: 2026-09-25 19:43:00
status: wip
---

# Let `sase plan approve` recover a failed coder

## Diagnosis

### What happened (2026-09-25)

| Time     | Event                                                                                                                                                                                                                                                                                                                    |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 19:04:03 | Planner `0sk` proposed `~/.sase/plans/202609/codex_backend_key_401_retry.md`, and gate shell `0sk--gate` opened plan gate `7ff58c91`.                                                                                                                                                                                    |
| 19:07:30 | The user approved it in the TUI as a **tale** (`selected_option_ids: ["approve", "commit"]`). The plan was archived as `plan:202609/codex_backend_key_401_retry.md`. The pending-action entry became `state: already_handled` with `handled_action: "approve"`, the protocol action shared by tale, approve, and commit. |
| 19:08:03 | The gate shell launched `0sk--code`. `0sk--gate`'s `agent_meta.json` recorded `gate_followup_agent: "0sk--code"`.                                                                                                                                                                                                        |
| 19:09:54 | `0sk--code` failed (`done.json` `outcome: "failed"`) on the OpenAI "backend key" 401 outage. That is the bug this very plan fixes.                                                                                                                                                                                       |
| ~19:11   | The user ran `sase plan approve ~/.sase/plans/202609/codex_backend_key_401_retry.md` to get a new coder. It refused with `✗ … is not awaiting approval … already approved (7m ago)`.                                                                                                                                     |

### Root cause

`resolve_direct_approval()` in `src/sase/main/plan_direct_approval.py` calls
`_handled_refusal()` whenever `classify_plan_gate_history()` returns `handled` (the gate
was answered) or `direct` (a CLI receipt exists). That call comes before any
kind-specific logic. So once a plan has been approved even once, it is terminal to this
command, no matter whether the coder that approval owed ever ran successfully.

The sase-18i epic (`plan:202609/plan_approve_gateless_tales.md`) wrote this down as
"Anything already handled refuses". The user's own request for that epic, however, named
failure as "the most common reason a user would invoke the `sase plan approve` command
manually". Failures after approval are all unrecoverable through this command today:

- the coder crashed or was killed
- a provider outage
- a coder launch error
- the gate shell died before its follow-up

Even `-k approve` is refused, although the docs describe it as "run a coder on this
plan". Every selector spelling (name, `<shard>/<name>`, `plan:` ref, path) hits the same
refusal.

### Secondary defects in the same output

1. **Wrong age.** `(7m ago)` is measured from the gate's `created_at_unix`, not from
   `handled_at_unix` (`classify_plan_gate_history`, `plan_pending_diagnosis.py`).
2. **Wrong action.** "was already approved" loses the fact that it was a tale. The
   stored `handled_action` is the protocol action; the gate bundle's `response.json`
   records the real choice.
3. **Dead-end hint.** `Inspect it with: sase plan show codex_backend_key_401_retry` is
   ambiguous once the plan is committed: the local proposal and the committed copy both
   match by name. That is intended, tested `plan show` behavior
   (`tests/plan_show/test_resolve.py::test_rung_name_ambiguous_across_repo_and_local`).
   On top of that, the ambiguity list shows both candidates as the identical
   `plan:202609/codex_backend_key_401_retry.md`, so neither row can select the local
   copy.
4. **Duplicated hint.** The hint prints twice. `render_direct_approval_refusal` only
   skips a hint that equals a whole detail line, but the detail line is
   `Inspect it with: <hint>`.
5. **Silent failure.** Nothing says the coder failed, so the user cannot tell that the
   situation is recoverable.

Note for the reviewer: until this lands, the equivalent manual recovery is
`sase run '#gh:gh_sase-org__sase %model:@small #coder(plan:202609/codex_backend_key_401_retry.md)'`
(pick another `%model:` while the Codex outage lasts). `sase agent restart 0sk--code`
also works, but it deletes the failed run's artifacts.

## Design

### Recovery route: a third outcome for already-approved plans

After the resolver locates a plan file, it looks at the plan's history. When that
history is an **approval that owed a coder**, it evaluates the coder evidence (next
section) and decides:

| Coder evidence                                                                          | Result                                    |
| --------------------------------------------------------------------------------------- | ----------------------------------------- |
| any coder is live (running, starting, queued, waiting, needs input)                     | refuse `coder_running`                    |
| any coder succeeded, or the committed plan has `status: done`                           | refuse `already_implemented`              |
| every coder ended without success, no coder was ever launched, or no coder can be found | **recover**: launch one replacement coder |

An approval that owed a coder is either of:

- a `handled` gate whose real action is `tale`, `approve`, or `commit`
- a `direct` receipt with one of those actions

A commit-only approval counts: running `approve` on it now means "run the coder".

Everything else keeps today's refusal, with the wording fixes below:

- epic approvals (point at `sase bead work <path>`, the idempotent epic launcher)
- reject, feedback, and cancel histories
- explicit `-k commit` and `-k epic` on an already-approved plan

Recovery never re-decides or re-archives. The coder runs against the plan exactly as it
was approved:

- the committed `plan:` ref, if that approval committed the plan
- otherwise the local plan path

It applies when `-k` is omitted, `tale`, or `approve`. The live-gate route and fresh
direct approvals (histories `none`, `orphaned`, `expired`) are unchanged.

### Coder evidence

Put this in a new module, `src/sase/main/plan_direct_approval_recovery.py`. Do not grow
`plan_direct_approval.py`, which is already 750+ lines. Gather every source, because
more than one can apply (for example, a receipt from a CLI run that lost a gate race,
where the gate's own coder is the real one).

1. **Receipt.** `read_direct_approval_receipt(local_plan)` gives:
   - `coder_agent` and `coder_pid`
   - `coder_error`: a launch failure with no coder means "never launched"
   - new `replaced_coders` (see Receipts)
2. **Gate follow-up.** For the newest pending-action entry with
   `state == "already_handled"` (`_gate_history_for_plan` already finds it):
   - **Real action:** load `<bundle_path>/response.json`, pass it through
     `translate_plan_gate_response()` (`src/sase/_plan_gate_envelope.py`), then
     `persisted_plan_action()` (`src/sase/_plan_approval_protocol.py`). Fall back to
     `handled_action`.
   - **Committed ref:** the primary option result's `plan_archive_ref`, when present.
   - **Gate shell:** `action_data["raw_suffix"]` is the notifying agent's artifact
     timestamp, which is the gate shell's for gate-owned proposals. Resolve it with
     `resolve_agent_artifact_timestamp_path(project, "ace-run", raw_suffix)` from
     `src/sase/core/agent_artifact_paths.py`. Accept it only if its `agent_meta.json`
     `gate_notification_id` equals the entry's notification id, then read
     `gate_followup_agent` and `gate_followup_outcome`.
   - If that does not verify, fall back to the registered `<agent_name>--gate`, checked
     the same way. As a last resort, use the conventional `<agent_name>--code` coder if
     it is registered.
3. **Classify one coder by name.** Use a cheap, targeted read. Do not use
   `find_named_agent`: it scans every project (~17 s) and silently skips agents that
   crashed without `done.json`.
   - Get `artifacts_dir` and `state` from
     `sase.agent.names.lookup_registered_name(name)`. No entry means **missing**.
   - If the registry state is `dismissed`, use the archived outcome from
     `load_archived_agent_completions` (`src/sase/core/dismissed_agent_completion.py`).
     A success outcome means succeeded; anything else means ended.
   - Otherwise, call
     `scan_agent_artifact_dirs(sase_projects_dir(), [artifacts_dir], wait_scan_options())`
     and
     `classify_wait_target(WaitTarget(kind=WaitTargetKind.AGENT, artifact_dir=...), records)`
     from `sase.agent.wait_watch`. Include the `retried_as_timestamp` successor
     directory when `done.json` names one. Precedent:
     `src/sase/ops/commands/_agent_drain_notify.py`.
   - Map the results:
     - `SUCCEEDED` → succeeded
     - `RUNNING`, `STARTING`, `QUEUED`, `WAITING`, `NEEDS_INPUT`, `NEEDS_REVIEW` → live
     - `FAILED`, `TERMINAL_OTHER`, `STALLED` → ended
   - A receipt coder with a PID but no name is live if the PID is alive, and ended
     otherwise.
4. **Committed status (best effort).** When the approval committed the plan, read the
   committed copy through the same plan-ref resolution `sase plan show plan:<ref>` uses,
   accepting only a path outside `~/.sase/plans`. `status: done` means succeeded:
   `src/sase/workflows/commit/plan_hooks.py` sets it when a coder commits with
   `SASE_PLAN`. This check is a safety net for dismissed or renamed coders.

Aggregate the result as live, else succeeded, else recover. Return a small frozen
result:

- the verdict
- the prior coders, each with name, state, outcome, and an age such as "failed 14m ago"
- the approved action, age, and gate id
- the plan argument (committed ref or local path)

The dry run, the refusal, and the executor all use this one result.

### Placement for the replacement coder

Reuse `_resolve_placement()`, with two changes:

- **No `planner_running` refusal during recovery.** The gate is already settled, so
  there is nothing to race. Fresh approvals keep the refusal.
- **Name collisions join the session instead of going standalone.** When `--code` is
  taken (normally by the failed coder), retry the attach with suffix `@` so the
  replacement joins the planner's session under the next free member name. This is the
  remedy the attach error itself suggests. Apply it to fresh approvals too.
  - Detect the collision with a structured field, not message text. Commit `11b9c56b0`
    fixed exactly this kind of text-matching bug. Give `AgentSessionAttachError`
    (`src/sase/agent/_agent_session_attach_types.py`) an optional `reason` attribute,
    and set `reason="name_taken"` where `_ensure_agent_session_name_available` raises.
    The type is Python-only, so no sase-core change is needed.
  - `CoderPlacement` gains `suffix: str = "code"`, and `compose_coder_prompt` emits
    `%id(<suffix>, session=<parent>)`.
  - Any other attach error still falls back to standalone, with its reason, as the user
    originally asked.

Model precedence, `-p`, `-w`, and `bead=` handling are unchanged.

### Execution

Add `execute_coder_recovery()` next to `execute_direct_approval()` in
`src/sase/main/plan_direct_approval_run.py`, or in the new module. Route to it from
`_approve_plan_from_cli()` when the resolver returns a recovery plan. The steps:

1. **Agent guard.** Keep the `SASE_AGENT` guard: `agent_launch_denied`. Dry runs are
   still allowed.
2. **Lock and re-check.** Take `file_lock(<receipt path>.lock, timeout=...)` from
   `sase.notification_gates.durability`. Re-evaluate the coder evidence under the lock;
   if it is now live or succeeded, raise the matching refusal. Two concurrent runs then
   launch one coder.
3. **Skip approval side effects.** No adopt, no archive, no gate retirement, and no
   planner-metadata rewrite: the approval already happened.
4. **Write the receipt before launching.** Leave the coder fields empty, and append the
   prior coder names to `replaced_coders`.
5. **Launch.** Call `_launch_coder(prompt, local_plan)`, which passes
   `SASE_PLAN=<local plan>`. Any exception becomes `coder_error`.
6. **Rewrite the receipt** with the new coder's name and PID, or the error.

Existing tests expect a crash between steps 4 and 6 to leave a receipt with no coder and
no error. The next run treats that as "no coder found" and recovers again, which is the
safe direction.

### Receipts (`src/sase/plan_approval_receipts.py`)

- Add optional fields `replaced_coders: tuple[str, ...] = ()` and
  `recovered_gate_id: str | None`, where the latter is the gate whose approval this
  recovery continues. Old receipts read unchanged.
- Recovery writes a receipt even for a gate-approved plan. That makes the replacement
  coder discoverable on the next run (a live coder is refused). It also makes
  `classify_plan_gate_history` return `direct` from then on. Update the `direct` wording
  wherever it is rendered (`diagnose_located_plan_miss`, `_handled_refusal`) so that a
  receipt with `replaced_coders` or `recovered_gate_id` reads "coder relaunched via sase
  plan approve", not "approved … via sase plan approve".
- Evidence gathering still consults the gate entry for such plans (step 2 of "Coder
  evidence"), so the original gate coder is never forgotten.

### Diagnosis and hint fixes (`src/sase/main/plan_pending_diagnosis.py`)

These also improve `sase plan reject` misses, which share the diagnosis code.

- **Age.** `PlanGateHistory.age` for handled entries uses `handled_at_unix`, falling
  back to `created_at_unix`.
- **Action.** Refine the handled action through the gate `response.json`, as above, so
  the text reads "was already approved as a tale".
- **Hint target.** The inspect hint names a target that resolves to exactly one plan:
  - the committed `plan:<shard>/<name>.md` ref when the approval committed it (it
    resolves on the ref rung)
  - otherwise the absolute local plan path

  Put the command in the hint tuple once, and print "Inspect it with:" only once.

- **Renderer dedupe.** In `render_direct_approval_refusal`, skip a hint when it appears
  inside any detail line, not only when it equals one.

### `plan show` ambiguity rows (`src/sase/plan_show/load.py`)

In `load_ambiguity_candidate`, show `canonicalize_plan_reference_from_roots(...)` only
when `resolve_plan_reference_from_roots(reference, roots=roots)` resolves back to that
same file. Otherwise show the absolute path. Then "pass one of the references above"
lists two distinct, working selectors. The existing ambiguity itself stays.

### Output (`src/sase/main/plan_approve_render.py`)

All output follows the existing color contract.

**Recovery success** (stdout, exit 0):

```
↻ Coder relaunched · codex_backend_key_401_retry
  Retry Codex's OpenAI-side backend-key 401

  plan    plan:202609/codex_backend_key_401_retry.md · approved as a tale 18m ago · gate 7ff58c91
  before  0sk--code · failed 14m ago
  coder   0sk--2 · agent session 0sk · %model:@small

  follow  sase agent show 0sk--2
```

For a standalone coder, add the dim `no agent session: <reason>` continuation line, as
today. With no prior coder found, the `before` line reads
`none found · the approval's coder never launched`.

**Launch failure** (exit 1): show the plan and `before` lines, then
`✗ Coder launch failed: <error>`, `Launch it yourself:`, and `_print_recovery_command`.

**Dry run** (exit 0):

```
◇ Dry run · codex_backend_key_401_retry would get a replacement coder
```

followed by the `plan`, `before`, `coder`, and `prompt` lines, and "Nothing was
changed."

**Refusals** (stderr, exit 2):

```
✗ codex_backend_key_401_retry is already approved and its coder is running
  Retry Codex's OpenAI-side backend-key 401
  plan:202609/codex_backend_key_401_retry.md · approved as a tale 18m ago
  coder   0sk--code · running
  sase agent show 0sk--code
```

```
✗ codex_backend_key_401_retry is already approved and implemented
  …
  coder   0sk--code · completed 2h ago        (or: plan status done)
  To run another coder anyway:
    sase run '<exact composed prompt>'
```

### Help and docs

- **Parser help.** Update the `approve` description in `src/sase/main/parser_plan.py` to
  add: "A plan already approved whose coder failed, was killed, or never launched gets a
  replacement coder; one whose coder is running or finished is refused."
- **Parser examples.** Add the example
  `sase plan approve my_plan   # relaunch a failed coder`.
- **CLI snapshot.** If the completion CLI-spec snapshot
  (`tests/completion/snapshots/cli_spec.json`) captures this text, regenerate it the way
  the repo normally does.
- **Prose docs.** Add one or two sentences on recovery to the direct-approval paragraph
  in `docs/xprompt.md` (the "Outside the TUI, `sase plan` shows…" paragraph), to the
  `sase plan approve` paragraph and table row in `docs/cli.md`, and to the approve
  mention in `docs/sdd.md`.

## Tests

### New: `tests/test_plan_direct_approval_recovery.py`

Patch the launch, the status reads, and the registry at their module seams; see
`tests/test_plan_direct_approval_run.py` for the style.

- **Gate-handled tale, follow-up coder failed** (this incident). The resolver returns a
  recovery plan with the committed ref and the `before` coder. The executor:
  - launches exactly one coder, with `SASE_PLAN` set
  - never calls `archive_approved_plan` or `cancel_gate`
  - writes a receipt with `replaced_coders == ("x--code",)` and the new coder
- **Refusals from coder state:**
  - a live coder refuses `coder_running`
  - a succeeded coder refuses `already_implemented`
  - a committed plan with `status: done` and an unknown coder refuses
    `already_implemented`
- **Recover when there is no working coder:**
  - the gate shell is missing and no `--code` is registered: recover, with "none found"
  - a commit-only approval: recover
  - a `direct` receipt with a launch `coder_error`: recover
- **Coder found through the gate, not the receipt.** A `direct` receipt that lost a gate
  race (route `none`, "answered concurrently") is evaluated through the gate's follow-up
  coder: failed recovers, running refuses.
- **Second run.** After a recovery whose new coder is running, the next run refuses
  `coder_running`.
- **Lock re-check.** The evidence flips to live between resolve and execute, and the run
  refuses without launching.
- **Unchanged refusals.** Epic, reject, feedback, and cancel histories, and explicit
  `-k commit` / `-k epic` on approved plans, still refuse.
- **Agent guard.** With `SASE_AGENT` set, the run is refused with `agent_launch_denied`,
  and a dry run still works.

### Placement

- A `name_taken` attach error retries with `@`, and the prompt contains
  `%id(@, session=<planner>)`.
- Recovery ignores `parent_is_running`.
- A fresh approval still refuses `planner_running`.

### Diagnosis

- Age comes from `handled_at_unix`.
- A v2 `response.json` with `approve+commit` reads "approved as a tale".
- The inspect hint is the committed ref (or the absolute path) and prints once.

### Rendering (`tests/test_plan_approve_render.py`)

Under `NO_COLOR`, cover the recovery card, the launch-failure card (exit 1), the dry
run, and both new refusals.

### `plan show`

`test_rung_name_ambiguous_across_repo_and_local` now also asserts that the two candidate
references differ and that the local one is its path.

### Existing tests to update

`tests/test_plan_direct_approval_run.py` (the race re-run near line 355 now depends on
coder evidence, so stub the gate coder as live to keep the refusal) and any
direct-history wording assertions.

Finish with the repo's standard lint and test check, as the `lint_and_test` memory
describes.

## Out of scope

- The live-gate route and TUI approval flow.
- Recovering epic approvals: `sase bead work` already does this idempotently.
- Overriding reject or feedback decisions.
- The Codex 401 retry itself (the `codex_backend_key_401_retry` plan).
- Moving plan-approval orchestration into sase-core. It is Python-owned today, for the
  same reasons given in the sase-18i plan.
