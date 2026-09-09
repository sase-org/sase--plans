---
tier: epic
status: done
title: Preserve pager link identity and resolve targets in their owning repositories
goal:
  Every currently rendered pager link retains a usable action and reaches the intended
  target when that target can be identified and accessed. Preserve typed references,
  source paths, URLs, commit links, and attached targets; repair the screenshot failures
  without suppressing links or guessing another repository.
parent_bead: sase-xy
phases:
  - id: document-targets
    title: Parse document links into faithful semantic targets
    size: medium
    depends_on: []
    description:
      "document-targets: add the Rust document-link scanner and binding contract,
      separating painted text from normalized destinations and covering generated
      Markdown."
  - id: repository-resolution
    title: Resolve targets using document ownership and repository identity
    size: medium
    depends_on:
      - document-targets
    description:
      "repository-resolution: add shared provenance and repository-aware candidate
      selection, retaining owner context, bounded lookup, ambiguity, and resolution
      evidence."
  - id: pager-integration
    title: Carry semantic targets through every pager entry and action
    size: medium
    depends_on:
      - document-targets
      - repository-resolution
    description:
      "pager-integration: integrate normalized targets and provenance into pager
      construction, follow, copy, edit, commit dispatch, reload, and navigation history."
  - id: rendered-link-contract
    title: Exercise every rendered link through real pager navigation
    size: medium
    depends_on:
      - pager-integration
    description:
      "rendered-link-contract: build the screenshot regression corpus and real
      scanner-to-action contract tests, close remaining integration defects, and
      document behavior."
proposed_by: bbugyi200.athena.03o--1
bead_id: sase-xy.5
create_time: 2026-09-09 19:52:34
---

- **PROMPT:**
  [prompts/202609/pager_target_integrity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/pager_target_integrity.md)
- **BEAD:**
  [sase-xy.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xy/sase-xy.5.md)

# Preserve pager target identity

## Evidence and root causes

The user supplied `.sase/artifacts/pool/0f744356aa04-file-ref.png` and
`.sase/artifacts/pool/cebc78d3792e-file-ref.png`. The first shows a generated plan's
Links table with `[plan:202609/capture_line_edge_cycling.md][2]`, and a toast reporting
`202609/capture_line_edge_cycling.md not found (searched 4 locations)`. The second
reports the same failure for `Sources/BobMacCapture/CaptureKeyCommandRouter.swift`. Both
are valid targets:

- Audited reads of `plan:202609/capture_line_edge_cycling.md` and
  `plan:202609/capture_ctrl_u_previous_line.md` succeeded from the opened bob-cli
  project checkout. Both plans identify bob-mac-capture as their implementation repo.
- The bob-cli configuration declares bob-mac-capture as a linked repository. An
  authorized checkout of `gh:bobs-org/bob-mac-capture` contains the router file, the
  adjacent controller, and the tests named in the screenshots. Git history shows the
  router existed before the line-above shortcut change, so this is not merely a
  not-yet-created planned file.
- Resolving the router path with only the bob-cli checkout as an anchor returns `None`;
  resolving it with bob-mac-capture as the anchor succeeds.

The source and direct Python reproductions establish these defects:

| Input or path through the code                  | Current behavior                                                | Cause                                                                                                                                 |
| ----------------------------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `[plan:202609/capture_line_edge_cycling.md][2]` | Scans only `202609/capture_line_edge_cycling.md` as `file_path` | `pager/link_scan.py` uses the prompt-only Rust scanner, which starts candidates only at `@`                                           |
| `@plan:202609/example.md`                       | A typed span is passed to a parser that rejects the leading `@` | `document.target_resolution_ref()` returns display text instead of the scanner's semantic `reference`                                 |
| `[@plan:202609/x.md][1]`                        | Scanned target includes `][1]`                                  | Prompt token boundaries do not model Markdown link syntax                                                                             |
| `@plan:"a b.md"`                                | Quotes remain part of the dispatched path                       | The scanner's decoded reference is discarded                                                                                          |
| `@src/sase/pager/resolve.py`                    | Link is painted, but the file resolver misses an existing file  | The display sigil is treated as part of the filename                                                                                  |
| `@src/sase/pager/resolve.py:42`                 | The file link disappears from scanning                          | A malformed artifact candidate occupies the entire valid file token                                                                   |
| A short SHA in `PagerOrigin.DIFF`               | A label is painted, but dispatch returns `None`                 | Bare-token dispatch implements only bead-origin tokens                                                                                |
| A source path in a linked repo                  | Direct and suffix searches miss it                              | Neither search enumerates the document owner's linked repo roots; the parent Git index does not enumerate nested ignored repositories |
| A resolved Swift source file                    | Opens a binary metadata card instead of readable source         | `_is_probably_text()` relies on MIME/suffix allowlists; this host has no MIME entry for `.swift`                                      |

