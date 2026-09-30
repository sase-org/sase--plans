---
tier: tale
title: Add a linker agent to the research swarm
goal:
  "#research_swarm gains an opt-in linker agent (implied by image=true) that publishes a
  restructured, link-checked <name>.md with the embedded infographic from the lead's
  <name>__final.md; the critique agent is removed and image_model is configurable."
size: medium
proposed_by: bbugyi200.athena.0uf
create_time: 2026-09-30 11:02:00
status: wip
---

# Add a `linker` agent to `#research_swarm` (and drop `critique`, add `image_model`)

## Goal

Give `#research_swarm` an opt-in **linker** agent that publishes the report the
`research-highlights` file hook turns into a Highlights PDF. The lead then writes an
intermediate `<name>__final.md` (ignored by the hook). The linker waits for the lead,
and for the image agent when `image=true`. It then writes the canonical, well-structured
`<name>.md`, with checked links, in-document jump links, and the embedded infographic.

This adds a **publication barrier**. Today the hook fires once, on the `ADD` of the
lead's `<name>.md`, before the image agent has drawn anything, so no Highlights PDF has
ever contained an infographic. `MODIFY` never fires the hook, so a later edit cannot fix
this. With the linker, the only hook-eligible file is created after the image exists.

In the same change, remove the unused `critique` agent and add an `image_model` input.

Context and inspiration (read it before starting):
`sase artifact read research:202609/research_swarm_linker_agent/research_swarm_linker_agent.md "<reason>"`.
That report was verified against the code. This plan adopts its recommendations,
including the ones that adjust the original request (see **Decisions**).

## Where the work lands

- **Linked `sase-research-artifacts` repo** holds almost everything: the swarm xprompt,
  `#research/image`, the hook spec, the default config, the docs, and the tests. Open it
  with `sase repo open sase-research-artifacts -r "<reason>"`, read its `AGENTS.md`, and
  work only in the printed path.
- **This `sase` repo** needs one test update: `tests/fakey/test_runner_slots_e2e.py`
  (see step 8).
- **No sase runtime code change is needed.** `%if`, `%wait`, `wait.artifacts`, image
  auto-capture, and the hook matcher already do everything this needs. Nothing moves to
  `sase-core`: this is prompt-template and plugin-config work.

## Decisions (read before implementing)

1. **`final` is NOT added to `_SWARM_RESEARCHER_SUFFIXES`.** The request asks to add the
   `__final` suffix to "the list of suffixes the file hook uses to ignore research
   files". The only such tuple in `provider.py` generates `agent_name_globs`, and those
   globs veto the _agent_ that committed a file, not the file name. Adding `final` would
   produce `!research.*.final`. That vetoes the lead agent, which publishes `<name>.md`
   on every swarm that skips the linker, so default swarms would silently lose their
   PDF.

   `<name>__final.md` is already ignored by the hook's path globs `!20*/*__*.md` and
   `!20*/*/*__*.md`. Honor the request's intent by making that explicit and pinning it
   with tests: rename the tuple and fix its comments (step 3). Do not replace the broad
   `__*` path globs with an enumerated list, because that would narrow today's behavior.

2. **`run_linker = linker or image`.** Compute this once in the template prelude and use
   it everywhere, so `image=true` always implies the linker.
3. **Add a `linker_model` input, default `@xlarge`.** Every other role has a model
   input. The linker rewrites the report the user actually reads, so silently losing a
   caveat or number is the main risk, which justifies the strongest tier at launch.
4. **`image_model` defaults to `"@image"`**, so current behavior is unchanged.
5. **The infographic is named `<name>_infographic.png`, not
   `<name>__final_infographic.png`.** `#research/image` drops a trailing `__final` from
   the source stem. It drops only `__final`, never researcher suffixes, so a standalone
   run on a draft cannot collide with the consolidated infographic.
6. **Image failure parks the linker, and this is accepted and documented.** Named
   `%wait`s release only when `done.json` has outcome `"completed"`. With `image=true`,
   a failed image agent leaves the linker parked, so there is no `<name>.md` and no PDF,
   and SASE posts the "Wait dependency can never self-resolve" notification. The linker
   fails soft in only one case: the image agent completed but produced no PNG. Then the
   linker publishes without the image and says so.
