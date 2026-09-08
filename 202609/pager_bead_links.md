---
tier: tale
title: Make pager bead hints open live details without published pages
goal:
  Existing beads open through pager hints and direct pager inputs using the correct
  project store, independently of generated pages, with safe navigation and regression
  coverage.
size: medium
proposed_by: bbugyi200.athena.08z
create_time: 2026-09-08 12:31:57
status: wip
---

# Make pager bead hints open live bead details without requiring published pages

Repair bead navigation in the SASE pager so that an existing, readable bead opens even
when its generated Markdown page is absent. Preserve project ownership, relationship
rendering, and pager history. One coding agent can implement and verify this bounded
adapter change; an epic is unnecessary.

## Diagnosis and evidence

The user's screenshot, `~/tmp/screenshots/20260908_121439.png`, shows a multi-bead pager
document. The `sase-xi` section contains both a typed `bead:sase-po` link in LINKS and a
bare `sase-po` mention in its reason. Both receive painted hints, and following the bead
produces `bead:sase-po could not be resolved.` This places the failure after scanning
and hint activation.

Read-only probes on September 8, 2026 produced this comparison:

| Reference        | Artifact/page resolution                        | Public pager resolution            | Direct bead-detail loader       |
| ---------------- | ----------------------------------------------- | ---------------------------------- | ------------------------------- |
| `bead:sase-po`   | `missing`, candidate `pages/sase-po/README.md`  | No target, screenshot's diagnostic | Correct `bead:sase-po` document |
| `bead:sase-xi`   | `missing`, candidate `pages/sase-xi/README.md`  | No target                          | Correct `bead:sase-xi` document |
| `bead:sase-y3.3` | `exact`, candidate `pages/sase-y3/sase-y3.3.md` | Correct document                   | Correct document                |

`sase bead show sase-po --format json` successfully returned the existing bead,
including its related link to `bead:sase-xi`. `sase artifact read bead:sase-xi` failed
with the published-page hint. The normal pager and direct-loader comparison used
`resolve_link()` and `_bead_link_target()` from this checkout, with the installed Rust
binding. No source files were changed for the diagnosis.

The causal chain is:

1. `src/sase/pager/document.py` normalizes a bare ID in a bead document to `bead:<id>`.
   Typed references reach the same resolver.
2. `src/sase/pager/resolve.py::_resolve_artifact_result()` calls
   `resolve_cli_reference()` and rejects non-success artifact statuses **before** its
   `kind_type == "bead"` branch.
3. Full bead IDs defer to the Rust artifact resolver. In the opened `sase-core`
   repository, `crates/sase_core/src/artifact_ref/mod.rs::resolve_bead()` explicitly
   resolves the generated page without reading the bead store. It returns `missing` when
   that file is absent. Shorthand's built-in adapter also ultimately consults this page
   resolver.
4. The pager's `_bead_link_target()` can already build the desired live view using
   `resolve_show_batch()` and `build_show_batch_document()`. The page gate prevents it
   from running. Requiring this separate publication projection for a live detail view
   is the root defect.
5. Existing successful tests in `tests/pager/test_resolve.py` invoke
   `_bead_link_target()` directly. The rendered-link corpus treats its bead examples as
   unavailable. Neither covers a painted hint navigating to a real bead that has no
   generated page.

Two adjacent seams must be handled while reconnecting this path:

- `_bead_link_target()` currently discards the passed document context and opens the
  ambient CLI store. Indexed bead targets also discard their project part.
- `get_read_view()` delegates to `get_project()` for writable locations; that can
  materialize or initialize a store. A pager keypress needs an existing-store read, and
  batch failures should retain their diagnostic instead of collapsing to None.

The absence of particular generated pages is established; its historical publication
cause was not established and is not needed to make live navigation correct.

## Intended implementation

### 1. Dispatch bead navigation to the live detail adapter

- Keep canonical artifact parsing. Dispatch a successfully parsed bead reference to the
  bead pager adapter before the generic published-file status gate. Use the parsed
  payload ID, including shorthand, rather than splitting an unparsed token; preserve the
  existing fragment behavior rather than accidentally including a fragment in the store
  lookup.
- Give this path one `LinkResolution` result containing a target or a useful failure.
  Keep `resolve_ref()` as the target-only convenience API. Extract the bead adapter into
  a focused pager module if necessary to keep `resolve.py` within repository size
  limits.
- Use this same adapter for `link_target_for_artifact_entry_target()`'s bead case,
  retaining a populated project part from `(project, kind, bead_id)` and its supplied
  `LinkResolutionContext`. Avoid two bead resolution implementations.
- `sase pager bead:<id>`, sigiled typed references, bare IDs scanned in bead sections,
  and hosted/attached pager targets must all reach this path. Bare-ID scanning remains
  limited to bead-origin sections.
- Leave generic artifact read/prompt publication semantics intact. Do not report a
  fabricated filesystem success or create a page as a side effect of navigation. A
  generated page is neither a prerequisite nor a substitute for a live bead.

### 2. Reuse existing store routing and detail rendering with explicit context

- Resolve the local read location from the originating document's owner/anchors,
  retaining workspace identity. Use the ordinary default context only when the caller
  provides no owner or anchors. Do not temporarily change process cwd or environment:
  multiple background resolutions can overlap.
- Open existing stores through
  `resolve_beads_location(..., require_existing=True, materialize=False)` and
  `open_bead_project_for_beads_dir()`, or the equivalent existing contextual store
  descriptors. Do not call the initializing CLI `get_read_view()`/`get_project()` path
  from this adapter.
