---
tier: epic
title: Durable feature-flag controls in the SASE Admin Center
goal: Users can inspect and persistently enable or disable every registered SASE feature
  flag from either a polished Config > Flags pane or the sase flag CLI, with both
  surfaces sharing one crash-safe state mutation path and applying changes through
  the established ACE and AXE restart flows without editing normal configuration files.
phases:
- id: core
  title: Rust feature-flag preference store and bindings
  depends_on: []
  size: medium
  description: 'core: add a versioned, locked, atomic machine-state store for persistent
    feature-flag booleans in sase-core, expose strict PyO3 read and set bindings,
    and prove concurrency, corruption, downgrade, and durability behavior.'
- id: floor
  title: Adopt the released core binding floor
  depends_on:
  - core
  size: small
  description: 'floor: wait for the core release, raise sase''s sase-core-rs dependency
    floor, refresh the lockfile and editable install, and smoke-test the published
    bindings before Python callers land.'
- id: runtime
  title: Shared Python resolution and mutation facade
  depends_on:
  - floor
  size: medium
  description: 'runtime: add the thin typed Python adapter, insert saved machine preferences
    into feature-flag resolution at the designed precedence, synchronize process transport
    after writes, and return one structured mutation outcome used by every frontend.'
- id: cli
  title: Persistent flag enable and disable commands
  depends_on:
  - runtime
  size: medium
  description: 'cli: add sorted sase flag disable and enable commands with rich and
    JSON results, completion, idempotent shared mutations, and the existing AXE restart
    machinery, including clear partial-success reporting.'
- id: tui
  title: Beautiful Config Flags pane and controlled restart flow
  depends_on:
  - runtime
  size: medium
  description: 'tui: create the default-on sunset rollout flag and its live call site,
    add the lazy Config > Flags list/detail pane with filtering, provenance, removal
    metadata, confirmation and narrow layouts, and apply successful toggles through
    the existing proc-aware ACE plus AXE restart path.'
- id: polish
  title: Integrated documentation, visual coverage, and release verification
  depends_on:
  - cli
  - tui
  size: small
  description: 'polish: align all user documentation and help, exercise the complete
    CLI and TUI journeys including both rollout-flag states and restart failures,
    refresh intentional PNG goldens, and run the repository''s exhaustive landing
    gates.'
proposed_by: bbugyi200.athena.09g
create_time: 2026-08-21 10:00:39
status: wip
bead_id: sase-rs
---

- **BEAD:** [sase-rs](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rs/README.md)

# Plan: Durable feature-flag controls in the SASE Admin Center

## Outcome

Add **Flags** as the first, alphabetically ordered child of the Admin Center's
**Config** tab. The pane is a keyboard-first control surface for every code-owned SASE
feature flag: it shows effective state, kind, default, provenance, saved machine
preference, description, removal bead, and removal horizon; lets the user filter and
inspect the catalog; and confirms an enable/disable action before saving it and
restarting ACE and AXE.

The same durable operation is exposed as:

```text
sase flag enable <flag>
sase flag disable <flag>
```

Neither surface edits `~/.config/sase/sase.yml`, an overlay, a project-local `sase.yml`,
or chezmoi source. They write one SASE-owned machine-state file under `SASE_HOME`, so
the choice survives logout, reboot, and upgrades while remaining separate from portable
user configuration.

## Why this is an epic

This crosses a repository and release boundary: shared state and mutation semantics
belong in `sase-core`, while resolution, CLI presentation, Textual presentation, and
restart integration live in `sase`. The core binding must be released and adopted before
the Python callers can land. CLI and TUI work can then proceed independently against one
shared runtime facade, followed by integrated visual and exhaustive verification.

## Existing seams to preserve

- Registry definitions live in `src/sase/feature_flags/registry.py`; defaults are
  derived from `beta` (off) and `sunset` (on).
- `src/sase/feature_flags/resolver.py` currently resolves registry defaults, user and
  overlay config, in-process overrides, `SASE_FEATURE_FLAGS`, and root
  `-f/--enable-feature` / `-F/--disable-feature` values in that order.
- `install_process_feature_flags()` pins one snapshot and writes the resolved values
  into `SASE_FEATURE_FLAGS` so children inherit the parent's behavior.
- `flag_views()` already joins definitions, decisions, removal beads, and due state for
  `sase flag list` and `show`; the TUI must build on that model instead of inventing a
  second registry projection.
