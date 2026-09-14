---
tier: tale
title: Follow xprompt skill references in the pager
goal:
  Pager hints for slash skills and explicit skill xprompts open the correct canonical
  skill source, with consistent navigation and useful failures.
size: medium
proposed_by: bbugyi200.athena.0ky.w0
create_time: 2026-09-14 16:28:48
status: wip
---

# Follow xprompt skill references in the pager

## Scope and size

Implement this as one medium tale across `sase-core` and `sase`. The defect is
reproduced, the shared definition/catalog machinery already exists, and the Rust
contract, Python adapter, and navigation tests form one bounded change. A single coding
agent should keep those pieces consistent. This planning turn changes no implementation
files; implementation begins after plan approval.

Open `sase-core` with `sase repo open sase-core` and use the returned checkout. Paths
below beginning with `crates/` are relative to that repository. Other source, test, and
documentation paths are relative to `sase`. Read each repository's agent instructions
and required reference memories before implementing.

## Confirmed cause and existing behavior

The following probes against the workspace virtualenv reproduce the failure:

| Input to `scan_links(..., PagerOrigin.FILE)` | Current result                                  |
| -------------------------------------------- | ----------------------------------------------- |
| `Use /sase_plan now.`                        | One `file_path` span targeting `/sase_plan`     |
| ``Use `/sase_plan` now.``                    | The same file-path target                       |
| `[plan](/sase_plan)`                         | One file-path span for the entire Markdown link |
| `Use #skill/sase_plan now.`                  | No span                                         |
| `Use #sase/skill/demo now.`                  | No span                                         |

`resolve_link('/sase_plan', context=...)` returns no target and the diagnostic
`/sase_plan not found (searched 1 locations)`. Meanwhile,
`load_skills_from_package()['skill/sase_plan']` has `skill_name='sase_plan'` and points
to `src/sase/xprompts/skills/sase_plan.md`.

The relevant flow is:

1. Rust's `crates/sase_core/src/artifact_ref/scanner.rs` recognizes slash skills
   incidentally through `scan_document_file_paths`. It has no skill-link branch.
2. `src/sase/pager/link_scan.py` converts Rust byte spans to character spans;
   `document.py` preserves the destination rather than using visible link text.
3. `src/sase/pager/resolve.py` tries artifact parsing, then file-path resolution.
   Neither path looks up skill definitions.
4. `_screen_actions.py` resolves on a background worker and applies a `LinkResolution`;
   successful documents already support follow, edit, and history.

Reuse the existing skill contracts rather than inferring filenames:

- `src/sase/xprompt/loader_skills.py` and Rust `content_layout.rs` require a canonical
  skill source and a truthy `skill` declaration. The provider name is `skill_name`;
  canonical xprompt names are `skill/<name>` and `<project>/skill/<name>`.
- Rust `xprompt_catalog.rs` already loads canonical skill sources, tracks project
  context, and exposes concrete `definition_path` separately from display paths.
- Rust `editor/definition.rs` already maps slash names through `skill_name` and explicit
  references through the canonical name. Factor reusable selection from this area where
  appropriate; editor token extraction is not a document scanner.
- Generated provider `SKILL.md` files are outputs. The landing is the canonical Markdown
  definition, including its frontmatter and template text.

## User-visible contract

| Reference or condition                                                         | Required behavior                                                                          |
| ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| `/sase_plan` in prose or inline code                                           | Follow the discovered canonical skill source when no real absolute path owns this spelling |
| `#skill/sase_plan`                                                             | Hint and follow the exact skill xprompt                                                    |
| `#project/skill/name`                                                          | Resolve that enabled project's skill with its normal source precedence                     |
| Canonical `__` namespace shorthand                                             | Normalize as existing xprompt naming does, including `#skill__sase_plan`                   |
| Markdown inline/reference link targeting a supported skill spelling            | Hint the whole visible link and follow its destination                                     |
| Existing `/tmp`, `/etc`, or `/name` file/directory                             | Preserve file/directory navigation, including a collision with a skill named `name`        |
| `/usr/bin/python`, `./name`, `~/name`, `@/name`, or an explicit `file:` target | Preserve path/artifact semantics; do not reinterpret them as skill invocations             |
| Missing explicit skill or unavailable source                                   | Name the skill and failure in the existing toast, leaving document and trail intact        |
| Multiple distinct visible skill references with the same slash name            | Explain the ambiguity and list qualified `#.../skill/...` alternatives                     |

