---
tier: epic
title: Recognize glossary terms that wrap across a line break
goal:
  A multiword glossary term stays recognized when a line break falls between its words,
  so it is highlighted, previewable, and jumpable in ACE and in LSP-backed editors
  exactly as it is when it fits on one line.
phases:
  - id: core
    title: Match phrases across one line break and expose per-line span segments
    depends_on: []
    size: medium
    description:
      "core: allow a single line break inside a multiword alias in the Rust matcher, add
      core-computed per-line display segments to the glossary span wire, and emit one
      LSP semantic token per segment."
  - id: core-release
    title: Publish a sase-core-rs release containing the matcher change
    depends_on:
      - core
    size: small
    description:
      "core-release: land the release-plz version bump for sase-core, record the
      published version, and confirm the wheel resolves from PyPI."
  - id: python
    title: Consume segments in ACE, guard the floor, and correct the docs
    depends_on:
      - core-release
    size: medium
    description:
      "python: move the sase-core-rs window, carry span segments through the facade into
      the prompt highlight map, normalize wrapped matched text in the preview card, add
      a published-minimum smoke gate, and fix the docs that promise line-local matching."
proposed_by: bbugyi200.athena.ws
create_time: 2026-08-09 15:53:19
status: wip
---

- **PROMPT:**
  [prompts/202608/glossary_line_break_matching.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/glossary_line_break_matching.md)

# Recognize glossary terms that wrap across a line break

## Problem

A hard-wrapped ACE prompt splits a multiword glossary term across two buffer lines, and
the term stops being recognized. In the reported case the prompt text is

```text
- Let's start describing all sase sizes in a new sase/memory/sase_sizes.md xprompt
  memory file that gets created automatically by the `sase memory init` command.
```

`xprompt memory` is the configured term `Xprompt Memory`, but only the bare word
`xprompt` is underlined: the shorter `Xprompt` entry wins because the longer phrase
never matched at all. The same text typed on one line highlights the full phrase.

Confirmed against the installed matcher:

```python
cat = compile_glossary_catalog([
    {"term": "Xprompt", "definition": "d1", "aliases": []},
    {"term": "Xprompt Memory", "definition": "d2", "aliases": ["sase memory file"]},
])
cat.scan("...sase_sizes.md xprompt\n  memory file that gets created")
# -> [Xprompt, matched_text='xprompt']
cat.scan("an xprompt memory file here")
# -> [Xprompt Memory, matched_text='xprompt memory']
```

This is not a niche case. Prose in sase is wrapped at 88 columns by the repo formatter,
prompts are wrapped by hand for readability, and the glossary is full of two- and
three-word terms (`Agent Instruction File`, `Sase Workspace`, `Xprompt Workflow`). Any
of them silently loses its affordance whenever the wrap lands in the wrong place, and
the failure is invisible: the reader sees a shorter term highlighted, or nothing, and
has no signal that a definition exists.

The requested behavior: a line break between the words of a term is treated like any
other run of whitespace between them.

## Current behavior

Matching is Rust-owned in the sibling `sase-core` repo and is shared by both frontends:
the ACE prompt widget compiles the catalog through `sase_core_rs`, and the xprompt LSP
compiles the same catalog from the launcher-materialized `glossary_catalog.json`.

- `crates/sase_core/src/glossary.rs` (sase-core)
  - `alias_regex(alias)` splits the normalized alias on whitespace, escapes each word,
    and joins them with `[\t ]+`. That character class is the whole bug: it cannot match
    a newline, so no compiled pattern can span one.
  - `CompiledGlossaryCatalog::candidate_spans()` drops candidates that intersect a
    `prompt_literal_zone_ranges()` literal, keeps only word-boundary hits via
    `is_word_boundary()`, then accepts the longest non-overlapping candidate at each
    start offset.
  - `GlossarySpanWire` carries `byte_start`, `byte_end`, `matched_text`, and one `range`
    (an `EditorRange` with `start`/`end` positions).
  - `validate_glossary_entries()` already rejects an authored term or alias that itself
    contains a line break (`multiline_term`, `multiline_alias`). That stays true: this
    epic changes how _document text_ is matched, never what may be authored.
- `crates/sase_xprompt_lsp/src/semantic_tokens.rs` (sase-core)
  - `raw_glossary_tokens()` emits exactly one raw token per span, and `push_token()`
    returns early when `range.start.line != range.end.line`. A multi-line glossary span
    would therefore be silently dropped rather than highlighted.