- Config children are declared lazily in `config_hub_catalog.py`, with their typed and
  remembered order in `config_hub_session.py`.
- `sase.ace.tui.update_restart.restart_after_update_when_ready()` already waits for
  tracked background procs and routes a controlled re-exec through
  `_restart_tui(restart_axe=True)`.
- `sase.main.update_restart.restart_after_update()` already checks whether AXE is
  running, invokes the robust daemon restart, and returns structured `RestartInfo`.

## Binding design decisions

### 1. Persist preferences as SASE state, not configuration

Use `sase_home() / "feature_flags.json"` (normally `~/.sase/feature_flags.json`,
honoring `SASE_HOME`) and a sibling lock file. The stable wire shape is deliberately
small:

```json
{
  "version": 1,
  "flags": {
    "epic_resume_gate": true,
    "prettier_enabled": false
  }
}
```

This is a bounded, machine-local preference store, not a new config layer file that a
human is expected to edit. Store exact booleans even when they equal a registry default:
`enable` and `disable` are durable user choices, not an optimization to “non-default
only.” A reset-to-config/default command is explicitly outside this request.

The Rust store owns read/modify/write concurrency, a short bounded lock wait, stable key
ordering, a strict size cap, UTF-8/JSON/schema validation, temp-file cleanup, file
flush, atomic replace, and parent-directory sync. Missing state is an empty snapshot. A
malformed or unsupported whole file is not silently deleted or overwritten: reads return
no preferences plus a diagnostic, and mutations fail with the path and a
recovery-oriented message. Syntactically valid keys unknown to the current binary are
preserved across writes and ignored with a diagnostic; that prevents a temporary binary
downgrade from destroying preferences written by a newer release.

The Rust store validates wire shape and snake_case keys but does not own the Python
registry. The Python facade rejects a requested mutation unless the key is currently
registered.

### 2. Keep existing override precedence

The complete resolution order becomes:

```text
registry default
  < user config
  < overlay config
  < saved machine preference
  < explicit in-process/test override
  < SASE_FEATURE_FLAGS transport / legacy env
  < root CLI -f/-F override
```

The source enum gains a stable `state` value, rendered to users as **SAVED**, with the
resolved state path as detail. Existing environment and root CLI precedence must not
change: those mechanisms intentionally pin a process family or force one invocation. The
pane therefore shows both **effective** and **saved** values. If a higher-precedence
environment or CLI override shadows the saved choice, show a prominent “forced for this
process” explanation rather than pretending the toggle already controls the process.

After a successful state write, the shared Python mutation facade invalidates the
process snapshot and merges the chosen key into `SASE_FEATURE_FLAGS`. This is essential
for ACE's `execv` restart: the current ACE process already exported its old pinned
snapshot, and without updating that transport the inherited old value would outrank the
new saved state on re-exec. Root CLI overrides still win when deliberately supplied.

### 3. One mutation facade, context-aware restart orchestration

Both frontends call one Python operation, conceptually:

```python
set_saved_feature_flag(key: str, enabled: bool) -> FeatureFlagMutationOutcome
```

The immutable outcome includes the key, requested value, previous saved value, whether
the store changed, before/after effective decisions, whether a higher-precedence source
shadows the saved value, the state path, and state diagnostics. It performs no TUI or
console rendering and no daemon lifecycle action.

Restarting remains context-aware while reusing established lifecycle code:

- From Config > Flags, a successful mutation always goes through the proc-aware ACE
  restart helper, generalized only enough to say “apply feature-flag changes” instead of
  the current hard-coded “load new code.” It then requests the existing controlled TUI
  re-exec with `restart_axe=True`. The confirmation, footer, queued-restart toast, and
  final restart toast all tell the user what will happen.
- From standalone CLI, no foreground TUI belongs to that process and SASE must not
  signal an unrelated terminal. The command uses the existing CLI restart planner to
  restart AXE when it is running and explicitly tells the user that any separately
  running ACE session must be restarted. When the command is invoked from the Flags
  pane, the pane owns the current TUI and performs the full ACE+AXE path directly.
- A saved preference is never rolled back because restart fails. Rich and JSON output
  distinguish `saved` from `restart`, so a partial success is actionable and safe to
  retry. Repeating an already-saved enable/disable remains idempotent but retries the
  applicable restart contract.

