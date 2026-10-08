---
tier: tale
title: De-duplicate
goal:
  The research_swarm macro is about 30% shorter, authors its queue directive,
  research-report loop, and researcher segment once each, and renders byte-identical
  prompts except a fixed solo-lead layout.
size: medium
proposed_by: bbugyi200.athena.0yh
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0yh](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.0yh.md)
- **COMMITS:**
  - [94938a9](https://github.com/sase-org/sase-research-artifacts/commit/94938a91df0f4673540744d718cb3bda5b76830f)
    — refactor(research-swarm): de-duplicate swarm with local helpers and a researcher
    loop

# De-duplicate `#research_swarm` with local helpers and a researcher loop

## Goal

Shrink and de-duplicate the `#research_swarm` macro swarm in the linked
`sase-research-artifacts` repo
(`src/sase_research_artifacts/xprompts/research_swarm.md`). Use file-local helper macros
(local xprompts) where they fit, and Jinja where the repetition is driven by derived
data. Every launched agent must see the same prompt as today, with one intentional
exception: the solo-lead layout fix described below.

All edits are in that linked repo. Open it with
`sase repo open sase-research-artifacts -r "<reason>"` and read its `AGENTS.md` before
editing. The sase repo itself needs no changes.

## What Local Helpers Can And Cannot Do (verified on sase master)

- Helpers declared under the Markdown frontmatter `macros:` key expand right after the
  outer macro's Jinja render (`expand_single_macro` → `_expand_local_macro_references`
  in `sase.macro.processor`). The swarm launch path
  (`sase.agent._macro_swarm_rendering.render_macro_swarm`) calls the same function
  before it splits segments.
- Helpers inherit the outer macro's bound inputs, including defaults
  (`_scope_for_local_macros`). They do **not** see the outer body's `{% set %}`
  variables (`researchers`, `run_linker`, layout lists). Helper arguments are
  macro-argument strings, so a list cannot be passed to a helper.
- The helper pass leaves unknown references (`#research(...)`, `#fork:...`,
  `#research/image`, `#tags` in user prompts) untouched. `%if(should_run=False)`
  segments are filtered before the helper pass runs.
- Use the canonical `macros:` key. Current sase rejects the retired `xprompts:` key
  unless a legacy flag is on. Published sase ≤ 0.17.1 only read `xprompts:`, but the
  plugin's floor (`sase>=0.17.2`) is satisfied only by releases cut after the rename, so
  the floor does not change.

Where each mechanism fits:

- **Local helpers** hold text snippets that depend only on the macro's inputs and are
  repeated across segments: the queue directive and the raw research-report loop.
- **Jinja loops and `{% macro %}`** handle repetition driven by derived data: the
  researcher fan-out and the layout trees.

Alternatives considered and rejected:

- **One `_researcher` helper per segment.** Peers and the model would have to be
  string-encoded into macro arguments and split again inside the helper, and about 20
  lines of prose would move into a YAML block scalar. A Jinja loop is shorter and keeps
  the prose in Markdown.
- **An `_after_lead` helper** for the one-line image/audio directive header, and a
  helper for the two-line "SASE derives…" note. Each adds indirection without saving
  meaningful text.

## Duplication Today

1. Five near-identical researcher segments of about 23 lines each. Each segment repeats
   the enable predicate that `researchers` already computes.
2. The `%q(...)` queue template is written out 9 times.
3. The raw `wait.artifacts` research-report loop appears twice (lead and linker).
4. The lead's "do not create `<name>.md`" note, step-5 registration, and "Final layout"
   are duplicated across the with-researchers and solo branches. The solo copy
   hard-codes a fenced layout. Fenced blocks are protected from Jinja, so its
   `{{ lead_report }}` renders literally (`└── {{ lead_report }}`), and it omits the
   narration file when `audio=true`.
5. Two imperative `namespace` loops build near-identical lead and linker layout trees.

## Target Design

