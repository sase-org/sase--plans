---
tier: tale
title: The research_swarm linker references every Mermaid figure it places
goal: Every Mermaid diagram the research_swarm linker places in the published report
  has a figure-<id> anchor and is introduced by one thoughtful sentence that links
  to it with a local link, on both GitHub and the Highlights PDF.
size: medium
decisions:
  lead_diagrams:
    ask: Should Mermaid diagrams the lead drew itself also get a figure anchor and
      a reference sentence?
    choices:
      proposed: Only figures placed from the image agent's manifest; runs without
        diagrams stay unchanged
      all: Every Mermaid figure in the published file, including the lead's own, in
        every linker run
    default: proposed
    why: Matches "diagrams the linker adds"; lead diagrams are rare and already sit
      in lead prose
    answer: proposed
  placement:
    ask: Should the reference sentence end the paragraph the figure illustrates?
    choices:
      inline: Appended to the anchor paragraph; its own paragraph only after lists
        and the bottom line
      lead_in: Always a one-sentence paragraph above the figure; simpler, but reads
        like a 2nd caption
    default: inline
    why: Reads like a paper's in-text figure reference; the caption below already
      labels the figure
    answer: inline
proposed_by: bbugyi200.apollo.94
proposed_with: claude/opus
decided_by: auto
status: done
---

# Plan: The research_swarm linker references every Mermaid figure it places

## Goal

When `#research_swarm` runs with diagrams (`diagrams=true`, or `image=true` without
`diagrams=false`), the linker agent places the image agent's validated Mermaid diagrams
into the published `<name>.md`. Today the placed figures cannot be linked to and nothing
in the text points at them. After this change, every figure the linker places:

1. carries a stable in-document anchor, `figure-<id>`; and
2. is introduced by one thoughtful sentence, written by the linker, that links to it
   with a local link (`[the publication-order diagram](#figure-publication-order)`) and
   tells the reader what to look for in it.

The linker is still an editor, not a researcher: the sentence points at the figure and
restates only what the manifest's `illustrates` point and the surrounding text already
say.

## Where the work lives

All edits are in the linked `sase-research-artifacts` repo. Open it with
`sase repo open sase-research-artifacts -r "<why>"`, use the printed path for every read
and write, and read its `AGENTS.md` first. Verify there with `sase tool run check`,
never bare `just check`. `CHANGELOG.md` is owned by release-please, so do not edit it.
No `sase`, `sase-core`, or bob change is needed.

Files:

- `src/sase_research_artifacts/xprompts/research_swarm.md` (the linker segment; the main
  change)
- `src/sase_research_artifacts/xprompts/research_diagrams.md` (two manifest-field
  clarifications)
- `docs/macros.md` and `README.md`
- `tests/test_macro_loading.py`

## Design: what the link targets, and why

A Mermaid fence has no anchor of its own, so the linker wraps each placed figure (fence
plus caption) in an HTML `<div>` carrying the id, exactly like the existing listen card
(`<div class="listen">` with blank lines inside):

```markdown
<div id="figure-publication-order">

<the manifest's mermaid fence, byte-for-byte>

_The linker publishes once, after every companion finishes._

</div>
```

This was verified on both reading surfaces while planning:

- **Highlights PDF (bob, pandoc to XeLaTeX).** pandoc's default `markdown` reader
  (`native_divs`) parses the wrapper as a `Div` with id `figure-…` whose content is the
  untouched `CodeBlock` with class `mermaid`. bob's `diagrams.lua` therefore still swaps
  it for the rendered vector figure. bob's `return_links.lua` treats a Div id as a
  return-capable target. In a pandoc 3.1.3 test run with that filter, both
  `#figure-publication-order` links were tagged and paired (`paired: 2, targets: 1`),
  and the `↩ p. N` return-pill row landed first inside the Div, directly above the
  figure.
