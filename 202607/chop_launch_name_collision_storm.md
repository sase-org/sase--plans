---
tier: tale
title: Stop the refresh_docs chop failed-launch prompt-stash storm
goal: 'Chop-proposed agent launches get per-run-unique names so repeat runs never
  collide with durably reserved agent names, failed chop launches stop flooding the
  user prompt stash, a failing run_every chop retries at its configured cadence instead
  of every lumberjack tick, and the ~1,500 junk stash entries already written are
  cleaned up safely.

  '
create_time: 2026-07-19 07:25:06
status: done
prompt: 202607/prompts/chop_launch_name_collision_storm.md
---

# Plan: Stop the refresh_docs chop failed-launch prompt-stash storm

## Incident summary

The user prompt stash (`~/.sase/prompt_stash.jsonl`) is flooding with `source: "failed_launch"` entries whose text is a
scaffolded chop prompt (`#gh:...`, `%name:chop.refresh_docs.<project>.1`, `%tribe:chop`, "Refresh the documentation for
..."). As of 2026-07-19 ~07:16 there are ~1,480 such entries and the storm is still live, appending roughly one entry
per enabled target project per minute (~300/hour) since ~00:51 on 2026-07-19.

The source is the `refresh_docs` lumberjack configured in `~/.config/sase/sase_athena.yml` (interval 60s, chop
`refresh_docs` with `script: sase_chop_refresh_docs`, `run_every: 3h`, `trigger: git.commits_since` with
`checkpoint: on_action_success`, `for_each: source: projects`). Every launch attempt fails with:

```
sase.agent.launch_validation.AgentNameLaunchCollisionError:
Agent name 'chop.refresh_docs.sase.1' is taken. Try 'chop.refresh_docs.sase.11'.
```

(see `~/.sase/axe/error_digests/digest_20260719_070858.txt`; raised from `src/sase/agent/launch_validation.py`
`validate_launch_name_requests`, reached via `src/sase/axe/chop_runner_script_result.py` →
`src/sase/axe/chop_proposals.py` `launch_chop_proposals` → `src/sase/agent/launch_cwd_agents.py`).

## Root cause

Three defects interact; each is worth fixing on its own.

1. **Chop proposal agent names are deterministic and collide forever (primary).** `derive_chop_agent_name` in the Rust
   core (`crates/sase_core/src/axe_chop/validation.rs` in the sibling `sase-core` repo, exposed through
   `src/sase/core/axe_chop_facade.py`) builds `chop.<chop>.<target>.<proposal_index + 1>` with no per-run uniqueness.
   The agent-name registry (`~/.sase/agent_name_registry.json`) durably reserves names even after an agent is dismissed
   — by design, since names remain addressable identities. So the first run of the composable-chops naming scheme
   (2026-07-18 21:51) succeeded and reserved `chop.refresh_docs.<project>.1` / `.2` for every target; every subsequent
   run collides on `validate_launch_name_requests` and the launch raises before spawning anything. The pre-6v scheme
   (`refresh_docs.<project>.<hash12>.update` / `.polish`, visible in the registry until 2026-07-18) embedded a per-run
   token and never had this problem; epic sase-6v's "stable default agent name" regressed it. This latent bug affects
   **every** script chop that does not set an explicit `agent_name` (e.g. the `code_quality` lumberjack's
   `recent_bug_audit` / `recent_improvement_audit` chops will hit it on their second triggered run), not just
   `refresh_docs`.

2. **Failed chop launches write the user prompt stash (amplifier).** Every launch failure in
   `launch_agents_from_cwd_impl` calls `record_failed_launch_prompt` (`src/sase/history/prompt_store.py`), which appends
   a `failed_launch` row to the prompt stash via `src/sase/agent/failed_launch_prompt_stash.py`. That safety net exists
   so a human's just-typed prompt survives an unmounted prompt bar. It is wrong for chop-proposed launches: the prompt
   is machine-generated, fully reproducible, already persisted in the chop run history (`finalize_script_chop_run`
   result files) and error digests, and is retried automatically — so each retry appends another junk stash row.

3. **`run_every` never advances on failure (amplifier).** In `src/sase/axe/lumberjack.py` `_outcome_to_result`, the
   `failure` / `check_error` / `action_failed` (and `timeout`) branches return `update_timestamp=False`, so a failing
   chop's `run_every: 3h` clock never moves and the lumberjack re-runs it on every 60-second tick. Combined with defect
   1 (a persistent, deterministic failure) and the trigger's `on_action_success` checkpoint (which intentionally keeps
   the trigger hot until an action succeeds), this turned one broken chop into a ~5-failures-per-minute storm of error
   digests and stash entries.