Provenance is also incomplete after the previous changes:

- `path_section()` retains a parent directory, but no owning project/repository or
  canonical artifact identity. A sidecar month's directory is not its project root.
- `artifact_cli/read.py::_pager_link_context()` uses indexed-file workspace metadata
  when present, then defaults to the viewer's cwd; ordinary document refs often have
  `result.file is None` even though `result.context` and the resolved path are known.
- `_resolve_artifact_result()` opens a file using `path_section()`, losing the canonical
  reference and its resolved artifact context. An indexed target's producer workspace is
  not consistently carried into this landing.
- `main/pager_handler.py::_build_pager_document()` flattens input documents into
  sections and assigns one default document context and combined origin. Input-specific
  context/origin can be discarded. ACE's link-index fast path also bypasses parts of
  context-aware resolution.

Existing tests validate these stages mostly in isolation; for example, the basic
follow-key test replaces the real resolver with a function that always returns a
document. A correct label and a correct manually supplied canonical ref can therefore
pass their respective tests while the painted label cannot actually be followed.

## Relationship to existing work and tier

This is an epic because shared Rust semantics, repository provenance, Python/Textual
integration, and a cross-entry regression corpus are substantial, separately bounded
changes. Each phase is direct implementation work of size medium. Dependencies are
deliberately sequential where files or wire contracts overlap.

The audited predecessor is `plan:202609/pager_link_reliability.md` (sase-xy). At
investigation time its three original phases were closed but the epic remained open. Its
child sase-xy.4, `plan:202609/pager_link_landing_corrections.md`, owns duplicate
dead-end searches, bounded Git output capture, pure context merging, off-thread ACE
preparation, and workspace-number-aware failure keys. Those fixes are prerequisites for
Python integration here. Phase 3 must incorporate the landed correction child before
editing its affected paths; if it is still active, use the normal SASE
dependency/handoff mechanism rather than racing its workers. Recheck the live status
before implementation. Do not duplicate that child's implementation or close its beads.
Associate this epic with sase-xy through `parent_bead` and leave parent completion to
its land agent.

The original plan explicitly excluded bare-token changes and some raw-file provenance;
it also assumed extra path anchors alone could repair the problem. This plan covers the
missing target semantics and repository identity required by the current request.

## Behavioral contract

1. Preserve every existing link occurrence and supported action. Repair or expand a
   mistakenly partial token into its full semantic link; this is not removal. Keep
   hosted URL occurrences in generated reference definitions as well as their typed
   reference occurrences. Keep links in code/prose contexts currently scanned. Do not
   make tests pass by disabling scanning, hiding dangling links, or deleting generated
   Links tables. Preserve `--links never` as the explicit user-controlled opt-out.
2. Separate the exact visible span from its semantic target. Painting operates on
   original character offsets; resolution receives a normalized artifact ref, path, URL,
   fragment, or caller-owned target. Prompt `@` and quoting are syntax, not path bytes.
   Do not strip a literal `@` from a quoted or explicitly literal filename.
3. Resolve the author's target in the owning repository. A same-named file in an
   unrelated cwd is not a successful resolution. Directory order remains meaningful
   within the same repository; distinct repositories are not interchangeable fallback
   directories. Equally plausible targets produce an ambiguity result with concrete
   candidate links the user can follow, not a first-hit guess.
4. URLs retain the documented copy-on-follow behavior. Copy the actual destination,
   without accidental Markdown delimiters or sentence punctuation. Keep explicit URL
   path punctuation, queries, fragments, and balanced parentheses intact. This work does
   not change URL activation into browser launch or automatic network validation.
5. Follow, copy, edit, and current-section actions use the same target/context
   interpretation. Copy cannot invent a nonexistent cwd-joined path and present it as
   resolved. For unavailable paths, copy the original logical token or an authoritative
   canonical reference with an honest status. Edit reports unavailable when the target
   has no editable file, preserving meaningful follow/copy actions.
6. Preserve line/column suffixes, artifact line fragments, media dispatch, directory
   navigation, attached commit handlers, labels, search, back/forward state, and bounded
   resolution. A line or heading fragment must land at its target or explicitly report
   that the fragment is unavailable; silently dropping it is not success.