- sase (this repo)
  - `src/sase/core/glossary_facade.py` mirrors `GlossarySpanWire` as `GlossarySpan`.
  - `src/sase/ace/tui/widgets/_prompt_glossary.py` `_build_highlight_map()` converts
    `span.range` to absolute offsets and calls `_append_highlight_span()`, which splits
    a range across rows through `_line_byte_spans()` in `_jinja_highlight.py`. A
    multi-line span would therefore already render, but it would underline the
    continuation line's leading indentation along with the word.
  - `src/sase/ace/tui/modals/glossary_preview_render.py` `build_glossary_title()`
    renders `matched "<matched_text>"` as one `Text` line and
    `glossary_definition_markdown()` embeds the matched text inside Markdown link text.
    Both assume `matched_text` has no newline in it.
  - `pyproject.toml` pins `sase-core-rs>=0.21.3,<0.22.0`.
- Documentation asserts the current limitation in two places, so both are part of the
  contract being changed: `docs/configuration.md` (`### memory.glossary`) says matching
  "allows horizontal whitespace runs inside multiword phrases, and does not cross a
  line", and `docs/xprompt.md` calls the LSP scanner "line-local".

## Design

### One line break is whitespace; two are a boundary

Replace the `[\t ]+` inter-word separator in `alias_regex()` with

```text
(?:[\t ]*\r?\n[\t ]*|[\t ]+)
```

applied independently to each gap between words. Each gap may absorb at most one line
break plus the horizontal whitespace around it, so a three-word term may span three
lines, and no gap may cross a blank line.

The narrowness is deliberate. Everything that should still fail keeps failing for free,
without a Markdown parser:

| Text                         | Matches | Why                                               |
| ---------------------------- | ------- | ------------------------------------------------- |
| `xprompt\n  memory`          | yes     | one break plus continuation indentation           |
| `xprompt  \n\tmemory`        | yes     | trailing spaces and a tab indent are whitespace   |
| `xprompt\r\n  memory`        | yes     | CRLF documents behave like LF ones                |
| `xprompt\n\nmemory`          | no      | a blank line needs two breaks in one gap          |
| `xprompt\n- memory`          | no      | `-` is not whitespace; a new list item intervenes |
| `xprompt\n> memory`          | no      | same, for a blockquote                            |
| `xprompt\n## Memory`         | no      | same, for a heading                               |
| `xprompt\n---\nmemory`       | no      | same, for a swarm separator or thematic break     |
| `xprompt.\nMemory files ...` | no      | the sentence period breaks the phrase             |

Everything else in `candidate_spans()` is unchanged, and each existing rule keeps
applying to the longer candidates this enables: a wrapped candidate that intersects a
fenced or inline literal zone is discarded (the shorter in-prose candidates around it
survive, because filtering happens before overlap resolution), word-like boundaries are
still required at both ends, and longest-match-at-earliest-start still decides ties —
which is exactly what makes the wrapped `Xprompt Memory` beat the bare `Xprompt` in the
reported case.

### Core-computed per-line segments

A span is now a matching unit that display surfaces cannot use directly, because both
frontends underline per line:

- LSP semantic tokens are single-line by protocol; today `push_token()` drops anything
  else.
- ACE would otherwise underline the continuation line's indentation, drawing a rule from
  the left margin to the word.

Both frontends need the same answer to "which runs of characters does this match paint",
so per the Rust core backend boundary rule that answer belongs in core rather than being
derived twice. Add to the span wire:

```rust
pub struct GlossarySegmentWire {
    pub byte_start: usize,
    pub byte_end: usize,
    pub range: EditorRange,
}
```

as `#[serde(default)] pub segments: Vec<GlossarySegmentWire>` on `GlossarySpanWire`.
Each segment is single-line and has its leading and trailing horizontal whitespace (and
any `\r`) trimmed off, so a single-line match yields exactly one segment equal to the
span itself and consumers can iterate `segments` uniformly instead of special-casing
wrapped matches. `range`, `byte_start`, `byte_end`, and `matched_text` keep their
current meaning and still describe the whole match, which is what `lookup()`, hover, and
go-to-definition need.

`GLOSSARY_WIRE_SCHEMA_VERSION` stays `1`. The version gate in `load_glossary_catalog()`
(`crates/sase_xprompt_lsp/src/server.rs`) applies to the _catalog_ payload written by
the sase launcher, and that payload carries entries, not spans; spans are computed
in-process on both sides. Adding a defaulted field to a struct that is never read from
disk is compatible in both directions.

