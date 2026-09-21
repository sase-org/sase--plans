---
tier: tale
title: Preserve one-shot launch handoffs across the post-wait runner code refresh
goal:
  A clan-declaring agent whose dependency wait crosses a sase code update keeps its clan
  membership and local xprompts after the runner re-exec instead of failing with "clan
  already exists" or silently losing prompt expansions.
size: medium
proposed_by: bbugyi200.athena.0ox
create_time: 2026-09-21 18:30:39
status: wip
---

# Plan: Preserve one-shot launch handoffs across the post-wait runner code refresh

## Problem

`research.24.final` (the lead agent of a `#research_swarm` dispatch) failed 22 seconds
after its 78-minute dependency wait finished:

```
Refreshing sase runner code after dependency wait: 04d35849… -> 2f15d0a7…
Error running agent: clan 'research.24' already exists; join it with %id(<id>, clan=research.24)
```

Its prompt declared the clan
(`%clan(research.24, tribe=research, summary=…) %id:research.24.final %wait:research.24.cld …`).
The launch itself had succeeded: the pre-wait `agent_meta.json` already recorded
`agent_clan: research.24` and `agent_clan_generation: 20260921162653` (the generation
founded by `research.24.cld`).

## Root cause

1. The multi-prompt launcher's clan prepass (`sase/agent/multi_prompt_launch_plan.py`)
   reserves each clan once per batch and hands every member the resolved
   `ClanMembershipPlan` through the one-shot `SASE_AGENT_CLAN_MEMBERSHIP` env var.
2. The first runner bootstrap pass consumes it:
   `consume_clan_membership_plan_from_env()` (`sase/agent/clan_membership.py`) does
   `os.environ.pop(...)`, so nested launches cannot inherit it.
3. During the long wait the editable sase checkout's HEAD moved, so
   `refresh_runner_code_after_wait()` (`sase/axe/run_agent_runner_refresh.py`) re-execs
   the runner with `os.execv(sys.executable, [sys.executable, *sys.argv])`. The re-exec
   inherits the already-popped environment, so the refreshed pass has no clan payload.
4. The refreshed pass runs `bootstrap_agent_run()` →
   `extract_directives_and_write_meta()` again. With `clan_membership_plan is None` and
   `directives.clan` set, `resolve_agent_identity()`
   (`sase/axe/run_agent_directive_identity.py`) calls `_resolve_clan_membership()`.
   Because the prompt _declares_ the clan (`directives.clan_declared`), it uses
   `declare_clan_membership()` → `create_only=True`.
