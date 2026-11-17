---
tier: tale
title: Stop macro guard markers from leaking and breaking swarms and typed args
goal:
  "#research_swarm(image=true) launches all of its segments again, typed macro arguments
  fed by Jinja interpolation validate and compare as their real values, trace evidence
  holds no guard tokens, and any future marker or token leak aborts loudly instead of
  silently corrupting a prompt, all without letting interpolated data reach command
  execution."
size: medium
proposed_by: bbugyi200.athena.1do
proposed_with: claude/opus
create_time: 2026-11-17 16:38:32
status: wip
---

# Stop macro guard markers from leaking and breaking `#research_swarm` and typed args

## Problem

Commit `efc66a420b` ("feat(macro): evaluate authored commands before interpolating
data", phase `sase-1tc.1.1.2` of the in-progress epic `sase-1tc.1.1`) wraps every Jinja
`{{ }}` output in NUL-byte interpolation markers, then hides each marked span behind a
guard token. That broke three things:

1. **Swarms silently lose agents.** `#research_swarm(..., image=true)` launched 6 agents
   instead of 8 on apollo (`research.58`, `research.59`). The image and linker segments
   were merged into the lead (`.final`) prompt as dead text. This is the bug in
   `research:202611/research_swarm_missing_image_linker_agents.md`. The cause is a Jinja
   macro called inside `{{ }}`. `#research_swarm` prints its file tree with
   `{{ tree(lead_files) }}`, and `tree`'s body prints its code fences through
   `{{ "```text" }}` and `{{ "```" }}`. So the output is marked twice: inner markers
   inside outer ones. `GeneratedTextGuard.hide_marked_spans` uses a non-greedy
   `START(.*?)END` regex, which pairs the outer START with the first inner END. That
   leaves an orphan START before the opening fence and an orphan END after the closing
   fence. After unwrap, the opening fence line no longer starts with a fence, and the
   closing fence reads as a new unclosed fence. That fence hides the last two `---`
   separators from `split_segments_protecting_fences`, so 8 segments become 6.
2. **Typed macro arguments fed by interpolation fail validation.** `sase macro expand`
   on the same prompt fails with
   `Macro '#research' argument error: Argument 'suffix' expects word ... got '\x00SASEDATA_<nonce>_18\x00'`.
   In `#research(suffix={{ r.short }})`, the argument is still a guard token when
   `expand_single_macro` calls `validate_and_convert_args`. The same tokens also break
   Jinja logic in the callee. For example, `#research` uses
   `{% elif suffix and suffix != 'false' %}`, which compares the token instead of the
   value. Swarm launches avoid the validation error only because they unwrap before
   per-agent expansion.
3. **Guard tokens leak into trace evidence.** `ExpansionTrace` records store
   `expanded_text` and the arguments before unwrap. Those records feed continuation
   capture (`macro_trace_segments`) and `sase macro expand -t`, so they carry raw
   `\x00SASEDATA_...\x00` tokens.

Nothing fails loudly. The swarm launched with missing agents and no error.

### Reproductions on current master (`be6adc20d0`)

All of these worked at the parent commit `71f0060927`. Use them as regression tests.

| Case                                                                                                                                                                                          | Result at `be6adc20d0`                                       |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `_render_jinja2_template` with a guard, macro `t()` printing fences through `{{ }}`, called as `{{ t() }}`                                                                                    | Output keeps orphan `SJSMARK` start/end markers              |
| Same, `{% set y %}{{ "a" }}-{{ "b" }}{% endset %}[{{ y }}]`                                                                                                                                   | Markers leak                                                 |
| Same, `{% macro w() %}<{{ caller() }}>{% endmacro %}{% call w() %}{{ "in" }}{% endcall %}`                                                                                                    | Markers leak                                                 |
| Same, `{% filter upper %}{{ "q" }}{% endfilter %}`                                                                                                                                            | Uppercased markers leak, because the regex is case-sensitive |
| Same, `{{ t() \| upper }}` where `t()` prints `{{ "a" }}`                                                                                                                                     | Uppercased markers leak                                      |
| Swarm macro whose lead segment calls `{{ tree([...]) }}`, followed by two `---` + `%if(should_run={{ img }})` segments, expanded with `expand_macro_swarms` (`tests/_macro_swarm_helpers.py`) | 1 segment instead of 3, with NUL markers                     |
| `#gen` → `#typed(suffix={{ s }})`, where `typed` has input `suffix: word`                                                                                                                     | `MacroArgumentError` (token fails the word regex)            |
| `#gi` → `#ti(n={{ 3 }})`, where `ti` has input `n: int` and content `N={{ n + 1 }}`                                                                                                           | `MacroArgumentError` (`expects int`)                         |
| `#gc` → `#cond(x={{ 'cdx' }})`, where `cond` is `{% if x == 'cdx' %}YES{% else %}NO{% endif %}`                                                                                               | `NO` (expected `YES`)                                        |
| Trace records for `#outer` → `#g(who={{ 'world' }})`                                                                                                                                          | `expanded_text` and `named_args` contain `\x00SASEDATA_`     |

## Design

The epic's invariants stay intact (`plan:202611/cmdsub_source_provenance.md`):

- Interpolated data never reaches an execution pass.
- Jinja sees the original typed values.
- Macro input validation, conditional segments, and trace capture keep working.
- Trace records are evidence, not authority to execute.

This is Python-side Jinja and guard orchestration, so no `sase-core` change is needed.

### 1. Markers never nest, and leftovers fail loudly (`src/sase/macro/cmdsub_source.py`)

- Add `class GeneratedTextLeakError(MacroError)`, importing `MacroError` from
  `sase.macro._exceptions`. Subclassing `MacroError` lets the existing macro-error
  handling (`raise_on_error`, or print-and-exit) surface it.
- In `GeneratedTextGuard.__init__`, precompile the patterns:
  - one that matches either of this guard's interpolation markers (from
    `interpolation_markers(self._nonce)`), with `re.IGNORECASE`;
  - the balanced-pair pattern for `hide_marked_spans`, with `re.DOTALL | re.IGNORECASE`.

  Case-insensitivity is needed because case-changing filters (`upper`, `lower`, `title`,
  `capitalize`) rewrite the letters in a marker.

