---
tier: tale
title: Stop the ACE Memory pane from crashing on human strand previews
goal:
  A human running `sase ace` can open the glossary with `<ctrl+g>G`, preview any strand
  body, and have the read recorded as an interactive read instead of crashing the app.
size: medium
proposed_by: bbugyi200.athena.0dn
---

# Plan: Stop the ACE Memory pane from crashing on human strand previews

## Problem

Opening the glossary from the prompt input (`<ctrl+g>G`) kills the whole ACE TUI:

```
AgentIdentityError: memory reads require agent attribution; set SASE_AGENT_NAME,
or provide SASE_ARTIFACTS_DIR/agent_meta.json with a name
```

## Root cause

Two independent defects compose into a full-app crash.

### 1. A human-facing surface calls an agent-only gate

`<ctrl+g>G` dispatches `request_open_glossary_panel`
(`src/sase/ace/tui/widgets/_prompt_input_bar_g_prefix_actions.py:59`, `:288`), which
posts `GlossaryPanelRequested` with a `glossary:<slug>` identity. The app opens
`MemoryPane` seeded with that identity, the strand row becomes selected, and
`MemoryPane._ensure_strand_read_for_current_selection`
(`src/sase/ace/tui/modals/memory_pane.py:497`) starts a thread worker that calls
`record_memory_panel_strand_read` (`src/sase/ace/tui/modals/memory_panel_load.py:158`).

That function calls `require_agent_identity()`
(`src/sase/ace/tui/modals/memory_panel_load.py:171` → `src/sase/memory/read_log.py:345`
→ `src/sase/agent/identity.py:65`), which raises `AgentIdentityError` whenever neither
`SASE_AGENT_NAME` nor a named `SASE_ARTIFACTS_DIR/agent_meta.json` is present.

ACE is the human's TUI. A human shell has neither, so **every** strand preview by a
human raises. The crash locals confirm it: `identity='glossary:memory-web'`, `env=None`,
`identity=None`.

Memory reads are the only audit log in the repo that hard-requires an agent. Three
sibling logs already solve exactly this problem with an interactive fallback that stamps
`agent_source="interactive"` and the local username:

- `src/sase/artifact_read_log.py:88-96`, `:225-232`
- `src/sase/repo_open_log.py:87-95`, `:250-258`
- `src/sase/core/artifact_consumption.py:94-101`, `:157-165`

Each is covered by a test (`tests/test_artifact_read_log.py:55`,
`tests/test_repo_open_log.py:53`, `tests/artifact_consumption/test_ledger.py:85`). The
memory read log is the outlier, not the rule.

### 2. A failing Memory pane worker exits the app

All five `run_worker` calls in `src/sase/ace/tui/modals/memory_pane.py` (`:336`, `:350`,
`:362`, `:379`, `:531`) rely on Textual's default `exit_on_error=True`. Textual's
`Worker._run` calls `app._handle_exception(WorkerFailed(...))` on that default, which is
what printed the rich traceback panel and tore the app down. Roughly 25 other
`run_worker` call sites in this repo pass `exit_on_error=False`; the Memory pane does
not.

This matters beyond the identity bug: the pane **already** renders a graceful failure
state. `_on_strand_read_state_changed` (`memory_pane.py:535`) stores `error:<detail>`
and `_body_preview_for_node` (`src/sase/ace/tui/modals/memory_panel_view.py:252-262`)
renders `_Could not record audited read: ..._`. That code is currently unreachable,
because the app dies before the ERROR branch can render. Any future worker failure (an
`OSError` while reading a memory directory, for instance) would kill ACE the same way.

### Why the tests missed it

`install_fake_strand_read` (`tests/ace/tui/modals/memory_panel_test_helpers.py:239-247`)
replaces the real recorder in every panel test, with the comment "Selecting a strand row
otherwise requires real agent-identity env".
`test_record_strand_read_uses_memory_read_audit_path`
(`tests/ace/tui/modals/test_memory_panel_load.py:210`) sets `SASE_AGENT_NAME` before
calling. No test ever ran the real recorder without agent env, and no test asserted that
a failing pane worker leaves the app alive.

## Design

**Memory reads join the established interactive-identity convention, and that convention
gets one implementation.**

`sase memory read` (the CLI) keeps requiring an agent identity: it is the agent surface,
and `sase memory show` is the documented human equivalent. Only the ACE pane changes.

## Steps

### 1. Add one shared interactive-capable identity resolver

In `src/sase/agent/identity.py`:

- Extend `AgentIdentitySource` (`:16`) to
  `Literal["SASE_AGENT_NAME", "agent_meta", "interactive"]`. Nothing in `src/` or
  `tests/` compares an identity's `source` against those literals, so this is additive;
  verify with a grep before and after.
- Add `interactive_user_name(env, *, login_user=None) -> str`, lifted verbatim from the
  three existing private `_interactive_user` helpers: explicit `login_user`, then
  `getpass.getuser()` (catching `KeyError`/`OSError`), then `env["USER"]`, then
  `"unknown"`.
- Add `resolve_audit_identity(env=None, *, login_user=None) -> AgentIdentity`: return
  `discover_agent_identity(env)` when it is not `None`; otherwise return
  `AgentIdentity(name=interactive_user_name(...), source="interactive", artifacts_dir=_clean_value(env.get("SASE_ARTIFACTS_DIR")))`.
