---
tier: tale
size: small
title:
  Alternation scanner parity after `{` and nested paren openers, then close sase-1co
goal:
  Every alternation opener that launch fans out (including `{%{a | b}`, `{%(a,b)`, and a
  paren opener right after another opener such as `%{%(a,b) | c}`) is reported by the
  shared core scanner, highlighted, and free of false Jinja diagnostics, and epic
  sase-1co is closed with its plan marked done.
proposed_by: bbugyi200.athena.sase-1co.land
bead: sase-1co
create_time: 2026-09-29 18:41:27
status: wip
---

- **PARENT:**
  [202609/midword_alternation.md](https://github.com/sase-org/sase--plans/blob/main/202609/midword_alternation.md)
- **BEAD:**
  [sase-1co](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1co/README.md)

# Plan: alternation scanner parity after `{` and nested paren openers, then close sase-1co

## Why this tale exists

Epic `sase-1co` ("Mid-word alternation (`%{...}`) everywhere") has all five phases
closed and verified. The landing audit found one remaining defect that the epic caused.
Commit `0266fe4a12` moved TUI alternation highlighting from the old Python tokenizer
onto the shared core scanner (`sase_core::editor::alternation::scan_alternations`,
exposed as the `alternation_scan` binding). That scanner now disagrees with launch
fan-out in two cases. The old Python tokenizer highlighted both of them, and launch
still fans them out.

1. **An opener right after a literal `{`.** `{%{a | b}`, `{%(a,b)`, `{%alt(a,b)` and
   `x {%{a | b} y` all launch as two slots (`{a` / `{b`), but `alternation_scan` returns
   `[]`.
   - **Cause:** `excluded_literal_and_definition_ranges` in the sase-core file
     `crates/sase_core/src/editor/exclusion.rs` treats the `{%` as an unclosed Jinja
     statement tag. `jinja_tag_ranges` / `next_jinja_tag` then extend that tag to the
     end of the text, and the scanner skips every opener inside it.
   - **Why Jinja never sees it:** launch splits alternations before any Jinja rendering.
     `render_toplevel_jinja2` runs per slot in `src/sase/llm_provider/preprocessing.py`,
     after fan-out.
   - **Not valid Jinja anyway:** `{%{`, `{%(` and `{%alt(` are never valid Jinja ("tag
     name expected").
   - **Same diagnostic in Python:** `sase.xprompt.jinja_inspect.diagnose("{%{a | b}")`
     reports `tag name expected`. The TUI prompt input shows that Jinja error on text
     that launches correctly.
2. **A paren opener directly after another opener's delimiter.**
   - **Symptom:** `%{%(a,b) | c}` launches `a`, `b`, `c` (the nested paren alternation
     expands through `expand_branch_value`). `scan_alternations` reports only the outer
     brace record, so the inner `%(`, its `,` separator, and its `)` are never
     highlighted. The same happens for `%{%alt(a,b) | c}` and `%(%(a,b), c)`.
   - **Cause:** `alt_directive_starts` in the sase-core file
     `crates/sase_core/src/agent_launch/directive_scan.rs` iterates `alt_directive_re()`
     (`(?m)(%\{)|(?:^|[\s\(\[\{"':])(%(?:alt)?\()`) with `captures_iter`. The paren
     branch must consume its one-character left boundary. Here that boundary is the
     `{`/`(` the previous match already consumed, so the non-overlapping iterator never
     matches it.

Current reproduction, using the installed binding from the sase repo root:

```text
$ .venv/bin/python -c "import sase_core_rs as r; print(r.alternation_scan('{%{a | b}'))"
[]
$ .venv/bin/python -c "import sase_core_rs as r; print([(d['marker_start'], d['depth']) for d in r.alternation_scan('%{%(a,b) | c}')])"
[(0, 0)]
```

This tale fixes both cases in sase-core and adds the matching Python mirrors in sase. It
then finishes the epic's closeout. Nothing resumes the `sase-1co` landing after this
tale, so its final step closes the epic.

## Grammar rules this tale must preserve

These rules from the epic plan (`plan:202609/midword_alternation.md`) are normative:

- `%{` opens anywhere outside literal/definition zones.
- `%(` and `%alt(` open only at a directive-valid position: start of text, or after
  whitespace, `(`, `[`, `{`, `"`, `'` or `:`. So `x%(a,b)`, `50%(approx)` and
  `fmt%alt(a,b)` stay plain text.
- Real Jinja tags stay editor definition zones, and alternations inside them stay inert:
  - `{% if x %}`, `{%- if x %}`, `{{ ... }}`, `{# ... #}`;
  - `{{%{a|b}}}` — the `{{` expression zone is unchanged.
- The existing `%{%` carve-out stays: `100%{% if x %}` is an alternation.
- Launch fan-out results must not change for any input. Only editor/scanner surfaces
  change.

## Step 1 — sase-core fixes

Open the linked checkout with `sase repo open sase-core -r "<why>"`, read its
`AGENTS.md`, and work only in the printed path.

1. **`crates/sase_core/src/editor/exclusion.rs` — `next_jinja_tag`.**
   - For the `{%` opener, add a second carve-out beside the existing
     `tail[..start].ends_with('%')` one: skip the `{%` when the text right after it
     starts with `{`, `(` or `alt(`. That `%` begins an alternation opener, which launch
     always fans out because `{` is a directive boundary.
   - Continue the search from `start + 1`, exactly like the existing carve-out.
   - Extend the comment to cover both carve-outs.
   - Leave the `{{` and `{#` openers, and `{%` followed by anything else (including
     `{%-`, `{%+`, whitespace and tag names), unchanged.
2. **`crates/sase_core/src/agent_launch/directive_scan.rs` — `alt_directive_starts`.**
   - Stop relying on non-overlapping regex iteration. Scan every `%` byte position `i`
     (for example with `match_indices('%')`) and classify it:
     - `%{` → `(i, i + 1, AltDelimiter::Brace)`;
     - `%alt(` → `(i, i + 4, AltDelimiter::Paren)`;
     - `%(` → `(i, i + 1, AltDelimiter::Paren)`.
   - For the two paren forms, emit only when the position is directive-valid: `i == 0`,
     or the previous `char` (`prompt[..i].chars().next_back()`) satisfies
     `char::is_whitespace` or is one of `(`, `[`, `{`, `"`, `'`, `:`. This is exactly
     what the regex allowed, since `(?m)^` after `\n` is covered by whitespace.
   - Keep the return type, the source order, and the doc comment's rule text.
   - Delete `alt_directive_re()` if it becomes unused (clippy runs with `-D warnings`),
     and move its explanatory comment onto `alt_directive_starts`.
   - **Why launch is unaffected:** `plan_alternative_slots` in `agent_launch/fanout.rs`
     already skips any start that falls inside an accepted outer span. The only other
     callers are `alt_inner_ranges` and the inline-literal masks. They build
     `[start, end)` ranges consumed only through `position_in_ranges`-style unions, so
     an extra nested range inside an outer range changes nothing.
3. **Tests, placed beside the code.**
   - **`tests` module in `crates/sase_core/src/editor/alternation.rs`:**
     - `scan_alternations` returns one depth-0 record for each of `{%{a | b}` (marker 1,
       close 8, separator 5), `{%(a,b)`, `{%alt(a,b)` and `x {%{a | b} y`.
     - `%{%(a,b) | c}` returns the outer brace record (depth 0) and the inner paren
       record (depth 1, with its `,` separator). The same holds for `%{%alt(a,b) | c}`
       and `%(%(a,b), c)`.
     - `x%(a,b)`, `%%(a,b)` and `fmt%alt(a,b)` still return `[]`.
     - `{% if x %}y{% endif %} %{a|b}` still reports only the trailing alternation.
     - `{{%{a|b}}}` still returns `[]`.
   - **Exclusion carve-out tests** (a new `#[cfg(test)] mod tests` in `exclusion.rs` is
     fine): assert that `{%{`, `{%(` and `{%alt(` start no Jinja range, while
     `{% if x %}`, `{%- if x %}` and an unclosed `{% if` still do.
   - **Launch regressions in `crates/sase_core/src/agent_launch/tests/fanout.rs`:**
     `{%{a | b}` fans out to `{a` / `{b`, and `%{%(a,b) | c}` fans out to `a`, `b`, `c`.
     Both already pass; they pin the unchanged launch behavior. Keep every existing
     fanout, alternation, `model_alias_shortcut`, diagnostics and LSP test green.
   - Run `just test -p sase_core alternation` and `just test -p sase_core exclusion` (or
     similar targeted filters) while iterating.
4. Run `sase tool run check` from the sase-core checkout. It takes about 5 minutes, so
   give it an explicit tool timeout of 10 minutes or more. Fix anything it reports.

## Step 2 — sase mirrors, tests, and docs

Work in this sase repo. Read the `lint_and_test` reference memory before finishing.

1. **Rebuild the binding** from the linked checkout (`just install` does this). Then
   confirm that
   `.venv/bin/python -c "import sase_core_rs as r; print(r.alternation_scan('{%{a | b}'))"`
   prints one brace record.
2. **`src/sase/xprompt/jinja_inspect.py` — `_alt_jinja_overlap_ranges`.**
   - Also mask the two characters of a `{%` whose `%` begins an alternation opener, that
     is, `{%` immediately followed by `{`, `(` or `alt(`. This mirrors the core
     carve-out from Step 1.
   - Keep the existing `%{%` masking.
   - Replace the `"%{%" not in text` fast path with one that also admits `{%`, so text
     without either substring still returns `[]` immediately.
   - Update the docstring.
   - `_mask_jinja_regions` / `_mask_inert_regions` then stop reporting
     `tag name expected` for `{%{a | b}` and `has_jinja("{%{a | b}")` becomes `False`
     (today it is `True`). Real `{% ... %}` tags keep their diagnostics.
3. **`src/sase/xprompt/_directive_alt.py` — `_ALT_DIRECTIVE_RE`.**
   - Mirror the core boundary fix. Switch the paren branch back to a zero-width
     lookbehind so it consumes no prefix character:

     ```python
     _ALT_DIRECTIVE_RE = re.compile(
         r"(%\{|(?:^|(?<=[\s(\[{\"':]))%(?:alt)?\()",
         re.MULTILINE,
     )
     ```

     Adjacent openers such as `%{%(a,b)|c}` then all match.

   - Group 1 is now the bare marker for both forms.
   - Rewrite the comment above the regex:
     - drop the "do not use `match.start(1)`" caveat;
     - say it mirrors `alt_directive_starts` in the Rust core.
   - Every caller uses only `match.end() - 1`, which is unchanged: `has_alt_directive`,
     `_plan_prompt_fanout`, `_directive_collect._alt_inner_regions`,
     `_directive_shorthand._alt_inner_ranges`,
     `_directive_edit_core.find_alt_inner_regions`, `directive_diagnostics`, and
     `jinja_inspect._alt_jinja_overlap_ranges`. Re-grep for `_ALT_DIRECTIVE_RE` to
     confirm.

4. **Tests.**
   - **`tests/test_xprompt_alt_inspect.py`:**
     - `tokenize("{%{a | b}")` yields delimiters `(1, 3)` and `(8, 9)` and separator
       `(5, 6)`;
     - `tokenize("%{%(a,b) | c}")` includes the inner `%(` / `)` delimiters and its `,`
       separator;
     - `groups("{%{a | b}")` returns one group with branches `("a ", " b")`.
   - **`tests/test_xprompt_jinja_inspect.py`:**
     - `diagnose` is clean (`ok=True`) for `{%{a | b}`, `{%(a,b)` and `x {%{a | b} y`;
     - a real unclosed `{% if x %}` without `{% endif %}` still reports an error;
     - `has_jinja("{%{a | b}")` is `False`.
   - **`tests/test_directives_has_helpers.py`** (or a sibling directive test):
     `_ALT_DIRECTIVE_RE` finds both openers in `%{%(a,b) | c}`, and `x%(a,b)` still does
     not match.
   - Keep `tests/xprompt/test_highlight.py::test_calls_each_scanner_once` and the
     binding-call-count test green.
5. **Docs.**
   - In `docs/xprompt.md`, section "Where `%{...}` Opens", next to the Jinja ambiguity
     paragraph, add a sentence: a `%{`, `%(` or `%alt(` right after a literal `{`
     (`{%{a | b}`) is an alternation, not a Jinja tag. It launches `{a` / `{b`.
   - Keep the existing wording.
6. **Core pin: do not edit `sase-core-revision.txt` by hand.** `sase/sase.yml` declares
   `repos.linked[].revision_pin: sase-core-revision.txt` for sase-core. When this turn's
   declaration commits both repos, the host commits sase-core first and writes its
   pushed SHA into the pin before the sase commit (see `docs/rust_backend.md`, "The CI
   source revision pin").
7. **Verify.** Run `just fmt`, then `sase tool run check`.
   - **Known pre-existing failure:** the patch/stitch terminology audit fails on 14
     lines in the sase-core fixture
     `crates/sase_core/tests/fixtures/note_attachment/at_bearing_notes.jsonl`. It is
     tracked on active epic `sase-1ck` and is not this tale's work. Confirm that the
     only failing items are exactly those, and record that in the close note.
   - Any other failure is yours to fix.
   - Do not run `just check-full`.

## Step 3 — close out epic sase-1co (final step)

Do this in the same turn as the code above. The host commits after the turn ends, so
nothing here may wait for this tale's commit, push, or CI.

1. **Epic symbols.** Run `sase bead epic-symbols sase-1co`. At landing-audit time it
   printed `No --epic-symbol entries for sase-1co.`
   - If entries appear, resolve each one: wire it up, privatize it, add a non-test
     pragma, or delete it, following the Symvision epic-whitelist policy in the
     `symvision` reference memory.
   - Re-key a Justfile line only to a still-open later bead that still needs it.
2. **Close the epic** with a note that states what was verified. Fill in the `<...>`
   parts from this turn's actual results:

   ```bash
   sase bead close sase-1co --note "Landed. Verified phases .1-.5 against sase-core 1160ea4 + 1e51ff3, sase 0266fe4a12 + f02c3273e4, sase-nvim 332b7ab: mid-word/adjacent/colon/nested %{ fan-out with no nested panic, glued-directive spacing, model-shortcut spacing, shared alternation_scan wire + code-point binding, unclosed-alternation diagnostic, LSP alternation/separator semantic tokens, _ALT_DIRECTIVE_RE brace-anywhere mirror, memoized alt_inspect adapter + project-tag groups, TUI mid-word padding/separators/innermost span/Jinja auto-pair guard, nvim LSP-token overlay, docs; 345 focused sase tests green at f02c3273e4. Integration: commits since the epic started (bead attachments, tool-run, finalizer revision_pin, prompt archive, core pin ratchet 339a67306b) needed no alternation changes. Remaining epic work (scanner missed openers after a literal { and paren openers right after another opener) fixed in sase-core <sha or 'this turn'> + sase mirrors (jinja_inspect carve-out, lookbehind _ALT_DIRECTIVE_RE); sase-core check <result>, sase check <result> (only the pre-existing sase-1ck terminology-fixture failure). Follow-ups: sase-1co.3/.4 terminology audit -> corroborated on active epic sase-1ck (owner); sase-1co.5 sase lsp self-exec loop -> new task sase-1ct."
   ```

   - If the close is rejected because `--epic-symbol` entries remain, finish that
     cleanup and close again.
   - Never use `--force` merely to make the close succeed.

3. Run `just symvision` and confirm the whitelist is clean.
4. **Mark the plan done.** Set `status: done` in the frontmatter of the epic's plan
   file, `plan:202609/midword_alternation.md`. It is the PLAN path printed by
   `sase bead read sase-1co -r "Need the plan path"` and lives in the plans sidecar
   checkout at `sase/repos/plans/202609/midword_alternation.md`. Today it says
   `status: wip`.
5. `sase-1co` has no `parent_bead`, so there is no ancestor to close; finish normally.
