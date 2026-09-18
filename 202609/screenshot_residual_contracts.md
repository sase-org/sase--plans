---
tier: epic
title: Finish screenshot failure paths and nested memory rendering
goal:
  Close the reproduced launch, retained-target, settling, and nested-memory gaps left in
  sase-123.7 without redoing its completed renderer and transport work.
parent_bead: sase-123.7
phases:
  - id: launch-failure-ownership
    title: Guard launch ownership and preserve timeout diagnostics
    depends_on: []
    description:
      "launch-failure-ownership: clean up claims and owned windows across launch
      exceptions, normalize CLI errors, and retain the last pane text on polling
      timeouts."
    size: medium
  - id: retained-target-contract
    title: Preserve remote window identity and usable iteration guidance
    depends_on:
      - launch-failure-ownership
    description:
      "retained-target-contract: carry the printed unique tmux target through SSH
      metadata and hints, prove both shell boundaries, and correct the authorized
      screenshot-memory target key."
    size: medium
  - id: nested-memory-listings
    title: Preserve unread descendants and suppress nested duplicate listings
    depends_on: []
    description:
      "nested-memory-listings: recursively normalize inline-note listings across
      Markdown, Rich, and JSON while preserving discoverable unread descendants and
      existing batch deduplication."
    size: medium
  - id: finite-visual-settling
    title: Bound finite visual settling and verify the repaired workflow
    depends_on:
      - launch-failure-ownership
      - retained-target-contract
      - nested-memory-listings
    description:
      "finite-visual-settling: distinguish pending finite visual work from recurring
      background work, bound awaited refreshes, and exercise the repaired local, remote,
      and memory contracts together."
    size: medium
proposed_by: bbugyi200.athena.sase-123.7.land
create_time: 2026-09-18 00:03:27
status: wip
---

- **PROMPT:**
  [prompts/202609/screenshot_residual_contracts.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/screenshot_residual_contracts.md)