Apply ordinary discovery precedence to definitions of the same canonical reference
before deciding slash-name ambiguity. Do not choose a source by directory order,
alphabetical catalog order, or a `<name>.md` filename guess. Restrict unqualified slash
lookup to the document's applicable project and global scopes, excluding unrelated
projects. Explicit qualification identifies the requested project.

Use the existing pager label keys, `E` edit prefix, `y` copy prefix, history, and
refresh. `E` opens the canonical source via its concrete path. `y` copies the skill
reference destination, not its expansion or an invented absolute source path; ordinary
existing paths retain their current copy behavior. A landed source uses normal
file-section copy/edit behavior. URLs continue copying their URL destination.

This change is source browsing: never expand arguments, render Jinja, execute a
workflow, record skill invocation, generate provider files, or deploy skills when
following a hint. General non-skill `#xprompt` navigation, provider-only installed
skills without a SASE source, new CLI options, and new keymaps are outside scope.

## Implementation

### 1. Add the shared skill-reference and definition contract in Rust

Add a small reusable core module for parsing a skill reference and resolving its
definition, backed by the existing catalog and content-layout rules. Keep shared
grammar, reference selection, ambiguity decisions, and source validation in Rust.

The lookup request needs the authored reference, document project/root context, and
package/plugin resource locations. Return a versioned typed outcome with status,
canonical reference, concrete source path when available, and diagnostic or ambiguity
candidates. Distinguish not-a-skill-candidate, missing skill, missing source, ambiguity,
and catalog/load failure so the adapter can preserve fallback and retry behavior. Reuse
existing wire types where they already express this.

Use `xprompt_catalog.rs`'s canonical source discovery and definition-path mapping.
Expose a narrow source lookup through `crates/sase_core_py`, with binding tests. Supply
resource paths as explicit per-call options from Python package/plugin discovery; factor
the existing catalog options only as much as needed. The native catalog currently gets
these paths from LSP environment variables, so simply calling it from the pager without
supplying them would omit packaged/plugin skills. Do not mutate process environment or
call the LSP environment-preparation routine, which also materializes unrelated
catalogs. Honor disabled plugins and current home/chezmoi layout through existing
configuration adapters.

Resolve against the supplied document workspace without changing process cwd. Retain
alternate-workspace overrides and project alias/lifecycle rules. An explicit project
reference can use its already-available registered workspace; an unavailable workspace
produces a diagnostic, never an automatic clone or workspace reset. Use a concrete
definition path, not `source_path_display` or a `plugin:` string.

### 2. Extend document scanning without breaking path recognition

In the shared Rust document scanner, add explicit canonical skill-xprompt recognition
and a document target kind such as `xprompt_skill`. This is a document link
classification, not a new persisted artifact kind. Carry it through the Rust wire, PyO3
serialization, Python `artifact_ref_models`, and `LinkSpanKind`; update schema
validation and affected fixtures together according to repository policy.

Recognize `#skill/<name>` and `#<project>/skill/<name>` plus existing namespace
shorthand. Match the reference token only, stopping before invocation arguments; viewing
a reference with arguments must never evaluate them. Preserve UTF-8 spans, punctuation
boundaries, Markdown destination/label metadata, URL precedence, and nonoverlap. Apply
the same classification to inline and reference-style Markdown destinations. Do not
infer a skill from the visible label when its destination is a URL or file. Keep the
pager's existing document-code/literal scanning policy.

Keep ambiguous slash spellings lexically classified as file paths: `/tmp` and
`/sase_plan` cannot be distinguished without lookup. The press-time core parser
identifies eligible single-segment slash tokens. Do not load catalogs, inspect the
filesystem, or resolve sources to paint hints. Audit all consumers of the document scan
enum so the new kind cannot raise in a non-pager reader.

### 3. Adapt shared results into existing pager targets

Add a thin validated Python facade under `src/sase/core/` and a focused pager adapter,
for example `src/sase/pager/_resolve_skills.py`. Reuse `file_link_target` for successful
source landings, including Markdown syntax, source ownership, relative-link anchors, and
`edit_path`. Keep all lookup and source I/O on the existing worker path.

In `resolve.py`, route explicit skill references before artifact/location parsing: the
leading `#` is skill syntax, not a file fragment. For a slash candidate, preserve normal
existing absolute-path resolution first, then attempt skill lookup only on a genuine
missing-path result. Do not override a denied, ambiguous, or unreadable real-path result
with a skill. Preserve ordinary missing-path handling when no skill matches, with enough
context to explain the attempted slash lookup.