## Fix design

### 1. Per-run-unique chop agent names (Rust core + Python threading)

Extend `derive_chop_agent_name` to accept a **run token** and produce:

```
chop.<chop>.<target>.<run_token>.<suffix>
```

- `run_token`: the chop runner's `run_id`, sanitized through the existing `sanitized_component` rules and truncated to 8
  characters. `run_id` is already generated per run and threaded through `process_script_chop_result` /
  `launch_chop_proposals`, so names are deterministic within a run (idempotent for previews vs. launches) and unique
  across runs.
- `<suffix>`: keep today's `<proposal_index + 1>`. (Optionally use the sanitized proposal `id` — `update` / `polish` —
  when present for readability; acceptable either way, coder's choice, but the index fallback must remain for id-less
  proposals.)
- Keep the existing 120-char truncation and trailing-separator trimming.

Wiring:

- **sase-core repo** (open via `/sase_repo`; `crates/sase_core/src/axe_chop/validation.rs`, re-exports in
  `axe_chop/mod.rs` / `lib.rs`, PyO3 wrapper in `crates/sase_core_py/src/lib.rs`, tests in
  `crates/sase_core/src/axe_chop/tests.rs`): add the run-token parameter. Make it an optional trailing argument at the
  binding layer (`None` preserves today's shape) so this is a backward-compatible minor release, per this repo's
  convention of releasing sase-core first and then bumping the consumer pin (`sase-core-rs>=0.8.0,<0.9.0` in
  `pyproject.toml`; raise the floor to the new minor release).
- **This repo**: `src/sase/core/axe_chop_facade.py` `derive_chop_agent_name` gains the run-token parameter;
  `src/sase/axe/chop_proposals.py` `prepare_chop_proposals` gains a `run_id` keyword and passes it through; its single
  call site in `src/sase/axe/chop_runner_script_result.py` already has `run_id` in scope.

Wait dependencies keep working unchanged: `%wait:` names are resolved from the prepared proposal names (previews) or the
launch results (`launch_chop_proposals`), and with deterministic literal names the preview, `%name:` directive, and
`%wait:` reference all agree within a run. Agents-tab nesting improves: each run groups under its own
`chop.<chop>.<target>.<run_token>` hood.

**Rejected alternatives:**

- _`@` name templates_ (`%name:chop.refresh_docs.sase.@`): the launcher's template allocator would avoid collisions, but
  preview names would no longer match launched names, the two proposals of one run would land on non-adjacent tokens
  (`.0`, `.3`), `%wait:` correctness would depend on the launch result propagating the rendered name, and run grouping
  is lost.
- _Forced reuse_ (`%name:!...`): requires interactive confirmation (`AgentNameReuseConfirmationRequiredError`) by
  design; wrong for headless launches, and it would destroy prior-run agent identity.
- _Releasing dismissed names from the registry_: durable name reservation is intentional (dismissed agents stay
  addressable); loosening it for this bug would be a far riskier semantic change.

### 2. Chop launches never write the user prompt stash

Gate the failed-launch stash on the launch's chop context. `launch_chop_proposals` already passes `extra_env` containing
`SASE_CHOP_NAME` / `SASE_CHOP_LUMBERJACK` / `SASE_CHOP_RUN_ID` (built by `build_chop_launch_env` in
`src/sase/axe/chop_agents.py`). In the from-cwd launch path (`src/sase/agent/launch_cwd_agents.py`, and
`launch_cwd_bead_work.py` if its failure sites are reachable with chop env), every `record_failed_launch_prompt` call
that has `extra_env` in scope must skip the call (or invoke a variant that skips the stash append) when the env marks a
chop launch — use the existing `ENV_CHOP_NAME` constant rather than a new string literal, via a small helper such as
`is_chop_launch_env(extra_env)`. Nothing is lost: the prompt is durably recorded in the chop run history and the failure
in the axe error digest. Interactive/TUI launch paths keep today's stash-on-failure behavior.

### 3. Failed runs advance the `run_every` clock

In `src/sase/axe/lumberjack.py` `_outcome_to_result`, return `update_timestamp=chop.run_every is not None` from the
failed branches (`failure` / `check_error` / `action_failed`) and the `timeout` branch, matching the success and
`skipped` branches. Rationale: `run_every` is a rate limiter on _attempts_; re-fire semantics after failure already
belong to the trigger checkpoint policy (`on_action_success` keeps the trigger hot, so a failing chop still retries —
but at its configured cadence, e.g. every 3h, not every 60s tick). Behavior change to call out in the commit message: a
transiently failing `run_every` chop now waits a full period before retrying; failure visibility via error
digests/notifications is unchanged.

## Testing

- **sase-core**: extend `axe_chop` Rust tests for the new name shape (with and without run token, sanitization of odd
  run ids, truncation) and keep the no-token compatibility shape.
- **This repo** (`just check` must pass):
  - Update `tests/test_core_facade/test_axe_chop.py` (currently asserts `chop.refresh_docs.sase-core.1`) and
    `tests/test_axe_chop_refresh_docs.py` (asserts `agent_name` / `wait_name` / `%wait:` previews) for the run-token
    shape — assert the token is derived from the supplied `run_id` and that two proposals in one run share it.
  - New test: two consecutive prepared runs with different `run_id`s produce non-colliding names (the regression this
    incident exposed).
  - New test: a failed launch whose `extra_env` carries `SASE_CHOP_NAME` does not append to the prompt stash (and one
    asserting the interactive path still does).
  - New test: a lumberjack tick whose chop returns `action_failed` (and `timeout`) updates the `run_every` timestamp so
    the next tick skips the chop.

## Remediation runbook (operator steps, after the code fix lands)

These accompany the fix but are host-state operations, not repo changes:

1. Release sase-core, bump the pin here, land this repo's change, then upgrade the installed tool and restart the axe
   daemon (`sase axe start` currently runs the released uv-tool install, so the storm continues until then; interim
   mitigation if desired: disable the `refresh_docs` lumberjack in `~/.config/sase/sase_athena.yml` and restart axe).
2. Purge the junk stash rows **through the Rust-backed store, not by hand-editing the JSONL**: read the snapshot via
   `sase.core.prompt_stash_facade.read_prompt_stash_snapshot`, select entries with `source == "failed_launch"` whose
   text contains `%tribe:chop` (equivalently `%name:chop.`), and remove them with `pop_prompt_stash` so pinned and human
   entries are untouched. (~1,480 rows as of 2026-07-19.)
3. Leave the eight reserved `chop.refresh_docs.<project>.1/.2` registry names alone — with run-token names they can
   never collide again.

## Out of scope / follow-ups

- `lowest_name_suggestion` (`src/sase/agent/names/_registry.py`) suggests `<name>1` by string concatenation, yielding
  the unhelpful `chop.refresh_docs.sase.11` for dotted numeric names; a numeric-leaf-aware suggestion would be a nice
  cosmetic follow-up.
- Successful chop launches currently also flow through `add_or_update_prompt` user prompt history; whether
  machine-generated prompts belong in human prompt history is a separate product question.
- Retry backoff policies for failing chops beyond the `run_every` cadence fix.