7. **Removing `critique` is non-breaking at runtime.** Stale `critique=true` arguments
   are preserved as unknown named values and silently ignored.

## Execution matrix (default researchers cdx + cld)

| `linker` | `image` | Agents               | Lead writes                    | Hook fires on                                    |
| -------- | ------- | -------------------- | ------------------------------ | ------------------------------------------------ |
| false    | false   | cdx, cld, final (3)  | `<name>.md`, unregistered      | lead's `<name>.md` (**byte-identical to today**) |
| true     | false   | + linker (4)         | `<name>__final.md`, registered | linker's `<name>.md`                             |
| false    | true    | + image + linker (5) | `<name>__final.md`, registered | linker's `<name>.md`                             |
| true     | true    | + image + linker (5) | `<name>__final.md`, registered | linker's `<name>.md`                             |

The authored segment count stays **8**: the linker replaces critique one for one and
stays last. Segment order is cdx, cld, grk, mus, gem, final, image, linker.

## Implementation steps

### 1. `src/sase_research_artifacts/xprompts/research_swarm.md`

**Frontmatter.**

- Update `description`: drop "critique"; mention the optional infographic and the linker
  that publishes the report.
- Delete the `critique` and `critique_model` inputs.
- After `image`, add these inputs, in this order:
  - `image_model` (word, default `"@image"`). Its description names `<clan>.image`.
  - `linker` (bool, default `false`). Its description says the linker runs after the
    lead (and after the image agent when `image=true`), and that the lead then writes
    `<name>__final.md` while the linker publishes `<name>.md`. It must also say the
    linker always runs when `image=true`.
  - `linker_model` (word, default `"@xlarge"`). Its description names `<clan>.linker`.
- Extend the `image` description with: "Implies the linker agent, which embeds the
  infographic in the published report."

**Prelude.** Add these two lines:

```jinja
{%- set run_linker = linker or image -%}
{%- set lead_report = "<name>__final.md" if run_linker else "<name>.md" -%}
```

Then:

- End the lead's `layout_body` with `"└── " ~ lead_report`.
- Delete the `critique_layout_*` namespace.
- Add a `linker_layout_body` namespace that renders these lines:
  - `<month-dir>/<name>/`
  - `├── <name>__<short>.md`, once per surviving researcher
  - `├── <name>__final.md`
  - `├── <name>_infographic.png`, only when `image`
  - `└── <name>.md`

**Lead segment.** Keep the default (`run_linker` false) render byte-identical to today.

- In both branches (with researchers and solo), step 4 writes to
  `<name>/{{ lead_report }}`, and the solo branch's hard-coded layout ends at
  `└── {{ lead_report }}`.
- Replace both `{%- if critique %}` blocks with `{%- if run_linker %}` blocks. Each
  block holds two things:
  - One sentence: do not create `<name>/<name>.md`, not even as a placeholder, because
    the linker agent `research.{@1}.linker` publishes it from your report.
  - The existing registration step, retargeted to the linker. Its example label becomes
    `research:202609/<name>/<name>__final.md`. Keep the same
    `sase artifact create -p ... -l "research:<repo-relative-report-path>"` command,
    with no `--move`, and keep the failure wording.
- Do **not** tell the lead that the linker will "clean up" its report. `__final.md` is
  the archival synthesis and the linker's only source, so the lead's quality bar must
  not drop.

**Image segment.**

- Replace `%model:@image` with `%m:{{ image_model }}`.
- Leave everything else unchanged: `%if(should_run={{ image }})`, the `%wait` on
  `.final`, the `%q(...)`, and `#fork:research.{@1}.final #research/image`.

**Linker segment.** Replace the critique segment entirely.

Directive lines:

```text
%if(should_run={{ run_linker }}) %id(linker, clan=research.{@1}) %m:{{ linker_model }}
%wait:research.{@1}.final {% if image %}%wait:research.{@1}.image {% endif %}%q({% if runners is not none %}{{ runners }}{% else %}1.5x{% endif %}, w=0.25{% if priority is not none %}, priority={{ priority }}{% endif %})
```