### 1. Frontmatter: add `macros:` after `input:` (inputs unchanged)

```yaml
macros:
  _queue:
    description: Runner-queue admission shared by every swarm member.
    content: |-
      %q({% if runners is not none %}{{ runners }}{% else %}1.5x{% endif %}, w=0.25{% if priority is not none %}, priority={{ priority }}{% endif %})
  _report_records:
    description:
      Launch-time list of the research reports registered by the awaited agents.
    content: |-
      {% raw %}{% for a in wait.artifacts if a.kind == "markdown" and a.label and a.label.startswith("research:") %}
      - wait_name={{ a.wait_name }} label={{ a.label }} source_path={{ a.source_path }} path={{ a.path }} ref={{ a.ref }}
      {% endfor %}{% endraw %}
```

`_queue` content is the exact current template; tests pin it via
`_WEIGHTED_QUEUE_TEMPLATE`.

### 2. Preamble

Keep the `researchers`, `run_linker`, and `lead_report` set statements verbatim. Replace
the two `namespace` layout loops (`layout_body`, `linker_layout_body`) with:

````jinja
{%- set drafts = [] -%}
{%- for r in researchers %}{% set _ = drafts.append("<name>__" ~ r.short ~ ".md") %}{% endfor -%}
{%- set narration = ["<name>_narration.md"] if audio else [] -%}
{%- set lead_files = drafts + [lead_report] + narration -%}
{%- set linker_files = drafts + ["<name>__final.md"] + (["<name>_infographic.png"] if image else []) + ["<name>.md"] + narration -%}
{%- macro tree(files) -%}
{{ "```text" }}
<month-dir>/<name>/
{%- for f in files %}
{{ "└──" if loop.last else "├──" }} {{ f }}
{%- endfor %}
{{ "```" }}
{%- endmacro -%}
````

Fence lines must be emitted through string expressions. A literal fence in the body is
protected from Jinja, which is what causes the solo-branch bug.

### 3. Researchers: one loop replaces the five segments

```jinja
{%- for r in researchers %}
{%- set peers = researchers | rejectattr("short", "equalto", r.short) | list %}
%id({{ r.short }}, clan=research.{@1})
%m:{{ r.model }} {% if wait %}%wait:{{ wait }} {% endif %}#_queue

You are researcher {{ r.short }} in a {{ researchers | length }}-researcher swarm.
{% if peers -%}
<the existing "The other {{ ... }}" line, verbatim except `__cdx.md` → `__{{ r.short }}.md`>
{% else -%}
You are the only independent researcher in this swarm. Your report will end in `__{{ r.short }}.md`.
{% endif %}
<the existing 9-line "Conduct your research independently ..." paragraph, verbatim>

{{ prompt }} #research(suffix={{ r.short }})

---
{% endfor %}
```

The loop needs no `%if(should_run=...)` gate. It visits only researchers whose flag is
true and whose provider is not hard-disabled, which is the same set today's `%if` gates
select.

### 4. Lead segment

- Second directive line:
  `{% for r in researchers %}%wait:research.{@1}.{{ r.short }} {% endfor %}#_queue`.
- Replace the raw research-report loop under "The researchers' registered reports:" with
  a line containing only `#_report_records`.
- Restrict the `{% if researchers %}` / `{% else %}` split to steps 1–4. Hoist the
  shared tail after it:

```
{% if researchers -%}
1. … (unchanged)
4. Write the consolidated report to `<name>/{{ lead_report }}`: merge … without unnecessary length.
{%- else -%}
1. No independent researcher reports … (unchanged)
4. Write the consolidated report to `<name>/{{ lead_report }}` based on your own research,
   adding missing critical context without unnecessary length.
{%- endif %}
{%- if run_linker %}

   Do not create `<name>/<name>.md`, … (unchanged, now written once)

5. After the write succeeds, … (unchanged, now written once)
{%- endif %}

Final layout:

{{ tree(lead_files) }}
---
```