7. A missing checkout, unavailable revision, ambiguous name, denied/filtered target,
   temporary lookup error, and proven missing target are distinct outcomes. Retain their
   links and explain the outcome. A temporary failure is retryable; reload must
   invalidate stale failures. An impossible target may be non-actionable only when its
   reason is established, and its text remains visible. A failed search is not proof
   that linking is impossible.
8. Every lookup, inventory read, stat, materialization, and subprocess stays off the
   event loop and serial message pump. Paint and key matching use prepared immutable
   data. Preserve generation checks and cancellation; each request searches once and
   returns diagnostics with its result. Never clone/reset/clean workspaces on a label
   press or scan the whole filesystem/all unrelated projects as a fallback.

## Phase 1: Parse document links into faithful semantic targets

Open `sase-core` with `/sase_repo` and use only its returned checkout path. Shared
grammar, reference normalization, and link-target interpretation belong in Rust core,
with a versioned PyO3 binding and a thin Python adapter. Textual label allocation and
character-offset conversion remain in sase. Do not copy the grammar into Python or
weaken prompt parsing globally to accommodate document display.

Relevant sources are Rust `artifact_ref/scanner.rs`, `artifact_ref/mod.rs`,
`artifact_ref/wire.rs`, `markdown_link_refs.rs`, `artifact_link/managed_table.rs`, and
the PyO3 exports; Python `artifact_ref_operations.py`, `pager/link_scan.py`, and
`pager/document.py` are their consuming seams. Reuse the existing Markdown reference
definition machinery and canonical artifact parser; extend them narrowly where needed.

Add a document-oriented scan API, separate from the unchanged prompt scanner contract.
Its versioned result must carry source byte spans, visible text, target category,
normalized destination, and any explicit Markdown destination/fragment needed for
interpretation. It must support sigiled and unsigiled typed refs, configured document
kinds and aliases, quoted payloads, generated reference-style links, inline links, and
the existing file/URL recognition surface. Keep code-zone discovery where the pager
already offers links; the publisher's rewrite exclusions are not pager exclusions.

For generated managed-table links, retain the canonical typed artifact identity from the
label and the associated hosted destination as repository provenance. Resolve a
consistent local artifact when available, while leaving the separately rendered URL
usable. For ordinary Markdown links, the declared destination controls the target; never
substitute unrelated label text. Recognizing SASE-generated tables must not require
changing old archived artifacts or their publishers. Validate agreement before using a
hosted destination to qualify a typed ref. An ordinary URL is not authorization to fetch
a repository.

Use explicit precedence and overlap rules so an artifact ref is not split into a file
path and a URL does not sprout inner file links. Handle Markdown brackets and escapes
without swallowing adjacent reference labels. Decode quotes from the scanner's semantic
reference, not by loosely stripping all quotes. Preserve existing valid `@path` file
syntax, including line suffixes, without letting a malformed artifact candidate mask it.
Handle Unicode byte-to-character conversion and original spans after normalization.

Deliver Rust and binding tests plus Python contract tests for every scanner reproduction
above, both screenshot table rows, repeated refs, punctuation, UTF-8, quoted spaces,
aliases/custom kinds, unknown/malformed kinds, URLs containing paths, and literal
filenames containing punctuation. Confirm existing prompt scanner fixtures are
unchanged.

## Phase 2: Resolve targets using ownership and repository identity

Extend the shared target/context contract to retain source canonical reference, owner
project key, repository identity and revision when known, source file directory, and
ordered workspace/checkout candidates. Keep enough provenance even if a recorded
workspace directory has been removed. Define resolution outcomes that include the
selected target, effective landing context, candidate evidence, failure category, and
retryability; do not collapse everything into `None` before the UI sees the reason.

Python may collect existing host metadata and adapt it to the Rust wire. Selection,
normalization, identity/ambiguity rules, and shared repository/path lookup policy belong
in Rust core. Reuse the repository inventory and marker/sidecar ownership facilities,
`ArtifactRefContext.repositories` and document roots, artifact-file provenance, and the
existing artifact resolver; avoid a new parallel repository registry. Preserve existing
permission/filter outcomes instead of falling through to unrestricted path lookup. Use
the project's installed bindings without a Python fallback.

Resolve in this order, with repository identity enforced at every stage:

- A caller-attached concrete target or explicit canonical repository-qualified target.
- The target's known document/artifact owner and source-relative directory, then its
  owning checkout root. Typed document refs use the appropriate kind's sidecar root, not
  the file's month directory or viewer cwd.
- For an unqualified source path, the owning project's already available linked repos.
  Use producer repo/open-log metadata where available; otherwise search full relative
  paths across owner-scoped repositories and require a unique repository match.
