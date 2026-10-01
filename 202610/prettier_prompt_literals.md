---
tier: tale
title: Stop launch-time prettier from rewriting _ and * literals in agent prompts
goal:
  Agents receive every authored and runtime-filled `_` / `*` character exactly as
  written (research-swarm handoff labels and paths, dunders, globs, Jinja), while agent
  prompts stay prettier-reflowed and `sase-1dx` is closed.
size: medium
proposed_by: bbugyi200.athena.0v0
create_time: 2026-10-01 15:12:31
status: wip
---

# Plan: Stop launch-time prettier from rewriting `_` / `*` literals in agent prompts

## Goal

Fix the defect described in the "Prettier corrupts the handoff list at launch" section
of
`research:202610/research_swarm_improvement_roadmap/research_swarm_improvement_roadmap.md`
(tracked as bug bead `sase-1dx`), in both places the report names:

1. **Plugin (`sase-research-artifacts`)**: render every field of the `#research_swarm`
   `wait.artifacts` handoff loops as inline code, so the lead and linker get exact
   labels and paths on any host sase version.
2. **sase**: make the shared agent-prompt Markdown formatter
   (`format_agent_prompt_markdown`) preserve every inline `_` and `*` exactly, while
   keeping prettier's whitespace reflow and block normalization.

## Root cause (reproduced 2026-10-01, prettier 3.8.1, sase `36df0770f0`)

- `preprocess_prompt_late` in `src/sase/llm_provider/preprocessing.py` renders top-level
  Jinja (step 5), which fills the swarm's `{% raw %}` `wait.artifacts` loops. It then
  runs `format_agent_prompt_markdown` (step 6) in `src/sase/file_references.py`, which
  calls `format_with_prettier`.
- Prettier's Markdown parser treats runs of `_` and `*` as emphasis delimiters,
  including inside words. It then normalizes the emphasis it found (`__x__` becomes
  `**x**`, and `*x*` becomes `_x_`). `format_with_prettier` only undoes `_` escapes, so
  it cannot repair this.
- Inline code and fenced blocks are safe at launch: `protect_fenced_blocks` swaps them
  for `\x00XPF_n\x00` placeholders before step 6. That is why backticked fields survive.
- The ACE prompt editor's `gf` / `Ctrl+G f` action calls the same
  `format_agent_prompt_markdown` (`src/sase/ace/tui/widgets/_prompt_format.py`), so it
  corrupts the same text.

The corruption is wider than paired `__`. Running `sase xprompt expand` today reproduces
all of these:

| Authored or rendered text                     | What the agent receives                       |
| --------------------------------------------- | --------------------------------------------- |
| `topic__cdx.md … topic__cld.md`               | `topic**cdx.md … topic**cld.md`               |
| `/x/gh_sase-org__sase/a__cdx.md`              | `/x/gh_sase-org**sase/a**cdx.md`              |
| `__init__.py`, `self.__dict__`                | `**init**.py`, `self.**dict**`                |
| `a___b and c___d`                             | `a***b and c***d`                             |
| `src/*.py and tests/*.py`                     | `src/_.py and tests/_.py`                     |
| `x * y * z`                                   | `x _ y _ z`                                   |
| `Snake_case_name and _leading and trailing_`  | `Snake*case_name and _leading and trailing*`  |
| `5 * 6`                                       | `5 \* 6`                                      |
| `{%- set _ = ns.layout_lines.append(...) -%}` | `{%- set * = ns.layout*lines.append(...) -%}` |

## Decision and rejected alternatives

Protect inline `_` and `*` with width-preserving Unicode private-use sentinels before
prettier runs, and restore them afterwards, inside `format_agent_prompt_markdown` only.
Prettier still reflows prose and normalizes block structure: list markers, thematic
breaks, headings, and tables. It can no longer see or rewrite inline emphasis
characters.

Prototype evidence: a Python prototype of the algorithm below ran over 142 real Markdown
inputs, including sase xprompts, `docs/`, the plugin xprompts, and 60 launched
`*main_prompt.md` prompts. It preserved every non-whitespace character except prettier's
block-level normalizations. It also wrapped by true width; raw prettier mis-wraps lines
whose `_` escapes are later stripped. Every case in the table above came back
byte-for-byte.