Delete the second copy of the linker note, step 5, "Final layout", and the hard-coded
solo fence.

### 5. Image, linker, and audio segments

Keep their `%if(should_run=...)` lines unchanged.

- Image directive line:
  `%wait:research.{@1}.final #_queue #fork:research.{@1}.final #research/image`
- Linker:
  - Directive line ends with
    `... {% if audio %}%wait:research.{@1}.audio {% endif %}#_queue`.
  - Replace its research-report loop with `#_report_records`. The image and audio
    `wait`/`agents` loops are used only once and stay inline.
  - Replace its final layout expression with `{{ tree(linker_files) }}`.
- Audio directive line:
  `%wait:research.{@1}.final #_queue #fork:research.{@1}.final #research/audio(edition={{ audio_edition }})`

Expected result: the body shrinks from about 440 lines to about 315. The source writes
the `%q(` template once (in `_queue`) and the research-report loop once.

## Behavior Contract

`expand_single_macro(macro, [prompt], args, preserve_segment_separators=True)` must
produce byte-identical output to the pre-change file for every argument combination,
with one exception. When no researcher runs, the lead's "Final layout" block now renders
the real tree instead of the literal `└── {{ lead_report }}`:

- `└── <name>.md` without the linker.
- `└── <name>__final.md` when the linker runs.
- With `audio=true`, the lead file line becomes `├── …` and is followed by
  `└── <name>_narration.md`.

A prototype of exactly this design was checked during planning:

- **Render matrix:** 14 argument combinations rendered; 10 were byte-identical, and the
  4 solo cases differed only in that block.
- **Plugin suite:** only the three raw-source tests listed below failed.
- **sase e2e:**
  `tests/fakey/test_runner_slots_e2e.py::test_installed_research_swarm_quarter_weights_fill_one_fakey_capacity_unit`
  passed.
- **Launch planning:** `plan_typed_launch_units` planned 3, 9, and 3 units with no
  diagnostics for the default, all-opt-in, and solo+audio prompts.

### Verification Harness (temporary; do not commit)

1. Before editing, save
   `git show HEAD:src/sase_research_artifacts/xprompts/research_swarm.md` to a temp
   file.
2. Load the old and new files with `sase.macro.loader_sources.load_macro_from_file`. Use
   the plugin's own `.venv` so the `sase-research-artifacts@audio_edition` input type
   resolves, and set `SASE_HOME` to an empty temp dir so machine-wide provider disables
   don't leak in. If loading returns `None`, wrap it in
   `sase.macro.load_issues.collect_macro_load_issues()` to see why.
3. Render both files with
   `expand_single_macro(..., preserve_segment_separators=True, raise_on_error=True)`
   across this matrix:
   - defaults
   - `grok`+`muse`+`gemini`
   - `claude=false` (single peer)
   - `codex=false claude=false`: alone, `+linker`, `+audio`, and `+image+audio`
   - `image`
   - `audio audio_edition=full`
   - `image audio` with all five researchers
   - `linker`
   - `wait=… priority=5 runners=8`
   - `wait="a,b" priority=0 runners=0 image audio`
   - custom models for every role
4. Use a prompt containing `#tags`, `%`, and `{braces}`.
5. Compare with `command diff`, because a shell alias can hide `diff`'s exit code. Only
   the documented solo differences are allowed.

## Test Updates

All in `tests/test_macro_loading.py` unless noted.

- **`test_research_swarm_has_nine_top_level_segments`** (splits the raw source): replace
  it with a rendered test. With all five researchers plus `image` and `audio`, the swarm
  renders nine segments in the order cdx, cld, grk, mus, gem, final, image, linker,
  audio.