Pass the originating section's merged `LinkResolutionContext` and owner identity to
lookup. An ownerless CLI/stdin document uses the existing default context. Do not let
the pager process's cwd or another section's project decide a match. Use the existing
per-context dangling cache and refresh invalidation; do not add a global catalog cache.
Ensure late worker results still obey document/generation guards. Successful skill
sources must not be reduced to the catalog's truncated content preview.

Teach `document.py` and the action/copy mapping about the explicit skill kind while
preserving the Markdown destination and stable cache identity. Slash-copy behavior
already retains an unresolved logical token; ensure following it as a skill does not
change that contract. Preserve enough original target metadata through worker dispatch
to distinguish an explicit `@/name` path from a slash candidate: today normalization
strips `@`, so the normalized string alone cannot enforce this distinction. Keep
injected host resolver compatibility, using a narrow dispatch adapter if necessary, and
cover both spellings in integration tests. In the CLI, make the same distinction before
normalizing the original input.

Reuse the same resolver for direct CLI inputs such as `sase pager /sase_plan` and quoted
`sase pager '#skill/sase_plan'`, including plain output. Standalone and ACE-embedded
pager hosts must exercise the same path.

### 4. Document and verify the behavior

Update `docs/pager.md` with supported skill spellings, canonical-source landings, path
precedence, qualified ambiguity resolution, and copy/edit examples. Update in-app help
only where it enumerates supported target behavior. No generated skill files or default
keymap changes should be needed.

## Regression coverage and acceptance

Use isolated temporary home/project/plugin sources, with distinctive bodies to assert
the selected definition. Do not rely on the developer's installed skills.

1. **Rust scanner and binding:** supported spellings, namespace shorthand,
   punctuation/backticks, Unicode offsets, invocation arguments, inline/reference
   Markdown destinations, and no duplicate spans inside URLs or file paths. Verify
   serialization and Python rejection of unsupported schema values.
2. **Core lookup:** home, current project, project-specific home, package, and plugin
   skills; disabled plugins; nonstandard filenames with declared names; duplicate
   canonical-reference precedence; slash ambiguity; explicit project selection;
   alternate checkout precedence; missing/unreadable definitions; rejected
   non-skill/misplaced definitions. Assert Rust/Python catalog parity for the fixtures
   that cross the adapter boundary.
3. **Pager resolution:** `/sase_plan` and `#skill/sase_plan` reach the same complete
   source. A real absolute path wins, ordinary multi-segment/explicit paths do not enter
   skill lookup, and missing/ambiguous results preserve document/history. Assert source
   edit paths, copied reference destinations, Markdown labels, and correct project
   selection when viewer cwd differs from document ownership.
4. **Headless interaction:** extend `tests/pager/test_rendered_link_navigation.py` or a
   focused sibling to press actual painted labels using the real resolver and
   fixture-backed catalog. Follow a nested relative source link, go back and forward,
   and verify section identity/scroll. Exercise a missing skill followed by refresh and
   retry, plus the same slash token in two differently owned sections. Verify slow
   lookup runs off the UI thread and a late result cannot replace a newer navigation.
   Include an embedded-pager contract test and CLI plain-output coverage in the existing
   pager command suites.

Relevant existing suites include `tests/pager/test_link_scan.py`,
`test_target_resolution_ref.py`, `test_resolve_targets.py`, `test_app_dangling.py`, the
rendered-link suites, `tests/main/test_pager_command.py`,
`tests/ace/tui/actions/test_view_files_pager_contract.py`, and
`tests/test_xprompt_skill_sources.py`. Add focused tests beside these rather than
duplicating the entire rendering corpus or introducing unnecessary PNG goldens.

Read `lint_and_test.md` through `sase memory read`. In `sase-core`, run its `just check`
or `scripts/check.sh`, which includes the PyO3 crate; `cargo test -p sase_core` alone is
insufficient. Build/install the changed binding into the isolated SASE virtualenv using
the existing `Justfile` Rust install target, explicitly pointing `sase_core_dir` to the
checkout opened through `sase repo open`, before Python integration tests. Then run the
focused Python integration suites and `just check` in `sase`. Use the repository test
runner for focused suites and review diff-scoped test selection so cross-repository
binding changes do not silently omit pager tests. Use `/sase_monitor` for long
verification; `just check-full`, if required by selection escalation or landing policy,
must run through that skill.

Coordinate the core revision/binding dependency with Python consumers through the normal
host-owned landing flow. Do not add a Python fallback or manually alter Rust release
versions. The change is complete when every supported skill spelling opens the intended
source through a real pager hint, ordinary paths retain their behavior, and both
repositories' required checks pass.
