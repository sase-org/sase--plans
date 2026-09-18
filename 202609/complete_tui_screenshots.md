---
tier: epic
title: Complete the screenshot and inline-memory contracts
goal: Live TUI captures reliably address and clean up their own window, remote captures
  preserve arguments across SSH, and inline memory reads render each target once with
  complete agent workflow guidance.
parent_bead: sase-123
phases:
- id: canonical-renderer
  title: Restore one renderer for screenshots and visual snapshots
  size: small
  depends_on: []
  description: 'canonical-renderer: remove the test-side rasterizer copy reintroduced
    by concurrent commits and repoint visual helpers and fingerprints to the packaged
    runtime renderer without changing pixels.'
- id: capture-lifecycle
  title: Make local capture ownership, deadlines, and settling reliable
  size: medium
  depends_on: []
  description: 'capture-lifecycle: correct tmux window ownership and failure cleanup,
    propagate the capture deadline through launch, and wait for a settled live frame
    without blocking Textual.'
- id: ssh-contract
  title: Preserve the remote shell contract and cleanup
  size: medium
  depends_on:
  - capture-lifecycle
  description: 'ssh-contract: quote the complete remote command correctly, prove literal
    argument forwarding and actual file cleanup, and preserve bounded failures and
    the shared SSH target resolver.'
- id: memory-deduplication
  title: Deduplicate inline memory across the complete read
  size: medium
  depends_on: []
  description: 'memory-deduplication: render shared inline notes once across roots,
    suppress already-rendered references and children consistently, and keep depth,
    cycle, and audit semantics correct.'
- id: workflow-acceptance
  title: Complete screenshot guidance and verify the integrated workflow
  size: medium
  depends_on:
  - canonical-renderer
  - capture-lifecycle
  - ssh-contract
  - memory-deduplication
  description: 'workflow-acceptance: complete the originally approved screenshot memory
    content and verify real local capture, retained-window iteration, remote shell
    behavior, and memory read integration against the newer TUI changes.'
proposed_by: bbugyi200.athena.sase-123.land
create_time: 2026-09-17 21:13:46
status: wip
bead_id: sase-123.7
---