### 4. Roll out the pane behind a default-on sunset flag

The UI itself is user-reaching behavior, so create `admin_center_flags` only through:

```text
sase flag new admin_center_flags --kind sunset \
  --when-enabled "The Config catalog exposes the Flags pane for persistent feature-flag control." \
  --when-disabled "The Config catalog omits Flags; sase flag enable and disable remain available." \
  --remove-when "The pane and CLI have soaked across both flag kinds, restart paths, and supported platforms without persistence or responsiveness regressions."
```

Paste the generated registry scaffold, sync the generated feature-flag schema, and add
the real non-test call site in the same phase so `tools/check_feature_flags` never sees
an unreferenced flag. `sunset` makes the completed pane visible by default while keeping
a rollback branch. Disabling `admin_center_flags` from its own row is supported: the
confirmation says the pane will disappear after restart and gives
`sase flag enable admin_center_flags` as the recovery command.

Do not gate the CLI commands with this flag. They are the stable recovery and automation
surface whether the pane is on or off.

### 5. Visual and interaction design

When the rollout flag is on, the alphabetized Config strip is:

```text
01 Flags · 02 Glossary · 03 Launch · 04 Memory · 05 Misc · 06 Snippets · 07 XPrompts
```

When it is off, retain today's six children and numbers exactly. The dynamic catalog
check must occur inside a function or instance path, never at module import time, and it
must read only the already-pinned process snapshot.

The pane uses the established Admin Center list/detail visual language:

```text
FLAGS  ·  9 registered  ·  5 on  ·  4 saved
┌──────────────────────────────┐ ┌──────────────────────────────────────────────┐
│ ● ON   β artifact_links      │ │ artifact_links                    ON  ·  BETA│
│ ○ OFF  β epic_resume_gate    │ │                                              │
│ ● ON   ↗ prettier_enabled    │ │ Agents add typed artifact links…             │
│ ...                          │ │                                              │
│                              │ │ EFFECTIVE  on       SOURCE  SAVED             │
│                              │ │ DEFAULT    off      SAVED   on                │
│                              │ │ BEAD       sase-rc  STATUS  open              │
│                              │ │ REMOVE BY  2026-… / 0.…                       │
└──────────────────────────────┘ └──────────────────────────────────────────────┘
/ filter  ·  enter toggle  ·  r refresh  ·  changes restart ACE + AXE
```

Use semantic theme colors: green/on, dim/off, beta and sunset chips, yellow for a
shadowing process override or approaching removal, and red only for actual errors or
overdue removal. Keep descriptions at a readable measure, preserve selection by flag key
across filtering/reloads, and use a guarded programmatic `OptionList` selection.
Debounce only the detail card; highlight movement paints immediately.

`/` opens an inline filter over key, description, kind, effective state, and provenance;
`Esc` closes/clears it before closing Admin Center. `j`/`k`, arrows, Home/End, mouse
clicks, and `Enter`/Space work consistently with neighboring panes. The confirmation is
cancel-first and shows `OFF -> ON` or `ON -> OFF`, the saved-state path, any shadowing
source, and “ACE and AXE restart after active procs finish.” No disk, bead, JSON, or
subprocess work runs on the Textual event loop or serial message pump.

## Explicitly out of scope

- Replacing the existing portable `feature_flags:` config surface. It remains valid and
  lower-precedence than a saved machine preference.
- A `sase flag reset` command or a three-state toggle. Enable and disable are the
  requested public contract; a future reset can remove one saved entry without changing
  this wire format.
- Editing flag definitions, kinds, descriptions, defaults, beads, or removal dates from
  the pane. Those remain code and bead lifecycle concerns.
- Remotely killing or re-execing an ACE process owned by another terminal from a
  standalone CLI command.
- Live-reconfiguring already-running agents. Process snapshots remain pinned; new
  processes inherit the new state after the controlled frontend/daemon restart.

## Phase `core`: Rust feature-flag preference store and bindings

Open the linked repo with `/sase_repo` and work only in the path it prints.

1. Add a `feature_flag_state` domain module in `crates/sase_core` with wire structs for
   the versioned snapshot and set outcome. Use `BTreeMap<String, bool>` for
   deterministic ordering and expose constants for schema version and state filename.
2. Reuse `store_lock` rather than adding another lock implementation. Set is one
   exclusive-lock read/modify/write transaction so concurrent commands changing
   different flags cannot lose each other's writes. Include useful lock-holder context
   in timeout errors.