- **Image wait.** The image `%wait` must use exactly the image segment's condition
  (`image`). `%if` gating deletes a skipped segment but does not rewrite other segments'
  waits, so a wait on a dropped segment would park the linker forever.
- **Lead wait.** Always wait on `.final`, even when `image=true`: `wait.artifacts` holds
  only the waiter's own `%wait` targets, not transitive ones.
- **No fork, no `wait` input.** There is no `#fork`, and the swarm's `wait=` input does
  not apply to the linker.

Body. The wording is the implementer's to polish, but it must cover every point below.

- **Role.** The linker publishes the lead's `<name>__final.md` as the canonical
  `<name>.md`: the file readers open and the one SASE renders into a Highlights PDF. It
  is an editor, not a researcher. The new file must carry exactly the lead's meaning and
  intent. It must not do research: no new claims or sources, no settling open questions,
  no softening or strengthening conclusions, no changing the recommendation. If the lead
  seems wrong, leave it as written.
- **Context.**
  - The same "SASE derives your plan's links from the artifacts you read this turn…"
    line the lead carries.
  - `{{ prompt }}`, labelled as context only ("do not research it").
- **Inputs.** A raw-protected `wait.artifacts` markdown loop **character-identical** to
  the lead's `_WAIT_ARTIFACTS_LOOP` text. Under `{% if image %}`, add a second
  raw-protected loop:

  ```text
  {% raw %}{% for a in wait.artifacts if a.kind == "image" %}
  - wait_name={{ a.wait_name }} label={{ a.label }} vcs_relpath={{ a.vcs_relpath }} path={{ a.path }} ref={{ a.ref }}
  {% endfor %}{% endraw %}
  ```

- **Steps**, with fixed numbering. The image instructions live inside the restructure
  step, so numbering does not change with `image`.
  1. **Identify the source.** Find exactly one entry with `wait_name`
     `research.{@1}.final` and a label of the form
     `research:<YYYYMM>/<name>/<name>__final.md`. Otherwise stop and report the missing
     or ambiguous input. Open the research repo with `/sase_repo` and read the report
     through its canonical reference (or the `ref` `file:<id>` fallback) with
     `sase artifact read`. Take `<YYYYMM>/<name>/` from the label, never from the date.
     Read no chat transcripts. Never modify, move, or delete `<name>__final.md` or the
     drafts.
  2. **Inventory what must survive.** Before writing, list every finding,
     recommendation, caveat, open question, confidence statement, number, date, version,
     code block, table, and link.
  3. **Restructure** into a well-thought-out organization:
     - Keep the frontmatter, updating `updated_time` if present.
     - One `#` title, then the bottom line or answer first.
     - `##`/`###` sections ordered by the questions a reader will ask, with duplicated
       passages merged.
     - **Never number headings.** The PDF renderer runs pandoc `--number-sections`, and
       69 of 88 recent reports are hand-numbered, so they render doubly numbered.
     - **No table of contents and no block of jump links.** The PDF already gets a TOC.
     - Keep the lead's wording where it works. Never drop a claim, caveat, or source to
       save space.
     - `{% if image %}` Embed the infographic exactly once, where it best supports the
       text (usually right after the bottom line). Use a relative link and descriptive
       alt text, for example `![<alt text>](<name>_infographic.png)`. Locate it by the
       `<name>_infographic.png` convention or the image entries above. Embed only a file
       you have confirmed exists beside the report in your research checkout. If the
       image agent completed without producing one, publish without it and say so in the
       final response. `{% endif %}`
  4. **Validate every link carried over.**
     - Relative links resolve from `<YYYYMM>/<name>/`, and in-document anchors resolve
       against the final headings. Both are hard requirements.
     - Check external URLs with `curl -fsSL -o /dev/null --max-time 20 <url>`, retrying
       a transient failure once. Treat 401, 403, 429, and timeouts as _unverified_ and
       keep those links.
     - Verify repository-file links through a `/sase_repo` checkout, not by fetching
       github.com.
     - Repair a link only when the right target is certain: a followed redirect, a moved
       file, an obvious typo, or a renamed heading. For an unrepairable link, keep its
       text, drop the dead URL, and list it in the final response. **Never search for a
       replacement source.**
  5. **Add in-document links** so readers can jump between parts of the file. Add them
     inline and sparingly: from summary points to the sections that back them, from "see
     above/below" phrases, and from mentions of a named option, phase, or finding to
     where it is discussed. Do not link every mention.
     - Every heading used as a link target must start with a letter, contain only
       letters, digits, spaces, and hyphens, and be unique. Its anchor is then the
       lowercased heading with spaces replaced by hyphens, for example
       `[the bottom line](#bottom-line)`. pandoc (the PDF) and GitHub then agree.
     - Move emoji, version numbers, and code out of such headings, into the section's
       first line.
     - When `pandoc` is available, confirm anchors with `pandoc <file> -t html`.
  6. **Re-check against the step-2 inventory** and restore anything missing or changed.
     Every URL in `<name>__final.md` must appear in the new file unless it was listed as
     unrepairable.
  7. **Write** `<YYYYMM>/<name>/<name>.md` without overwrite. On a collision, stop and
     report it.
  8. **Register** it with the same
     `sase artifact create -p ... -l "research:<repo-relative-report-path>"` command,
     for example `research:202609/<name>/<name>.md`. Use no `--move`. If registration
     fails, report it and do not claim full completion.