### Presentation of a wrapped match

`matched_text` deliberately keeps the raw document slice, including the newline and the
continuation indentation, because it is the literal text that matched. The ACE preview
card is the one place that renders it as prose, so it normalizes whitespace at the point
of display: `matched "xprompt\n  memory"` becomes `matched "xprompt memory"`, which then
compares equal to the term and correctly suppresses the disclosure line entirely. That
is presentation, so it stays in Python and no wire field is added for it.

### Why nothing else has to change

- Soft wrapping is already fine and always was: Textual wraps a long buffer line for
  display while the document row and the highlight offsets stay intact. This epic is
  only about hard line breaks in the buffer.
- `sase memory init` output does not change. The generator renders entries from config
  and never scans document text, so no memory file or agent instruction file is
  regenerated by this work.
- `sase-nvim` does not change. It consumes whatever semantic tokens the server sends; a
  wrapped phrase simply arrives as two adjacent tokens.
- `crates/sase_core_py` does not change. `glossary_to_py()` serializes spans through
  serde, so the new field crosses into Python automatically.

## Phase 1: Match phrases across one line break and expose per-line span segments (`core`)

Use `/sase_repo` to open `sase-core` before reading or writing any file in it, and use
the path that skill prints. Do not hand-edit any crate version or `Cargo.toml` version
field — release-plz owns those.

In `crates/sase_core/src/glossary.rs`:

1. Add a private `const PHRASE_GAP: &str = r"(?:[\t ]*\r?\n[\t ]*|[\t ]+)";` next to
   `alias_regex`, and join the escaped words with it. Keep the newline alternative first
   so the intent reads in source order; both orders are correct because the branches are
   near-disjoint.
2. Add `GlossarySegmentWire` as specified in **Core-computed per-line segments**,
   deriving the same traits as `GlossarySpanWire`, and add the defaulted `segments`
   field to `GlossarySpanWire` after `range`.
3. Add a private helper that builds the segments for a byte range against the
   `DocumentSnapshot`: split the range at each `\n` inside it, trim ASCII spaces, tabs,
   and a trailing `\r` from both ends of every piece, drop pieces that become empty, and
   convert each surviving piece with `document.byte_range_to_range()`. Populate it from
   `CandidateSpan::to_wire()`. A single-line match must produce exactly one segment
   whose byte range equals the span's own.
4. Update the module doc comment and the `alias_regex` behavior note to state that a
   multiword phrase may be separated by horizontal whitespace or by a single line break
   with its surrounding indentation, and never by a blank line.
5. Leave `validate_glossary_entries`, `normalize_phrase`, `pluralize_phrase`,
   `is_word_boundary`, and the literal-zone filter untouched.

In `crates/sase_xprompt_lsp/src/semantic_tokens.rs`:

6. Change `raw_glossary_tokens()` to emit one `RawSemanticToken` per segment of each
   scanned span instead of one per span, using the segment's byte offsets. Leave
   `non_overlapping_tokens()` alone — it sorts by byte start at the end, so the split
   tokens stay in the ascending order the delta encoder requires — and keep the
   single-line guard in `push_token()` as a backstop rather than deleting it.

Unit tests (in the existing `#[cfg(test)] mod tests` in `glossary.rs`, and a new
`#[cfg(test)]` module in `semantic_tokens.rs`), covering at minimum:

7. A two-word term matches across `\n` plus continuation indentation; `matched_text`
   preserves the raw slice including the newline; `segments` is two single-line ranges
   with the indentation trimmed off.
8. The same term does not match across a blank line, nor when the continuation line
   starts with `- `, `> `, `## `, or `---`.
9. A wrapped multiword term beats a shorter single-word entry that starts at the same
   offset — assert the reported `Xprompt` / `Xprompt Memory` pair directly.
10. `\r\n` behaves like `\n` and no `\r` survives inside a segment.
11. A three-word term wraps across three lines.
12. A single-line match still yields exactly one segment equal to the span, so
    `segments` is safe to iterate unconditionally.
13. A wrapped candidate overlapping a fenced or inline literal is skipped while an
    in-prose match elsewhere in the same text still resolves.
14. `lookup()` with a position on the continuation word returns the whole span.
15. `document_semantic_tokens()` for a wrapped phrase emits two tokens on consecutive
    lines with the trimmed lengths and correct `delta_line`/`delta_start`, and the
    artifact-ref token path is unaffected.

Verification, from a sase workspace (these recipes drive the sase-core checkout):