Rejected alternatives (the report's other two SASE options, plus a narrower one):

- **Format the authored prompt before runtime values are substituted.** This does not
  protect authored prose (dunders, globs, or the swarm's `{{ prompt }}` request text).
  It would also require reordering late-phase steps 2–5, which all inject values.
- **Protect only `__…__` spans.** This leaves the `*`-glob, cross-token `_…_`, `\*`, and
  Jinja corruptions above.
- **Drop formatting from the launch path.** This removes a deliberate behavior. Commit
  `5da193482e` pinned the launch prompt, the editor's `gf`, and the published prompt
  document to one wrap policy. It would also leave `gf` corrupting the same text. The
  chosen fix removes no behavior, so it needs no feature flag: there is no old branch to
  keep reachable.

Scope boundary: do **not** change `format_with_prettier` itself. Plans, SDD files,
generated skills, and notifications go through it, and they must stay byte-identical to
the plain prettier CLI (`just fmt-md`, CI prettier checks). Changing the shared helper
would create formatter drift. The fix stays in Python because sase-core owns no Markdown
formatting: the transform must wrap the existing Python prettier subprocess call.

## Part 1 — `sase-research-artifacts` plugin

Open the linked repo with `sase repo open sase-research-artifacts -r "<why>"`. Read its
`AGENTS.md`, and make every edit in the printed checkout.

### 1a. Backtick the handoff fields

In `src/sase_research_artifacts/xprompts/research_swarm.md`, change the three loop
bodies. Keep the `{% raw %}…{% endraw %}` wrappers, the `for … if …` filters, and the
`{% endfor %}` lines exactly as they are.

- Lead segment, markdown loop:
  ``- wait_name=`{{ a.wait_name }}` label=`{{ a.label }}` source_path=`{{ a.source_path }}` path=`{{ a.path }}` ref=`{{ a.ref }}` ``
- Linker segment, markdown loop: the same line.
- Linker segment, image loop:
  ``- wait_name=`{{ a.wait_name }}` label=`{{ a.label }}` vcs_relpath=`{{ a.vcs_relpath }}` path=`{{ a.path }}` ref=`{{ a.ref }}` ``

The surrounding step text already refers to these fields by name (`wait_name`, the
label's `__<suffix>.md`, the `ref` field's `file:<id>`), so the instructions need no
rewording.

### 1b. Tests (`tests/test_xprompt_loading.py`)

- Update the `_WAIT_ARTIFACTS_LOOP` and `_WAIT_IMAGE_ARTIFACTS_LOOP` constants to the
  new loop text, character for character. `_without_wait_artifacts_loop` strips them
  before the `"{%" not in …` assertions, so a mismatch breaks many tests.
- Update `test_research_swarm_lead_lists_wait_artifacts_not_transcripts` to assert the
  backticked field templates, for example ``"wait_name=`{{ a.wait_name }}`"``.
- Update `test_research_swarm_lead_renders_registered_reports_via_wait_artifacts` and
  `test_research_swarm_linker_renders_registered_lead_via_wait_artifacts` to expect
  backticked rendered values, for example
  ``"wait_name=`research.m.cdx` label=`research:202609/topic/topic__a.md`"``,
  ``f"source_path=`{report_a}`"``, ``f"path=`{artifact_a.path}`"``,
  ``f"ref=`file:{artifact_a.id}`"``, and
  ``"vcs_relpath=`202609/topic/topic_infographic.png`"``.
- Add a structural regression test that runs everywhere. Render the lead and linker
  segments, with image enabled, against artifacts whose labels and paths contain `__`,
  such as `research:202610/t/t__cdx.md` and a path under `gh_sase-org__sase`. Assert
  that every `wait_name=`, `label=`, `source_path=`, `vcs_relpath=`, `path=`, and `ref=`
  in the rendered loop lines is immediately followed by a backtick.
- Add a formatter regression test, skipped when `shutil.which("prettier") is None`. Pass
  that rendered lead text through `sase.file_references.format_agent_prompt_markdown`
  and assert that every label and path survives byte-for-byte. Because the fields are
  backticked, this passes against both the current and the fixed host formatter.

### 1c. Docs

In `docs/xprompts.md`, the paragraph about the lead's `wait.artifacts` loop lists the
printed fields. Add one sentence: each field value renders as inline code, so
launch-time Markdown formatting cannot rewrite the `__` in labels and paths. Do not edit
`CHANGELOG.md` (release-please owns it). Do not bump the `sase` floor: backticks work on
every host version.

## Part 2 — sase: literal-preserving agent-prompt formatter

### 2a. New module `src/sase/markdown_literals.py`

Add a small, dependency-free module. Like `markdown_width.py`, it imports nothing from
`sase` at module scope. Public API:

```python
@dataclass(frozen=True, slots=True)
class EmphasisSentinels:
    underscore: str
    star: str
    glue: str

def protect_emphasis_literals(text: str) -> tuple[str, EmphasisSentinels]: ...
def restore_emphasis_literals(text: str, sentinels: EmphasisSentinels) -> str: ...
```

Algorithm. Keep it line-based and order-sensitive, exactly as below.

1. **Choose sentinels.** Pick three distinct code points from the Private Use Area
   (U+E000–U+F8FF), scanning upward from U+E000 and skipping any code point that already
   occurs in `text`. Each sentinel is one code point. Prettier measures PUA characters
   as width 1, so wrapping width is unchanged.
2. **Split on `"\n"`** and treat each line independently. Treat `\r` as whitespace in
   the lookaheads below.
3. **Thematic-break lines are left untouched.** These are lines matching
   `^ {0,3}(?:(?:\*[ \t]*){3,}|(?:_[ \t]*){3,})\r?$`, such as `***`, `___`, and `* * *`.
4. **Leading bullet marker stays real.** If the line matches
   `^([ \t]*(?:>[ \t]*)*)\*(?=[ \t\r]|$)`, keep that prefix, including the `*`,
   unchanged, and protect only the remainder of the line. Prettier may still normalize
   `*` bullets to `-`; that is semantic-preserving block formatting.
5. **Glue standalone star runs.** In the remainder, replace the single space in
   `(?<=\S) (?=\*+(?:[ \t\r]|$))` with the `glue` sentinel. This covers a run of `*`
   that is preceded by exactly one space after a non-space character and is followed by
   whitespace or the end of the line. Without the glue, prettier could wrap `* …` onto
   the start of a line, and restoring it there would create a list item.
6. **Protect the rest.** Replace every remaining `_` with `underscore` and every
   remaining `*` with `star`.
7. **Restore** maps `glue` back to `" "`, `underscore` back to `"_"`, and `star` back to
   `"*"`.

Invariants to document in the module docstring:

- `restore(protect(x)) == x` for every string.
- The protected text contains no `_` or `*` outside bullet prefixes and thematic-break
  lines.
- Protection is bijective inside code spans, fences, URLs, HTML, YAML front matter, and
  the `\x00XPF_n\x00` placeholders, so those regions also round-trip exactly.

### 2b. Wire it into `format_agent_prompt_markdown` (`src/sase/file_references.py`)

```python
def format_agent_prompt_markdown(text: str) -> str:
    protected, sentinels = protect_emphasis_literals(text)
    return restore_emphasis_literals(format_with_prettier(protected), sentinels)
```

- Call `format_with_prettier` by its module-global name, as today. Existing tests
  monkeypatch `sase.file_references.format_with_prettier`, so those patches still take
  effect. The argv-equality test
  (`test_agent_prompt_formatter_matches_default_formatter_argv`) must keep passing
  unchanged.
- Fallback stays correct with no extra code. When prettier is missing, fails, or times
  out, `format_with_prettier` returns the protected text, and restoring it yields the
  original.
- Intended behavior change: an authored `_` in an agent prompt now survives, because
  `format_with_prettier`'s unescape loop never sees a protected `_`. Pin this in a test.
- Rewrite the docstring. Agent-prompt formatting is prettier reflow plus block
  normalization, but inline `_` / `*` characters are preserved literally. Launch-time
  preprocessing, the editor `gf` action, and the published prompt document still share
  this one helper. Plain `format_with_prettier` intentionally keeps raw prettier parity
  for repo-tracked Markdown.

### 2c. Use the same policy for the published prompt document

`src/sase/agents_sync/prompt_archive/render.py` formats the published prompt document,
which is the agent's authored `raw_xprompt.md` plus plan-header link sections, with
`format_with_prettier`. Switch that call to `format_agent_prompt_markdown`. The archive
is an agent prompt, and its artifact labels routinely contain `__`. The existing tests
in `tests/agents_sync/test_prompt_archive_publish.py` monkeypatch
`sase.file_references.format_with_prettier` to identity; confirm they still pass.

### 2d. Docstring touch-up

In `src/sase/llm_provider/preprocessing.py`, update the step-6 docstring line and
comment in `preprocess_prompt_late` to say the formatting is the literal-preserving
agent-prompt policy. No behavior change there.

### 2e. Tests (new `tests/test_markdown_literals.py`, plus additions to `tests/test_format_with_prettier.py`)

These run without prettier, so CI lacks it and still covers them:

- Round-trip identity over a parametrized corpus. Include every row of the table above,
  bullets (`* a`, `  * b`, `> * c`, `* [ ] d`), thematic breaks, tables, a fenced block,
  an inline code span, front matter, Jinja, `\x00XPF_0\x00` placeholders, CRLF lines,
  and text that already contains U+E000–U+E002. The last case proves that sentinel
  selection skips occupied code points.
- Protected-text shape: no `_` or `*` remains except bullet prefixes and thematic-break
  lines, and glue appears only before standalone star runs.
- `format_agent_prompt_markdown` with a fake `subprocess.run` that simulates prettier's
  rewrites: it replaces `__` with `**`, replaces `*` with `_`, and adds `\*` escapes.
  The original literals must come back intact.
- `format_agent_prompt_markdown` when prettier is missing, fails, or times out returns
  the input exactly.
- An authored `_` survives `format_agent_prompt_markdown`.

These need real prettier and are skipped when `shutil.which("prettier") is None`:

- Every row of the table, through both `format_agent_prompt_markdown` and
  `preprocess_prompt_late(..., file_ref_mode="skip")`, comes back unchanged. Include a
  `wait.artifacts`-style list item with bare `__` labels, which is the bead's repro.
- Block formatting still applies. For example, `+ item` and `* item` lists normalize,
  and a long paragraph wraps at `markdown_print_width()`.
- Wrap parity: a long paragraph that repeats intraword-only words such as
  `snake_case_name other_word x_y` wraps at exactly the same line breaks as raw
  `format_with_prettier`. First assert the precondition that raw prettier leaves those
  words unchanged; it does with prettier 3.8.1.
- A standalone `*` placed exactly at a wrap boundary never starts an output line.
- Line joining keeps a space: `"ends foo_bar\nbaz__qux"` becomes
  `"ends foo_bar baz__qux"`.

Existing suites that must stay green: `tests/test_format_with_prettier.py`,
`tests/ace/tui/widgets/test_prompt_ordered_formatter_agreement.py` (uses real prettier),
`tests/ace/tui/widgets/test_prompt_format.py`,
`tests/test_preprocessing_code_blocks.py`, `tests/test_disabled_regions.py`,
`tests/test_fork_workflow.py`, and `tests/agents_sync/test_prompt_archive_publish.py`.

## Verification

1. Before the fix, confirm the repro:
   `printf 'Compare topic__cdx.md with topic__cld.md and update __init__.py and src/*.py and tests/*.py.\n' | sase xprompt expand`
   prints `topic**cdx.md`, `**init**.py`, and `src/_.py`. After the fix, the same
   command prints the input literals unchanged.
2. Run the targeted pytest files above with the workspace virtualenv.
3. Run `sase tool run check` in the sase checkout and again inside the
   `sase-research-artifacts` checkout. Do not run `check-full`.
4. After both repos pass, close the bug with
   `sase bead close sase-1dx --note "<what you verified>"`.

## Out of scope

- Raw `format_with_prettier` callers (plans, SDD files, generated skills, notifications)
  can still rewrite bare `__` in prose. Fixing them needs a parity story with
  `just fmt-md` and CI prettier checks. If you think it is worth tracking, file it
  through `/sase_new_task`; do not widen this change.
- The roadmap's other recommendations: the researcher delegation contract, the lead
  adjudication protocol, measurement, failure handling, and the rest.