- Export both from `__all__`.

Keep `discover_agent_identity` and `require_agent_identity` semantically unchanged —
they must never return an interactive identity, because agent-only gates depend on that
(`require_proposal_author` in `src/sase/memory/proposals/identity.py:37`,
`sase memory read`, `src/sase/skills/cli_use.py:24`).

Carrying `SASE_ARTIFACTS_DIR` on the interactive branch matches all three existing logs
and is deliberate: that branch is only reachable when the dir is set but carries no
usable `agent_meta.json` name, and in that case the read really did happen inside that
run.

### 2. Use it for the ACE strand preview

In `src/sase/ace/tui/modals/memory_panel_load.py`:

- Replace the `require_agent_identity` import with `resolve_audit_identity` (re-exported
  through `sase.memory.read_log` alongside the other identity symbols, so the module
  keeps importing its audit helpers from one place).
- In `record_memory_panel_strand_read` (`:171`), call `resolve_audit_identity()` instead
  of `require_agent_identity()`.

Everything downstream is unchanged: the same selector resolution, the same
`build_memory_read_event_for_view`, the same `append_memory_read_event`. The pane still
gates the body preview on a successfully recorded read.

### 3. Never let a Memory pane worker exit the app

In `src/sase/ace/tui/modals/memory_pane.py`, pass `exit_on_error=False` to all five
`run_worker` calls (`:336`, `:350`, `:362`, `:379`, `:531`).

No new stuck states result. `_on_initial_load_state_changed` (`:395`) and
`_on_scope_load_state_changed` (`:415`) already clear `_loading` on `WorkerState.ERROR`;
the picker (`:434`) and restat (`:482`) handlers return on non-SUCCESS, which leaves
their state untouched and is correct.

### 4. Collapse the duplicated interactive fallback

Point the three existing logs at the shared resolver so there is one implementation
rather than four:

- `src/sase/artifact_read_log.py:88-96`
- `src/sase/repo_open_log.py:87-95`
- `src/sase/core/artifact_consumption.py:94-101`

Each becomes `identity = agent or resolve_audit_identity(current_env, login_user=...)`
followed by plain `identity.name` / `identity.source` / `identity.artifacts_dir` reads
(`artifact_read_log` has no `login_user` parameter; omit it there). Delete each
now-unused private `_interactive_user`; keep `_optional_text` wherever it still has
other callers, and delete it where it does not, so Symvision's unused-symbol lint stays
clean.

This step must be behavior-preserving: the three existing interactive tests
(`tests/test_artifact_read_log.py:55`, `tests/test_repo_open_log.py:53`,
`tests/artifact_consumption/test_ledger.py:85`) must pass untouched.

## Tests

1. `tests/agent/test_identity.py` — `resolve_audit_identity` returns the agent identity
   when `SASE_AGENT_NAME` is set; with no agent env it returns `source == "interactive"`
   and the interactive username, including the `USER` fallback when `getpass.getuser()`
   raises. Assert `discover_agent_identity` and `require_agent_identity` still return
   `None` / raise for the same no-agent env, so the agent-only gates are provably
   unchanged.

2. `tests/ace/tui/modals/test_memory_panel_load.py` — a sibling of
   `test_record_strand_read_uses_memory_read_audit_path` that `delenv`s
   `SASE_AGENT_NAME` and `SASE_ARTIFACTS_DIR` (`raising=False`), calls
   `record_memory_panel_strand_read`, and asserts it returns normally, that one event is
   appended with `kind == "strand"` and the expected selectors, and that
   `agent_source == "interactive"`. This is the regression test for the reported crash.

3. `tests/ace/tui/modals/test_memory_panel.py` — a pilot test using the existing
   `MemoryPanelTestApp` harness that monkeypatches `record_memory_panel_strand_read` to
   raise, selects a strand, and asserts the app is still running and
   `_strand_read_status["decisions:alpha"]` starts with `error:` while
   `_body_preview_for_node` renders "Could not record audited read". This covers defect
   2 independently of defect 1 and makes the pane's existing error branch reachable.

4. Leave `install_fake_strand_read` in place for the other panel tests, but update its
   docstring: the fake now exists for speed and determinism, not because the real
   recorder needs agent env.

## Verification

- `just check` for the lint gates plus the diff-scoped test lane.
- `just check-full` through `/sase_monitor` before landing, since
  `src/sase/agent/identity.py` is broadly imported and step 4 touches three audit-log
  modules.
- Manual: run `sase ace` from a plain human shell with `SASE_AGENT_NAME` and
  `SASE_ARTIFACTS_DIR` unset, press `<ctrl+g>G` in the prompt input, and confirm the
  glossary strand body renders. Then confirm the read landed as interactive:
  `sase memory log` should show the row with agent source `interactive` and the local
  username.

## Non-goals

- **`sase memory read` keeps requiring an agent identity.** It is the agent-facing
  audited command; humans use `sase memory show`, which records nothing.
- **Memory writes are unchanged.** The pane's add/delete/publish flows
  (`src/sase/ace/tui/modals/memory_panel_write.py`) call the mutation API directly and
  already capture failures into a typed `error` field; they do not touch
  `require_proposal_author`.
- **No feature flag.** This is a crash fix; there is no deprecated branch that must stay
  reachable, and nothing here is behavior a user is meant to opt into.
