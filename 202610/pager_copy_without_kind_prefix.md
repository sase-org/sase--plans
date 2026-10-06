---
tier: tale
title: Pager copies never carry a kind label
goal:
  Every pager clipboard copy (y<label> and yy) yields the bare path, bead ID, SHA, or
  reference argument instead of a `file:`/`bead:`-style artifact-kind prefix.
size: small
proposed_by: bbugyi200.athena.0xe
create_time: 2026-10-06 13:40:48
status: wip
---

# Plan: Pager copies never carry a `<kind>:` label

## Problem

The pager (standalone `sase pager` and the copy embedded in ACE, which share
`PagerScreen`) often puts an artifact-kind label such as `file:` or `bead:` on the
clipboard. The user reports that this prefix is wrong every time they have seen it: they
want the path, the bead ID, or the SHA, not a canonical artifact identity. There are
three independent sources:

1. **`yy` (copy this section).** `PagerActionSectionMixin._dispatch_section_action` in
   `src/sase/pager/_screen_actions_section.py` copies `section.subject_ref` verbatim
   (`self._copy_ref(ref, label="this section")`). Subject refs are canonical identities:
   - `file:<absolute path>` for every file-backed section (`path_section` in
     `src/sase/pager/adapters.py`, directory listings in
     `src/sase/pager/_resolve_common.py`).
   - `bead:<id>` for `sase bead show` documents (`src/sase/bead/cli_show_batch.py`).
   - `result.canonical_reference` for resolved artifacts (`plan:…`, `research:…`,
     `stitch:<repo>@<sha>`, `attachment:<bead>/<name>`, `file:explicit:<hex24>` …) from
     `src/sase/pager/_resolve_artifact_refs.py`, `src/sase/pager/landings.py`, and
     `src/sase/artifact_cli/read.py`.
   - The CLI input value for media documents (`src/sase/main/pager_handler.py`).
2. **`y<label>` on an artifact-reference span.** `target_action_destination`
   (`src/sase/pager/document.py`) returns the scanner's canonical `kind:arg` destination
   (`semantic.artifact_reference`, a location-preserving `@`-stripped form, or a
   Markdown destination such as `[x](bead:sase-2)` → `bead:sase-2`).
   `copy_text_for_target` (`src/sase/pager/_resolve_file_paths.py`) only post-processes
   `FILE_PATH` spans and returns every other kind verbatim.
3. **`y<label>` on a bare-token span.** `target_resolution_ref` (`document.py`)
   _synthesizes_ `bead:<token>` for bare IDs in BEAD/AGENT-origin documents and
   `commit:<token>` for short SHAs in DIFF-origin documents. The user sees `sase-uk.5`
   painted, presses `y<label>`, and gets `bead:sase-uk.5`.

Existing tests lock the wrong behavior in and must flip:

- `tests/pager/test_app_actions.py::test_yy_copies_the_current_sections_subject_ref`
  asserts `["file:/tmp/source.py"]`.
- `tests/pager/test_resolve_targets.py::test_copy_text_for_target_returns_artifact_refs_unchanged`
  asserts `"bead:sase-uk.5"`.
- `tests/pager/_rendered_link_expected.py` declares `copy_text` values such as
  `PLAN_CYCLING`, `PLAN_PREVIOUS`, `"plan:spaced plan.md#L3"`,
  `f"{PLAN_LINE_TARGET}:12"`, `f"{PLAN_LINE_TARGET}:12-20"`,
  `f"{PLAN_LINE_TARGET}#L3C2"`, and `f"{DESIGN_LINE_TARGET}:40"` (constants from
  `tests/pager/_rendered_link_fixtures.py`).

## The Copy Rule

**Clipboard text from the pager never begins with an artifact-kind label.** For anything
that is an artifact reference, the pager copies the reference's argument exactly as
written: everything after the first `<kind>:` (and after one leading `@`), keeping any
location suffix or fragment. One exception exists, because its argument is not a
standalone identifier: digest-form indexed files.