```bash
just rust-fmt
just rust-check   # rust-fmt-check + rust-clippy + rust-test
```

and in the sase-core checkout:

```bash
cargo test --workspace glossary
cargo test --workspace
```

Acceptance: `cargo test --workspace` is green, a two-word term wrapped over one line
break scans as one span with two trimmed segments, a blank line still separates, and the
LSP emits one token per segment.

## Phase 2: Publish a sase-core-rs release containing the matcher change (`core-release`)

The Python phase moves `pyproject.toml`'s `sase-core-rs` window, and
`tools/validate_sase_core_rs_version --published-minimum` fails unless the floor exists
on PyPI, so the release must land first.

1. Confirm the `core` phase commit is on `sase-core` `master`.
2. Find the release-plz release pull request for `sase-core` and confirm its changelog
   entry covers the glossary matcher change.
3. Note the expected version before merging. At planning time `sase-core` `master`
   already carried an unreleased
   `feat!: drop legacy top-level glossary path from glossary_scope_paths`, so
   release-plz is expected to cut **0.22.0**, which is outside the `<0.22.0` ceiling
   sase declares today. That is a pre-existing consequence of that commit, not of this
   epic. Verify the expectation against the actual release PR rather than assuming it:
   if the published version is a `0.21.x`, the `python` phase moves only the floor; if
   it is `0.22.x`, the `python` phase moves the floor and widens the ceiling to
   `<0.23.0`. If release-plz proposes something outside both of those shapes, stop and
   raise it with the project owner instead of editing either side unilaterally.
4. Merge that release PR if the project owner has not already, then wait for the publish
   workflow to finish.
5. Record the exact published version on this phase's bead and confirm it resolves:

   ```bash
   uv pip download --no-deps --dest /tmp/core-release-check "sase-core-rs==<version>"
   ```

Acceptance: a `sase-core-rs` version containing the `core` phase change is installable
from PyPI, and its number and the required window shape are recorded on this phase's
bead for the `python` phase to consume.

## Phase 3: Consume segments in ACE, guard the floor, and correct the docs (`python`)

All work in this phase is in this repo.

1. `pyproject.toml`: set the `sase-core-rs` window to the version published in
   `core-release`, following the shape that phase recorded — `>=<version>,<0.22.0` for a
   `0.21.x` release, or `>=<version>,<0.23.0` for the expected `0.22.x` one. Widening
   the ceiling to `<0.23.0` is safe for the breaking change it carries: sase resolves
   glossary config exclusively through `resolve_glossary_config()` in
   `src/sase/glossary_config.py`, which reads only `memory.glossary`, there is no
   `glossary_scope_paths` caller in this repo, and `docs/configuration.md` already
   documents the legacy top-level key as removed. Confirm all three before widening.

   If by implementation time this repo has moved the `sase-core-rs` window ratchet to
   the release boundary — check `pyproject.toml`, `tools/validate_sase_core_rs_version`,
   and the current `just check` guidance rather than assuming — let that mechanism own
   the version edit and skip only this step. Every other step, including the smoke gate,
   still applies.

2. `src/sase/core/glossary_facade.py`: add `segments: tuple[Mapping[str, Any], ...]` to
   `GlossarySpan` after `range`, and read it in `from_wire` with a strict
   `payload["segments"]`. Do not add a tolerant fallback that synthesizes a single
   segment from `range`: the version window is what guarantees the field, and a silent
   fallback would hide a stale wheel while quietly restoring the indentation-underlining
   bug.
3. `src/sase/ace/tui/widgets/_prompt_glossary.py`: in `_build_highlight_map()`, iterate
   `span.segments` and call `_editor_range_to_offsets()` / `_append_highlight_span()`
   once per segment instead of once per span, skipping segments the conversion rejects.
   Leave the surrounding cache, context, and overlay-size guards alone.
4. `src/sase/ace/tui/modals/glossary_preview_render.py`: collapse whitespace runs in the
   matched text before it is compared and rendered — apply it in
   `_should_show_matched_text()` and `build_glossary_title()` so a wrapped match that
   normalizes to the term suppresses the `matched "..."` line instead of printing a
   newline into the card, and in `glossary_definition_markdown()`'s link text so a
   wrapped reference cannot break the generated Markdown link.
