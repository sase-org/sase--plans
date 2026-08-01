---
tier: epic
title: Restore green GitHub Actions CI for sase
goal: 'Every job in the sase `CI` workflow (lint, test 3.12/3.13/3.14, visual-test,
  bead-backend, published-core-minimum-smoke) passes on master again, with each of
  the five independent root causes fixed at its source rather than muted.

  '
phases:
- id: lint-agents-sidecar
  title: Stop the agents sidecar from blocking `init repo --check`
  depends_on: []
  size: small
  description: '''Phase: lint-agents-sidecar'' section: make the read-only repo-init
    plan skip the machine-level agents sidecar when no owning SASE project key can
    be resolved, instead of raising a hard blocker.'
- id: keymap-log-level
  title: Stop `install_tui_file_logging` from leaking the `sase` logger level into
    other tests
  depends_on: []
  size: small
  description: '''Phase: keymap-log-level'' section: restore the `sase` logger level
    in the TUI log-setup fixture and scope the keymap registry test''s caplog to the
    emitting logger.'
- id: soak-attribution
  title: Attribute residual-freeze soak failures to the fixed paths, not the wall-clock
    window
  depends_on: []
  size: small
  description: '''Phase: soak-attribution'' section: make the lowered-threshold soak
    assertion fail only on watchdog events implicating the deliberately blocked paths,
    so unrelated bounded renders on loaded runners stop failing it.'
- id: visual-home-path
  title: Remove the host `$HOME` dependency from the axe editor visual snapshots
  depends_on: []
  size: small
  description: '''Phase: visual-home-path'' section: derive the axe writable-scope
    fixture path from the real home directory so the tilde abbreviation renders identically
    on every host, then regenerate the seven affected goldens.'
- id: visual-capture-races
  title: Gate the neighbor-badge and prompt-caret snapshots on rendered output
  depends_on: []
  size: medium
  description: '''Phase: visual-capture-races'' section: reproduce and fix the two
    CI-only PNG capture races by waiting on the painted frame instead of internal
    footer layout inputs and widget focus state.'
- id: core-sqlite-busy
  title: Fix spurious SQLITE_BUSY in the sase-core telemetry store
  depends_on: []
  size: medium
  description: '''Phase: core-sqlite-busy'' section: in the sase-core repo, make write
    transactions use BEGIN IMMEDIATE and make the WAL journal-mode pragma contention-tolerant
    so concurrent telemetry writers stop failing with `database is locked`.'