| Copied today                            | Copied after this change                                 |
| --------------------------------------- | -------------------------------------------------------- |
| `file:/home/u/x.py`                     | `/home/u/x.py`                                           |
| `file:/home/u/x.py:12`                  | `/home/u/x.py:12`                                        |
| `bead:sase-uk.5` / `@bead:sase-uk.5`    | `sase-uk.5`                                              |
| `commit:abc1234` (bare SHA in a diff)   | `abc1234`                                                |
| `stitch:sase@abc1234`                   | `sase@abc1234`                                           |
| `agent:foo`, `patch:foo`, `tool:<id>`   | `foo`, `foo`, `<id>`                                     |
| `goal:<project>@<id>`                   | `<project>@<id>`                                         |
| `plan:202609/x.md#L3`                   | `202609/x.md#L3`                                         |
| `attachment:sase-1/a.png`               | `sase-1/a.png`                                           |
| `file:explicit:<hex24>` (digest form)   | stored file path; canonical ref only if it can't resolve |
| `/abs/path`, URLs, `#skill`, `sha:path` | unchanged                                                |

Already-correct copies stay as they are: URL copies, `FILE_PATH` copies (resolved
absolute path with `:line[:col]`), the time band's SHA copy, the diff view's
unified-diff copy, and the pinned-version `yy` copy in git `sha:path` form (documented
in `docs/memory_history.md`).

### Deciding what counts as a reference

Do **not** gate on "the Rust parser accepts it". `parse_artifact_ref` treats any
`[a-z][a-z0-9_-]*:<relative path>` as a document kind, so `abc1234:src/x.py` (the
pinned-version `sha:path` form) and `ambiguous:src/x` both parse. The compiled kind
catalog (`parsable_artifact_ref_kinds()` in `src/sase/artifact_ref_kinds.py`: `stitch`,
`patch`, `bead`, `goal`, `agent`, `file`, `attachment`, `tool`, `job`, `commit`,
`plans`, `chat`, `bug`) also leaves out project-configured document kinds such as
`plan`, `research`, and `designs`. Gate as follows:

- **Span destinations (`ARTIFACT_REF`, `BARE_TOKEN`) are references by construction.**
  The Rust document scanner painted the span only for a known kind, or the origin rule
  synthesized the label. Always strip the label, provided it matches the kind grammar
  `[a-z][a-z0-9_-]*`. Do not re-validate the whole ref: `commit:abc1234` fails a full
  parse (the `commit` alias requires `<repo>@<sha>`), but it must still copy `abc1234`.
- **`yy` subject refs come from adapters, not a scanner.** Strip only when the label is
  in the compiled catalog, `section.known_kinds`, or
  `known_kinds_from_link_context(context)` (`src/sase/pager/known_kinds.py`). Compute
  the context lookup lazily, only when the cheaper sets miss and the text looks labeled,
  because it can touch the filesystem. Anything else (plain paths, `ambiguous:` path
  text, unknown labels) copies unchanged.
- **Digest-form `file:` refs** are those where
  `parse_artifact_ref(ref).payload.type == "file"` (as opposed to `"file_path"`).
  Resolve them to the stored file path with the existing artifact resolver:
  `resolve_cli_reference` plus `resolved_file_path` (see `_resolve_artifact_result` in
  `src/sase/pager/_resolve_artifact_refs.py`), using
  `artifact_context_for_link_context(context)` from `src/sase/pager/owner.py` when a
  context exists. If resolution fails, copy the canonical ref unchanged. This is the
  only case where a label survives: `explicit:<hex24>` on its own identifies nothing,
  and the canonical form still works with `sase artifact read`.

## Changes