3. Make reads bounded and non-destructive. Missing is empty; malformed, oversized, or
   unsupported state returns a typed diagnostic/error without deleting the file. Keep
   unknown but valid snake_case keys in the map.
4. Write pretty JSON plus a final newline through a same-directory temporary file, flush
   and sync it, atomically persist it, then sync the parent directory. Ensure failed
   writes leave the previous state intact and no temp litter.
5. Export Rust APIs and PyO3 bindings named consistently with existing store bindings,
   for example `feature_flag_state_get(sase_home)` and
   `feature_flag_state_set(sase_home, flag, enabled)`. Return JSON-shaped wire objects;
   do not import or duplicate the Python registry in Rust.
6. Add Rust and binding tests for missing state, both booleans, stable order, exact
   round trips, same-value idempotence, two concurrent writers, lock timeout, malformed
   JSON, wrong version/type/key, size limit, unknown-key preservation, atomic failure,
   temp cleanup, and Python-wire conversion.
7. Update the binding contract inventory/checks in the core repo.

Verification in `sase-core`: `cargo fmt --all -- --check`,
`cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`.
Commit as a `feat(core):` change; release-plz owns versions.

## Phase `floor`: Adopt the released core binding floor

1. Confirm the core phase is on the core repo's default branch and wait for the
   release-plz release containing both bindings. Verify the actual published version; do
   not predict it from today's `sase-core-rs>=0.29.5,<0.30.0` floor.
2. Raise only the inclusive `sase-core-rs` floor in `pyproject.toml` while preserving
   the compatible ceiling, refresh `uv.lock`, update the local linked core checkout, and
   run `just install`.
3. Add the two binding names to `tools/check_sase_core_rs_bindings` and its tests.
4. Probe the published wheel in a throwaway environment: read an empty temporary home,
   set two distinct flags, verify both survive, and verify a same-value set reports an
   idempotent outcome. This catches a published-wheel mismatch that name-only binding
   checks cannot.
5. Run the published-core floor/version probes and `just check`.

Do not land Python imports of the new bindings before this phase; every supported SASE
install must have the required core API.

## Phase `runtime`: Shared Python resolution and mutation facade

### Typed adapter and state projection

1. Add a thin `src/sase/feature_flags/state.py` adapter around the two required Rust
   bindings. Validate every wire field/version defensively, resolve the path through
   `sase_home()`, and map domain/binding failures to a feature-flag state error with the
   real path. The adapter must participate in pytest's redirected `SASE_HOME` boundary.
2. Represent the saved mapping and any read diagnostic separately. Unknown stored keys
   must not enter the effective snapshot, but must remain visible as diagnostics and
   survive a later set operation.
3. Add `state` to `FlagSource`, render it as `SAVED`, and update list/show JSON and
   provenance views without changing existing keys. Ensure `show` can explain the saved
   value even when env or CLI wins effectively.

### Resolver and process transport

1. Load the saved mapping once while building a process snapshot and apply it after
   overlay config but before in-process overrides and environment transport. Keep the
   pure resolver pure by passing explicit saved-state input; no filesystem access
   belongs in `resolve_feature_flags()` itself.
2. Preserve the current local-config rejection and plugin-layer exclusion. Saved state
   is global to the machine, not project scoped.
3. Add `set_saved_feature_flag()` as the only mutation facade. Validate registry
   membership, capture previous state/decision, call Rust, merge the selected value into
   `SASE_FEATURE_FLAGS`, invalidate the cached/pinned snapshot safely, resolve the
   after-decision, and return the structured outcome. Never edit config files.
4. Make same-value calls idempotent and keep enough result detail for frontends to retry
   lifecycle work. If a root CLI override still shadows the saved state, record that
   fact rather than reporting the effective value incorrectly.
5. Keep imports lazy enough that registry modules still perform no feature-flag
   resolution or state I/O at import time.

### Runtime tests

- Exhaustively cover beta and sunset defaults across user config, overlays, saved state,
  test overrides, env, and root CLI values, including source/source_detail.
- Prove a successful mutation changes only the SASE state path, leaves representative
  user/overlay/local config bytes and mtimes untouched, and preserves unrelated stored
  keys.
- Prove the environment merge makes an exec-style rebuild see the chosen value while a
  deliberate root CLI override still wins.