- **`test_research_swarm_dependency_graph_preserved`**: rewrite it against those nine
  rendered segments. Assert:
  - each segment's `%id(...)` and clan;
  - the lead's `%clan(` declaration and its waits on all five researchers;
  - image and audio wait on and fork from final and run `#research/image` /
    `#research/audio(edition=brief)`;
  - the linker waits on final, image, and audio and has no `#fork:`;
  - exactly one `%q(` per segment;
  - raw artifact-loop placement: the research-report loop appears in final and linker,
    the image/audio loops only in linker, and none in audio.

  Keep authored-source assertions only where they still mean something:
  - `xp.content` still contains `%if(should_run={{ image }})`,
    `%if(should_run={{ run_linker }})`, and `%if(should_run={{ audio }})`;
  - `"%q(" not in xp.content` and
    `xp.local_macros["_queue"].content == _WEIGHTED_QUEUE_TEMPLATE`;
  - `xp.local_macros["_report_records"].content` contains `_WAIT_ARTIFACTS_LOOP`.

- **`test_research_swarm_lead_lists_wait_artifacts_not_transcripts`** and
  **`test_research_swarm_lead_mentions_artifact_read_derivation`**: assert on the
  rendered final segment from `_swarm_segments({})`. The second test currently passes
  only because of the raw `---` split.
- **`_authored_swarm_segments()`**: delete it once nothing uses it.
- **`test_research_macros_keep_deferred_jinja_out_of_inline_code`**: also scan the
  content of every macro's `local_macros`, because the raw region now lives in
  `_report_records`.
- **Solo-lead regression**: extend
  `test_research_swarm_all_researchers_off_yields_lead_only` or add a test. Assert that
  `{{ lead_report }}` is absent and the lead's tree ends with `└── <name>.md` followed
  by the closing fence. With `audio=true`, the lead layout lists `├── <name>__final.md`
  and `└── <name>_narration.md`.
- **`tests/test_wheel_contract.py`**: replace the stale
  `research_swarm.content.count("%q(1.5x, w=0.25") == 9` (that count is already 0 today)
  with `assert set(research_swarm.local_macros) == {"_queue", "_report_records"}`. This
  proves the packaged helpers survive the wheel build.

## Docs

In the `#research_swarm` section of `docs/macros.md`:

- Change "up to nine authored segments" to "up to nine launched agents", and note that
  each enabled researcher is rendered from one shared researcher template.
- In list item 1, change "Gated by `%if(should_run=...)` on the `codex` flag plus the
  `provider_enabled` filter" to say the researcher is rendered only when `codex` is true
  and the provider is not hard-disabled.

Do not edit `CHANGELOG.md`; release-please owns it.

## Verification

- Run the harness above; only the documented solo differences may appear.
- Run `sase tool run check` in the plugin repo. Its `AGENTS.md` makes `check` a guarded
  recipe.
- Optional cross-check from a sase checkout: run
  `tests/fakey/test_runner_slots_e2e.py::test_installed_research_swarm_quarter_weights_fill_one_fakey_capacity_unit`
  with `PYTHONPATH` pointing at the plugin checkout's `src`.
- `sase macro show research_swarm` should list `#_queue` and `#_report_records` under
  LOCAL MACROS. Its header segment count drops from 9 to 5 because it counts raw `---`
  lines; this is cosmetic.

Suggested commit subject:
`refactor(research-swarm): de-duplicate swarm with local helpers and a researcher loop`.
Mention the solo-lead layout fix in the body.

## Out Of Scope

If these are not already tracked, file `bug` task beads for them with `/sase_new_task`.

- **Stale publish smoke.** The published-minimum smoke in
  `.github/workflows/publish.yml` is already out of date: it asserts 4 rendered segments
  and the same raw `%q(1.5x, w=0.25` count of 9. Leave it unchanged here.
- **Pre-existing `sase doctor` failure.** The config checks fail on
  `#research/audio(edition={{ audio_edition }})` because the model-macro scan validates
  the unrendered raw source. The failure is identical before and after this change.
- **Singular wording for multiple peers.** The researcher independence paragraph says
  "the other researcher's report" even when there are several peers. It is now a
  one-place fix, but it would change the prompt, so leave it out of this refactor.