1. **New module `src/sase/pager/copy_text.py`** owns the rule (pure Python presentation
   glue over the Rust-owned kind catalog and parser):
   - `strip_reference_kind(ref, *, known_kinds=None) -> str`: removes one leading `@`
     plus a `<kind>:` label. When `known_kinds` is `None` (span destinations), any label
     matching the kind grammar is stripped. Otherwise the label must be in
     `parsable_artifact_ref_kinds()` or `known_kinds`. If nothing is stripped, or
     stripping would leave an empty argument, return `ref` unchanged.
   - `copy_text_for_reference(ref, *, context=None, known_kinds=None) -> str`: the
     digest `file:` branch first (resolve, else canonical), then `strip_reference_kind`.
     Import resolver modules lazily inside the digest branch to avoid a cycle with
     `sase.pager.resolve`.
   - Give the module a short docstring stating the rule ("clipboard text never carries a
     `<kind>:` label") so later callers find one owner.
2. **`copy_text_for_target`** (`src/sase/pager/_resolve_file_paths.py`): keep the
   `FILE_PATH` branch as is. Route `ARTIFACT_REF` and `BARE_TOKEN` through
   `copy_text_for_reference(ref, context=context)`, leaving `known_kinds=None` because
   they are references by construction. Every other kind (for example `MACRO_SKILL`)
   stays verbatim. Update its docstring. `_copy_target` in `_screen_actions_section.py`
   already passes `kind` and `context` and needs no change, and URL spans still bypass
   it.
3. **`yy` in `_dispatch_section_action`** (`src/sase/pager/_screen_actions_section.py`):
   - Replace `self._copy_ref(ref, label="this section")` with a `schedule_copy_delivery`
     call whose value is a callable. `deliver_copy` runs callables via
     `asyncio.to_thread`, so resolution and context lookups stay off the UI thread. The
     callable returns
     `copy_text_for_reference(ref, context=<section link context>, known_kinds=section.known_kinds)`.
     Get the context from
     `self._link_context_for_section_index(self._current_section_index())`, as the `EE`
     branch does. Keep `copied_label="this section"`, `task_name="sase-pager-copy"`, and
     `on_failure="toast"`.
   - Pinned-version branch: replace `live_ref.removeprefix("file:")` with
     `strip_reference_kind(live_ref, known_kinds=section.known_kinds)` so that only the
     bare path follows `<commit>:`. The output stays `sha:path`.
   - `_copy_ref` stays for the unified-diff and time-band SHA copies, which are already
     bare.
4. **Docs** (`docs/pager.md` key table):
   - `y<label>`: "Copy a painted link's bare value: a resolved file path, a URL, or a
     reference's argument such as a bead ID or SHA (never a `kind:` label)".
   - `yy`: "Copy the current section's path or bare reference (a bead ID rather than
     `bead:<id>`), or `sha:path` / a unified diff in memory history".
   - Run `just fmt` afterwards, since that table is width-aligned Markdown. Do not
     change the in-pager help string in `src/sase/pager/_trail_chrome_help.py`; it is
     still accurate, and editing it would churn PNG goldens.

No `sase-core` change: the kind catalog and parser are already Rust-owned, and this only
decides how the pager presents a ref on the clipboard. No feature flag: this is a
correctness fix with no old branch to keep reachable. No keymap or `default_config.yml`
change.

## Tests

- **New `tests/pager/test_copy_text.py`** (unit, no Pilot):
  - Span mode (`known_kinds=None`) strips: `file:/tmp/x.py` → `/tmp/x.py`;
    `file:/tmp/x.py:12` → `/tmp/x.py:12`; `@bead:sase-1` → `sase-1`; `bead:sase-uk.5` →
    `sase-uk.5`; `commit:abc1234` → `abc1234`; `stitch:sase@abc1234` → `sase@abc1234`;
    `plan:202609/x.md#L3` → `202609/x.md#L3`.
  - Gated mode: `plan:202609/x.md` with `known_kinds=("plan",)` → stripped, and with
    `()` → unchanged. `abc1234:src/x.py` and `ambiguous:src/x` with `()` → unchanged.
    `bead:sase-1` with `()` → stripped (compiled kind).
  - Never touched: `/tmp/x.py`, `src/x.py:12`, `https://example.test/a`, `#sase_plan`,
    `""`.
  - Digest: monkeypatch the resolver so `file:explicit:<hex24>` returns a stored path →
    that path is copied. When the resolver raises or returns no path → the canonical ref
    is copied unchanged. Assert that the stripped `explicit:<hex24>` form is never
    produced.