- Add `GeneratedTextGuard.mark(value: object) -> str`. It converts the value with
  `str(value)`, strips every occurrence of this guard's markers (any case), and returns
  `start + text + end`. This is the only marking implementation. A nested macro, a
  `caller()`, or a `{% set %}` block capture becomes one balanced data span. The whole
  outer value stays data, so this cannot relax the "data never executes" rule.
- `hide_marked_spans`: use the case-insensitive pair pattern. After substitution, if any
  of this guard's markers remain (orphaned or unbalanced), raise
  `GeneratedTextLeakError` instead of returning corrupted text.
- Add a module function
  `assert_no_generated_text_residue(text: str, *, boundary: str) -> None`.
  - It raises `GeneratedTextLeakError` if `text` contains interpolation-marker text or
    guard-token text from **any** guard: `\x00SJSMARK_` (any case) or `\x00SASEDATA_`.
  - Fast path: return immediately when `"\x00" not in text`.
  - The message names the boundary, calls this an internal invariant violation, and says
    the prompt was not launched. It must not print the nonce-bearing text.
  - Call it only at true outermost boundaries (section 4). Nested calls can legitimately
    carry an outer guard's tokens.

### 2. Both marking `finalize` hooks use `guard.mark`

- `src/sase/macro/_jinja.py` `_render_jinja2_template`: `_marking_finalize` returns
  `guard.mark(value)`. Wrap the `guard.hide_marked_spans(rendered)` call so a
  `GeneratedTextLeakError` is re-raised as
  `MacroArgumentError(f"Macro '#{macro_name}' template error: {e}")`, with the original
  chained.
- `src/sase/macro/source_render.py` `_marking_env`: `_marking_finalize` returns
  `guard.mark(finalized)`. This is the twin workflow and early-preprocessing renderer,
  and it has the same nesting bug.
- Drop any `interpolation_markers` imports that become unused (ruff and symvision).
- Update the `_marking_env` docstring and the `hide_marked_spans` docstring to state the
  flattening and fail-loud contract.