- Cover missing/corrupt/unknown stored state, binding failures, snapshot invalidation,
  immutable outcome models, stable diagnostics, and list/show rich plus JSON provenance.
- Extend the static import-time and binding-contract checks as needed.

Run focused feature-flag tests, `tools/check_feature_flags --static`, and `just check`.

## Phase `cli`: Persistent flag enable and disable commands

1. Extend `parser_flag.py` with subcommands in alphabetical order: `disable`, `enable`,
   `list`, `new`, `show`. Both new commands take the registered flag key as the required
   positional `flag_key`; add `-j/--json` for automation. Help and examples must say
   that the choice is machine-persistent, identify the state file conceptually, describe
   precedence, and state the AXE restart behavior.
2. Route both commands to one handler parameterized by the target boolean. The handler
   calls only `set_saved_feature_flag()`; it must not duplicate JSON writes, config
   targeting, validation, precedence, or environment synchronization.
3. Human output is concise and colored: key, `enabled`/`disabled`, previous saved value,
   effective value/source, state location, shadowing warning if any, then the AXE
   restart result. JSON gets a versioned envelope with separate `mutation` and `restart`
   objects and exactly one JSON document on stdout.
4. Use `is_axe_running`, `restart_axe_daemon_result`, and
   `sase.main.update_restart.restart_after_update()` with source attribution such as
   `sase flag enable`. Generalize shared rendering only where necessary so flag output
   says “apply the saved feature flag,” not “load updated code.” If AXE is stopped,
   report the established skipped-not-running state; do not start a daemon the user had
   stopped.
5. Persisted success survives restart failure. Return/report partial success clearly and
   make an idempotent repeat retry the AXE restart. Unknown flags are usage errors (exit
   2); store or restart failures use normal operational failure semantics.
6. Because both positional arguments use `flag_key`, reuse the existing dynamic flag
   completion provider. Refresh the structural CLI spec snapshot and command catalog
   tests; update dispatcher usage and compact/full help.

Test both commands through the public parser/handler boundary for changed, idempotent,
unknown, corrupt-store, shadowed, AXE stopped, AXE restarted, restart failure, rich,
JSON, help ordering, and completion. Run `just check`.

## Phase `tui`: Beautiful Config Flags pane and controlled restart flow

### Rollout flag and catalog integration

1. Run the exact `sase flag new admin_center_flags --kind sunset ...` workflow from the
   binding decision, paste its generated registry member/definition, and sync the
   generated schema. Do not use `/sase_new_task` or hand-create the flag bead.
2. Add the non-import-time call site that filters the Config child catalog. When on,
   insert Flags first and renumber all seven labels. When off, preserve today's catalog,
   numbers, default XPrompts child, and resume validation.
3. Extend `ConfigSubTab`, catalog factories, and Config-hub session state with a Flags
   `SelectionBookmark`; keep the pane lazy and cached like its siblings. Recalibrate
   compact/micro strip thresholds for seven labels rather than allowing silent clipping.

### Pane implementation

1. Build a `FeatureFlagsPane` with a header, `OptionList` rail, scrollable detail card,
   hidden inline filter, and one-line contextual footer. Split pure row/card/footer
   rendering from widget state so colors and text are unit-testable.
2. Load `flag_views()`, saved-state details, and bead metadata in one thread worker.
   Paint a lightweight loading shell immediately. No synchronous file, bead, JSON, or
   subprocess work may run in compose, render, key, message, timer, or serial
   `call_after_refresh` paths.
3. Sort by key, preserve selection by key, and filter key/description/kind/effective
   state/provenance. Use `ProgrammaticSelectionGuard`; repaint highlight immediately and
   route the detail card through `DetailPanelDebouncer`. Cancel workers/debouncers on
   teardown and ignore stale load generations.
4. Render effective on/off and source separately from saved on/off. Include
   kind/default, full registry description, state path, removal bead/status, remove-by
   date/release, due chip, and resolver/state diagnostics. Empty, no-match, loading,
   corrupt-state, and no-registered-flags states must be intentional rather than blank.
5. Wire keyboard and click behavior described above, forwarding Config's armed numeric
   prefix before child bindings. Preserve Config hub Tab/Shift+Tab ownership rules.

### Toggle and restart