- **PARENT:**
  [202609/complete_tui_screenshots.md](https://github.com/sase-org/sase--plans/blob/main/202609/complete_tui_screenshots.md)

# Finish screenshot failure paths and nested memory rendering

This plan contains only remaining work found while landing `sase-123.7` at `80336097ad`.
Its accepted plan is `plan:202609/complete_tui_screenshots.md`; the original ancestor
contract is `plan:202609/tui_agent_screenshots.md`. All five phases of the immediate
parent are closed, but the concrete failures below prevent its successful landing.
Detailed evidence and reproduction recipes are in
`file:explicit:e8b9c4338d74ab54ee9e348b`; read it with `sase artifact read`.

The `parent_bead: sase-123.7` relationship is the landing continuation. Parent close,
post-close Symvision, and parent-plan status changes are land-agent duties, not phases
of this plan. Do not force a successful nested landing.

## Established behavior to preserve

- There is now one canonical packaged rasterizer. All 273 focused renderer, glyph, and
  fingerprint tests passed. Do not fork a renderer, change fonts, or rebaseline
  unrelated PNGs.
- Local launch returns unique `@window_id` targets and records request directories in
  tmux metadata. Normal keep/recapture and resize/metadata failure tests pass.
- Remote probe/capture/fetch/cleanup correctly serialize a remote command at the SSH
  boundary, and the shell-executing cleanup regressions pass. Keep the shared sudo
  target resolver, local rasterization, remote version reporting, and bounded cleanup.
- Inline note bodies deduplicate across requested roots, and top-level suppression
  works. Preserve requested order, depth caps, cycle safety, always-reference core/web
  descriptors, audit included targets, and byte counts.
- Newer Agents capacity/attention/loading, stable-search/fleet projection, and
  grouping/detail pickers must remain intact. Do not introduce broad refreshes or
  synchronous I/O on the Textual event loop or serial pump.

The landing audit fetched origin/master and found it equal to HEAD. Non-epic commits
since the first child implementation were `df0090f040`, `6e06a3e24c`, and `e91fa138b0`;
`76df54778f` also landed after epic creation. They change gate/core failure semantics,
supervision, and recovery notifications. No competing screenshot implementation or new
renderer consumer needs migration. Recheck drift before editing.

## Phase 1: Launch ownership and diagnostics {#launch-failure-ownership}

Files: `src/sase/main/ace_tmux.py`, `src/sase/screenshot/local.py`, their existing
tests, and handler error mapping if needed.

`_claim_window()` reserves the claim and calls both `_window_name_in_use()` and
`new-window` before entering its exception guard. A runner that first creates the window
and then raises `subprocess.TimeoutExpired` leaves one owned window and one claim, calls
no cleanup, and exposes raw `TimeoutExpired` through the CLI. The audit used the
existing `_FakeRunner` subclass, so this is a demonstrated exception path, not a
hypothetical tmux duplicate-name theory.

Extend the ownership guard to the whole reserved-claim lifetime. On exceptions before
creation, release the reservation. On ambiguous post-creation failure, recover/clean
only the uniquely named owned window using the existing temporary-name/ID mechanism;
never kill a supplied or unrelated window. Keep bounded best-effort cleanup and the
documented keep behavior. Do not silently consume launch errors. Map launch timeouts and
relevant OS failures into actionable screenshot CLI errors instead of tracebacks.
Preserve `sase tui --tmux` behavior and the one capture budget through all launch calls.

Also retain last-known pane text when the next capture subprocess times out during
startup or `--wait-for`. Both loops currently lose that text because `_run_tmux` raises
before their timeout-diagnostic branch. Cover errors after a nonblank frame, without
starting unbounded diagnostic calls or hiding the original failure.

Acceptance: inject failures before creation, after server-side creation but before
successful return, and during metadata/resize. Verify no leaked reservation/window, safe
treatment of existing targets, bounded subprocess budgets, and CLI error text. Use an
isolated real tmux socket to corroborate the post-create failure case, then remove only
test-owned resources. Retain existing successful two-window tests.

## Phase 2: Retained targets through SSH {#retained-target-contract}

Files: `src/sase/screenshot/remote.py`, `src/sase/main/screenshot_handler.py`,
`tests/main/test_screenshot_command.py`, and the authorized screenshot reference note.

Remote metadata includes `sase_tmux_target=@42`, but `_parse_remote_metadata()` drops it
and `_RemoteScreenshotResult.tmux_target` reconstructs a display-name target. Carry the
actual unique target through the result, printed contract, recapture flow, and send-keys
hint. Prove behavior after rename and with duplicate display names; do not weaken local
identity guarantees on the remote leg. Handle an older remote contract deliberately with
an actionable diagnostic or an explicitly tested compatible path.

The send-keys hint applies `shlex.join` only to the outer SSH argv. SSH then joins its
remote argv again, losing target/key quoting. Build a complete quoted remote command and
then quote the outer command; keep placeholder replacement clear. Extend the existing
shell-boundary harness to actually run the printed hint after substituting a key, with
target/key spaces, quotes, pipes, and harmless metacharacters. Assert the intended argv
reaches tmux. Keep the capture/fetch/remove regressions and shared sudo
target-resolution tests passing.

Use `sase_memory_write` before editing `sase/memory/tui_screenshot.md`. This is a
correction within the original plan's explicit screenshot-memory authorization and the
immediate parent's phase-5 scope: the keep/iterate paragraph currently tells agents to
copy `sase_tmux_window` while calling it the unique identity. Name `sase_tmux_target`
instead and keep the examples consistent with the real local and remote output. Keep
this concise; preserve `tui_perf.md` and note hierarchy. Run `sase memory init` and
verify no new memory warnings.

## Phase 3: Nested memory listing parity {#nested-memory-listings}

Files: `src/sase/memory/selector_postprocess.py`, `render.py`, `selector_render.py`, and
the existing selector/read/render tests. This is the current Python read/show
presentation pipeline; do not introduce shared domain behavior in Python that belongs in
Rust core. Use `sase repo open sase-core` if core changes prove necessary.

Two minimal fixtures reproduce the gaps:

1. `root.md` embeds `child.md`; `grandchild.md` has parent `child.md` and is not
   embedded. The single-root Markdown, Rich, and JSON outputs hide grandchild entirely.
   The unread grandchild must remain discoverable as a reference/child row even though
   its parent body is embedded. Do not inline its body without a requested link.
2. Child embeds grandchild, and an unrelated second root selects the batch JSON path.
   `notes[0].inline_notes[0]` both embeds grandchild and lists it under children.
   Top-level suppression does not recurse into nested note objects. The same omission
   leaves nested linked-reference records inconsistent with batch-rendered identities.

Normalize every rendered note in the inline tree, suppressing only bodies actually
present anywhere in the batch. Carry non-inlined descendants into the visible output
without restoring duplicate rows. Unify single-note and multi-note JSON semantics with
Markdown/Rich. Retain real references, core/descriptors, and depth-truncated targets.
Add regressions for nested children and references, diamonds/cycles, multiple roots, and
depth zero/one. Assert no target body duplicates, no already-rendered child rows, and
preserved unread-grandchild visibility. Recheck audit event targets and byte counts and
the actual `tui.md` read.

## Phase 4: Finite visual settling and acceptance {#finite-visual-settling}

Read the TUI and performance memory. Primary file:
`src/sase/ace/tui/actions/screenshot_export.py`; inspect current selected-detail
hydration in `actions/agents/_display_detail_render.py` and relevant task registries.

The audit ran a real Textual App with ScreenshotExportMixin and a Static label. A finite
worker sleeps 0.5 seconds then changes BEFORE to AFTER. Immediate export returns BEFORE
while that worker is still running; the later live frame shows AFTER. Ignoring every
worker permits a temporarily stable stale frame. Waiting for every live worker was also
wrong: automatic updates and broad recurring refreshes can run longer than a bounded
capture. Establish an explicit, bounded notion of finite visual work relevant to the
current screen/selection. Include selected-detail hydration/pending visual effects where
applicable, while retaining the regression that long nonvisual background work does not
block capture. Preserve selection revalidation across awaits and never trigger a broad
reload for screenshots.

The deadline check also occurs only after `await wait_for_refresh()`. A host whose
refresh await never finishes outlives a 50 ms settle deadline until an external 250 ms
timeout cancels it, leaving only `screen_1.pending`. Bound the awaited work itself,
produce a useful error marker on settle timeout, restore cursor state, and handle
teardown/cancellation coherently. Do not block Textual's serial pump. Keep live
timestamps and recurring clocks visible instead of applying golden-only time pins.

Acceptance must prove the eventual visual state, not only several identical frames or a
PNG signature. Add a delayed selection/detail update test that fails on today's code and
checks the exported final content, a stalled-refresh deadline test, and a
nonvisual-worker exclusion test. Include a real live-signal/PTY capture where feasible.
Run the repaired local pipeline with two retained windows, drive them independently,
recapture by the printed unique target, inspect the PNG, and verify the other window is
unchanged. Run both-shell-boundary remote regressions, original capture cleanup cases,
memory parity/read checks, and the shared sudo resolver tests. If an authorized remote
host is reachable, document a capture/version; do not deploy changes remotely.

## Verification and follow-up disposition

Each coding phase runs `just fix` and `just check` per project memory; use
`sase_monitor` for long checks. The land agent runs governed `just check-full` on the
combined tree before any successful close, through `sase_monitor` only. The final
acceptance phase runs the visual lane through a monitor when lengthy and attributes
failures before accepting goldens. Do not add parent closure to a phase.

All three previous proposals have a disposition:

- `sase-123.7.1` note 1 (three empty-attempt gate fixtures) was fixed in `df0090f040`;
  all three named cases pass in the 117-test audit. Existing active gate epic
  `sase-zr.7.1.1.5` already tracked it. No new task is justified.
- `sase-123.7.1` note 2 and `sase-123.7.5` note 1 report the same visual backlog.
  Forwarded phase counts and fresh one-node evidence to `sase-x5`, and the missed
  integrated golden to `sase-126`, whose phase 2 last authored it. Actual
  `agents_list_120x40` adds `(p)` to the view hint; expected omits it. Old and current
  rasterizers produce identical actual bytes. Audit run: 273 passed, one failed. Actual:
  `file:explicit:89b3629330021159e922d710`; expected:
  `file:explicit:a276711d01074f55e3581a95`. Do not infer a shared cause for every
  phase-reported 106/107 failure or blanket-update snapshots. Preserve the original
  parent's already-forwarded `sase-123.1` note-1 disposition on `sase-x5` too.

At the audit, both `sase bead epic-symbols sase-123.7` and the same command for
`sase-123` reported no entries. Recheck at landing. Once this child's implementation is
complete, its land agent resumes `sase-123.7` through `parent_bead`, rechecks all
descendants/notes/plans and new drift, then follows the user's normal ancestor landing
rules for directly parented plan `sase-123`.
