---
tier: tale
title: Complete sase-1dr.6 pager time axis and read view
goal:
  Give memory and instruction-file pager sections a responsive, revision-pinned read
  view with correct navigation, trail restoration, historical links, and flag-gated CLI
  entry.
size: medium
proposed_by: bbugyi200.apollo.sase-1dr.6
bead: sase-1dr.6
create_time: 2026-10-01 03:06:34
status: wip
---

- **PARENT:**
  [202609/memory_history.md](https://github.com/sase-org/sase--plans/blob/main/202609/memory_history.md)
- **BEAD:**
  [sase-1dr.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1dr/sase-1dr.6.md)

# Pager time axis and read view

## Ownership and scope

Implement the assigned phase **sase-1dr.6**, `pager-axis`, of
`plan:202609/memory_history.md`, especially §4.2–4.3, §4.9–4.11, §5.4–5.5, and §11.
Phase sase-1dr.5 has supplied the Rust bindings, scope builders, `HistoryService`,
vocabulary, and text/JSON CLI. This is a tale because one coder can complete this
bounded pager integration against those existing APIs; the parent epic already assigns
the remaining feature surfaces to other phases.

The phase is already assigned and `in_progress`. Preserve its assignment and never set
its status by hand. Close only sase-1dr.6 after verification. Do not close sase-1dr or
any ancestor or create discovered-work beads. Record discoveries using
`sase bead note sase-1dr.6 'PROPOSED FOLLOW-UP: <summary — evidence and detail>'`. The
prescribed `sase flag new` command is the epic's flag-registration workflow; do not
manually create its flag bead or any additional plan/task beads.

This phase owns the generic history protocol, memory provider, `( ) { }` actions,
version pins and trails, scroll/search preservation, read-view gutter and subject chip,
history palette, revision-aware links, copy/edit semantics, and selector-based pager
CLI. The full time band/sparkline belongs to sase-1dr.7, word-diff rendering and
`[ ]`/`=` to sase-1dr.8, picker to the next phase, and the feed and Memory panel to
their assigned phases. Provide usable seams for those consumers without implementing
their UI. Add footer/help entries only for available actions.

## Existing code and contracts

- `src/sase/pager/screen.py` composes the action/body/trail/chrome/search/syntax mixins.
  Its syntax worker starts after paint; teardown already calls
  `cancel_pump_free_tasks(self)`.
- `_screen_body.py:_apply_refreshed_document` replaces a document without pushing the
  trail, but exits search and goes to the section header. A history swap must preserve
  the search query and map the visible passage instead.
- `PagerSection` is frozen; add optional history metadata with defaults so existing
  producers remain compatible. Its `owner` is an `ArtifactRefDocumentOwner` with an
  existing `revision` field. Owner revision participates in dangling-link cache keys.
- `_screen_syntax.py` currently indexes prepared and attempted sections by identity
  alone. Its underlying lex/styled caches already use content digests.
- `_screen_actions.py` resolves links in pump-free tasks, generation-checks results,
  pushes a trail entry on navigation, and takes section owner context. Extend this path
  rather than adding a second navigation pipeline.
- `PagerTrailEntry` already snapshots the document, search, line mark, label anchor, and
  scroll position. Explicit version/view pins must join that snapshot.
- `HistoryService` exposes `sync`, `resolve(at_commit=...)`, `timeline`, `version`,
  `compare`, and `resolve_in_scopes`. Build project scopes from section provenance, and
  use `map_deployed_home_path` for home files; viewer cwd is not ownership.
- The existing Rust wire returns timelines newest-first, committed ordinals starting at
  1, and pseudo-versions with ordinal 0. `version('now')` reads the worktree; its row
  class alone does not prove dirtiness. Use timeline OIDs/status for that. Deleted
  committed versions return their last-content blob, and `body_missing` distinguishes
  unavailable content from an empty file. Resolve returns `existed: false`/no version
  before creation. Comparisons wrap the prose result in `comparison`, with 1-based
  `line_marks`, separate `removal_anchors`, and `line_map.base_to_target` /
  `target_to_base`.
- The linked core already exposes `compare_prose` and `prose_diff_wire_schema_version`,
  though sase has no prose-diff facade yet. Add thin local binding glue only if needed
  for comparison against an empty creation base; retain the Rust wire unchanged and
  require the binding without a fallback.
- The canonical shared vocabulary is `src/sase/memory/history/vocabulary.py`. Justfile
  currently exempts `is_hidden_by_default` and `label_for` under sase-1dr.6.
- The parser already accepts `-f pager`, but defaults to text and the handler rejects
  pager output. Existing explicit text/JSON behavior must remain supported.

Read the applicable audited memory notes before implementation: `tui.md`, `tui_perf.md`,
`sase_flags.md`, `cli_rules.md`, and `lint_and_test.md`. Read `symvision.md` before
repairing unused-symbol diagnostics. Read all relevant local AGENTS.md instructions. If
a genuine shared backend gap appears, open sase-core with
`sase repo open sase-core -r '<specific reason>'` and follow its instructions; do not
implement git, lineage, selector semantics, or diff algorithms in Python.

## Implementation sequence

### 1. Register the temporary flag and generic provider seam

Create the beta with `sase flag new memory_history --kind beta` and authored
`--when-enabled`, `--when-disabled`, and `--remove-when` sentences. Enabled means
memory/instruction pager sections receive history, TTY history defaults to the pager,
and later panel consumers may expose history. Disabled means ordinary pager behavior,
text TTY default, explicit pager rejection, and no panel history. Removal means the
epic's remaining feature phases and end-to-end verification have passed. Paste the
command's exact enum/registry entry into `src/sase/feature_flags/registry.py`; retain
its removal bead for the launch phase. Do not toggle the user's saved flags while
testing; use `override_flags`.

Add small modules under `src/sase/pager/history/`:

- `VersionPin`: subject identity, version ordinal/selector, full commit, blob OID, view
  (`read`/`diff`), and optional comparison base. Distinguish live and staged states from
  committed ordinals without inventing durable version numbers. Include enough
  owning-scope information for stable lookup or keep it on the provider state.
- `SectionTimeState`: provider/subject, loading/error/status, full timeline and visible
  ordinals, current pin, immutable live section, pending time intent, generation, and
  bounded body/comparison caches. Keep per-section state isolated across scopes.
- `SectionHistoryProvider`: worker-safe methods to recognize/load a section's timeline,
  load a version, compare versions, resolve a same-scope historical link, and
  refresh/re-sync. Normalize wire results into generic pager presentation records; no
  memory-specific vocabulary or core query algorithm belongs in this package.
- A testable provider registry with `history_provider_for_section(section)`, invoked
  off-thread after paint. Providers may decline sections and failures are isolated.

Use a lazy generic registration mechanism so every `PagerScreen`, standalone or in ACE,
can discover providers without importing memory modules in pager core. Concretely, a
`sase_pager_history` package entry point in `pyproject.toml` can load a memory-provider
factory; registry discovery/factory invocation runs after paint off-thread. The memory
factory returns no registration when the beta is off, before constructing
`HistoryService` or inspecting files. Cache discovery without freezing a disabled
decision across test flag overrides. Keep injection hooks for deterministic tests and
verify entry-point visibility in the installed environment.

### 2. Implement the memory provider and document construction

Add `src/sase/memory/history/pager_provider.py`, backed exclusively by the existing
`HistoryService` and scope helpers. Recognize canonical and legacy project memory, web
descriptors and strands, root/subdirectory AGENTS.md and shims, chezmoi source
templates, and deployed home files. Inspect subject/owner provenance rather than title
alone, and decline unrelated files. Recover the owning checkout from owner checkout
candidates/source directory; never substitute the current viewer project. Use the
existing scope mapper for deployed paths and label unsupported home as NO VCS without
preventing the live document from being read.

Return historical bodies as derived `PagerSection`s retaining logical identity,
raw-source eligibility, known kinds, and link provenance. Replace the body, title/path
at the selected version, and optional `version_pin`; set owner `revision` to the
selected commit and source directory to that historical path's parent. Retain the live
current path separately for edit/return-now. Exact shim divergence must use the
version's own blob/source path, never today's generated shim contents.

Load all timeline metadata (`include_hidden=True`) once and derive the ordinary step
list from the wire's hidden bit with the shared vocabulary helper as appropriate. Keep
hidden rows available for later picker/band consumers. Default now for an existing file,
or the newest deletion tombstone when the subject is deleted. Treat `body_missing` as a
labelled unavailable result rather than valid empty text. Classify dirty-now from
worktree/index/HEAD status, not the synthetic row class. For v1 or a recreation after a
deletion gap, compare against empty content using the existing Rust `compare_prose`
binding so newly created content gets correct added-line marks. Ordinary parent
comparisons use the preceding subject version, including hidden rows. A clean now uses
the latest committed change; dirty now compares worktree to HEAD. Keep tombstone content
and its deletion rule distinct from the empty post-deletion file state.

Provide a reusable selector-to-pager document builder in the memory history package that
accepts scope, subject, optional initial revision, and view/comparison base. Later
panel/feed phases should call it. Keep selector/date conversion shared with the existing
CLI instead of duplicating its private parsing logic.

### 3. Add history state, worker scheduling, and modeless navigation

Compose `PagerHistoryMixin` from `_screen_history.py` into `PagerScreen` and bind
Textual punctuation keys for `(`, `)`, `{`, and `}`. Initialize lightweight state
synchronously, then schedule provider discovery/indexing after first paint with a
synchronous callback that only calls `spawn_pump_free_task`. Run service calls and file
work through `asyncio.to_thread`; cancel history tasks at unmount through the existing
task registry and ensure a failed spawn releases coalescing guards.

The state machine must implement:

- `(` from now opens the newest visible committed version; subsequent `(`/`)` moves skip
  hidden rows. `)` past the newest committed row restores now, even if now has identical
  bytes. Boundaries give an explicit harmless notice.
- `{` selects the first version; `}` follows now or its deletion tombstone.
- During provider discovery/indexing, store the last time intent and execute it when
  metadata is ready. Distinguish pending discovery from a confirmed unsupported section;
  do not silently discard a history key while recognition is in flight.
- Capture document/section identity, scope, generation, and intended target for each
  request. Advance generation on selection, refresh, navigation, and restoration. After
  every await revalidate them and current UI state; stale loads/prefetches cannot
  overwrite a newer view, footer, or section. Loading failure keeps/falls back to the
  live document with one concise notice.
- Preserve a live section snapshot, but `r` explicitly re-syncs and re-reads now
  off-thread. Refresh follows HEAD and clears stale pin/body/status state without
  disturbing existing refresh providers on unrelated sections.
- Prefetch version bodies, parent comparisons, and step-anchor comparisons for ±2
  neighbours after each selection. Use bounded session caches keyed by owning scope,
  subject, blob/commit or live freshness token, and comparison pair. Cache hits avoid
  git/file I/O on key/render paths; cache misses remain responsive worker requests.
  Parent comparisons feed the read gutter; old-to-new comparisons anchor a step that
  skips hidden versions. Do not confuse those comparison purposes.

Replace only the active section in a multi-section document using refresh-style
replacement without a trail push. Capture the top visible logical line and wrapped row
offset before replacement; map it in the proper direction through the core line map,
then translate to the composed section's new row and clamp. Account for document section
offsets, removed lines, empty bodies, and resized wrapping. Reapply the persisted search
query against the new corpus and styled base, preserving direction and selecting a valid
nearby result rather than reusing stale character offsets.

### 4. Restore pins and make syntax caching version-safe

Add `PagerSection.version_pin = None` and defaulted `PagerTrailEntry.version_pins`.
Snapshot pins for all sections and restore the exact ordinal, commit, view, compare
base, search and scroll position. A step does not alter either trail stack. Historical
link/picker/feed-style jumps use the existing single trail push. Ensure navigation and
back/forward invalidate obsolete history work and rehydrate appropriate states before
after-paint provider tasks can overwrite restored pins.

Change `_syntax_prepared` and `_syntax_attempted` keys to `(identity, blob_oid)` for
committed pins, with a content-digest/freshness key for live sections and safe legacy
behavior for non-history sections. Update every consumer, including hint lookup,
prepared-text collection, attempted checks, refresh and theme invalidation. Preserve the
existing cross-document syntax caches. A same-identity version swap must never paint
syntax from another body; tests must include pending lexing during rapid steps.

### 5. Add read-view marks and history chrome

Thread the parent comparison's line marks and removal anchors into `_layout.py` and
`_gutter.py:apply_gutter`. Reuse the rail column: green `▌` for added lines,
modified-accent `▌` for changed lines, and red `╴` at removal anchors, including
before-first and after-last removals. Preserve gutter width, logical/visual maps, wrap
behavior, label/search offsets, and existing goto emphasis. Define and test priority
when a goto rail and change mark coincide. Show a deleted version's retained content
under the muted-red deletion rule with date/author and `last content shown`.

Extend `_chrome.py:subject_line` and the screen chrome calls with optional generic
history state: violet `⟲ PAST vN/total · age` for committed pins, dim version count at
clean now, amber `◌ uncommitted` for dirty now. Truncate/crop without wrapping at narrow
widths. Footer exposes applicable version/now actions plus `E edits now` in the past; do
not advertise unfinished diff/picker controls. Add a Time help group through
`_trail_chrome_help.py`/`_help.py`. Preserve non-history rendering defaults.

Add `history_palette_from_theme` beside the syntax palette, with past accent seeded
around `#9d7cd8`, insert foreground/tint, deleted-word style, gutter styles, sparkline
roles and tombstone rule for later phases. Reuse contrast utilities; if needed expose a
small public contrast helper rather than importing private symbols across modules. Text
contrast must be ≥4.5, mark contrast ≥3.0, and past hue must stay ≥60° from each
built-in theme's warning hue. Preserve hue when correcting contrast and invalidate
history styling when the host theme changes.

### 6. Resolve historical links and keep edit/copy honest

Offer followed file-path and memory-selector links in a pinned section to its provider
before the ordinary resolver, inside the existing off-thread resolution worker. Resolve
relative paths against the source's historical directory and use
`HistoryService.resolve(..., at_commit=pin.commit)` within the same scope. Recognize
memory artifact refs as well as scanned paths without parsing artifact grammar
independently of existing helpers. Carry fragment/range landing information through
existing landing APIs. Beads, agents, URLs, plans and other-scope targets continue
through their normal resolvers.

When the target existed, build a pinned document from its as-of version and navigate
through the normal trail push. When absent, show a one-section notice with source
commit/date, creation date where known, and attached labels to open at creation or now.
Missing/ambiguous history stays an explicit notice; a same-scope historical miss must
never fall through to today's file. A deletion gap must be represented honestly using
the resolved version's deletion state, not treated as live content. Use stable
revision/scope-aware dangling-cache identities.

For section `yy` in a committed read view, copy
`<sha>:<repo-relative path at that version>`. `E` always resolves/edits the current live
path with an unpinned owner context, including when a note has been renamed since the
displayed version. Deleted targets offer a clear edit-unavailable notice if no live file
exists. Keep current labeled-target copy behavior; reserve unified-diff copy for the
diff phase.

### 7. Wire selector-based CLI pager entry

Update `parser_memory.py` help and default to an unset format, letting the handler
choose pager on a TTY only when the beta is on, otherwise text. Explicit `text` and
`json`, piped defaults, existing option aliases, and no-read-audit behavior stay intact.
Explicit pager while off exits with a clear flag notice. While on, explicit pager and
the TTY default open `SasePager` with the history builder for each selector. No `-A`
means now/tombstone; `-A` reuses `vN`, `~N`, SHA and date resolution. Multiple selectors
remain independent sections with independent scopes and pins.

This phase supplies the initial view/compare-base seam for `-d`; actual word-diff pager
rendering belongs to sase-1dr.8. Until that renderer exists, give an explicit read-view
notice or preserve the existing text diff path rather than claiming a diff view was
painted. Similarly, keep no-selector feed output using the existing text path with a
concise interim notice until the assigned feed phase wires its document; do not
implement a competing feed here. Document these temporary boundaries in the phase notes
so downstream workers replace them. Share service/provider state between CLI preparation
and the screen where practical and leave all history IO off the UI.

## Verification and completion

Add meaningful tests using an injected generic fake provider for deterministic
state-machine/race coverage and real git fixture histories for memory integration:

1. Flag off leaves unrelated and memory pager rendering/navigation unchanged, registers
   no provider, does not instantiate/query the service, keeps text TTY default, and
   rejects explicit pager. Flag on supports plain file/ACE-hosted pager discovery and
   selector CLI, with correct TTY/piped/explicit format selection and no read-audit
   rows.
2. Project/legacy memory, webs/strands, subdirectory instructions/shims, source and
   deployed home paths, alternate-owner checkout, unsupported/no-VCS files, renames,
   diverged blobs, deletion/tombstone and body-missing cases.
3. Stepping boundaries, hidden-row skipping, first/now, queued indexing intent, rapid
   repeated keys, refresh while loads run, section switches, trail return, failed
   workers/spawns, and teardown. Use controlled futures/events to prove stale work
   cannot publish; do not rely on timing sleeps.
4. Bidirectional anchors through insertions/deletions, hidden-version jumps, empty
   bodies, wrapped lines, multi-section offsets and narrow resize; persistent search on
   new content. Exact trail pin/view/base round trips with no trail push on steps.
5. Same-scope historical links, relative historical paths, selector refs, fragment
   landings, pre-creation/gap notices and their creation/now labels; normal other-kind
   links; pinned copy and live-path edit after rename.
6. Syntax/body correctness for different blobs sharing one section identity, including
   in-flight syntax and theme changes; gutter add/change/removal anchors and goto rail
   coexistence; all-built-in-theme contrast and warning-hue assertions.
7. New deterministic PNG goldens under `tests/pager/visual/` for past read view, dirty
   now, and tombstone, each at 120×40 and 60×30 in dark and light themes. Use fixture
   providers and fixed dates/OIDs so screenshots never depend on checkout history or
   today's time. Inspect every new image and any changed golden group.

Run targeted pager/history/CLI tests during implementation. After formatting with
`just fix` (or at least `just fmt`), run
`just fix-tui-screenshots -- tests/pager/visual/test_history_png_snapshots.py` for the
new cases, plus affected existing visual selectors. Read the unique visual report and
its skipped/partial warnings; generation alone is not approval. Run final
`sase tool run check`, the recorded form of `just check`. Do not run `just check-full`
without an explicit instruction. Use `/sase_monitor` for commands that outlast the
single turn, with a follow-up that reviews results/screenshots and finishes the bead;
wait for the monitor-start command itself to exit. A mutating screenshot run must not
use a prepared completion that omits image/report inspection.

Verify after-paint history startup with a blocked fake provider: first paint and
ordinary scroll/search/quit must remain responsive while indexing. Measure warm
prefetched version-step latency with `SASE_TUI_PERF=1`/`SASE_TUI_TRACE=1`, targeting p95
≤30 ms; record the measurement and any constraints in the bead's notes. The launch phase
owns final end-to-end performance verification.

Before closure run `sase bead epic-symbols sase-1dr.6`. Consume the existing
`is_hidden_by_default` / `label_for` helpers in provider filtering/notices where
appropriate and remove their Justfile exemptions once no longer needed. Any symbols
reserved solely for a later consumer must be re-keyed to the still-open parent or the
correct later phase after verifying that bead is open. Re-run the symbol audit and lint
so closure cannot leave stale exemptions.

Fix failures attributable to the change. For a failure reproduced identically on the
clean base tree, record a `PROPOSED FOLLOW-UP:` on sase-1dr.6 with command/evidence and
any existing tracking bead; that failure does not keep this phase open. Use an isolated
base comparison without disrupting the working changes and document the comparison. Do
not create follow-up beads yourself.

Finally run
`sase bead close sase-1dr.6 --note '<implemented behavior; targeted, visual, and check results; any verified clean-base failures; epic-symbol clearance>'`.
Leave all ancestors and the temporary flag bead open for their owning agents. Use
`/sase_final` as the last action before a normal final response; host finalizers own
commits, branches and PRs. A successful plan/monitor handoff instead follows its
mechanical termination contract.