1. On Enter/Space, open a cancel-first confirmation with key, current-to-target state,
   saved path, description, shadowing warning, and the ACE+AXE restart consequence.
   Special-case `admin_center_flags` disable copy with the CLI recovery command.
2. Run `set_saved_feature_flag()` in a thread worker, disable duplicate submission while
   it is active, and keep failure in the pane with an error toast/log. There is no
   optimistic state repaint because a successful path intentionally exits the process.
3. On success, dismiss the confirm surface and call the existing proc-aware restart
   helper with a generalized restart-reason string. Preserve its 60-second wait,
   background-proc summary, controlled exit, prompt stash, state flush, and
   `_restart_tui(restart_axe=True)` behavior. The user sees both the up-front warning
   and the queued/immediate restart toast.
4. A shadowed saved value still restarts and then renders according to the higher
   source; its confirmation and CLI output explain why. Never mutate `_cli_values` to
   bypass an explicit root option.

### TUI tests and visuals

- Both states of `admin_center_flags`: exact child order/numbering, lazy factory count,
  resume fallback, prefix selection, and no import-time resolution.
- Loading/success/error/empty/no-match rendering; filter focus and Escape behavior;
  keyboard, mouse, selection restoration, guarded highlights, and detail debounce.
- Beta/sunset, on/off, saved/config/env/CLI sources, shadowing, missing bead,
  approaching/due removal, and corrupt-state cards.
- Confirmation copy, cancellation, one mutation per confirmation, duplicate suppression,
  mutation failure without restart, success with `restart_axe=True`, active-proc queue,
  restart-wait expiry, and the self-disabling recovery message.
- PNG snapshots at the normal 120x40 Admin Center size plus a narrow 70x32 layout, in
  light/dark theme coverage where neighboring Config snapshots require it. Inspect
  actual/expected/diff/source artifacts before accepting any golden.
- A focused responsiveness check proving navigation performs no state reads and keeps
  key-to-paint p95 under the existing 16 ms target.

Run focused widget/app tests, the affected visual snapshot lane, and `just check`.

## Phase `polish`: Integrated documentation, visual coverage, and release verification

1. Update `docs/configuration.md` with the seven-child Config strip, state-file purpose,
   complete precedence, saved-vs-effective explanation, corruption behavior, and restart
   contract. Keep the existing portable config examples and clarify that the pane/CLI do
   not edit them.
2. Update `docs/ace.md` with the pane layout, filter/navigation/toggle keys,
   confirmation, active-proc wait, shadowing, and self-disable recovery.
3. Update `docs/cli.md` and CLI reference tables for `flag enable`/`disable`, JSON, exit
   behavior, AXE restart/skip/partial success, and the separately-running ACE notice.
4. Re-run every focused core/runtime/CLI/TUI test against the combined tree. Exercise
   public command flows in temporary `SASE_HOME` directories and an app-level Textual
   flow for both enable and disable, observing the saved file, effective provenance,
   restart request, and post-restart catalog state rather than only calling internals.
5. Run `just install` first. Run `just check-full` only through `/sase_monitor` with
   `TESTING`/`TESTED` statuses and a next action that inspects failures. Run the
   complete visual suite through the monitor as well if the focused lane changed shared
   Admin Center chrome. Inspect and accept only intentional PNG diffs.
6. Confirm `tools/check_feature_flags` passes against the live rollout bead, every new
   flag has both-states tests, committed-plan/core-binding gates are green, config files
   remain byte-identical in end-to-end mutations, no real user state escaped pytest's
   sandbox, and no unrelated dirty-worktree changes were included.

## Final acceptance checklist

- Fresh installs show **Config > 01 Flags** by default through the sunset rollout flag.
- Every registered flag appears once with accurate effective, default, saved, source,
  description, bead, and due information.
- A confirmed TUI toggle writes only the machine-state store, warns before the action,
  waits safely for tracked procs, and performs one controlled ACE+AXE restart.
- `sase flag enable <flag>` and `disable <flag>` use the identical mutation facade,
  persist across fresh processes/reboots, restart running AXE, and produce trustworthy
  rich and JSON results.
- User, overlay, project-local, plugin, and chezmoi configuration files are untouched.
- Environment and one-shot root CLI overrides retain their documented precedence and are
  visibly explained when they shadow a saved preference.
- Corruption, concurrent writers, restart failure, self-disabling the pane, both flag
  kinds, both rollout states, narrow terminals, and visual/theme variants are covered.