- Reuse `ShowStoreRouter` and `resolve_show_batch()` for their established rules: local
  lookup first, full-ID fallback to the enabled owning project, shorthand scoped to the
  selected project, and ambiguity reported. Preserve explicit project pinning on indexed
  targets. Permit a foreign full-ID lookup when there is no local store without creating
  a dummy on-disk store; any small optional-local-view adaptation belongs in the
  existing router rather than a second routing algorithm.
- Render via `build_show_batch_document()` with rich style, full details, and the
  existing non-exiting `enrich_with_artifact_link_neighborhood()` callback. Retain
  canonical section identity/subject ref, bead origin, LINKS/REFERENCED BY, and
  destination project render context so subsequent bead, plan, and source-file links
  remain anchored correctly. Reuse the render-context resolver's injection points;
  extend only the thin context glue if necessary.
- Preserve lookup failures from the batch/router, including unknown IDs, ambiguous
  shorthand/project prefixes, and unavailable stores. Leave the current document and
  trail unchanged on failure. Treat unavailable/transient store failures as retryable
  using existing `LinkResolution` conventions; reload must allow another attempt.
  Ambiguity must never select an arbitrary store.
- Keep all store/config access and rendering inside the existing background resolution
  task. Retain generation checks and stale-result suppression. Following a hint must not
  initialize a store, generate pages, start a sync, clone/reset a checkout, or issue
  publication commands.

This work composes existing Rust-backed reads, canonical ID handling, and existing CLI
routing/rendering adapters. Do not implement bead-store parsing, shorthand matching, or
relationship computation in Python. No Rust wire change is expected. If an actually
missing shared domain operation is discovered, implement that operation and its binding
in `sase-core` through `/sase_repo`, with the corresponding core and binding tests,
rather than adding a Python fallback.

### 3. Add regression coverage through the public navigation path

Use hermetic temporary stores built with the real Rust-backed bead APIs. Reuse
`tests/test_bead/resolution_test_helpers.py` and existing rendering fixtures where
useful. Stub inventory, host metadata, and unrelated network/publication work only; do
not stub the public pager resolver, bead lookup, or detail builder in the main
regression tests. Do not read the developer's live stores from pytest.

- Create related root/task beads and a phase bead with no `pages/` projection.
  Demonstrate the public `resolve_link()` failure on the original implementation, then
  assert correct live documents after the fix. Include an existing-page control and a
  stale page whose bead is absent; the latter must not produce a successful live detail
  view.
- Exercise both canonical `bead:<id>` and `@bead:<id>` references, a local shorthand,
  and bare IDs normalized from a bead-origin document. Verify canonical identity and the
  expected relationship content on the landed document.
- Build a multi-section document resembling the screenshot, obtain labels from the
  rendered label layer, and press their actual hints with Textual Pilot. Cover both a
  typed relationship reference and a bare mention. Follow another bead from the landing,
  then verify back/forward restores the prior document, section/scroll state, and
  correct subject. Include a public CLI input test for `sase pager --plain bead:<id>`
  through the command handler.
- Exercise explicit document ownership when ambient cwd belongs to a different project,
  a foreign full-ID destination, and an indexed target with a project hint. A shorthand
  must never resolve to an unrelated ambient project's bead. Check that a subsequent
  source/plan link retains the landed project's context.
- Cover unknown beads, ambiguous shorthand/ownership, missing stores, and an unavailable
  store becoming available after reload. Assert meaningful failure messages and
  unchanged history on failure. Keep a sentinel around store
  initialization/materialization/publication entry points to prove keypresses do not
  call them. Preserve the existing background/stale-result tests.

Likely test locations: a focused `tests/pager/test_bead_resolution.py` and
`tests/pager/test_bead_navigation.py`, existing `tests/pager/test_resolve.py`,
`tests/main/test_pager_command.py`, and router tests only if its adapter changes. Update
the successful direct-helper tests as needed, but do not use them as the only evidence
for the fix. Keep the existing unavailable corpus examples hermetic and unavailable
rather than letting them discover a live project.

### 4. Document and verify

- Add a short explanation to `docs/pager.md`'s resolution section: bead hints open
  current store-backed details, independently of generated published pages, using the
  document's project context. No keymap, default-config, or new CLI option is required.
- Read `lint_and_test.md` via `/sase_memory_read`. Prepare the workspace's isolated
  environment with `just install` if necessary; diagnosis found that a workspace
  interpreter could not load its editable Rust extension. Do not mistake an environment
  failure for the application regression.
- Run focused resolver, real-store navigation, pager-command, and any affected bead
  router/detail tests. Then run `just check`, the mandatory agent gate. If it escalates
  or selects an unusual scope, follow the memory's rules. Run `just check-full` only
  through `/sase_monitor` when the landing rules require it. If Rust changes become
  necessary, run `sase-core`'s required full `just check` including PyO3 tests and
  rebuild the Python binding before testing callers.
- Rendering is reused, so new PNG goldens should not be necessary. If intentional visual
  changes occur, run the relevant pager snapshot suite and inspect diffs before
  accepting goldens.
- Finally reproduce the user's flow with existing `sase-xi` -> `sase-po` links and
  `sase-y3.3` as a control, using audited artifact reads for any additional artifact
  context. Confirm follow and back work without first refreshing generated pages. Record
  the focused tests and `just check` result in the completion report.

## Completion criteria

A bead visible to the relevant project's live read API opens from a pager hint and from
`sase pager bead:<id>` even if its generated page is absent. Both typed and bare
references work in the screenshot's flow. Subsequent links retain the destination
project, failures retain actionable diagnostics without corrupting history, and
navigation performs no store initialization or publication. Public-path regression tests
and required repository checks pass. Changes to publication/reconciliation, artifact
prompt expansion, keymaps, and unrelated link kinds are outside this fix.