- Other available checkouts of that same identified repository, including its primary,
  when the recorded workspace is absent or a stale absolute path must be re-rooted.
- The existing bounded unique-suffix search inside those repositories. Deduplicate
  copies by logical repository/path identity before deciding whether distinct targets
  are ambiguous. Cache inventory/index reads within the request and enforce a total work
  budget as well as per-process bounds.

For input with no recoverable owner, preserve ordinary cwd-relative behavior. Never
silently switch an explicitly identified target to another project's same-named file. An
authoritative Markdown hosted destination may identify a configured local sidecar and
exact path/revision; only map it locally when repository and revision semantics match.
Otherwise retain its usable URL action. Do not invent a GitHub branch or treat a
different local revision as exact.

Prove the screenshot with fixtures containing a primary bob-cli-like repo, its plans
sidecar with a month subdirectory, and a separate linked capture-like repo containing
the router/controller/test files. None may be accidentally available under test cwd. Add
unrelated same-name files, two ambiguous linked repos, a deleted producer workspace, a
live alternate checkout, an absent checkout, and denied/filtered refs. Recover a
provable historical/indexed target through existing artifact materialization where
available; do not guess renames or perform an unbounded history search.

## Phase 3: Carry targets through every pager entry and action

First incorporate sase-xy.4's completed correction work. Preserve its single-attempt
result, bounded process capture, pure context merge, and off-thread preparation APIs.
Extend them rather than reviving the older helper that reruns resolution for a toast.

Adapt the pager model to carry semantic targets and per-section origin/owner context.
Keep the existing public convenience entry points usable and update injected resolver
protocols and attached-handler adapters consistently. In `target_resolution_ref()` or
its replacement, consume normalized target data, including decoded artifact references
without `@`, rather than reparsing display text. A custom attached target continues to
go to its registered handler and retains its object identity.

Wire the context through all construction/landing paths:

- `pager/adapters.py`: path and multi-path inputs, with source parent, containing
  checkout/sidecar owner, and any existing artifact provenance recovered off-thread.
- `artifact_cli/read.py`: keep `result.context`, canonical identity, resolved file
  location and producer metadata even for document refs with no `result.file` row.
- `pager/resolve.py`: ordinary files, typed documents, indexed/VCS-backed files,
  directories, beads, entity cards, and media. Followed documents inherit the landed
  target's owner before any parent defaults; `subject_ref` retains canonical identity.
- `main/pager_handler.py`: combined inputs preserve each section's effective context and
  origin rather than replacing them with one default.
- ACE hints/view construction and `link_target_for_artifact_entry_target()`: retain
  context on the link-index fast path as well as resolver fallback. Materialized reports
  must not become contextless temporary files.
- Bead-show sections and generic `page_or_print()`/stdin: preserve available origins and
  context; unowned stdin retains the bounded cwd fallback.

Implement the existing diff-origin SHA links using their repository context and the
existing commit/stitch resolution and view facilities. A successful commit action must
show identifiable commit content/details, not only a generic card linking to itself.
Preserve attached `CommitViewSpec` follow/copy/edit behavior. An ambiguous abbreviated
SHA is reported explicitly; unrelated hexadecimal prose must not acquire a commit
interpretation outside the existing diff-origin rule.

Use the same normalized target for follow, copy, edit, `yy`, and `EE`. Preserve file
line and column information through editor dispatch, and use the pager's layout to land
on the requested line/fragment. Retain usable URL actions with exact destinations. Use
shared bounded content classification to open readable source such as `.swift`, `.rs`,
and `.go` as text even when MIME lookup is absent or misleading. Keep known
image/video/PDF dispatch ahead of text fallback and do not decode binary files as prose.
Unknown non-file entity types should use their available structured payload/card and
meaningful actions rather than attempting to open the label as a file.

Include owner/repo/revision, normalized destination, and workspace information in
failure/cache identity. Keep failure diagnostics computed by the worker. Reload and an
explicit retry must allow a previously unavailable target to resolve; temporary errors
must not permanently poison a label. Back/forward restores the entire section context,
search, and scroll state. Prevent stale worker results from replacing a newer document.

## Phase 4: Enforce the rendered-link contract

Create a deterministic fixture corpus from the screenshot text rather than depending on
private live repositories or on the prompt's staging-image paths. Include the exact two
plan refs and three capture source paths from the screenshots. Add typed bead, agent,
Patch, stitch/commit, indexed file, configured document kind, directory, media, URL,
quoted/Unicode path, line/column and supported fragment cases; both sigiled and
unsigiled references; explicit Markdown destinations; and caller-attached targets.

