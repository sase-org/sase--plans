---
tier: tale
title: Research swarm linker opens reports with the research query and infographic
goal:
  Every linker-published research report opens with title, research-query summary, then
  the infographic (when generated) directly above the bottom-line/overview section.
size: small
proposed_by: bbugyi200.athena.0v3
create_time: 2026-10-01 15:25:06
status: wip
---

# Plan: Research swarm linker: research query and infographic at the top of the published report

## Goal

Tighten the `#research_swarm` linker agent's instructions so every published `<name>.md`
opens the same way:

1. frontmatter (if any), then the one `#` title;
2. a short **research query**: a summary of the `prompt` input passed to
   `#research_swarm`. This is always required.
3. the **infographic**, only when the image agent produced one, placed directly above
   the bottom-line section;
4. the **bottom-line / overview section** (`## Bottom line` or `## Overview`).

Today the linker is told "One `#` title, then the bottom line or answer first", with the
infographic embedded "where it best supports the text (usually right after the bottom
line)". It is never asked to state the research query. Recently published reports show
this: the infographic sits _below_ `## Bottom line`, and the question appears only in a
later "Question and inputs" section, when the lead happened to write one.

## Where the code lives

The swarm is not defined in the sase repo. It ships in the linked
`sase-research-artifacts` plugin repo. Open it first and work only in the printed path:

```bash
sase repo open sase-research-artifacts -r "Improve #research_swarm linker top-of-file instructions"
```

Read that repo's `AGENTS.md` before editing. No file in the sase repo itself changes.

Files to touch (all in `sase-research-artifacts`):

- `src/sase_research_artifacts/xprompts/research_swarm.md`: the last swarm segment (the
  one starting `%if(should_run={{ run_linker }}) %id(linker, clan=research.{@1})`).
- `tests/test_xprompt_loading.py`: linker assertions.
- `docs/xprompts.md`: `#research_swarm` agent list, item 8 (`<clan>.linker`).
- `README.md`: the `#research_swarm` bullet's `image=true` parenthetical.

Do not hand-edit `CHANGELOG.md` (release-please owns it).

## Design decisions

- **The title stays first.** "Very top of the file" means above the image and the bottom
  line, not above the `#` title. YAML frontmatter must stay at byte 0. The `#` title is
  the document heading in Obsidian and in the pandoc Highlights PDF, so the research
  query goes directly under the title.
- **The research query is a blockquote, not a heading.** Use one line in the form
  `> **Research query:** <summary>`. The PDF renderer runs pandoc with
  `--number-sections`, so a `## Research query` heading would get a section number and a
  TOC entry, and would push the bottom line out of first-section position. A blockquote
  stays visually distinct and unnumbered.
- **The research query summarizes the original `prompt`, not the lead's restatement.**
  The linker already receives `{{ prompt }}` under "Research request (context only; do
  not research it):". Keep the user's own terms. Use a request that is already one short
  sentence verbatim.
- **The editor-only rule gets one explicit exception.** The linker is told to add
  nothing of its own. The research-query summary is the single piece of prose it
  authors, and the prompt must say so; otherwise the two instructions conflict.
- **The bottom-line section gets a fixed name.** The first `##` section is
  `## Bottom line`, or `## Overview` when the report surveys options rather than giving
  one answer. This matches the user's own naming. It also keeps the existing anchor
  example `[the bottom line](#bottom-line)` valid.
- **Without an image, the prompt never mentions an infographic.** When `image=false`,
  the rendered linker prompt must not contain the word "infographic". The current
  template already behaves this way; keep it.

## Implementation

All edits are inside the linker segment of `research_swarm.md`. Keep the existing Jinja
structure, raw-protected `wait.artifacts` loops, directives, queue template, and step
numbering (1–8) unchanged. The wording below is the target text. Small rewording for
flow is fine, but keep the quoted anchor phrases exactly, because the tests assert them.

### 1. Intro paragraph: carve out the one authored line

Append a sentence to the paragraph that ends "If the lead seems wrong, leave it as
written.":

> The only prose you write yourself is the short research-query summary of the request
> that opens the file (step 3).

### 2. Research-request label

Change `Research request (context only; do not research it):` to:

> Research request (context only; do not research it, but summarize it as the file's
> research query in step 3):

### 3. Step 3 (**Restructure**): replace the opening bullets and the infographic bullet

Target structure of step 3, in this order:

```text
3. **Restructure** the lead's report into a well-thought-out organization:
   - Keep the frontmatter, updating `updated_time` if present.
   - **Open the file in this exact order**, with nothing else between these parts:
     the frontmatter (if any), one `#` title, the research query, <IMAGE-ONLY: "the infographic, ">and then
     the bottom-line section.
   - **Research query.** Directly below the title, add one blockquote that summarizes
     the research request above in one to three sentences, for example
     `> **Research query:** <summary>`. Phrase it as the question or task being
     answered, in the requester's own terms: keep the questions, named subjects, and
     explicit scope or constraints; drop instructions aimed at agents, such as output
     paths, xprompt or directive syntax, and formatting requests. Summarize what was
     asked, not material the request quotes or attaches. Use a request that is already
     one short sentence verbatim. Never fold findings, answers, or scope the request
     does not state into it. It is not a heading, so it gets no section number and no
     TOC entry.
   <IMAGE-ONLY bullet>
   - **Embed the infographic** exactly once, directly above the bottom-line section:
     after the research query and before that section's `##` heading, never further
     down. Use a relative link with descriptive alt text, for example
     `![<alt text>](<name>_infographic.png)`. Locate it by the
     `<name>_infographic.png` convention or the image entries above. Embed only a file
     you have confirmed exists beside the report in your research checkout. If the
     image agent completed without producing one, publish without it (the research
     query then sits directly above the bottom-line section) and say so in the final
     response.
   </IMAGE-ONLY bullet>
   - **Bottom-line section.** The first `##` section is `## Bottom line` (or
     `## Overview` when the report surveys options rather than giving one answer) and
     gives the answer first.
   - Below it, `##` and `###` sections ordered by the questions a reader will ask, with
     duplicated passages merged.
   - **Never number headings.** (unchanged)
   - **No table of contents and no block of jump links.** (unchanged)
   - Keep the lead's wording where it works. Never drop a claim, caveat, or source to
     save space. If the lead's report restates the question or lists its inputs, keep
     those details in a later section; the research query summarizes the request but
     does not replace them.