5. Tests:
   - `tests/test_core_glossary_facade.py`: build a real compiled catalog through
     `compile_glossary_catalog()` for a two-word term and assert (a) text wrapped at the
     term's internal space produces one span for the full term, (b) its `segments` are
     two single-line ranges with the continuation indentation excluded, (c) a blank line
     between the words produces no span for the phrase, and (d) a single-line match has
     exactly one segment. This is the assertion that fails loudly on a core downgrade.
   - `tests/ace/tui/widgets/_prompt_glossary_helpers.py`: add `segments` to the span
     wires the fake compiled catalogs return, and add a helper that builds a wrapped
     two-segment span so the widget tests can exercise the multi-segment path.
   - `tests/ace/tui/widgets/test_prompt_glossary_highlighting.py`: a wrapped term yields
     exactly two `glossary.term` highlights on consecutive rows, the continuation row's
     highlight starts at the first non-space column, and nothing is emitted for the
     newline itself.
   - `tests/ace/tui/widgets/test_prompt_glossary_navigation.py`: with the cursor on the
     continuation word, `K` previews and `Ctrl+]` targets the full term's entry.
   - `tests/ace/tui/modals/test_glossary_preview_render.py`: a wrapped `matched_text`
     that normalizes to the term renders no disclosure line, and one that normalizes to
     a different alias renders it on a single line with no embedded newline.
   - `tests/ace/tui/visual/_ace_prompt_png_snapshot_helpers.py`: teach
     `_VisualCompiledGlossary` to find a term across one line break and to emit trimmed
     per-line segments, mirroring the core rule, and add a
     `GLOSSARY_WRAPPED_HIGHLIGHT_PROMPT` constant whose text wraps a term mid-phrase.
     Leave `GLOSSARY_HIGHLIGHT_PROMPT` and its two existing snapshots untouched.
   - `tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py`: add one
     dark-theme snapshot for the wrapped prompt, generated with
     `--sase-update-visual-snapshots`, and assert a `glossary.term` highlight exists on
     both rows the way the existing glossary snapshot test asserts its own.
6. Add `tools/smoke_sase_core_rs_glossary_line_break`, modeled on
   `tools/smoke_sase_core_rs_plan_header`: a `#!/usr/bin/env python3` script importing
   only `sase_core_rs`, exposing a `validate_line_break_matching(module)` function that
   compiles a two-word-term catalog, asserts the wrapped scan returns the full term with
   two trimmed single-line segments, asserts a blank line does not match, and returns a
   small dict summary. Wire it into `.github/workflows/ci.yml` as one more step of the
   `published-core-minimum-smoke` job:

   ```yaml
   - name: Smoke glossary line-break matching
     run:
       /tmp/published-core-smoke/bin/python tools/smoke_sase_core_rs_glossary_line_break
   ```

   and add `tests/test_sase_core_rs_glossary_line_break_smoke_tool.py` following the
   shape of `tests/test_sase_core_rs_plan_header_smoke_tool.py`
   (`pytestmark = pytest.mark.contract`, load the script through `SourceFileLoader`,
   assert the summary against the installed core). This gate is what keeps the fix from
   silently regressing for published installs: the change adds no new binding name, so
   `tools/check_sase_core_rs_bindings` cannot see it.

7. Documentation:
   - `docs/configuration.md`, `### memory.glossary`: replace "allows horizontal
     whitespace runs inside multiword phrases, and does not cross a line" with the new
     rule — horizontal whitespace runs or a single line break with its surrounding
     indentation separate the words of a phrase, a blank line or any non-whitespace line
     prefix ends it, and a wrapped match is underlined per line with the indentation
     excluded.
   - `docs/xprompt.md`, the LSP glossary paragraph: drop "line-local" from the scanner
     description and note that a phrase wrapped across a line break is emitted as one
     semantic token per line.

Verification:

```bash
just install
just check-full
just test-visual
pytest tests/test_core_glossary_facade.py \
       tests/test_sase_core_rs_glossary_line_break_smoke_tool.py \
       tests/ace/tui/widgets/test_prompt_glossary_highlighting.py \
       tests/ace/tui/widgets/test_prompt_glossary_navigation.py \
       tests/ace/tui/modals/test_glossary_preview_render.py
python tools/validate_sase_core_rs_version --pyproject pyproject.toml --published-minimum
```

`just check-full` rather than `just check` because this touches `pyproject.toml`, CI
workflow YAML, and docs; `just test-visual` because the prompt highlighting snapshots
change. Then confirm the smoke script against a venv holding the exact published
minimum, the way CI does:

```bash
core_minimum="$(python3 tools/smoke_sase_core_rs_telemetry --print-minimum pyproject.toml)"
uv venv --python 3.12 /tmp/published-core-smoke
uv pip install --python /tmp/published-core-smoke/bin/python "sase-core-rs==${core_minimum}"
/tmp/published-core-smoke/bin/python tools/smoke_sase_core_rs_glossary_line_break
```