- **Final layout.** End the body with
  `{{ "```text\n" ~ linker_layout_body ~ "\n```" }}`.

**Authoring gotchas.**

- Keep every `%{…}`-like or `#anchor` example inside inline code. A bare `%{` opens an
  alternation that fans the swarm out, which is why the probe uses `curl -f` rather than
  `-w '%{http_code}'`. A bare `#bottom-line` triggers an unknown-xprompt warning.
- Keep the `{% raw %}` wrappers, or the swarm-level render consumes the runtime loops.
- Exactly one `%q(` per segment.

### 2. `src/sase_research_artifacts/xprompts/research_image.md`

Change the output rule:

- Write `<stem>_infographic.png` in the same directory, where `<stem>` is the source
  file's stem with any trailing `__final` removed (so `topic__final.md` becomes
  `topic_infographic.png`; other stems are unchanged).
- Create it without overwrite. If it already exists, stop and report the collision.

### 3. `src/sase_research_artifacts/provider.py` (no behavior change)

- Rename `_SWARM_RESEARCHER_SUFFIXES` to `_SWARM_RESEARCHER_AGENT_SUFFIXES`. Reword its
  comment to say it vetoes commits _by those researcher agents_. The comment must also
  say that it must never include `final`, because the lead agent `research.<N>.final`
  publishes `<name>.md` on swarms without the linker.
- Reword the depth-2 path-glob comment from "Drafts and critique moved into
  <month>/<name>/ by the lead agent" to cover researcher drafts plus the lead's
  `__final.md` intermediate, which the linker republishes as `<name>.md`.
- Mention the linker's `__final.md` in the module docstring's glob-divergence paragraph
  only if it reads naturally there.

### 4. `src/sase_research_artifacts/default_config.yml`

In the `researchers` bucket description, change "the optional critique agent" to "the
optional linker agent". Keep the word "infographic", because
`tests/test_default_config.py` asserts it.

### 5. Plugin docs

- **`docs/xprompts.md`**, `#research_swarm` section:
  - Input table: remove `critique`/`critique_model`; add `image_model`, `linker`, and
    `linker_model`.
  - Segment list: item 8 becomes `<clan>.linker`, and item 7 says the image agent uses
    `image_model`.
  - Replace the critique handoff-contract paragraph with the lead/linker contract: the
    lead registers `__final.md` only when the linker runs.
  - Add the execution matrix above, and state that "image implies linker".
  - Add a short **image failure recovery** note: the linker parks. A later _successful_
    run of the same `research.<N>.image` name releases it; alternatively kill the parked
    linker. `__final.md` stays in the repo either way.
  - Add a warning that setting `image_model` to a single model gives up the `@image`
    alias's fallback chain.
  - Update "The seven model inputs" count and the example sentences.
  - In the `#research/image` section, note the `__final` stem rule.