Build the document through real production adapters, allocate real labels, press those
labels with Textual Pilot, and exercise the real scanner, semantic target normalization,
resolver, and landing adapter. Fixtures may replace inventory/store roots, clipboard,
external editor/media launch, and network boundaries. Do not replace the resolver with
an unconditional successful `LinkTarget` in these regression tests. Expected targets
must be independently declared so scanner omissions cannot silently shrink the test set.

For every expected rendered occurrence, assert the destination's canonical identity,
actual file/body or entity payload, owner and line/fragment, or exact copied URL. Assert
each supported follow/copy/edit action. Assert both the canonical plan links and the
separately rendered hosted URL definitions remain available. Check section isolation
using the same text in two projects, mixed-origin input, navigation several hops deep,
media return, reload/retry after target creation, failure then success in a different
context, and back/forward restoration. Keep the intentional label-capacity window
behavior and `--links never` tests.

For impossible/unavailable targets, independently assert why resolution is impossible or
presently unavailable, that useful remaining actions persist, and that failed navigation
leaves the current document/trail untouched. Assert no I/O on the event loop, bounded
requests, no duplicate searches, no interactive subprocess prompts, and no checkout
allocation/cleanup. A source path that exists only in a linked repo must resolve, even
when the main repo's Git index has no matching entry.

Add or update focused files under `tests/pager/`,
`tests/ace/tui/actions/test_view_files_pager.py`, `tests/main/test_pager_command.py`,
the artifact-read tests, and bead-show pager tests. Keep files within the project's size
limits. Update `docs/pager.md` to describe real resolution, copy/edit, ambiguity, and
retry semantics. Do not change keymaps or defaults unless required; any such change must
also update help and `src/sase/default_config.yml`.

## Verification and landing

Planning baseline: monitored `just check` (`720ecjzxc361`, 2026-09-07) exited 1.
Formatting, lint gates, and SASE validation passed; pytest reported 39,137 passed, 14
skipped, and these three failures on the unchanged tracked source tree:

- `tests/test_xprompt_directive_completion_parity.py::test_ace_and_lsp_include_dispatch_directive_when_enabled`:
  ACE offered the enabled dispatch directives, but the LSP omitted them.
- `tests/test_xprompt_directive_completion_parity.py::test_ace_and_lsp_include_dispatch_machine_rows_when_enabled`:
  the LSP returned no dispatch machine rows while ACE returned the fixture machine.
- `tests/main/test_parser_machine.py::test_machine_help_renders_sorted_subcommands_and_defaults_to_list`:
  rendered help included `attention`, which the expected subcommand string omitted.

These failures concern completion parity and machine-command help, outside this pager
repair. They were inspected but not repaired or classified as flakes. The core-floor
probe also reported 12 missing capabilities in `sase-core-rs==0.32.34`, including
capabilities without a containing release tag. This baseline does not establish a
passing repository check; plan-schema validation is a separate result. No implementation
or tracked source files were changed during planning.

- Every implementation phase reads `lint_and_test.md` with `/sase_memory_read`, runs
  `just install` when needed for its ephemeral checkout, and runs `just check` after
  sase changes. Use `/sase_monitor` if verification is long-running.
- Changes to sase-core require its full `just check`/`scripts/check.sh`, including PyO3
  binding tests; `cargo test -p sase_core` alone is insufficient. Rebuild/install the
  binding into the sase development environment before Python integration tests. Follow
  its release-plz version ownership; do not manually bump crate release versions.
- Run the focused behavioral suites in each phase. Phase 4 runs `just test-visual`
  including the pager snapshots, inspecting actual/expected/diff images before accepting
  intentional changes. Extra/corrected labels may change snapshots; fewer available
  semantic targets are a regression unless an impossible target is documented.
- The land agent verifies the combined Python/Rust tree and uses `/sase_monitor` with
  `TESTING`/`TESTED` for `just check-full`. Confirm the active correction child has
  landed, required binding symbols are available, and no upstream changes were lost.
- Release success requires the screenshot fixture and every row of the rendered-target
  corpus to pass through real navigation. Report any genuinely impossible targets
  specifically, rather than claiming that all links work from a scanner-only suite.

No implementation, artifact-page rewrite, memory edit, or target-repository source
change is authorized by the planning turn. The scratch plan is the only authored file
before proposal. The bob-cli and bob-mac-capture repositories are investigation context;
the repair belongs in sase and its shared Rust core.