5. `reserve_registered_clan_name()` (`sase/agent/names/_registry_group_mutations.py`)
   finds the existing `research.24` clan entry (owned by the founder's artifacts dir,
   different generation than this agent's artifacts dir name), so the
   planned-reservation short-circuit does not match, and `create_only` raises
   `NameCollisionError`.

The durable state needed to avoid this already exists: `preserved_agent_metadata()`
(`sase/axe/run_agent_directive_metadata.py`) explicitly carries `agent_clan`,
`agent_clan_generation`, `clan_tribe`, and `clan_summary` across the re-exec. However,
nothing rebuilds the membership plan from those fields. The batch predecessor context
already follows the correct pattern: `consume_batch_predecessor_context_from_env()`
falls back to `preserved_batch_predecessor_context(preserved_metadata)`.

The failure is deterministic whenever all three conditions hold: a clan-_declaring_
member (every research swarm lead and any `%clan` fan-out member) has a blocking
`%wait`, and sase HEAD moves during that wait. Given how often master moves, long
research waits hit this routinely. Joiners (`%id(x, clan=…)`) do not crash, because
`resolve_or_create_clan_membership()` returns the existing generation. They still
re-resolve instead of reusing their launch identity.

### Same root cause, second instance: local xprompts

`extract_directives_and_write_meta()` also pops `SASE_AGENT_LOCAL_XPROMPTS`, which
carries multi-prompt frontmatter xprompts, and unlinks its temp file. On the refreshed
pass `info.local_xprompts` loses those definitions. Both
`expand_deferred_launch_xprompts(...)` in `_admit_and_launch()` and the launch phase
then leave `#<local_name>` references unexpanded. This was verified:
`process_xprompt_references("#_x\nplease", extra_xprompts=None)` returns the reference
verbatim. The agent silently runs a wrong prompt.

### Audit of the other one-shot inputs consumed before the refresh point

These are already safe and need no behavior change. Record them in the refresh module
docstring (step 4):

- `SASE_LAUNCH_HOLD_KEY`: `arm_bootstrap_hold()` returns early on a refreshed pass.
- Epic-work env (`epic_work_metadata_from_env()`): its fields are preserved metadata
  keys.
- `SASE_EPIC_CLAN_SUMMARY_SCRIPT`: the resolved `clan_summary` is preserved metadata.
- `SASE_AGENT_PREDECESSOR_CONTEXT`: already uses the preserved-metadata fallback.
- `SASE_AGENT_PLANNED_NAME`: `refresh_runner_code_after_wait()` restores it.
- `SASE_AGENT_GENERATED_NAME`: lost, but harmless. The refreshed claim targets a name
  already owned by the same artifacts dir, and `claim_registered_name()` accepts
  same-owner claims regardless of `explicit`.

## Changes

### 1. Rebuild clan membership from preserved metadata (`sase/agent/clan_membership.py`)

- Add
  `preserved_clan_membership_plan(preserved: Mapping[str, Any]) -> ClanMembershipPlan | None`.
  It returns a plan only when both `AGENT_CLAN_FIELD` and `AGENT_CLAN_GENERATION_FIELD`
  are non-empty strings, and `None` otherwise. Add it to `__all__`.
- Extend the `consume_clan_membership_plan_from_env()` docstring. The env payload is
  first-pass only; a refreshed runner pass recovers the same plan through
  `preserved_clan_membership_plan()`.

### 2. Use the fallback during directive extraction (`sase/axe/run_agent_directives.py`)

Right after `clan_membership_plan = consume_clan_membership_plan_from_env()`, if the
result is `None` **and** `directives.clan is not None`, set it to
`preserved_clan_membership_plan(preserved_metadata)`. This mirrors the batch-predecessor
fallback a few lines above.

- The env payload keeps precedence over preserved metadata.
- Do not fall back when `directives.clan is None`. Family-attach continuations inherit
  clan metadata via `agent_meta.update(inputs.preserved)`, and handing them a plan would
  trip the existing "payload requires a %clan directive" error.
- Add a short comment explaining why the fallback is safe. `bootstrap_agent_run()` is
  the only caller, and its first pass overwrites `agent_meta.json` via
  `_write_bootstrap_agent_meta(refreshed=False)`, so preserved clan fields exist only on
  a refreshed pass. That pass replays the identical submitted prompt.
- Effect: `resolve_agent_identity()` skips `_resolve_clan_membership()`.
  `_claim_agent_identity()` → `claim_registered_clan_name()` becomes a no-op for a
  non-founding member (the entry is a claimed `clan` owned by another artifacts dir) and
  an idempotent rewrite for the founder. `_add_clan_metadata()` still enforces the
  `<clan>.<suffix>` hood check.
- The create-only guard stays intact for genuinely new launches. A fresh `%clan(x)`
  declaration of an existing clan, with no env payload and no preserved metadata, must
  still raise `ClanMembershipError`.

### 3. Re-materialize local xprompts before the refresh exec

- In `sase/agent/multi_prompt_xprompts.py`, add
  `LOCAL_XPROMPTS_ENV = "SASE_AGENT_LOCAL_XPROMPTS"` next to `serialize_local_xprompts`.
  Use it instead of the string literal in `run_agent_directives.py`.
- In `sase/axe/run_agent_runner_refresh.py`, add a keyword parameter
  `local_xprompts: Mapping[str, Any] | None = None` to
  `refresh_runner_code_after_wait()`. After the prompt-file restore succeeds and before
  `os.execv`, when `local_xprompts` is non-empty, serialize it with
  `serialize_local_xprompts()` (lazy import) and set `LOCAL_XPROMPTS_ENV` to the new
  path. This follows the module's documented contract of re-materializing one-shot
  resources before exec, just as it does for the prompt file.
  - Serialization failure: print a warning and skip the refresh. This matches the
    prompt-file-restore failure path, and continuing on old code with the correct prompt
    beats running new code with a wrong one.
  - Exec failure: restore the previous `LOCAL_XPROMPTS_ENV` value (normally absent) and
    unlink the newly serialized file. Do this alongside the existing planned-name and
    refresh-guard restoration.
  - Success: the refreshed pass's existing pop-and-unlink in
    `extract_directives_and_write_meta()` cleans the file up, so nothing leaks.
- In `sase/axe/run_agent_runner.py` `_run_agent()`, pass
  `local_xprompts=bootstrap.info.local_xprompts` to `refresh_runner_code_after_wait()`.
  This includes frontmatter-defined entries. Re-serializing them is harmless because
  frontmatter still wins the merge.

### 4. Document the invariant so the next handoff does not regress

Rewrite the `run_agent_runner_refresh.py` module docstring to state the general rule:
every one-shot launch handoff consumed by the pre-wait pass must survive the refresh
re-exec in one of two ways.

- **Durable identity:** the refreshed pass recovers it from `preserved_agent_metadata()`
  (clan membership, batch predecessor context, epic work, model selection).
- **One-shot file or env resource:** this module re-materializes it before exec (the
  prompt file, local xprompts, the planned name).

List the audited-safe inputs from the "Audit" section above. Anyone adding a new
`consume_*_from_env()` / `os.environ.pop(...)` in the bootstrap path then has a
checklist to follow.

## Tests

Use `tests/_agent_names_extract_fixtures.py` (`run_extract`, `mock_provider`) and the
existing refreshed-extract tests in `tests/test_agent_names_extract_metadata.py`
(`test_preserved_batch_predecessor_context_rebinds_refreshed_wait`) as the pattern. For
registry setup, reuse the clan registry fixtures used by
`tests/test_parallel_agent_family_metadata.py`,
`tests/_clan_summary_persistence_helpers.py`, or
`tests/test_agent_generated_name_guard.py`. Registry state must be isolated in
`tmp_path`, never the real `~/.sase`.

Add, in a new focused test module or next to the existing refreshed-extract tests:

1. **Exact regression.** Reserve and claim a declared clan (for example `rc`, generation
   `G`) for a _founder_ artifacts dir. Write this agent's `agent_meta.json` with
   `agent_clan: rc` and `agent_clan_generation: G`. Run extraction for
   `%clan(rc, tribe=research)\n%id:rc.final\n...` with no `SASE_AGENT_CLAN_MEMBERSHIP`.
   Assert no error, `meta["agent_clan"] == "rc"`, `meta["agent_clan_generation"] == G`,
   and a clan registry entry still owned by the founder with generation `G`.
2. **Two-pass replay.** Run the first extraction with the env payload
   (`encode_clan_membership_plan`) as the launcher would, then a second extraction for
   the same artifacts dir with the payload absent. The second pass mimics the re-exec
   environment: set `SASE_RUNNER_CODE_REFRESHED=1` and `SASE_AGENT_PLANNED_NAME` to the
   first pass's name. Assert both passes produce the same `name`, `agent_clan`,
   `agent_clan_generation`, `clan_tribe`, and `wait_for`. Cover both a non-founding
   declared member and the founding member.
3. **Guard intact.** With no env payload and no preserved metadata, re-declaring an
   existing clan still raises `ClanMembershipError`, and the message still says to join
   with `%id(<id>, clan=…)`.
4. **Precedence.** An env payload wins over conflicting preserved metadata.
5. **No clan directive.** Preserved `agent_clan` fields combined with a prompt that has
   no `%clan` / `clan=` must not raise, and must not fabricate a membership plan.
6. **Helper unit test.** `preserved_clan_membership_plan()` returns `None` for missing,
   empty, or non-string fields.

In `tests/test_run_agent_runner_refresh.py`, following the existing
`test_changed_identity_reexecs_original_argv` / `test_exec_failure_*` patterns with
`os.execv` patched to capture `os.environ`:

7. When the refresh fires with non-empty `local_xprompts`, `SASE_AGENT_LOCAL_XPROMPTS`
   points at a file that `deserialize_local_xprompts()` round-trips to the same names
   and content.
8. On exec failure, the env var is restored (absent, or its prior value) and the
   serialized file is removed.
9. With no or empty `local_xprompts`, the env var is left untouched.
10. If serialization raises, the refresh is skipped with a warning and `os.execv` is
    never called.
11. **Boundary replay.** A first `extract_directives_and_write_meta()` with
    `SASE_AGENT_LOCAL_XPROMPTS` set yields `info.local_xprompts`. Feed that to the
    refresh with `execv` patched, then run a second extraction on the captured env. The
    local xprompt names and content must match the first pass. Reuse the setup from
    `test_batch_predecessor_context_binds_local_xprompt_wait`.

Update any existing test that asserts the exact keyword arguments of
`refresh_runner_code_after_wait()`. Check `tests/test_run_agent_runner_wait_queue.py`.

## Verification

- `just install` if the workspace venv is stale.
- Targeted runs:
  `pytest tests/test_agent_names_extract_metadata.py tests/test_run_agent_runner_refresh.py tests/test_run_agent_runner_wait_queue.py`
  plus the new test module.
- `just fmt`, then `sase tool run check` (the agent default; do not run `check-full`).

## Out of scope

- No registry semantics change. `reserve_registered_clan_name(create_only=True)` stays
  strict, because the bug is a missing launch handoff, not an overly strict guard.
- No Rust core change. This is Python runner-bootstrap glue plus the Python clan
  registry.
- Recovering the failed run is a user action. A TUI retry of `research.24.final` already
  demotes its `%clan(...)` declaration into a clan join (see
  `test_retry_edit_agent_demotes_clan_declaration`), so the retry does not hit the
  collision.