- **`README.md`**:
  - File-hook section: replace the `__critique.md` sentence. `__final.md` is excluded by
    the same draft glob, so only the linker's `<name>.md` gets a PDF. Add that the agent
    veto intentionally never lists `final`.
  - Xprompts bullet: replace the critique mention with `linker=true` / `linker_model`,
    "`image=true` implies the linker (five agents with the default set)", and
    `image_model`.
  - Defaults paragraph: `image_model` defaults to the `image` alias.
- **`docs/configuration.md`**: swap `critique_model` for `image_model` and
  `linker_model`, and change "The optional critique agent also defaults to `@xlarge`" to
  the linker.
- **`AGENTS.md`**: change the "optional critique agent … defaults to `@xlarge`" sentence
  to the linker. Update `CLAUDE.md` too if it mirrors that text.
- **`CHANGELOG.md`**: do **not** hand-edit it; release-please generates it. Mention the
  `critique`/`critique_model` removal in the commit message body.

### 6. Plugin tests: `tests/test_xprompt_loading.py`

**Helpers.**

- In `_swarm_segments`, replace the `critique` kwarg with `linker: bool = False`.
- Add `_WAIT_IMAGE_ARTIFACTS_LOOP`, the exact raw image loop text.
- Make `_without_wait_artifacts_loop` strip it too. Otherwise the stray-`{%` assertion
  in `_assert_each_segment_has_one_queue` fails for `image=true` linker segments.

**Update existing tests.**

- Typed inputs: the list ends
  `..., lead_model, image, image_model, linker, linker_model`, with defaults `False`,
  `"@image"`, `False`, `"@xlarge"`, and no critique inputs.
- Authored segments: still 8. Destructure as
  `cdx, cld, grk, mus, gem, final, image, linker`, including every
  `*_researchers, final, _image, _critique` site.
- Image segment:
  - Authored: `%m:{{ image_model }}`, and no `%model:@image`.
  - Default render: `%m:@image`.
  - Every `*_researchers, image = _swarm_segments(..., image=True)` destructure becomes
    `*_researchers, image, linker = ...`. The segment count for `image=True` is now 5.
- Default render (3 segments):
  - No `%id(linker` and no `%id(image`.
  - The lead contains none of `__final`, `linker`, or `sase artifact create`.
  - Lead step 4 and the layout still end at `<name>.md`.
- Dependency graph: the authored linker has
  - `%if(should_run={{ run_linker }})` and `%id(linker, clan=research.{@1})`;
  - `%m:{{ linker_model }}` and `%wait:research.{@1}.final`;
  - the conditional `.image` wait;
  - no `#fork:` and no `%clan(`;
  - one `%q(`, `_WEIGHTED_QUEUE_TEMPLATE`, and `_WAIT_ARTIFACTS_LOOP`.

**Replace every critique test with linker tests.**

- `linker=true` alone gives 4 segments.
  - The linker waits on `.final` only, and contains `__final.md`, `sase artifact read`,
    `sase artifact create`, and `Final layout:`. It does not end with `__final.md` in
    the layout; the layout ends `└── <name>.md`.
  - The lead writes `<name>__final.md`, registers it (the command is present), and
    mentions `research.{@1}.linker`.
- `image=true` alone gives 5 segments, and so does `image=true, linker=true`, with no
  duplicate linker.
  - The linker waits on both `.final` and `.image`.
  - It includes the embed instructions and `<name>_infographic.png` in its layout.
  - The image segment still waits on and forks `.final`.
- `linker=true, image=false`: no `.image` wait, and no embed or infographic text.
- Solo lead plus `linker=true` (all five researchers `false`) gives 2 segments. The solo
  lead writes `__final.md` and registers it.
- `image_model` routes only to the image segment and `linker_model` only to the linker;
  neither leaks into `final`.