Finally, reproduce the original report by hand in `sase ace`: paste a prompt in which a
configured multiword term wraps mid-phrase, confirm both halves are underlined and the
indentation is not, and confirm `K` on the continuation word opens the card for the full
term with no `matched "..."` line.

Acceptance: the wrapped term from the report highlights across both lines in ACE, the
per-segment highlights exclude indentation, the preview card shows no newline, and the
smoke gate passes against the published minimum.

## Explicitly out of scope

- Matching across more than one line break in a single gap, i.e. across a blank line. A
  paragraph boundary is a real boundary; joining across it would produce matches no
  reader perceives as one phrase.
- Markdown-aware continuation prefixes. A phrase whose continuation line begins with a
  blockquote marker, a list marker, or a heading will not match, by design. Teaching the
  matcher to strip block prefixes means teaching it Markdown, and the payoff is a rare
  case that reads as two blocks anyway.
- Hyphenation and other punctuation-split phrases (`agent-\nclan`). Only whitespace
  separates words in a glossary phrase.
- Soft-wrap behavior. A long line that Textual wraps for display already highlights
  correctly; nothing here changes it.
- `sase/sase.yml` glossary entries and `sase memory init` output. Neither is affected;
  no memory file or agent instruction file is regenerated by this epic.
- `sase-nvim`. Two adjacent semantic tokens already render with the existing
  `SaseGlossaryTerm` highlight group.
- Glossary hover and go-to-definition Markdown in the xprompt LSP. Both go through
  `lookup()` and pick up wrapped spans without a change; their rendered content is not
  touched.

## Risks

- **False positives across a sentence boundary.** Text like `... a new xprompt` newline
  `memory of the run ...` would now match where it did not before. The single-break
  rule, the surviving word-boundary requirement, and the fact that sentence-ending
  punctuation blocks the gap keep this rare, and a false positive costs one spurious
  underline that `K` immediately explains — strictly less harmful than the false
  negative being fixed.
- **A stale core wheel silently disables glossary highlighting.** The render path
  swallows exceptions, so the strict `segments` read in step 2 degrades to no glossary
  overlay rather than crashing. That is the intended failure mode, but it is quiet; the
  version window plus the published-minimum smoke gate are what make it impossible to
  ship, and both must land together in the `python` phase.
- **Cross-repo ordering, and a ceiling that has to move.** The `python` phase cannot
  land its version edit before `core-release` publishes, and the pending breaking commit
  on `sase-core` `master` means the ceiling likely moves too. Phase 2 step 3 forces that
  to be checked against the real release PR. Development and local testing of phase 3
  can start as soon as phase 1 lands, because `just install` builds the extension from
  the sase-core checkout rather than from PyPI.
- **Matcher cost in the prompt render path.** Pattern count is unchanged; each gap gains
  a two-branch alternation, and the linear-time `regex` engine keeps the scan linear.
  `_build_highlight_map` still refuses to scan past `_MAX_OVERLAY_BYTES` /
  `_MAX_OVERLAY_LINES`. No perf work is planned; if `SASE_TUI_PERF=1` key-to-paint p95
  regresses past 16 ms on the prompt tab, report it as a finding rather than fixing it
  inside this epic.
- **Visual snapshot churn.** Adding a new prompt and a new snapshot rather than editing
  `GLOSSARY_HIGHLIGHT_PROMPT` keeps the two existing goldens byte-identical, so an
  unexpected diff in them is a real regression signal and not expected noise.

## Definition of done

- A multiword glossary term separated by one line break is matched, and the longest such
  phrase wins over a shorter entry starting at the same offset — the reported `xprompt`
  / `xprompt memory` case resolves to `Xprompt Memory`.
- A blank line, or a continuation line starting with a list, quote, heading, or
  separator marker, still ends the phrase.
- ACE underlines each line of a wrapped match and never its indentation; `K` and
  `Ctrl+]` work from either half.
- The xprompt LSP emits one semantic token per line of a wrapped match instead of
  dropping it.
- `tools/smoke_sase_core_rs_glossary_line_break` passes against the exact published
  minimum in CI.
- `docs/configuration.md` and `docs/xprompt.md` no longer promise line-local matching.
- `just rust-check` and `cargo test --workspace` pass in sase-core; `just check-full`
  and `just test-visual` pass in sase.