- **PROMPT:** [prompts/202609/complete_tui_screenshots.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/complete_tui_screenshots.md)
- **PARENT:** [202609/tui_agent_screenshots.md](https://github.com/sase-org/sase--plans/blob/main/202609/tui_agent_screenshots.md)
- **BEAD:** [sase-123.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-123/sase-123.7.md)

# Complete the screenshot and inline-memory contracts

This is remaining work from landing `sase-123`, whose accepted plan is
`plan:202609/tui_agent_screenshots.md`. All six original phases are closed, but the
landing audit at `c199dcb6ca` reproduced the failures below. The original epic stays
open. Its `parent_bead` relationship is the continuation after this child completes. Do
not add an original-epic close, post-close Symvision run, or original-plan status update
as a phase of this plan.

Detailed audit and reproduction outcomes: `file:explicit:66624d11a0d4d2c8d4fbead9`
(consume with `sase artifact read`).

## Established work and integration constraints

The runtime renderer and packaged fonts exist, but concurrent changes restored a
separate test rasterizer; unify them while preserving their pixel behavior. The new
screenshot CLI, live signal export, SSH target resolver, flat-note embeds, TUI memory
hierarchy, and generated instructions exist. Repair the specific incomplete contracts
instead of rebuilding those features.

The two original epic issue notes are resolved by concurrent commits:

- `73e4318edf` added the optional `resvg_py` mypy overrides. A focused type check with
  site packages disabled and imported modules skipped passes on the renderer. Keep
  runtime imports lazy and the actionable visual-extra error.
- `5e4c866eb5` removed the renderer's stale epic exemption when it relocated the
  renderer back into tests. Phase 3 later restored the runtime module, which now has a
  real screenshot caller. `sase bead epic-symbols sase-123` reports no entries.

Preserve the subsequently landed refresh/capacity/attention changes from `sase-124` and
`sase-124.8`, stable-search and fleet projection work from `sase-127`, the
grouping/layout/detail pickers, and `sase-126`'s current core floor and visual fixture
repairs. Screenshot settling must not request broad reloads or wait forever on recurring
background refreshes. Recheck newer commits when implementing.

The sole proposed follow-up, `sase-123.1` note 1 about ambient golden drift, was
forwarded as inherited evidence to existing task `sase-x5` through `sase_new_task`.
`sase-126.2` subsequently reports a passing full visual lane in `fd626ec222`. No new
task or current failure claim was created. Attribute any fresh failures individually; do
not undo or blanket-regenerate that reviewed corpus.

## Phase 1: One canonical renderer {#canonical-renderer}

Phase 1 originally promoted the implementation correctly in `7aef3364e2`. Later,
`5e4c866eb5` deleted `src/sase/ace/tui/visual_render.py` and pointed the visual suite at
a test-local implementation. `37d1b2592e` inlined that copy into
`tests/ace/tui/visual/png_diff.py`. Epic phase 3 (`729fe7cae1`) restored the runtime
module without repointing the visual suite, leaving two implementations today.

Delete only the duplicated renderer/font-discovery implementation from test helpers.
Have the visual fixture, glyph audits, font fingerprint code, and relevant tests use
`src/sase/ace/tui/visual_render.py` and its packaged fonts. Retain the test-side PNG
comparison/artifact facade. Preserve existing font bytes, renderer arguments, lazy
dependency behavior, and approved goldens. Confirm the built wheel contains every font.
Verify both callers reach the same renderer and run the renderer-focused tests plus the
visual lane, using a monitor for lengthy verification. Do not use a dead symbol
whitelist to preserve the duplicate.

## Phase 2: Local capture lifecycle {#capture-lifecycle}

Primary files: `src/sase/main/ace_tmux.py`, `src/sase/screenshot/local.py`,
`src/sase/ace/tui/actions/screenshot_export.py`, the request-directory protocol, and
their tests. Read the TUI performance memory first.

Confirmed failures:

- Calling `create_agent_tmux_window()` twice against a private tmux socket creates two
  windows both named `sase_tmux_1`. tmux accepts duplicate window names; the current
  claim loop assumes it refuses them. Both results share one request dir, and the second
  returned name target resolves to the first process. This breaks `--keep` iteration and
  overlapping captures. The bad assumption predates the epic, but integrating the reused
  launcher is required for this feature's ownership contract.
- An injected resize failure after successful `new-window` produces no `kill-window`:
  `created_window` is set only after the launch helper returns. Malformed post-creation
  metadata needs equivalent ownership handling.
- Launch subprocesses receive no timeout, so the advertised overall deadline does not
  cover creation or geometry checks. Timeout error handling can also lose the last pane
  text while attempting a new capture with almost no time left.
- Startup accepts any nonblank frame followed by a fixed 100 ms delay. Export waits for
  one refresh and suppresses the no-running-node error; it does not establish the
  approved settled-frame contract. New detached refresh work makes a single refresh
  particularly weak evidence.

Implement a race-safe identity/claim mechanism using tmux's actual unique IDs or another
atomic reservation. Preserve useful printed target fields and deterministic request-dir
recovery for `--window`; preserve existing `sase tui --tmux` callers. Never use an
ambiguous display name to send keys, resize, or kill a newly owned window. Clean up
every partially created owned window on failure unless the documented keep behavior
intentionally retains it, and never clean up an existing window supplied by the caller.
Propagate one bounded capture budget through launch, drive, settle, and export; allow
only a small explicit best-effort cleanup budget.

Make the live export settle on finite visual work and meaningful compositor progress,
off the serial Textual pump. Reuse the production-safe ideas in
`tests/ace/tui/visual/_ace_png_snapshot_waits.py`, without importing test code or
waiting on unbounded recurring workers. Preserve real live timestamps/state and cursor
restoration. Report failures with the last known pane text.

Tests must exercise two actual tmux windows on an isolated socket, identity and
request-dir isolation, keep/recapture behavior, resize/metadata failures, deadlines, and
delayed visual changes. Fake tmux runners must model duplicate names accurately.

## Phase 3: Remote shell contract {#ssh-contract}

Primary files: `src/sase/screenshot/remote.py`, `tests/main/test_screenshot_command.py`.
Retain `src/sase/dispatch/ssh_target.py` as the shared target adapter and verify its
sudo callers remain correct.

The current `_ssh_argv()` appends an argv array after the host. SSH joins those
arguments into shell text; it does not preserve their original boundaries. Replaying
that boundary with `/bin/sh -c` and a harmless argument-printing executable shows
`-w 'Agents Ready|Loading'` invokes a command named `Loading` and exits 127. Spaces,
quotes, dollar signs, and forwarded TUI queries are equally affected.

The cleanup argv `sh -c 'rm -f PATH'` becomes `sh -c rm -f PATH`, which exits 1 with
`rm: missing operand`; the SVG remains. Current mocks falsely pass both paths by
treating the post-host words as a preserved argv and counting cleanup calls.

Serialize each complete remote command with correct shell quoting at the SSH boundary.
Verify literal forwarding of press keys, wait regexes, paths, and TUI arguments,
including spaces, quotes, pipes, and harmless shell metacharacters. Use a shell-boundary
harness that actually parses the transmitted command; prove actual removal of the
intended temporary SVG after success, capture failure, fetch failure, and version
failure. Preserve original errors when cleanup fails. Keep screenshot commands free of
privilege escalation or remote deployment.

Carry the local-capture overall deadline through probe/capture/fetch/version, with
bounded cleanup. Distinguish SSH connection/authentication/offline failures from an
installed remote sase lacking the screenshot contract. Keep local rasterization, remote
version reporting, enrollment-optional target selection, and a correctly quoted usable
send-keys hint. Exercise the existing sudo target-resolution regression tests.

## Phase 4: Whole-read inline deduplication {#memory-deduplication}

Primary files: `src/sase/memory/selector.py`, `render.py`, `selector_render.py`,
`cli_read.py`, and focused memory tests. This concerns the existing read/show
presentation pipeline; honor the Rust backend boundary for any shared domain logic
introduced, using `sase repo open sase-core` before accessing that repo if needed.

Confirmed isolated fixture results:

- Reading roots `a.md` and `b.md`, each containing `![[shared]]`, prints `SHARED_BODY`
  twice because each root receives its own seen set.
- A note containing `[[shared]]` before `![[shared]]` both embeds the target and lists
  it in Linked References. The real `tui.md` read has the same kind of duplicate for
  `tui_perf.md` through the screenshot child's reference.
- JSON rendering still lists an embedded child under `children`, while Markdown and Rich
  apply a suppression filter.

Use deterministic, batch-wide rendered identities and a final suppression step so
requested roots, shared inline descendants, and mixed note/web targets are rendered
once. Remove a target from Linked References and Children whenever its body is already
present, independent of link ordering. Preserve the user's requested root order, cycle
safety, depth caps, and always-reference handling for core notes and web descriptors.
Keep audit included-targets/byte counts consistent with actual bodies. Retain visibility
of references and non-inlined descendants; do not lose a grandchild merely because its
parent was embedded.

Add meaningful regressions for multiple roots, diamond/cycle paths, reference-before-
inline ordering, mixed note/web inputs, depth zero/one, and Markdown/Rich/JSON parity.
Verify the actual TUI hub read contains each body once with no duplicate listing.

## Phase 5: Guidance and real workflow acceptance {#workflow-acceptance}

The original approved phase 6 explicitly authorized `sase/memory/tui_screenshot.md` and
its hierarchy. Use `sase_memory_write` before editing. Complete that existing reference
note concisely: when to capture after a TUI change; how live captures complement the
golden lane; keys plus wait and `-- -t axe` examples; the keep, send-keys, recapture
loop; checking `remote_sase_version`; determinism versus live timestamps; missing visual
extra, tmux, old remote sase, and timeout troubleshooting. Keep exhaustive flags in CLI
help. Preserve the existing `tui_perf.md` body and note parentage, authored links, and
regenerate instructions with `sase memory init`.

Run real local PNG capture with keypresses and inspect the PNG. Run two retained
captures, drive them independently, then recapture the intended one using the printed
target and confirm the other is untouched. Use a private tmux socket/server and remove
only test-owned resources. Exercise a delayed render/selection so the final image proves
settling instead of merely checking a PNG signature.

Run a remote capture against a reachable authorized host if available, documenting the
exact invocation and reported version; do not deploy local changes remotely. Regardless
of host availability, run the real shell-boundary transport regressions. Confirm memory
init/read behavior and absence of new memory warnings.

Each coding phase runs `just fix` and `just check` under the normal verification rules.
Integrated acceptance also runs the visual lane through a monitor if lengthy, inspects
any failing images, and attributes failures before accepting new goldens. The land agent
performs the governed `just check-full` landing check. Carry the parent audit and the
`sase-x5` proposal disposition forward when landing this child.