- **GitHub.** The GFM renderer (checked through GitHub's markdown API) keeps the wrapper
  as `<div id="user-content-figure-publication-order">`. That is GitHub's standard
  prefixed in-page anchor, which `#figure-publication-order` links resolve to. The fence
  inside still renders as `highlight-source-mermaid`, the same as a top-level Mermaid
  fence.

Rejected alternatives (the prompt should forbid them briefly so the linker does not
"simplify" the wrapper away):

- **An `<a id="…"></a>` anchor.** pandoc's LaTeX writer drops raw HTML, so bob finds no
  target and turns the link into plain text with a dead-link warning.
- **Linking the enclosing heading.** It only reaches the section, and it would usually
  link to the very section the sentence is in.
- **Pandoc attributes on the fence** (` ```{.mermaid #id} `). GitHub then stops
  rendering it as Mermaid, and the fence must stay byte-identical anyway.
- **A bracketed span** (`[]{#id}`). GitHub prints it literally.

## Changes to the linker segment (`research_swarm.md`)

Unless a decision callout says otherwise, everything below is gated on `run_diagrams`,
using the file's existing `{%- if run_diagrams %}` whitespace-control patterns. Rendered
linker prompts for runs without diagrams must stay byte-for-byte unchanged. Treat the
wording below as near-final: tighten it, but keep every rule.

### 1. Opening editor paragraph

The current gated sentence is "The only other content you add is the image agent's
diagrams and captions, copied verbatim." Extend it to:

> The only other content you add is the image agent's diagrams and captions, copied
> verbatim, plus a figure anchor and one short reference sentence for each diagram you
> place (step 3).

Keep the phrases "The only prose you write yourself" and "copied verbatim", which tests
pin.

### 2. Step 3, the "Place proposed diagrams" bullet

Keep its rules: byte-for-byte fence, caption as one italic line, the `illustrates` and
contradiction check, the forbidden locations, bottom-line placement, manifest order, and
no editing or inventing. Change two things:

- The fence still goes after the block that now carries the tagged content, but only
  that figure's reference sentence may sit between the block and the figure (see the
  placement rule below).
- The fence and caption are wrapped in the figure anchor described in the next bullet.

### 3. Step 3, new bullet after it: **Anchor every placed figure.**

- Wrap each placed fence and its caption, and nothing else, in `<div id="figure-<id>">`
  and `</div>`. `<id>` is the diagram's manifest `## <id>`. Leave a blank line after the
  opening tag and before the closing tag, so GitHub and pandoc parse the fence and
  caption as Markdown, as with the listen card. Show the example block from the Design
  section above, inside a fenced `markdown` block.
- Keep the id to lowercase letters, digits, and hyphens: replace any other run of
  characters with one hyphen. It must be unique in the file. If it matches a heading's
  anchor or another figure's id, append `-2`, `-3`, and so on.
- Explain in one sentence why the wrapper exists: it is the figure's link target on both
  reading surfaces, because GitHub keeps the id and jumps to it, and the Highlights PDF
  gives it return links.
- Never put an id or attribute on the fence itself, and never substitute an `<a>` anchor
  or a heading. Make clear that this wrapper and the listen card are the only HTML
  blocks the linker writes. The table rule that forbids `:::` divs is about table
  substitutes and does not cover this wrapper.

### 4. Step 3, new bullet after that: **Reference every placed figure.**

Write exactly one sentence per placed figure that links to it with a local link,
`[<link text>](#figure-<id>)`, so no figure goes unreferenced. Make it worth reading:

- **Say what to look for.** Tell the reader what the figure makes visible that the prose
  alone does not, such as the path, order, branch, or handoff it traces. Build the
  sentence from the manifest's `illustrates` point, phrased in the surrounding text's
  terms. Never write a bare pointer such as "See the diagram below."
- **Point, don't claim.** The sentence is a pointer, not a finding:
  - add no claim, number, actor, step, or causal link that `illustrates` and the
    surrounding final text do not already state;
  - keep conditional wording conditional, and keep a proposed design distinct from an
    implemented one;
  - do not repeat the caption word for word.
- **Name the figure in the link text,** for example `the publication-order diagram`.
  Never use "here", "this", or a figure number. Captions carry no numbers, and an
  omitted diagram would leave gaps.
- **Keep it short:** about 12 to 30 words of plain prose. It is not italic, not a
  heading, and not a list item of its own. It never goes inside the figure's `<div>`,
  its caption, or a blockquote.
- **Placement.** When the figure directly follows the paragraph that carries its anchor,
  make the sentence that paragraph's last sentence. Otherwise give the sentence its own
  short paragraph directly above the figure's `<div>`. That happens when the anchor sits
  in a list (so the figure follows the whole list) or when the figure closes the
  bottom-line section. Two figures after one block get one sentence each, in manifest
  order.
- **Omitted figures lose their sentence.** If you omit a figure, delete its sentence
  too. No link may point at a figure that is not published.
- **Example**, written in inline code inside the prompt:
  `… so the link must be in the linker's first commit. [The publication-order diagram](#figure-publication-order) traces why the linker waits for every companion agent: the PDF hook fires only once, on the ADD of name.md.`

> [!decision] placement = lead_in Replace the placement sub-bullet with this: "Give the
> sentence its own short paragraph directly above the figure's `<div>`, after the block
> it follows. Two figures after one block get one sentence each, in manifest order." In
> the example, show the sentence as its own line rather than continuing the paragraph.
> The "Place proposed diagrams" bullet then always allows the reference-sentence
> paragraph between the block and the figure. Docs and tests assert the lead-in wording
> instead of "end that paragraph".

> [!decision] lead_diagrams = all Make the figure anchor and reference rules apply to
> every Mermaid figure in the published file, in every linker run, not only when
> `run_diagrams` is true.
>
> - **Ungate the generic rules.** Rename the two new bullets to cover "every Mermaid
>   figure", move them out of the `run_diagrams` gate, and keep only the
>   manifest-specific parts gated: the id source and the `illustrates` basis.
> - **Add an ungated step-3 bullet for the lead's own figures.** It treats every Mermaid
>   fence carried over from the lead's report as a figure:
>   - wrap the fence, plus a caption line the lead put directly under it if any, in a
>     figure-anchor `<div>` whose id is `figure-<slug>`, a short descriptive slug of
>     what the diagram draws, unique in the file;
>   - never move, edit, or redraw the lead's fence; it travels with its block during
>     restructuring;
>   - if the lead's prose already points at the diagram ("the diagram below shows"),
>     turn that phrase into the local link instead of adding a sentence;
>   - otherwise add one reference sentence under the same rules, drawing only on what
>     the lead's surrounding text says the diagram shows.
> - **Opening paragraph.** Name "one short reference sentence per Mermaid figure" as
>   part of the prose the linker writes, in every linker variant.
> - **Steps 4–6.** Ungate the figure-anchor wording. The step-6 grep count must equal
>   the published Mermaid fence count (the lead's plus the accepted ones).
> - **Wording limits.** The ungated text must never contain the word "infographic",
>   which a test already forbids in the non-image linker. The "Open the file in this
>   exact order" line must never mention diagrams.
> - **Tests.** Assert the lead-figure bullet appears in a `linker=True` swarm without
>   diagrams.

### 5. Step 4 (link validation)

In the first bullet, in-document anchors must resolve "against the final headings
{% if run_diagrams %}and figure anchors{% endif %}". Mind the whitespace control so the
sentence reads correctly both ways.

### 6. Step 5 (in-document links)

Append, gated: a summary point that discusses a figure's content may also link to its
`#figure-<id>` anchor, sparingly. Always link a figure through that anchor, never
through a nearby heading.

### 7. Step 6 (re-check) and the final response

Add, gated: every accepted diagram sits inside exactly one figure-anchor `<div>` holding
only its fence and caption, and its reference sentence links to it. Concretely:

- `grep -c '^<div id="figure-' <file>` equals the number of accepted diagrams;
- every `(#figure-` link names an id that exists in the file;
- the `pandoc <file> -t html` check that step 5 already runs shows each
  `id="figure-<id>"`.

The final-response sentence becomes: list accepted diagram IDs with their `figure-<id>`
anchors, and omitted IDs with reasons. Keep "Never write \"diagrams pending\"."

### Gotchas the implementer must respect

- **Keep every `#figure` inside backticks or a fenced block.** SASE's macro parser
  treats a bare `#figure-…` in prompt text as an unknown `#figure` macro reference and
  prints a warning. Planning confirmed that inline code spans and fenced blocks are left
  alone. `plan_typed_launch_units` diagnostics do **not** catch this, so add the test
  described below.
- **No whitespace-only lines in step 3.**
  `test_research_swarm_linker_opens_with_research_query_then_infographic` rejects any
  whitespace-only line in step 3, and it already renders the diagrams branch through
  `image=True`. Blank lines inside the indented example fence must be truly empty.
- **Keep the pinned strings:** "copied verbatim", "**Place proposed diagrams.**",
  "diagrams pending", "Every Mermaid diagram from the lead's report is still present",
  "or a diagram", and the "Mark every diagram after the fifth…" sentence.
- **Keep runtime Jinja out of the new text.** The new text must not introduce `{{`/`{%`
  that would be evaluated at agent runtime. The deferred-Jinja test sweeps every
  research macro.

## Changes to `#research/diagrams` (`research_diagrams.md`)

In step 7's field bullets:

- "Each `## <id>` is a unique lowercase slug" becomes "Each `## <id>` is a unique slug
  of lowercase letters, digits, and hyphens; the linker publishes the diagram under the
  anchor `figure-<id>`."
- Extend the `illustrates` description: the linker also builds the figure's in-text
  reference sentence from it, so it should name what the diagram lets a reader see.

Keep the macro free of runtime Jinja and of bare `#figure`.

## Docs

- **`docs/macros.md`, "Diagram handoff":** add a short paragraph. Each placed diagram is
  published as a figure: the fence and caption are wrapped in `<div id="figure-<id>">`
  (blank lines inside, like the listen card), which GitHub and the Highlights PDF both
  treat as a link target. One linker-written sentence links to it with
  `[the <topic> diagram](#figure-<id>)`, says what to look for, and restates only the
  manifest's `illustrates` point. An omitted diagram takes its sentence with it.
- **`docs/macros.md`, "Manifest":** document the id character set and its use as the
  anchor, and that `illustrates` also seeds the reference sentence.
- **`docs/macros.md`, linker list item 8:** after "in-document jump links follow", add
  that every placed diagram is an anchored figure introduced by one linking sentence.
- **`README.md`:** in the linker sentence that "places any accepted Mermaid diagrams
  from the image agent's companion manifest", add that each one is anchored and
  referenced by one in-text local link.

If `lead_diagrams = all`, phrase all three as applying to every Mermaid figure in the
published file.

## Tests (`tests/test_macro_loading.py`)

- **Extend `test_research_swarm_diagrams_image_matrix`.**
  - When `run_diagrams` is true, assert the linker segment contains:
    - `'<div id="figure-<id>">'`;
    - `"**Anchor every placed figure.**"` and `"**Reference every placed figure.**"`, in
      that order, after `"**Place proposed diagrams.**"`;
    - `"[<link text>](#figure-<id>)"`.
  - Otherwise assert `"figure-<id>"` is absent from every segment.
- **New test: figure-anchor text is gated.** For `_swarm_segments({}, linker=True)` and
  for `image=True, diagrams=False`, assert the linker contains no `figure-` text. Under
  `lead_diagrams = all`, assert the lead-figure bullet is present instead.
- **New test: reference-sentence rules are present.** On the diagrams linker's step-3
  slice, assert:
  - the "See the diagram below." prohibition;
  - the no-figure-number rule;
  - the "delete its sentence too" omission rule;
  - the chosen placement wording ("end that paragraph" for `inline`).

  On the step-6 slice, assert the `grep -c '^<div id="figure-'` check and the
  final-response anchor listing.

- **New test: `#figure` stays out of macro syntax.**
  - Render every swarm variant: linker only, image, diagrams only,
    `image=True, diagrams=False`, audio, and image plus audio.
  - From each segment, strip fenced blocks (a `re.DOTALL` match on backtick fences) and
    then inline code spans.
  - Assert `"#figure"` never appears in what remains.
- **Extend the research_diagrams tests** with assertions for the new id character-set
  sentence and the `figure-<id>` anchor mention.

## Verification

1. `sase tool run check` passes in `sase-research-artifacts`.
2. Render the swarm with diagrams on (through the test helpers) and read the whole
   linker segment end to end. Confirm that it reads coherently, that the
   gating/whitespace is clean in the `run_diagrams` true and false variants, and that
   runs without diagrams match the pre-change rendering exactly.
3. **Sanity render.** Write a toy published file in a fresh `mktemp -d` directory. It
   should hold one paragraph ending in a reference sentence, then a figure-anchor div
   around a small Mermaid fence and an italic caption, then a later heading.
   - Confirm that `pandoc <toy> -t html` contains `id="figure-…"` around the `mermaid`
     code block.
   - Optionally, open `bob-cli` with `sase repo open bob-cli -r "<why>"`. Then run
     `pandoc <toy> -t latex --lua-filter <bob-cli>/src/native/highlights_ref/return_links.lua -M bob-return-links-report=<tmp>/r.json`
     and confirm the report shows the link `paired` (not dead).

## Out of scope

- Referencing the infographic: it is an image at the top of the file, not a Mermaid
  diagram.
- Any change to bob's renderers, SASE core, or published research reports (existing
  reports are not backfilled).
- Figure numbering.