- **Flip the locked-in tests:**
  - Rename `test_yy_copies_the_current_sections_subject_ref` to
    `test_yy_copies_the_current_sections_path_without_a_kind_label` and expect
    `["/tmp/source.py"]`.
  - Rename `test_copy_text_for_target_returns_artifact_refs_unchanged` to
    `test_copy_text_for_target_strips_the_artifact_kind_label` and expect `"sase-uk.5"`.
  - In `tests/pager/_rendered_link_expected.py`, set every `ARTIFACT_REF` occurrence's
    `copy_text` to the argument form, e.g. `"202609/capture_line_edge_cycling.md"`,
    `"spaced plan.md#L3"`, `"202609/line_target_plan.md:12"`,
    `"202609/line_target_plan.md:12-20"`, `"202609/line_target_plan.md#L3C2"`,
    `"line_target_design.md:40"`. Leave `resolution_ref` and `identity_contains` alone:
    they are resolution identities, not clipboard text. Derive the values with a small
    test helper (e.g. `PLAN_CYCLING.partition(":")[2]`) rather than retyping literals.
    Check whether any `ExpectedOccurrence` falls back to `copy_text` for `edit_path`
    (`edit_path = occurrence.edit_path or occurrence.copy_text`); if so, give it an
    explicit `edit_path`.
- **New Pilot tests** in `tests/pager/test_app_actions.py`, using the existing
  `copy_to_system_clipboard` monkeypatch pattern:
  - `yy` on a `bead:sase-1` subject section with `kind="bead"` copies `sase-1`.
  - `y<label>` on a bare bead token in a `PagerOrigin.BEAD` document copies the visible
    token, not `bead:<token>`.
  - `y<label>` on an `@bead:sase-1` artifact-ref span copies `sase-1`.
  - If a pinned-version fixture is cheap to build from existing memory-history test
    helpers, assert that `yy` copies `<sha>:/tmp/x.py`. Otherwise cover the pinned
    expression with a direct unit assertion on `strip_reference_kind`.
- Search for other clipboard assertions that expect a label before finishing:
  `rg -n 'copied == \[.*"(file|bead|plan|commit|stitch|agent|designs):' tests` and
  `rg -n 'copy_text=' tests/pager`.

## Verification

- `sase tool run check`. This is the guarded `just check` recipe: lint plus the
  diff-scoped tests. Read the `lint_and_test` memory first and do not run `check-full`.
- Run the touched pager test files directly as a fast loop:
  `tests/pager/test_copy_text.py`, `test_app_actions.py`, `test_resolve_targets.py`,
  `test_copy_owned.py`, `test_rendered_link_contract.py`,
  `test_rendered_link_navigation.py`, `test_rendered_link_failures.py`.
- Manual sanity check, if a TTY-less Pilot run is not enough: open a file with
  `sase pager <file>`, press `yy`, and confirm the clipboard holds the bare absolute
  path. Then run `sase bead show <id>` in the pager, press `yy`, and confirm the bare
  bead ID.

## Out of Scope

- Resolving document-kind refs (`plan:`, `research:`, …) to absolute filesystem paths on
  copy. That is a separate behavior choice. This change only removes the label.
- ACE copy targets outside the pager (`src/sase/ace/tui/actions/clipboard/…`).
- Subject-ref identities themselves. `subject_ref`, section `identity`, history keys,
  and `EE`/follow resolution keep their canonical `kind:arg` form. Only clipboard text
  changes.