- id: core-release-window
  title: Align sase's `sase-core-rs` window with a published core release
  depends_on:
  - core-sqlite-busy
  size: medium
  description: '''Phase: core-release-window'' section: land a sase-core release that
    contains the 20 bindings sase requires, then point sase''s pyproject window at
    that published version and stop the speculative minor ratchet.'
create_time: 2026-07-25 07:27:05
status: wip
bead_id: sase-93
---

- **PROMPT:** [prompts/202607/restore_green_ci.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/restore_green_ci.md)
- **BEAD:** [sase-93](https://github.com/sase-org/sase--beads/blob/main/pages/sase-93/README.md)

# Plan: Restore green GitHub Actions CI for sase

## Context

`actstat --repo sase-org/sase` shows master red. Most runs on master report `cancelled` rather than `failure` because
`CI` uses `concurrency: cancel-in-progress: true` and push volume is high, so a superseding push kills the run. The two
most recent runs that settled on their own (`ecc07b6a`, run 30144483410; `cd6fc9fd`, run 30154992915) plus a per-job
sweep of the last ten runs give the full picture.

There are **five independent root causes**, not one:

| Job                            | Failing step                         | Root cause                                               | Frequency                   |
| ------------------------------ | ------------------------------------ | -------------------------------------------------------- | --------------------------- |
| `lint`                         | SASE validation                      | `init repo --check` blocker on the agents sidecar        | 10/10 runs checked          |
| `bead-backend`                 | Run Rust bead checks                 | spurious `SQLITE_BUSY` in sase-core telemetry store      | intermittent (~46% locally) |
| `published-core-minimum-smoke` | Install exact published core minimum | declared core window points at an unpublished version    | every run                   |
| `visual-test`                  | Run visual tests                     | 7 host-`$HOME`-dependent goldens + 2 capture races       | 9 snapshots                 |
| `test (3.14)`                  | Run tests                            | leaked `sase` logger level + over-broad soak attribution | intermittent                |

Phases `lint-agents-sidecar`, `keymap-log-level`, `soak-attribution`, `visual-home-path`, `visual-capture-races` and
`core-sqlite-busy` are mutually independent and may run in parallel. Only `core-release-window` has a dependency: it
needs `core-sqlite-busy` first, because the flaky Rust test is currently failing CI on sase-core's own open release PR
and therefore blocks the core release that `core-release-window` consumes.

Work in the `sase-core` repo MUST be opened through the `/sase_repo` skill (`sase repo open sase-core -r "<reason>"`)
and edited only under the path that command prints.

### Repository-wide requirements

- Run `just install` before any other `just` recipe; workspace virtualenvs go stale.
- Run `just check` before reporting completion for any phase that changes files in this repo.
- Phases touching TUI responsiveness or snapshots should read `sase/memory/tui_perf.md` through the `/sase_memory_read`
  skill first.

---

## Phase: lint-agents-sidecar

### Evidence

`lint` fails at step `SASE validation` (`just validate` → `.venv/bin/sase validate`):

```
  fail   init repo --check
Blockers:
  init repo: could not resolve the owning SASE project key for the agents sidecar from /home/runner/work/sase/sase
```

This has failed on every one of the last ten master runs, so it is fully deterministic and is the single highest-value
fix.

### Root cause

`sidecar_clone_root()` in `src/sase/sdd/_sidecar_init.py` (the `role == AGENTS_SIDECAR_ROLE` branch, around lines 63-82)
resolves the _machine-level hidden_ clone path for the agents sidecar by calling
`infer_project_name_from_cwd(str(workspace))`. That inference (see `src/sase/bead/project_name.py`,
`infer_project_name_from_cwd`) needs one of: a nearest `.sase/checkout.json` marker, a workspace-provider workspace name
backed by an existing `~/.sase/projects/<key>` spec, or a project whose registered workspace matches the cwd. A plain
`actions/checkout` tree at `/home/runner/work/sase/sase` has none of those, so the function returns `None` and
`sidecar_clone_root` raises `SddMaterializationError`.

The read-only plan path calls it unconditionally: `plan_sidecar_actions()` in `src/sase/main/_repo_init_sidecars.py`
(around line 268) invokes `sidecar_clone_root` for every configured sidecar spec, and `plan_repo_init()` in
`src/sase/main/repo_init_handler.py` (around lines 202-208) converts the raised `SddMaterializationError` into a plan
blocker. `run_init_check()` in `src/sase/main/init_onboarding.py` returns exit 1 when any plan has blockers.

The agents sidecar spec is always synthesised as a default entry (see `src/sase/_linked_repo_config.py`, the
`agents_identity` block around lines 256-285), so every non-workspace checkout hits this. The regression entered with
commit `44ccbe84c` ("feat(repo-init): initialize agents sidecars with explicit consent (sase-8k.5)").

### Reproduction

From a checkout, with no clone needed:

```bash
.venv/bin/python -c "
import os, tempfile
from sase.sdd._sidecar_init import sidecar_clone_root
d = tempfile.mkdtemp(prefix='plainclone-'); os.makedirs(os.path.join(d, '.git'))
print(sidecar_clone_root(d, 'plans'))   # resolves fine
print(sidecar_clone_root(d, 'agents'))  # raises SddMaterializationError
"
```

### Required change

Make the _read-only_ plan path degrade instead of blocking when the owning project key cannot be resolved. A plain
checkout has no machine-level project registry, so the agents sidecar is simply not plannable there — that is a
legitimate state, not an error.

Constraints on the fix:

- Do **not** make this a plan "change". `run_init_check()` returns exit 1 for pending changes as well as blockers, so
  emitting a "create the agents sidecar" action in CI would keep `lint` red. Skipping (optionally with a `warnings`
  entry, which does not affect the exit code) is the only shape that turns the job green.
- Keep the hard failure for the _mutating_ paths. `run_materialized_sidecars()` and `initialize_sidecars()` in
  `src/sase/main/_repo_init_sidecars.py` and `src/sase/sdd/_sidecar_init.py` must still raise when asked to actually
  create or initialize an agents sidecar without a resolvable project key — silently skipping real initialization would
  be a behaviour regression.
- Scope the tolerance to the unresolvable-project-key case only. Do not broaden it into a blanket
  `except SddMaterializationError` around all sidecar planning; the "configured sidecar is not materialized at ..."
  blocker must still surface.

Suggested shape: give `sidecar_clone_root` a non-raising companion (or an explicit `missing_ok`/`required=False`
variant) that returns `None` when the project key is unresolvable, and have `plan_sidecar_actions` skip specs whose root
resolves to `None`, recording a warning.

### Verification

- New unit coverage: `plan_sidecar_actions` (or `plan_repo_init`) over a non-workspace project root yields no blockers
  and no agents-sidecar action, while the mutating entry point still raises.
- `just check`.
- Confirm the `lint` job's `SASE validation` step passes on the follow-up PR.

---

## Phase: keymap-log-level

### Evidence

`test (3.14)`, step `Run tests`:

```
FAILED tests/test_keymaps_registry_loading.py::test_app_query_override_is_honored_while_retired_help_is_dropped
  - AssertionError: assert 'Ignoring retired app keymap action: show_help' in ''
```

The test passes in isolation, which is why this reads as a flake.

### Root cause

Deterministic cross-test pollution of global `logging` state:

1. `install_tui_file_logging()` in `src/sase/ace/tui/log_setup.py` ends with
   `if logger.level == logging.NOTSET or logger.level > level: logger.setLevel(level)` on the **`sase`** logger, where
   `level` defaults to `logging.WARNING`.
2. The autouse `_cleanup_sase_handler` fixture in `tests/ace/tui/test_tui_log_setup.py` removes the handler it added but
   never restores `logging.getLogger("sase").level`. The `sase` logger stays pinned at `WARNING` for the remainder of
   that xdist worker's session.
3. `load_keymap_registry()` logs the retired-action message with `log.debug(...)` on `sase.ace.tui.keymaps.registry`
   (`src/sase/ace/tui/keymaps/registry.py`, around line 166).
4. The failing test wraps its call in `caplog.at_level(logging.DEBUG)`, which adjusts the **root** logger and caplog's
   handler — not the `sase` logger. With the `sase` logger's effective level at `WARNING`, the DEBUG record is never
   emitted, so `caplog.text` is empty.

Because it depends on whether `tests/ace/tui/test_tui_log_setup.py` happens to land in the same xdist worker before
`tests/test_keymaps_registry_loading.py`, it fails intermittently. `src/sase/main/ace_handler.py` also calls
`install_tui_file_logging()`, so any test exercising the ace command entry point can leak the same level.

### Reproduction

Deterministic, in this repo:

```bash
.venv/bin/python -m pytest tests/ace/tui/test_tui_log_setup.py \
  tests/test_keymaps_registry_loading.py -p no:randomly -q
# -> 1 failed, 34 passed
```

### Required change

Fix both the leak and the fragile assertion:

1. In `tests/ace/tui/test_tui_log_setup.py`, have the autouse fixture capture and restore
   `logging.getLogger("sase").level` alongside the handler list. This is the actual pollution source and fixing it
   protects every other test that logs below `WARNING` under the `sase` logger.
2. In `tests/test_keymaps_registry_loading.py`, scope the capture to the emitting logger —
   `caplog.at_level(logging.DEBUG, logger="sase.ace.tui.keymaps.registry")` — so the assertion no longer depends on the
   ambient level of an ancestor logger.

Do not "fix" this by promoting the `log.debug` call to `info`/`warning`; the message is intentionally debug-level
diagnostics for an ignored retired key.

### Verification

- The reproduction command above passes.
- `just check`.
- Run the two files in both orders and with `-p randomly` to confirm order independence.

---

## Phase: soak-attribution

### Evidence

`test (3.14)`, step `Run tests`:

```
FAILED tests/ace/tui/test_residual_freeze_soak.py::test_lowered_threshold_soak_keeps_fixed_paths_responsive
  - AssertionError: watchdog events during controlled slow-work windows:
    [('prompt_history', {'event': 'tui_stall', 'stall_seconds': 1.026,
      'threshold_seconds': 1.0, ...})]
```

The recorded `main_thread_stack` is not the deliberately blocked path. It is the commits pane rendering a diff detail:

```
commits_pane.on_worker_state_changed -> commits_detail._on_diff_worker_changed
  -> _update_detail -> _build_detail -> commits_rendering.build_commit_detail
  -> lazy_syntax.lazy_renderable -> get_plain -> build_plain
  -> _truncate_plain_content -> _line_count
```

### Root cause

`_assert_no_fixed_path_freezes()` in `tests/ace/tui/test_residual_freeze_soak.py` attributes a watchdog event to a fixed
path using **only** the wall-clock window:

```python
for window in windows:
    if window.started_ts <= event_ts <= window.ended_ts:
        observed.append((window.label, record))
```

Any watchdog event that happens to land inside the window fails the test, regardless of what actually stalled the loop.
The test also runs the app with `startup_policy="real"`, so real artifact loading — including the commits-pane diff
worker and its synchronous UI-thread render — proceeds concurrently with the deliberately blocked worker bodies.

`_configure_lowered_watchdog()` lowers `SASE_TUI_STALL_THRESHOLD_SECONDS` to `1.0`, against a production default of
`5.0` (`DEFAULT_THRESHOLD_SECONDS` in `src/sase/ace/tui/util/_stall_watchdog_config.py`). The observed stall was
`1.026 s` — a 2.6% overshoot of an artificially hair-trigger threshold, on a runner sharing CPU with several xdist
workers. That is not a production-visible freeze, and it is not the fixed path the soak targets.

### Required change

Make the assertion prove what it claims: fail only on watchdog events that implicate the deliberately blocked paths.
Options, in order of preference:

1. Attribute by stack, not just by clock. `_FREEZE_EVENTS` records carry `main_thread_stack`; require that the recorded
   stack implicate the blocked callable for that window (the patched `_read_notifications_for_startup` /
   `load_prompt_record_page` frames) before counting it. Keep the window filter as an additional narrowing condition.
2. If stack attribution proves too brittle, keep the window filter but explicitly exclude events whose stack sits
   entirely in unrelated startup/artifact rendering, and assert on that exclusion list so the intent stays visible.

Whichever shape is chosen, keep the existing unit coverage (`test_watchdog_windows_ignore_unrelated_session_hitches`,
`test_watchdog_windows_reject_fixed_path_hitches`) meaningful and extend it with a case that feeds a record whose stack
is unrelated work inside the window and asserts it is _not_ counted.

Do not simply raise `SASE_TUI_STALL_THRESHOLD_SECONDS` back toward the production default; that would blunt the soak's
whole purpose. Do not delete or `xfail` the test.

### Out of scope, worth recording

The captured stack shows `commits_detail._on_diff_worker_changed` building the diff renderable synchronously on the UI
thread when the loader worker completes. The render is bounded (`PLAIN_RENDER_MAX_LINES` and byte caps in
`src/sase/ace/tui/util/lazy_syntax.py`), and no production-threshold stall has been observed, so this phase must not
restructure it. If the implementing agent judges it a real responsiveness risk, file a separate bead rather than
widening this phase.

### Verification

- The soak test passes locally, and passes when re-run under CPU contention (for example, alongside a busy parallel
  `just test` run) to confirm the marginal-overshoot sensitivity is gone.
- `just check`.

---

## Phase: visual-home-path

### Evidence

`visual-test` reports nine mismatches. Seven of them are the axe editor family, all with the same diff geometry:

```
axe_chop_editor_advanced_120x40:       947/1520532 changed (800 material)
axe_chop_editor_basics_120x40:         947/1520532 changed (800 material)
axe_editor_diff_preview_120x40:        947/1520532 changed (800 material)
axe_editor_multiline_yaml_cell_120x40: 947/1520532 changed (798 material)
axe_editor_single_line_cell_120x40:    947/1520532 changed (800 material)
axe_editor_validation_failure_120x40:  947/1520532 changed (800 material)
axe_generated_instance_warning_120x40: 913/1520532 changed (776 material)
```

Cropping the diff bounding box out of the CI artifact makes it unambiguous:

- golden renders `~/.config/sase/sase.yml`
- CI renders `/home/bryan/.config/sase/sase.yml`

### Root cause

Two facts combine into a host dependency:

1. `tests/ace/tui/visual/test_ace_png_snapshots_axe_editor.py` (around line 201) hard-codes the writable scope path as
   the literal string `"/home/bryan/.config/sase/sase.yml"`.
2. `src/sase/ace/tui/modals/axe_entry_editor_rendering.py` captures `_HOME = os.path.expanduser("~")` at **import time**
   and abbreviates paths under `$HOME` to `~`.

On the host that generated the goldens `$HOME == /home/bryan`, so the literal abbreviates to `~/.config/sase/sase.yml`.
On a GitHub runner `$HOME == /home/runner`, no abbreviation applies, and the full path renders. Per
`docs/development.md`, PNG comparison is byte-exact locally _and_ in every visual-bearing CI lane, so this is a hard
failure rather than absorbed drift.

### Reproduction

Proven — this reproduces CI's exact pixel counts on a Linux host:

```bash
mkdir -p /tmp/fakehome
HOME=/tmp/fakehome .venv/bin/python -m pytest \
  tests/ace/tui/visual/test_ace_png_snapshots_axe_editor.py::test_axe_chop_editor_basics_png_snapshot \
  -m visual -q -p no:randomly
# -> Changed pixels: 947/1520532 (0.062281%); materially changed pixels: 800/1520532
```

The same suite passes with the ambient `$HOME`, which is why local runs are green.

### Required change

Make the fixture host-independent. Because `_HOME` is bound at import time, monkeypatching `HOME` inside the test is not
reliable — the fix belongs in the fixture data.

Preferred: build the fixture path from the real home directory, e.g. `str(Path.home() / ".config/sase/sase.yml")`, so
the renderer always abbreviates it and every host renders the identical `~/.config/sase/sase.yml`. Sweep the whole
visual fixture tree for other hard-coded `/home/<user>` literals while here.

Then regenerate the seven affected goldens with `--sase-update-visual-snapshots` (Linux only, per the regeneration gate)
and confirm the regenerated PNGs differ from the current ones only in that path region.

Add a guard so this class of bug cannot silently return: a test (or a check in the visual conftest) asserting no visual
fixture embeds an absolute path under a host home directory.

### Verification

- `just test-visual` passes with the ambient `$HOME`.
- `HOME=/tmp/fakehome just test-visual` also passes — this is the decisive check, and it must be run, because it is what
  distinguishes a real fix from re-baking the same host dependency.
- `just check`.

---

## Phase: visual-capture-races

### Evidence

The two remaining `visual-test` mismatches do **not** reproduce locally, and do **not** reproduce under a foreign
`$HOME` (both pass in that configuration), so they are CI-timing sensitive rather than environment-path sensitive.
Rebuilding `sase_core_rs` from current `sase-core` master does not reproduce them either, so they are not core-version
drift.

1. `agent_neighbor_modal_descendants_dismissed_60x30` — 83 changed pixels (76 material). The golden footer paints
   `[neighbors: 2 (~)]`; the CI frame paints `[neighbors: 1 (~)]`.
2. `prompt_xprompt_highlight_solo_light_120x40` — 288 changed pixels, a single 12x24 cell. The golden paints the prompt
   `TextArea` caret on line 5; the CI frame has no caret there.

### Root cause

Both are captures gated on **internal state** rather than on the **painted frame**.

For the neighbor badge, `test_agent_neighbor_modal_dismissed_descendant_png_snapshot` in
`tests/ace/tui/visual/test_ace_png_snapshots_agents_neighbors.py` injects private state (`_dismissed_agent_objects`,
`_dismissed_agents`, `_dismiss_revive_epoch`), calls `_refresh_agent_footer_bindings_only()`, and then waits on
`footer._last_layout_inputs` containing the label `"neighbors (2)"`. Because that wait inspects the footer's computed
layout inputs, it can be satisfied while the _painted_ footer still shows the pre-injection value. The wait therefore
cannot time out, and the test fails as a pixel mismatch instead. (Cache invalidation itself is correct: the neighbor
index signature in `src/sase/ace/tui/actions/agents/_neighbors.py` includes `_dismiss_revive_epoch` and
`_dismissed_agents`.)

For the caret, `_draw_cursor()` in Textual's `TextArea` reduces to `has_focus` once `cursor_blink` is `False`, which
`_disable_cursor_blink()` in `tests/ace/tui/visual/_ace_png_snapshot_waits.py` sets. `mount_prompt_bar()` waits for
`bar.active_text_area().has_focus` and then `wait_for_visual_idle()`. The absent caret in the CI frame therefore means
focus was not on the prompt text area at capture time — most plausibly a background refresh moving focus after the wait
resolved. Note only the light-theme variant failed; the sibling stack variant using the same helper passed, which is
consistent with a race rather than a systematic error.

### Required change

Start by reproducing. This phase is sized `medium` primarily because the reproduction is the hard part; do not change
fixtures until a failure is reproduced locally.

Suggested reproduction levers, in order:

- run the visual suite under heavy CPU contention (more xdist workers than cores, plus a CPU hog) to widen the
  scheduling gaps a shared GitHub runner exhibits;
- run the two tests in a loop for many iterations;
- temporarily instrument the two capture sites to log the painted footer text and `active_text_area().has_focus` /
  `_draw_cursor()` immediately before `assert_page_png`, and confirm the failing frame is the stale/unfocused one.

Then fix by gating on rendered output:

- Neighbor badge: replace (or supplement) the `footer._last_layout_inputs` wait with a wait on the rendered frame — the
  suite already has `wait_for_svg_contains`, so wait for `"neighbors: 2"` in the SVG before capture. Keep the
  internal-state wait only as a fast precondition.
- Prompt caret: re-assert focus (and `_draw_cursor()`) on the active text area immediately before `assert_page_png`, and
  if focus can legitimately be stolen by a background refresh, quiesce that refresh in the fixture rather than papering
  over it with a sleep. If the caret genuinely cannot be made deterministic through focus, pin the caret-visibility
  state explicitly in `_ace_png_snapshot_waits.py` for snapshot captures.

Do not widen the PNG tolerance to absorb these. `docs/development.md` is explicit that comparison is byte-exact in every
visual-bearing CI lane and that the tolerance environment variables are local escape hatches only; note also that
`sase/memory/build_and_run.md` currently claims CI "allows a small ratio-only renderer drift tolerance", which the
workflow and the code contradict — `.github/workflows/ci.yml` sets no `SASE_VISUAL_PNG_*` variables, and
`resolve_png_diff_tolerance()` falls through to the exact-equality default. Flag that stale memory line to the user
rather than editing it; SASE memory files require explicit user permission.

### Verification

- Each fixed test survives a high-iteration loop under CPU contention.
- `just test-visual` passes, including under `HOME=/tmp/fakehome` once `visual-home-path` has landed.
- `just check`.

---

## Phase: core-sqlite-busy

### Evidence

`bead-backend`, step `Run Rust bead checks`:

```
---- telemetry::store::tests::concurrent_writers_preserve_every_delta stdout ----
thread '<unnamed>' panicked at crates/sase_core/src/telemetry/store.rs:2396:18:
called `Result::unwrap()` on an `Err` value: "database is locked"
test result: FAILED. 888 passed; 1 failed
```

Reproduced locally: **12 of 26** runs of that single test fail. The same test also failed CI on sase-core commit
`77c88ac` and is failing CI on sase-core's open release PR #29 (`chore: release v0.9.1`), which is what currently blocks
the core release consumed by `core-release-window`.

### Root cause

Two distinct SQLite contention defects in `crates/sase_core/src/telemetry/store.rs` (sase-core repo). Both were
reproduced independently in a standalone Python `sqlite3` harness driving four threads through the same sequence, which
produced both error shapes:

1. **Deferred write transactions.** `record_batch()` (and the other mutators) call `conn.transaction()`, which issues a
   plain `BEGIN` (deferred). The transaction starts as a reader and upgrades on the first write. When two connections
   both hold read transactions and both try to upgrade, SQLite returns `SQLITE_BUSY` **immediately without invoking the
   busy handler**, because retrying would deadlock. The 5-second `busy_timeout` the test passes is therefore never
   consulted.
2. **`PRAGMA journal_mode = WAL` under concurrent open.** `open_store()` executes `PRAGMA journal_mode = WAL;` inside
   the same `execute_batch` as the schema DDL. Changing the journal mode needs an exclusive lock, and concurrent opens
   against a fresh database return `database is locked` — again not covered by `busy_timeout`.

### Required change

Open the repo with the `/sase_repo` skill and work only under the printed path:

```bash
sase repo open sase-core -r "Fix spurious SQLITE_BUSY in the telemetry store"
```

Then, in `crates/sase_core/src/telemetry/store.rs`:

1. Use `TransactionBehavior::Immediate` (rusqlite `Connection::transaction_with_behavior`) for every write transaction,
   so the write lock is taken at `BEGIN` and the configured `busy_timeout` actually governs contention. Audit all
   `with_store` callers that write — `record_batch`, the prune/rollup path, `cleanup_matching_labels`, and the retention
   fold — not just the one the test exercises. Leave read-only paths deferred.
2. Make the journal-mode switch contention-tolerant: run `PRAGMA journal_mode = WAL` separately from the schema DDL and
   treat a busy/locked result as retryable (bounded retry) or benign when the mode is already `wal`. Do not swallow
   genuine errors, and do not drop WAL mode.

Respect the existing bounded-lock-wait convention in the crate — `bounded_busy_timeout()` and the `lock_wait_is_bounded`
tests show the codebase deliberately caps waits; any retry loop added here must stay bounded.

Then harden the test so a regression is caught rather than flaked: keep `concurrent_writers_preserve_every_delta`, and
run it repeatedly (or add a companion that loops the contended open plus write) so a reintroduced deferred transaction
fails reliably.

### Verification

- `cargo test -p sase_core --lib telemetry::store::tests::concurrent_writers_preserve_every_delta -- --exact` passes 50
  consecutive times (it currently fails roughly half the time).
- The full sase-core check suite passes (`cargo fmt`, `clippy`, `cargo test --workspace`).
- sase's `bead-backend` job passes against the updated core.
- Because this lands in sase-core, follow that repo's own commit and PR workflow; do not commit it as part of a sase
  change.

---

## Phase: core-release-window

### Evidence

`published-core-minimum-smoke`, step `Install exact published core minimum`:

```
uv pip install sase-core-rs==0.12.0
  × No solution found when resolving dependencies:
  ╰─▶ Because there is no version of sase-core-rs==0.12.0 ...
```

The `lint` job's setup step reports the same divergence from the other direction:

```
[validate_sase_core_rs_version] sase-core checkout is behind: source version 0.9.0 ...
  does not satisfy `sase`'s `sase-core-rs>=0.12.0,<0.13.0`
```

### Root cause

`pyproject.toml` declares `sase-core-rs>=0.12.0,<0.13.0`, but the latest version published to PyPI is **0.9.0**.
Versions 0.10.0, 0.11.0 and 0.12.0 were never published, so the declared minimum can never be installed.

The divergence is structural, not a one-off typo:

- sase ratchets a **minor** on each core-touching change. `git log -L` on the dependency line shows `>=0.9.0` (commit
  `e2a294dc0`) → `>=0.10.0` (`65bcc9391`) → `>=0.11.0` (`0ee641f6c`) → `>=0.12.0` (`8e7851ede`), each bump landing with
  no corresponding core release.
- sase-core's `release-plz.toml` sets `features_always_increment_minor = false`, so on a `0.x` crate `feat` commits
  produce **patch** bumps. Its open release PR #29 is `chore: release v0.9.1`. Core's cadence can never reach 0.12.0 by
  this route.

Lowering sase's window to the published 0.9.0 does **not** work either. Verified against a real `sase-core-rs==0.9.0`
install:

```
sase_core_rs 0.9.0 is missing 20 of 179 required binding(s):
  agent_link_target, agent_local_hood, agent_name_ancestors, agent_name_in_hood,
  bead_claim_for_agent_wait, bead_needs_size_check_relax_migration,
  bead_release_agent_claim, bead_remove_many, bead_size_check_relax_migration_sql,
  classify_agent_ownership, globalize_agent_name, localize_agent_name,
  normalize_agent_archive_name, parse_agent_family_name,
  rewrite_agent_relationship_batch, strip_global_agent_name, validate_agent_name,
  validate_agent_owner, validate_agent_relationship_batch, validate_agent_username
```

Those bindings exist in sase-core master (`crates/sase_core_py/src/lib.rs`) but in no published wheel. So this job
cannot be made green from the sase repo alone: **core must publish a version containing those bindings, and sase's
window must then name that version.**

### Required change

1. Confirm `core-sqlite-busy` has landed on sase-core master and its CI is green, including on the release PR branch.
   That PR's CI is currently red solely because of the flaky telemetry test.
2. Land the sase-core release so a wheel containing the 20 bindings reaches PyPI. Decide, and state in the PR
   description, which version policy is intended:
   - **Recommended:** let release-plz publish the version it computes (currently `0.9.1`) and set sase's window to the
     version actually published, e.g. `sase-core-rs>=0.9.1,<0.10.0`. This stops the speculative ratchet and makes
     `published-core-minimum-smoke` a real check again.
   - **Alternative:** if the minor-per-core-change convention is deliberate, set
     `features_always_increment_minor = true` in sase-core's `release-plz.toml` so core's numbering matches sase's
     expectation, and reconcile the two skipped minors explicitly. This is a policy call with user-visible versioning
     consequences; surface the recommendation and get confirmation rather than deciding silently.
3. Update `pyproject.toml` to the published version window, and update the fixtures that encode it
   (`tests/test_sase_core_rs_telemetry_smoke_tool.py` has been edited alongside every previous bump).
4. Verify the published wheel actually satisfies the binding contract before declaring done.

### Verification

```bash
core_minimum="$(python3 tools/smoke_sase_core_rs_telemetry --print-minimum pyproject.toml)"
uv venv --python 3.12 /tmp/published-core-smoke
uv pip install --python /tmp/published-core-smoke/bin/python "sase-core-rs==${core_minimum}"
/tmp/published-core-smoke/bin/python tools/smoke_sase_core_rs_telemetry
/tmp/published-core-smoke/bin/python tools/check_sase_core_rs_bindings
```

All four commands must succeed, with `check_sase_core_rs_bindings` reporting zero missing bindings. Then
`just install && just check`, and confirm both `published-core-minimum-smoke` and the `lint` job's
`validate_sase_core_rs_version` step are clean.

### Guard against recurrence

The recurring failure mode is a sase-side window bump with no matching core release. Add a check that fails fast and
locally — for example, extend `tools/validate_sase_core_rs_version` (or add a `just` target wired into `just validate`)
to assert the declared minimum exists on PyPI, so the next speculative bump is caught before it reaches CI.