- The swarm `wait=` does not leak into the linker (or the image segment).
- `runners=8, priority=5` renders `%q(8, w=0.25, priority=5)` on the linker, and
  `_assert_each_segment_has_one_queue` passes for the linker/image combos.
- Runtime render, mirroring the removed critique end-to-end test:
  - Register `topic__final.md` from `research.m.final` via
    `store_explicit_artifact_file` and render the `image=true` linker with
    `bind_runtime_template_vars`.
  - Assert the `__final.md` entry appears and no `{{`/`{%` remains.
  - For the image loop, if the explicit-store helper cannot produce an `image`-kind
    record, append a plain dict (`kind="image"`, `wait_name`, `vcs_relpath`, …) to the
    artifacts list instead.
- Add a small `#research/image` test asserting the `__final` stem rule text.

### 7. Other plugin tests and CI

- **`tests/test_filters.py`**:
  - Add `202608/widgets/widgets__final.md` and `202608/widgets__final.md` to the
    candidates. Assert the hook filters both, and that the ref inventory keeps both.
    Keep the exact-tuple assertions consistent.
  - In the agent-veto parametrization, replace `research.26.critique` with
    `research.26.linker` (expected `True`). Keep `research.26.final` `True` and the five
    researchers `False`.
  - Leave the `agent_name_globs` list assertion unchanged.
- **`tests/test_wheel_contract.py`**: change `(4 if image else 3)` to
  `(5 if image else 3)`. The authored `%q(` count assertion (8) stays.
- **`.github/workflows/publish.yml`**: no change expected. Default rendering and the
  authored count are unchanged. Its published-floor `len(segments) == 4` assertion
  predates the optional image and is out of scope; do not touch it.

### 8. sase repo: `tests/fakey/test_runner_slots_e2e.py`

sase's `just` setup installs `plugins.required` editable from the linked
`sase-research-artifacts` checkout, and
`test_installed_research_swarm_quarter_weights_fill_one_fakey_capacity_unit` plans the
_installed_ swarm. It asserts that `image=true` gives 4 units, which becomes 5.

Make the test tolerant of both plugin shapes. It is `importorskip`'d, and CI may still
install the pre-linker PyPI release.

- Detect the shape with `swarm_has_linker = "%id(linker" in research_swarm.content`.
- The `explicit_one_plan` (`image=true`) count becomes `5 if swarm_has_linker else 4`.
  Keep its wait assertions on units 0 and 1.
- The fakey harness must still see exactly four quarter-weight units filling `cap=1`.
  Build `capacity_plan` from `linker=true` (cdx, cld, final, linker) when
  `swarm_has_linker`, else from `image=true`. `max_active_roots == 4` and the four 0.25
  weights then stay valid.
- The `runners=0` rejection case keeps working with `image=true` unchanged.

## Verification

1. **Before editing**, capture the rendered default swarm body (`_swarm_body({})`) for
   `{}`, `{"grok": "true"}`, and the all-researchers-off solo case. **After editing**,
   confirm all three are byte-identical. The non-linker path must not change.
2. In the plugin repo, run `sase tool run check` (the guarded `just check`). Then run
   `just test-wheel` for the wheel contract.
3. Spot-check the expansion from the sase workspace. Each command must exit 0, and the
   agent counts must match the matrix:
   - `sase xprompt expand '#research_swarm(linker=true):: topic'`
   - `sase xprompt expand '#research_swarm(image=true):: topic'`
   - `sase xprompt expand '#research_swarm(image=true, image_model=@foo, linker_model=@bar):: topic'`
4. In the sase repo, run `sase tool run check`. The fakey test must pass against the
   edited linked plugin.

## Out of scope

Do not do these. They are already tracked or deliberately deferred.

- Research-lineage deriver `__a`/`__b` staleness in sase's `_research_lineage.py`, and
  the research README's `__a` layout text. Already tracked as bead `sase-15r`.
- `bob highlights create --shift-heading-level-by=-1`.
- A deterministic anchor/URL checker script.
- Flipping `linker` to default `true`.
- A standalone `#research/link` xprompt.