```

Notes:

- The inline conditional must render as
  `the research query, the infographic, and then the bottom-line section.` when
  `image=true`, and `the research query, and then the bottom-line section.` when it is
  false. Keep each of those phrases unbroken on one rendered line, wrapping around them
  as needed, because the tests assert them as literal substrings.
- Gate the infographic bullet with `{% if image %}` as today, and remove the old "where
  it best supports the text (usually right after the bottom line)" wording. Use
  `{%- if image %}` / `{%- endif %}` trimming, the same idiom as the lead segment's
  `{%- if run_linker %}`, so the rendered step 3 no longer contains the whitespace-only
  lines that the current `   {% if image %}` lines leave behind.
- Do not put any literal `{{`, `{%`, a line that is exactly `---`, or a line starting
  with `#name:` in the new prose. The segment is Jinja-rendered twice (at swarm
  expansion and at agent runtime) and xprompt-parsed. `` `#` title `` inside prose is
  fine; it is already there.

### 4. Step 6: re-check the opening order

Append to step 6 (**Re-check against the step-2 inventory**) a final sentence:

> Then confirm the file opens in the step-3 order: title, research query, <IMAGE-ONLY:
> "infographic, ">bottom-line section.

Use the same Jinja-conditional style so the no-image render never says "infographic".

### 5. Frontmatter input description (same file)

Update the `image` input's description, "Implies the linker agent, which embeds the
infographic in the published report.", to say the linker embeds it "directly above the
published report's bottom line". Leave the other inputs alone.

### 6. Tests (`tests/test_xprompt_loading.py`)

Keep every existing assertion passing. Existing assertions on `"Embed the infographic"`
(present with image, absent without) still hold with the bolded
`**Embed the infographic**`. Add one focused test, for example
`test_research_swarm_linker_opens_with_research_query_then_infographic`, that renders
the linker via `_swarm_segments` and asserts:

- **Linker only** (`_swarm_segments({}, linker=True)[-1]`):
  - `"> **Research query:** <summary>"` is in the prompt.
  - `"the research query, and then the bottom-line section"` is in the prompt.
  - `"## Bottom line"` and `"## Overview"` are both in the prompt.
  - `"infographic"` is not in the prompt.
- **Image** (`_swarm_segments({}, image=True)[-1]`):
  - `"the research query, the infographic, and then the bottom-line section"` is in the
    prompt.
  - `"directly above the bottom-line section"` is in the prompt.
  - `"right after the bottom line"` is not in the prompt.
  - The research-query instruction comes before the infographic bullet, which comes
    before the bottom-line bullet. Assert this by comparing the `str.index` values of
    `"**Research query.**"`, `"**Embed the infographic**"`, and
    `"**Bottom-line section.**"`.
- **Both modes**:
  - The intro carve-out (`"The only prose you write yourself"`) is in the prompt.
  - The research-request label mentions summarizing it as the research query.
  - Neither rendered step 3 contains a whitespace-only line. Check this on the text
    between `"3. **Restructure**"` and `"4. **Validate every link carried over.**"`.

The existing runtime-render test
(`test_research_swarm_linker_renders_registered_lead_via_wait_artifacts`) must keep
passing. It asserts that no `{{` or `{%` survives the runtime render.

### 7. Docs

- `docs/xprompts.md`, item 8 (`<clan>.linker`): replace "writes and registers the
  canonical, well-structured `<name>.md` with checked links, in-document jump links, and
  the embedded infographic" with a description of the new opening order. The file opens
  with the title, then a short research-query summary of the swarm's `prompt`, then the
  infographic when one was generated, then the `## Bottom line` / `## Overview` section.
  Restructured sections with checked links and in-document jump links follow.
- `README.md`, `#research_swarm` bullet: change "(`image=true` implies the linker, which
  embeds the infographic in the published report)" so it says the linker embeds it
  directly above the published report's bottom line. Optionally add that the linker
  opens the report with a research-query summary, if it fits the bullet without bloating
  it.

## Verification

1. From the opened plugin checkout, run `sase tool run check` (lint plus tests). Do not
   run bare `just check`; it is guarded.
2. Eyeball both rendered linker prompts. Use the same helpers the tests use
   (`load_xprompts_from_plugins()["research_swarm"]`,
   `expand_single_xprompt(xp, ["some topic"], {"image": "true"} or {"linker": "true"}, preserve_segment_separators=True)`,
   then `split_segments_protecting_fences(...)[-1]`) with `SASE_HOME` pointed at a temp
   dir. Confirm that steps 3 and 6 read naturally in both modes, with no dangling commas
   and no blank-but-indented lines, and that the no-image prompt never mentions an
   infographic.

## Out of scope

- The lead-published `<name>.md` in runs without the linker (`linker=false`,
  `image=false`). The user asked only about the linker's instructions.
- Retrofitting already-published research reports.
- The separately known launch-time rewrite of `__` to `**` in lead and linker handoff
  lists.

## Commit

One commit in `sase-research-artifacts`, declared through `/sase_final`, for example
`feat(research-swarm): open linker reports with the research query and lead with the infographic`.
