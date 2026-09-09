---
tier: epic
status: done
title: Reliable pager links
goal: "Pager link presses land on the file or artifact the text's author meant: file
  paths and typed refs resolve against the workspace directory where the corresponding
  agent ran when knowable, fall back through reliable anchors, and spans carry :line
  suffixes faithfully.

  "
phases:
  - id: resolve-search
    title: Context-aware hard search in the resolver
    depends_on: []
    size: medium
    description:
      "resolve-search: add LinkResolutionContext with ordered workspace anchors, rework
      file-path resolution into a candidate/probe ladder (line suffixes, trailing-dot
      and diff-prefix candidates, stale-workspace re-rooting, bounded unique-suffix git
      ls-files search), walk anchors for typed refs through resolve_cli_reference
      contexts, and thread context through resolve_ref and copy_text_for_target."
  - id: context-threading
    title: Thread real contexts from every pager entry point
    depends_on:
      - resolve-search
    size: medium
    description:
      "context-threading: carry link context on PagerDocument and per-section anchors on
      PagerSection, merge them on the press path and widen the ResolveRef signature,
      make followed documents inherit context, and construct honest contexts at every
      entry point (ACE agents/patches v, sase artifact read, sase bead show, sase pager,
      adapters)."
  - id: span-fidelity
    title: Scanned spans carry line suffixes and shed trailing dots
    depends_on:
      - resolve-search
    size: small
    description:
      "span-fidelity: add an opt-in shared matcher variant that extends file-path spans
      over :LINE/:LINE:COL suffixes and drops sentence-ending dots, adopt it only in the
      pager's link scanner, and pin the old matcher's behavior for ACE hint callers."
proposed_by: bbugyi200.athena.01q
bead_id: sase-xy
create_time: 2026-09-09 19:52:33
---

- **PROMPT:**
  [prompts/202609/pager_link_reliability.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/pager_link_reliability.md)
- **BEAD:**
  [sase-xy](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xy/README.md)

# Reliable Pager Links

## Problem

The link-traversing pager (`src/sase/pager/`) resolves every scanned file-path span
against exactly one base directory: `Path.cwd()` of the process hosting the pager
(`src/sase/pager/resolve.py`, `_resolve_file_path_target`). Typed artifact refs have the
same anchor bug in softer form: `_resolve_artifact_ref_target` calls
`resolve_cli_reference(ref)` without the `context=` parameter it already accepts
(`src/sase/artifact_cli/references.py`), so `plan:`/`chat:`/`file:` refs also resolve
relative to process cwd. That is wrong for most of what the pager shows:

1. **Agent documents describe work done elsewhere.** SASE agents run in ephemeral
   numbered workspace clones. A bead body, agent chat, or diff shown in the pager
   routinely says `src/sase/pager/resolve.py` meaning _the file in the workspace where
   that agent ran_ — but the pager probes cwd, which either dead-ends or silently opens
   a same-named file from an unrelated checkout.
2. **Absolute paths go stale.** A document that captured an absolute path inside a
   numbered workspace clone dead-ends forever once that clone is recycled, even though
   the same repo-relative file exists in the primary workspace and in the current
   checkout.
3. **`path:line` is promised but not implemented.** The pager design doc
   (`plan:202608/link_traversing_pager.md`, section D6) says "File path with `:line`
   opens as a pager document scrolled to that line", but `_resolve_file_path_target`
   never parses a `:line` suffix and the scanner's span never includes one.
4. **`y` copies guesses.** `copy_text_for_target` joins the span text onto `Path.cwd()`
   without an existence check, so the copied path frequently does not exist.
5. **Dead ends are mute.** The toast says only "could not be resolved", with no hint
   about what was searched.

The context needed to fix this exists at every construction site and is thrown away:

- **ACE Agents-tab `v`**: the live `Agent` has `workspace_dir`, `workspace_num` /
  `effective_workspace_num`, and `project_file`; the hint render already computes the
  correct base via `resolve_agent_workspace_dir(...)`
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_hint_render.py`), then
  `build_pager_document` (`src/sase/ace/tui/actions/hints/_files.py`) drops it —
  `document_from_paths(files)` is called without the `cwd=` parameter the adapter
  already exposes (`src/sase/pager/adapters.py`).
- **`sase artifact read`** (`src/sase/artifact_cli/read.py`, `_page_markdown`): the
  resolved `result.file: ArtifactFile` carries `workspace_dir`, `agent_artifacts_dir`,
  `agent_name`, and `project` (`src/sase/core/artifact_file_types.py`) — the exact
  "workspace where the corresponding agent ran" — and only the canonical reference plus
  body survive into the `PagerDocument`.
- **`sase bead show`** (`src/sase/bead/cli_show_batch.py`): each entry's
  `BeadStoreOrigin` (`src/sase/bead/cross_project.py`) carries `primary_workspace` and
  `project_key`, threaded through `_ShowRenderContext` (whose `design_cwd` _is_ that
  primary workspace) — none of it reaches the document.
- Nothing on `PagerDocument`/`PagerSection` (`src/sase/pager/document.py`) can carry any
  of this, and the press path (`src/sase/pager/_screen_actions.py`,
  `_resolve_and_dispatch`) forwards only the bare ref string
  (`ResolveRef = Callable[[str], LinkTarget | None]` in `src/sase/pager/app.py`).

## Goal

A pressed link should land on the thing the author of the text meant: file paths and
typed refs resolve against the workspace directory where the corresponding agent ran
when that is knowable, then fall through a short, ordered list of reliable anchors
(primary workspace, current checkout) plus a bounded repo-aware search — searching very
hard before declaring a dead end. Copy (`y`) and edit (`E`) must use the same
resolution, and scanned spans must carry `:line` suffixes faithfully.

## Design

### A. `LinkResolutionContext`: ordered anchors, not one base dir

A new module `src/sase/pager/link_context.py` with two frozen dataclasses:

- `LinkAnchor`: `directory: Path`, `workspace_num: int | None = None`.
- `LinkResolutionContext`: `anchors: tuple[LinkAnchor, ...]` (ordered, first match wins;
  a convenience `base_dirs` property yields the directories).

An anchor carries `workspace_num` so the same context serves both file probing and
typed-ref resolution (`artifact_ref_context(anchor_dir, workspace_num)` is what
`resolve_cli_reference(..., context=...)` wants; see
`src/sase/artifact_ref_context.py`). This mirrors the proven two-anchor retry in
`repair_artifact_read_path` (`src/sase/ace/tui/actions/hints/_artifact_ref_repair.py`),
which is the closest existing analogue and the pattern to follow.

Factory helpers, all existence-checked, order-preserving, deduplicating, and degrading
silently to a shorter anchor list on any failure:

- `default_link_context()` — anchor 1: `Path.cwd()` with its workspace number recovered
  from the checkout marker when present (`find_marker_from_cwd` in
  `src/sase/workspace_provider/marker.py`; primaries deliberately have no marker);
  anchor 2: the primary workspace via `get_primary_workspace_dir(cwd, workspace_num)`
  (`src/sase/sdd/_paths.py`, purpose-built ephemeral→primary: marker → project spec
  `WORKSPACE_DIR` → numeric suffix strip), when different from cwd.
- `agent_link_context(workspace_num, project_file, workspace_dir)` — anchor 1: the
  agent's own workspace via the existing `resolve_agent_workspace_dir` from
  `src/sase/ace/tui/widgets/prompt_panel/_file_path_hints.py`; anchor 2: the primary via
  `parse_workspace_dir(project_file)` (`src/sase/workspace_provider/utils.py`); then the
  `default_link_context()` anchors.
- `inherited_link_context(landed_path, parent_context)` — the landed path's parent
  directory (with marker-recovered workspace number) prepended to the parent context's
  anchors, for documents synthesized by a follow.

**Never call `get_workspace_directory_for_num` (`src/sase/running_field/_workspace.py`)
from this code: it cleans managed workspaces by default.** Every helper here must be
strictly read-only.

This machinery stays in Python rather than `sase-core`: it encodes which machine-local
directories to probe for one TUI surface (the `rust_core_backend_boundary` litmus test —
no other frontend consumes pager press resolution), and typed-ref parsing/resolution
already routes through the Rust binding via
`scan_artifact_refs`/`resolve_cli_reference`, untouched here.

### B. The hard search for file-path refs

`_resolve_file_path_target(text, context)` in `src/sase/pager/resolve.py` becomes:

1. **Split a trailing `:LINE` or `:LINE:COL` suffix** (digits only); keep the parsed
   line for `scroll_line`/`edit_line`, feeding the existing
   `_file_link_target(path, requested_line=...)`.
2. **Build candidate texts**, in order: the raw text; the text with trailing dots
   stripped (the shared path regex's character class includes `.`, so a sentence-ending
   `foo.py.` matches with the dot); the text with a leading `a/` or `b/` git-diff prefix
   stripped (diff output reaches the pager as piped stdin, so this is not origin-gated;
   tried only when the raw text does not resolve).
3. **For each candidate**, probe in order and return the first existing path:
   - _Absolute_: the path itself; if missing, **re-root it** — find the deepest prefix
     that is either a known anchor directory or a component matching the
     numbered-checkout naming contract (`<name>_<digits>`, the same convention
     `get_primary_workspace_dir`'s suffix-strip and `normalize_workspace_path` in
     `src/sase/bead/cli_common.py` already parse), take the remainder, and probe
     `anchor_dir / remainder` for every anchor.
   - _Relative_: `anchor_dir / candidate` for every anchor. `Path.cwd()` is always among
     the anchors, preserving today's behavior as a fallback rung.
4. **Bounded unique-suffix search**: only if every direct probe missed and the candidate
   has at least two components, ask each anchor's git index (`git -C <dir> ls-files`,
   subprocess with a short timeout, output size capped) for tracked files whose path
   ends with the candidate. Exactly one match across all anchors → use it; zero or
   several → dead end. Cache `ls-files` output per `resolve_ref` call so one press never
   runs the subprocess twice per anchor.
5. Nothing found → `None` (the existing D6 dead-end contract: toast, `⊘`, no navigation,
   trail untouched).

### C. Typed refs get anchors too

`_resolve_artifact_ref_target(ref, context)` tries
`resolve_cli_reference(ref, context=artifact_ref_context(anchor.directory, anchor.workspace_num or 1))`
for each anchor in order, accepting the first result whose resolution status is in
`_RESOLVED_STATUSES` — the same ladder `repair_artifact_read_path` walks. A `None`/empty
context preserves today's exact call shape.

### D. `resolve_ref` and `copy_text_for_target` take a context

Both gain keyword-only `context: LinkResolutionContext | None = None` (a `None` lazily
becomes `default_link_context()`, computed only when a file path or typed ref actually
needs anchoring). `copy_text_for_target` runs the same search as B and copies the first
existing resolution, falling back to today's cwd-joined absolute string when nothing
exists. Resolution already runs off the UI thread (`asyncio.to_thread` in
`_screen_actions.py`), so the extra `stat` probes and the bounded `ls-files` rung are
latency-safe; the whole search stays synchronous and side-effect free per the module's
existing contract.

### E. Context travels with the document

- `PagerDocument` gains `link_context: LinkResolutionContext | None = None`;
  `PagerSection` gains `link_anchors: tuple[LinkAnchor, ...] = ()` (prepended to the
  document context for spans pressed inside that section — one bead-show batch can
  render beads from different stores, and each section's store has its own primary
  workspace). Both default, so no existing constructor breaks, and `PagerTrailEntry`
  (`src/sase/pager/trail.py`) stores whole documents, so context rides the trail for
  free.
- The press path threads it: `PagerLabel` already records `section_index`
  (`src/sase/pager/_labels.py`), so the action path in `_screen_actions.py` merges
  section anchors + document context and passes the merged context to the resolver and
  to `copy_text_for_target`. The `ResolveRef` alias in `src/sase/pager/app.py` becomes
  `Callable[[str, LinkResolutionContext | None], LinkTarget | None]`; the one injected
  implementation (`_resolve_ref_from_link_index` in
  `src/sase/ace/tui/actions/hints/_files.py`) forwards the context to its `resolve_ref`
  fallback (the `LinkIndex` fast path ignores it — indexed targets are already
  concrete).
- **Followed documents inherit.** The documents synthesized inside `resolve.py`
  (`_file_link_target`, `_directory_link_target`) get
  `inherited_link_context(landed_path, pressing_context)`, so a hop into another
  workspace keeps resolving inside that workspace, trail-deep.

### F. Entry points construct honest contexts

- **ACE Agents-tab `v`** (`src/sase/ace/tui/actions/hints/_files.py`,
  `build_pager_document` and its caller):
  `agent_link_context( agent.effective_workspace_num, agent.project_file, agent.workspace_dir)`
  — the same triple the hint render already computes.
- **ACE Patches-tab `v`**: the patch's workspace directory via the existing
  patch-workspace lookup in `src/sase/core/patch.py`, then default anchors.
- **`sase artifact read`** (`src/sase/artifact_cli/read.py`, `_page_markdown`): when
  `result.file.workspace_dir` is set, that directory is the first anchor (with
  `parse_workspace_dir`-derived primary second); otherwise `default_link_context()`.
  This is the highest-value site: chat transcripts, plans, and research documents open
  through it, and the `ArtifactFile` names the exact agent workspace.
- **`sase bead show`** (`src/sase/bead/cli_show_batch.py`): per-section `link_anchors`
  from the entry's `BeadStoreOrigin.primary_workspace` (already threaded through
  `_ShowRenderContext`); document context `default_link_context()`.
- **`sase pager` CLI** (`src/sase/main/pager_handler.py`), the `page_or_print` stdin
  fallback (`src/sase/cli_pager.py`), and `src/sase/pager/adapters.py`
  (`document_from_paths`/`path_section`): `default_link_context()`; `path_section`
  additionally sets `link_anchors=(anchor for the file's parent directory,)`.

### G. Dead ends say what was tried

When a `file_path`-kind span resolves to `None`, the notify message includes how many
locations were probed (e.g. `src/x.py not found (searched 3 locations)`), sourced from a
small resolver-side helper so the screen does not re-derive the search. Other kinds keep
the existing message. The D6 contract is unchanged: toast, mark `⊘`, never navigate,
never touch the trail.

### H. Scanned spans carry their line suffix

An opt-in variant of the shared matcher in
`src/sase/ace/tui/widgets/prompt_panel/_file_path_hints.py` extends a matched path span
over a trailing `:LINE`/`:LINE:COL` and excludes trailing dots followed by whitespace or
end-of-line. Only the pager's `scan_links`/`scan_bounded_links`
(`src/sase/pager/link_scan.py`) adopt it, so ACE hint callers keep byte-identical
behavior. The span text then reaches `resolve_ref` with the suffix attached, which
design point B already parses — so the painted label, the copied text, and the resolved
target all agree.

## Explicitly out of scope

- No change to URL or bare-token handling, and no change to which spans exist for ACE
  hint surfaces.
- No feature flag: every change is an additive fallback ladder whose final rung is
  today's behavior (cwd-anchored probe), so existing links can only resolve at least as
  often as before.
- No fuzzy or basename-only matching beyond the unique-suffix `ls-files` rung; ambiguity
  is a dead end, never a guess.
- No workspace-provenance lookup for _raw_ chat transcript paths opened as plain files
  (the `chat_catalog_provenance` join from transcript → `agent_meta.json`). Chat
  transcripts opened as `chat:` refs already get exact context via
  `ArtifactFile.workspace_dir`; the raw-path join is real work with its own failure
  modes and belongs in a follow-up if wanted.
- No persistence or index of historical workspace layouts.

## Phases

### Phase 1: Context-aware hard search in the resolver (resolve-search)

Implement design points A, B, C, D, and G in `src/sase/pager/link_context.py` (new) and
`src/sase/pager/resolve.py`:

1. Add `LinkAnchor`, `LinkResolutionContext`, `default_link_context()`,
   `agent_link_context()`, and `inherited_link_context()` — read-only, existence
   checked, silently degrading; never touch `get_workspace_directory_for_num`.
2. Rework `_resolve_file_path_target` into the candidate/probe ladder of design point B:
   `:line[:col]` parsing into `scroll_line`/`edit_line`, trailing-dot and `a/`/`b/`
   candidates, stale-absolute re-rooting, anchor-ordered relative probing, and the
   bounded unique-suffix `git ls-files` rung with per-call caching and a subprocess
   timeout.
3. Thread anchors into typed-ref resolution (design point C):
   `_resolve_artifact_ref_target` walks anchors as `artifact_ref_context`s through
   `resolve_cli_reference(..., context=...)`.
4. Add `context` keyword-only parameters to `resolve_ref` and `copy_text_for_target`
   (design point D), defaulting to `None` → lazy `default_link_context()`, leaving every
   existing caller compiling unchanged.
5. Add the dead-end description helper (design point G).
6. Tests in `tests/pager/test_resolve.py` plus a new `test_link_context.py` (mind the
   repo's file-size cap): tmp-path fixtures building two fake workspace trees asserting
   — relative path found in a later anchor; stale absolute numbered-clone path
   re-rooted; `:line` suffix parsed into scroll/edit line; trailing-dot candidate; `a/`
   diff prefix; unique-suffix hit and ambiguous-suffix dead end; typed-ref anchor walk
   (stub `resolve_cli_reference`); `copy_text_for_target` returns the existing
   resolution; `context=None` still resolves cwd-relative paths as before. Note:
   `test_copy_text_for_target_resolves_a_relative_file_path` currently pins the cwd-only
   behavior via `monkeypatch.chdir` — update it deliberately, not incidentally.

Verification: `just check` (run `just install` first if the workspace is a fresh clone).

### Phase 2: Thread real contexts from every pager entry point (context-threading)

Implement design points E and F:

1. Add `PagerDocument.link_context` and `PagerSection.link_anchors` in
   `src/sase/pager/document.py` (both defaulted).
2. Update the `ResolveRef` alias in `src/sase/pager/app.py` and the press path in
   `src/sase/pager/_screen_actions.py`: merge section anchors + document context via the
   pressed label's `section_index`; use the merged context for the resolver call, for
   `_copy_target`'s `copy_text_for_target`, and for the file-path dead-end message.
   Update `_resolve_ref_from_link_index` in `src/sase/ace/tui/actions/hints/_files.py`
   to the new signature.
3. Make followed file/directory documents inherit context (`_file_link_target`,
   `_directory_link_target` in `resolve.py`).
4. Construct contexts at every entry point per design point F: ACE Agents-tab `v`, ACE
   Patches-tab `v`, `sase artifact read` (`_page_markdown`, from
   `result.file.workspace_dir`), `sase bead show` (per-section anchors from
   `BeadStoreOrigin.primary_workspace`), `sase pager`, the `page_or_print` stdin
   fallback, and `document_from_paths`/`path_section` (per-section parent-dir anchor).
5. Tests: extend `tests/pager/test_resolve.py`, `tests/pager/test_document.py`, and the
   press-path tests in `tests/pager/test_app.py` (which already follow/copy/edit a
   path-link document) to assert a pressed relative span in a section with
   `link_anchors` resolves against the section anchor before the document context;
   update `tests/ace/tui/actions/test_view_files_pager.py` (which exercises
   `_resolve_ref_from_link_index` directly) for the new signature and assert the agent
   context triple is attached; extend `tests/main/test_pager_command.py`,
   `tests/test_bead/test_cli_show_pager.py`, and the `sase artifact read` pager tests
   for their new contexts.

Watch `tests/pager/test_rail_parity.py` and any other test constructing a `ResolveRef`
callable — apply the signature change consistently rather than shimming. (Two
`test_rail_parity` parametrizations are in `tests/reproducible_flake_baseline.txt`; do
not mistake that pre-existing flake for a regression.) No rendering changes in this
phase: if a PNG snapshot under `tests/pager/visual/` shifts, treat it as a regression,
not a golden update.

Verification: `just check`.

### Phase 3: Scanned spans carry line suffixes and shed trailing dots (span-fidelity)

Implement design point H:

1. Add an opt-in matcher variant in
   `src/sase/ace/tui/widgets/prompt_panel/_file_path_hints.py` that (a) extends a
   file-path match span over a trailing `:LINE`/`:LINE:COL` and (b) excludes trailing
   dots followed by whitespace/end-of-line. Existing exported matchers
   (`iter_file_path_matches` and friends) must keep byte-identical output for all
   current callers.
2. Use the variant from `scan_links` and `scan_bounded_links` in
   `src/sase/pager/link_scan.py` (both the scan and the `bound_hint_content` matcher
   argument, so budget bounding and span scanning agree on what a path is).
3. Tests in `tests/pager/test_link_scan.py`: span includes `:12` and `:12:5`; span
   excludes a sentence-ending dot; a path inside a URL is still not a file span; one
   regression test pinning the unchanged output of the old matcher on the same fixture
   text.
4. If label-painting PNG goldens under `tests/pager/visual/` legitimately change (spans
   grow where `:line` suffixes appear in fixture text), update them via
   `just test-visual --sase-update-visual-snapshots` and inspect the diffs in
   `.pytest_cache/sase-visual/`.

Verification: `just check`, plus `just test-visual` because this phase can move painted
spans.

## Risks and mitigations

- **False positives from the suffix search**: unique-match-only rule and the
  two-component minimum; ambiguity is a dead end.
- **Latency on press**: all probing is off-thread behind the existing generation-checked
  resolve task; the only subprocess is one capped `ls-files` per anchor per press, and
  only after every direct probe missed.
- **Accidental workspace mutation**: the one dangerous API
  (`get_workspace_directory_for_num`, which cleans by default) is named and banned in
  design point A; every context helper is read-only.
- **`ResolveRef` signature ripple**: exactly one injected implementation exists
  (`_resolve_ref_from_link_index`); tests that stub resolvers are updated in the same
  phase that changes the alias.
- **Shared-matcher drift**: ACE hint rendering reuses the same regex module; phase
  span-fidelity adds a variant instead of changing shared behavior, with a regression
  test pinning the old matcher.