- First rebase onto the latest master and grep `src/sase` for `interpolation_markers(`
  and `finalize=`. The in-progress phase `sase-1tc.1.1.3` ("Preserve provenance across
  every workflow rendering route") may have added marking sites. Route every one of them
  through `guard.mark`.

### 3. Reveal this boundary's tokens before validating arguments (`src/sase/macro/_processor_render.py`)

In `expand_single_macro`, after resolving `guard` and before
`validate_and_convert_args`:

- Build revealed arguments:
  - `revealed_positional = [guard.unwrap(a) for a in positional_args]`
  - `revealed_named = {k: guard.unwrap(v) for k, v in named_args.items()}`

  Validate and convert these instead of the raw lists. Render with the converted values
  exactly as today.

  This is safe for three reasons:
  - The engine already ran `_resolve_command_substitution_in_args` before this call.
  - Validation never executes anything.
  - Every value the template outputs is re-marked by the same guard: through `{{ }}` and
    `guard.mark` on the Jinja path, or through `guard.wrap` for legacy positionals.

  `guard.unwrap` restores only this guard's tokens. Tokens from another guard stay
  opaque. This is intentional (see Non-goals).

- **Do not widen the local-helper channel.** `_expand_local_macro_references` passes the
  converted arguments to local helpers as scope. That nested call mints its own guard
  and unwraps on return. So a revealed value rendered by a helper would come back to the
  outer pass as plain text, and the outer first pass could then run a `$(...)` inside
  it.

  Build `revealed = {rev for raw, rev in pairs if rev != raw}` over both positional and
  named pairs. Pass the helpers re-hidden copies through a small `_rehide` helper:
  - a `str` value in `revealed` becomes `guard.wrap(value)`;
  - `list` and `tuple` recurse;
  - a `CodeValue` (`sase.macro.code_value`) whose `source` is in `revealed` becomes
    `dataclasses.replace(value, source=guard.wrap(value.source))`;
  - everything else passes through (`bool`, `int`, `float`, `None` cannot carry
    commands).

  Helpers therefore receive exactly the protection they had at `be6adc20d0`.

### 4. Residue checks at the outermost boundaries, and clean trace evidence

- `src/sase/agent/_macro_swarm_rendering.py` `render_macro_swarm`: after
  `expand_single_macro(...)` returns and before `split_segments_protecting_fences`, call
  `assert_no_generated_text_residue(substituted, boundary=f"macro swarm #{macro_def.name}")`.
  With this check, the apollo failure would have stopped the launch with an error
  instead of launching 6 agents.
- `src/sase/llm_provider/preprocessing.py` `preprocess_prompt_early`: after the final
  `prompt = guard.unwrap(prompt)`, call
  `assert_no_generated_text_residue(prompt, boundary="prompt preprocessing")`. This also
  covers `sase macro expand` and agent launches.
- `src/sase/macro/_processor_engine.py` `_expansion_fixpoint`: record trace evidence
  unwrapped with this pass's guard:
  - `positional_args=[guard.unwrap(a) for a in positional_args]`
  - `named_args={k: guard.unwrap(v) ...}`
  - `expanded_text=guard.unwrap(expanded)`

  Do not change the `expanded` value that is spliced into the prompt.

## Tests

Put new tests in a new file, `tests/test_cmdsub_guard_nesting.py`.
`tests/test_cmdsub_source_provenance.py` is already 660 lines. Reuse the patterns in
that file (the `_catalog`, `_human`, `_agent`, and `counting_executor` fixtures) and
`tests/_macro_swarm_helpers.py` (`expand_macro_swarms`, `patch_catalog`, `xp`).

1. **Marker flattening round-trips.** Parametrize over the five Jinja shapes in the
   reproduction table (macro in `{{ }}`, `{% set %}` block, `{% call %}`/`caller()`,
   `{% filter upper %}`, `{{ t() | upper }}`). Run each through both:
   - `_render_jinja2_template(..., guard=g)` followed by `g.unwrap(...)`;
   - `render_source_template(..., provenance=generated_provenance(), guard=g)` followed
     by `g.unwrap(...)`.

   Assert the expected text and that it contains no `\x00`.

2. **Swarm keeps every segment.** Build a synthetic swarm macro shaped like
   `#research_swarm`:
   - a `{%- macro tree(files) -%}` whose body prints its fences through
     `{{ "```text" }}` and `{{ "```" }}`;
   - a lead segment that calls `{{ tree(["a.md", "b.md"]) }}`;
   - two more `---` segments, each starting with `%if(should_run={{ img }})`, where
     `img` is a bool input defaulting to true.

   `expand_macro_swarms(["#!sw"])` must return all 3 segments. The fenced tree must be
   intact in segment 1, and no segment may contain `\x00`. Add a second case with
   `img=false` so the `%if` filter still drops both conditional segments.

3. **Typed arguments from interpolation.** Through
   `process_macro_references_with_catalog`:
   - the word case returns `T[cdx] T[cld] `;
   - the int case returns `N=4`;
   - the callee conditional returns `YES`;
   - a `#research`-style `{% elif suffix and suffix != 'false' %}` branch is taken for
     `suffix={{ "cdx" }}` and not taken for `suffix={{ "false" }}`.

   Also run the word case through `preprocess_prompt_early`, patching
   `sase.macro.processor.get_all_macros` as
   `test_macro_command_assembled_from_adjacent_data_stays_literal` does. That mirrors
   the failing `sase macro expand` path.

4. **Revealing never makes data executable.** Use `counting_executor` and `tmp_path` in
   a human launch:
   - A typed `text` argument whose interpolated value is `$(touch marker)` is validated
     and rendered, but `marker` is not created and the text stays literal.
   - **Local-helper non-widening.** `outer` (content `#main(v={{ data }})`, scope
     `data="#echo($(touch helper-marker))"`) → `main` (input `v: text`, content `#_h`,
     `local_macros={"_h": Macro(content="{{ v }}")}`), plus a global `echo`. The marker
     is not created. This test fails if `_rehide` is removed (verified in a prototype),
     so keep it as the guard for section 3.
5. **Fail loudly.**
   - `GeneratedTextGuard.hide_marked_spans` on text containing a lone start marker
     raises `GeneratedTextLeakError`.
   - `assert_no_generated_text_residue` raises for both a foreign `\x00SASEDATA_x_0\x00`
     and an uppercased `\x00SJSMARK_...` marker, and passes for clean text and for text
     with ordinary NUL-free content.
   - `render_macro_swarm` raises `GeneratedTextLeakError` (not a silent mis-split) when
     `expand_single_macro` is patched to return residue.
6. **Trace evidence is clean.** For the `#outer` → `#g(who={{ 'world' }})` case, every
   `trace.records[*].expanded_text`, `positional_args`, and `named_args` is free of
   `\x00`, and `#g`'s record shows `who == "world"`.

Existing suites that must stay green include:

- `tests/test_cmdsub_source_provenance.py`
- `tests/test_macro_swarm_expansion.py`
- `tests/test_prompt_authorship_safety.py`
- `tests/test_preprocessing_code_blocks.py`
- `tests/test_disabled_regions.py`
- `tests/test_macro_processor_args.py`
- `tests/test_macro_jinja_and_standalone.py`
- `tests/test_file_references_substitution.py`
- `tests/test_macro*.py`, `tests/test_*swarm*.py`, `tests/test_*preprocess*.py`, and
  `tests/test_multi_prompt*.py`

A scratch prototype of sections 1–3 passed all 1,381 tests in those globs. It covered
case-insensitive flattening and hiding plus revealing and `_rehide` for `str` and
`list`, but not the residue helper or `CodeValue`. It fixed every reproduction row
except the trace row, which section 4 addresses. With `_rehide` disabled, the
local-helper case in test 4 executed `touch`; with `_rehide`, it stayed literal, as it
does on master.

## Verification

1. Run the new file and the suites above with `pytest` from the checkout's `.venv`.
2. Run the repo's agent gate, `just check`, through `sase tool run check`, as
   `sase memory read lint_and_test.md` instructs. Do not run `just check-full`.
3. Optional end-to-end check. Do this only if the checkout's `.venv` has a
   `sase-research-artifacts` plugin whose `#research_swarm` contains `macro tree` (check
   with `sase macro show research_swarm --format raw | grep "macro tree"`). With
   `SASE_PROMPT_AUTHORSHIP=agent` set, so that no `$(...)` runs, call
   `expand_macro_swarms_with_metadata(["#research_swarm(prompt=[[x]], image=true)"])`.
   It must yield the lead, `image`, and `linker` segments with no `\x00`. Do not
   globally reinstall sase or the plugin.

## Non-goals and coordination

- **Do not touch `sase-research-artifacts`.** `{{ tree(lead_files) }}` is valid Jinja
  and must keep working. Fixing the engine is the fix.
- **Local-helper guard isolation.** Two problems are already recorded as a
  `DISCOVERED ISSUE` note on epic `sase-1tc.1.1`, and their fix belongs to that epic's
  nested-guard design:
  - command stdout minted inside a local helper is unwrapped on return and can execute
    in the outer pass;
  - typed local-helper arguments fed by a _parent's_ interpolation still see the parent
    guard's tokens.

  This tale must not make either worse. Section 3's `_rehide` and test 4 enforce that.

- **Known remaining limits** (no shipped macro hits them). Jinja comparisons or `length`
  on a `{% set %}` block capture or a macro's return value still see the inner markers.
  Slicing a marked value can split a marker. Fixing either would need a Jinja
  compiler-level change. Mention both in the `mark()` docstring.
- **Operations are the user's call, not this tale's.** The degraded in-flight swarms
  `research.58` and `research.59` on apollo, and reinstalling or rerunning after this
  lands.
- `CHANGELOG.md` is release-generated, so do not edit it. No memory notes or docs become
  false.
