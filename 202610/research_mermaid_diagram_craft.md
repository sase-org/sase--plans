---
tier: epic
title: Mermaid diagram craft for the research swarm
goal: 'The research swarm''s diagrams agent proposes fewer, better Mermaid figures.
  Each one answers a named reader question, reads on its own from figure, in-figure
  title, and caption, uses a small semantic palette, and passes a look at a PDF-settings
  render that prints at 75% scale or more. The craft lives in one shared `mermaid_diagrams`
  plugin skill. `#research/diagrams` inlines it, and the swarm lead loads it on demand.

  '
decisions:
  lead_reach:
    ask: Should `#research` also tell standalone and swarm researchers to use the
      skill?
    choices:
      lead: Only the swarm lead gets the pointer; researcher drafts skip render time
      all: Also add it to `#research`; reaches standalone reports, slows unpublished
        drafts
    default: lead
    why: The research's Adjustment 7 names the lead, which writes most published figures
    answer: lead
phases:
- id: skill
  title: Ship the mermaid_diagrams plugin skill and point the lead at it
  depends_on: []
  size: medium
  description: 'skill: add `skills/mermaid_diagrams.md` to sase-research-artifacts.
    It holds positive rules with reasons, a verified classDef palette, the PDF-mirroring
    render, look, and revise loop, and two or three rendered worked examples. Point
    the swarm lead at the skill. Add load, expansion, and wheel tests, and list the
    skill in the README and architecture docs.'
- id: diagrams
  title: Rewire research/diagrams around reader questions and the skill
  depends_on:
  - skill
  size: medium
  description: 'diagrams: retarget candidate selection to named reader questions.
    Replace the drawing bans with an inlined `#skill/mermaid_diagrams`. Swap the max-width
    grep for root-SVG fit-scale validation under the PDF config. Redraw the manifest
    example and add `question`/`review` fields. Update tests and docs.'
proposed_by: bbugyi200.athena.research.54.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-20 11:52:07
status: done
bead_id: sase-1nd
---

- **PROMPT:** [prompts/202610/research_mermaid_diagram_craft.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/research_mermaid_diagram_craft.md)
- **BEAD:** [sase-1nd](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nd/README.md)

# Plan: Mermaid diagram craft for the research swarm

## Context

The Mermaid diagrams that the `#research_swarm` image agent proposes are hard to read on
their own, rarely useful, and visually flat. The consolidated research report
`research:202610/research_swarm_mermaid_diagram_craft/research_swarm_mermaid_diagram_craft.md`
diagnoses three causes in today's `#research/diagrams` macro. The user accepted every
requirement adjustment in that report. Read it with `sase artifact read` before starting
either phase. Its "Draft outline of the skill", "What the PDF renderer actually allows",
and "Phase 1" sections are the primary spec. This plan records the decisions that turn
that spec into two phases.

1. **Wrong target.** Candidates are drill-downs into infographic panels, ranked by how
   much detail they reveal. The macro should rank figures by how much the report's
   conclusion depends on them.
2. **Flat rules.** The macro bans `classDef`, decision shapes, and labels longer than
   about four words. It never asks for a title, a gloss, grouping, or emphasis.
3. **No visual check.** The macro renders with Mermaid defaults, never looks at the
   output, and greps a `max-width` that reads 200 px under the PDF settings. That check
   cannot fail.

Requirement adjustments this plan implements (numbering follows the report):

- Adj 1: add a render, look, and revise loop, plus a size check that matches the PDF.
- Adj 2: the advice is a plugin skill source, inlined as `#skill/mermaid_diagrams` and
  loadable as `/mermaid_diagrams`.
- Adj 3: replace drill-down ranking with reader questions. Keep the candidate-list-first
  rule and the anti-zero rule for infographic overlap.
- Adj 4: "understandable in isolation" means figure plus title plus caption.
- Adj 5: "beautiful" means structure, grouping, consistent shapes, and at most three
  semantic colour classes under the locked neutral theme.
- Adj 6: keep the 3/5 caps but raise the bar. Zero figures for a survey is a success.
- Adj 7: point the lead at the skill.
- Adj 8: do not split the agent yet.

## Where the work happens

All changes land in the linked `sase-research-artifacts` repo. Open it with `/sase_repo`
(`sase repo open sase-research-artifacts -r "<why>"`), work only in the path it prints,
and read that repo's `AGENTS.md` before editing. These rules apply:

- Neither phase changes the sase repo. Commit the plugin repo through `/sase_final` as a
  repository obligation.
- Verify with `sase tool run check` from the plugin repo root. A raw `just check` is
  refused. If the build backend allows it, also run the slow wheel contract test
  (`just test-wheel`, through `sase tool run` when guarded) after changing packaged
  files.
- Do not edit `CHANGELOG.md`; release-please owns it. Use conventional commit subjects,
  for example `feat(research-diagrams): …` and `feat(skills): …`.
- `tests/test_no_bob_dependency.py` forbids the word "bob" in plugin sources, docs,
  README, and AGENTS.md. Call the binding surface "the Highlights PDF renderer" or "the
  research PDF renderer". Never name its repo or its font alias.
- Read the PDF renderer facts from the research report. Do not copy from or depend on
  the renderer's repo.

### Authoring gotchas for macro and skill bodies

Both bodies are Jinja templates that SASE expands as prompt text:

- **Fences protect.** SASE protects fenced code blocks from Jinja and from macro and
  directive parsing. Inline code is not protected. These must appear only inside fences:
  - Mermaid hexagons, whose syntax opens with a double curly brace (a Jinja expression
    opener);
  - Mermaid frontmatter `---` lines;
  - `%%` comments;
  - hex colours such as `#E8F1FB`;
  - any `#word` text.

  In prose, a `#word` after whitespace or a quote expands as a macro, and a bare `---`
  line splits swarm segments.

- **Wrap double-brace fences in raw tags.** Wrap any fence that contains a double curly
  brace in Jinja `raw` / `endraw` block tags placed _outside_ the fence. Then the text
  survives whether or not a given render path protects fences. A raw tag inside a fence
  would leak into the output.
- **Prove it with tests.** One test should expand the skill as a macro reference.
  Another should render it as a generated skill, or preview `sase skill init --dry-run`
  where the edited plugin is the one sase loads. Both must preserve the hexagon text,
  hex colours, and frontmatter lines byte for byte, with no raw tag left over.
- **Keep the segment count.** `test_research_swarm_has_nine_top_level_segments` must
  still pass with the skill inlined.

### Renderer facts both phases rely on

These facts come from the research report's renderer section, verified against Mermaid
12.2.0 with the PDF renderer's configuration:

- **Works in the PDF:**
  - `classDef`, `class`, and `style` fills, strokes, and text colour;
  - frontmatter `title:` on flowcharts and state diagrams;
  - `<br/>` line breaks;
  - backtick Markdown labels with `**bold**`;
  - stadium and hexagon shapes, `-.->`, and `==>`;
  - non-secure frontmatter `config:` keys (`layout`, `flowchart.rankSpacing`,
    `nodeSpacing`, `wrappingWidth`).
- **Ignored in the PDF:** `%%{init}%%`, plus frontmatter `theme`, `themeVariables`,
  `themeCSS`, and `fontFamily`, all locked by the `secure` list.
- **Gotchas:**
  - `class B a,b` applies nothing, and `B:::a:::b` does not parse. Use two `class`
    lines.
  - Index-based `linkStyle` colours the wrong edges once subgraph edges shift the
    indices.
  - An edge that ends on a subgraph border can hide its arrowhead.
  - Long hexagon or diamond text wraps at 200 px and balloons the shape.
  - `timeline` drops its title and prints wide.
- **Page fit:** fit scale = min(1, 652.8 / W, 892.8 / H), where W and H are the natural
  root-SVG dimensions in pixels.
  - Hard floor: 0.75, so W ≤ ~870 and H ≤ ~1190.
  - Soft target: ≤ ~650 × 700 px.
- **PDF-mirroring Mermaid config.** Phase `skill` puts this config in the skill, written
  inline via a heredoc so no package path is needed. Keep only keys that affect layout
  or styling:

  ```json
  {
    "securityLevel": "strict",
    "theme": "neutral",
    "fontFamily": "DejaVu Sans",
    "themeVariables": {
      "fontFamily": "DejaVu Sans",
      "noteBkgColor": "#f8f9fa",
      "noteTextColor": "#24292e",
      "noteBorderColor": "#9ca3af",
      "dropShadow": "none"
    },
    "themeCSS": "* { filter: none !important; }",
    "secure": [
      "secure",
      "securityLevel",
      "startOnLoad",
      "maxTextSize",
      "suppressErrorRendering",
      "maxEdges",
      "theme",
      "themeCSS",
      "themeVariables",
      "fontFamily"
    ],
    "flowchart": { "useMaxWidth": false },
    "sequence": { "useMaxWidth": false },
    "state": { "useMaxWidth": false },
    "class": { "useMaxWidth": false },
    "er": { "useMaxWidth": false },
    "quadrantChart": { "useMaxWidth": false }
  }
  ```

  Confirm two behaviours with a real render. First, a deliberately broken diagram must
  still make `mmdc` exit non-zero under this config. Second, a frontmatter `theme:`
  override must render neutral grey, which shows the `secure` list works.

## Phase `skill`: ship the mermaid_diagrams plugin skill and point the lead at it

### Skill source

Add `src/sase_research_artifacts/skills/mermaid_diagrams.md`. sase 0.18.0, the plugin's
floor, already loads plugin `skills/` resource directories, and `#skill/<name>` expands
what `/<name>` invokes. Hatch packages the whole package directory.

Frontmatter:

```yaml
---
name: mermaid_diagrams
description: <one or two lines: draw Mermaid figures a reader understands from the
  figure, title, and caption alone, that render cleanly in the research PDF and on
  GitHub; choose the type, label to stand alone, style with a small palette, then
  render, look, and revise>
skill: true
---
```

Write about 100–150 lines of positive rules, each with a short reason, followed by the
examples. Follow the research report's draft outline, with its corrections. Give the
sections named headings, such as `## Know the one thing the figure says`. Do not number
them, because phase `diagrams` inlines this body after the macro's numbered steps and
cites the sections by name. The body must read correctly both as a standalone `SKILL.md`
and inlined into a prompt. Cover each of the following.

**Know the one thing the figure says.** Write down three things before drawing:

- the reader question the figure answers;
- a one-sentence takeaway, which becomes the caption;
- the source passages that support every entity and relation.

Do not draw when you cannot name the question, or when one sentence of prose already
answers it. A picture earns its place by showing order, branching, state, ownership, or
before versus after.

**Choose the type from the content's shape:**

- Decision or routing rule: `flowchart TD`, with questions as hexagons of about five
  words or fewer and answers on the edges.
- Modes or lifecycle: `stateDiagram-v2` with `state "Readable name" as ID`, or a
  flowchart with one subgraph per mode.
- Actors over time: `sequenceDiagram` with at most four participants and `autonumber`.
- Today versus proposed: two subgraphs that reuse the same node names.
- Ownership or layers: `flowchart TB` with one subgraph per owner.
- `classDiagram` and `erDiagram` only for real structure.
- `quadrantChart` only when the source itself places items on two axes.
- If a list or table answers better, draw nothing.

Avoid `timeline`, `pie`, `mindmap`, `@{ shape: … }`, and newer diagram families. Their
GitHub support is unverified, and `timeline` loses its title.

**Make it stand alone.** The unit is figure, title, and caption together.

- Phrase the frontmatter `title:` as the question or the claim.
- Write node labels as a bold name plus a short gloss in a backtick Markdown string, at
  most about eight words on two lines.
- Label every non-obvious edge with a verb or a condition.
- Never mix data-flow, ordering, and dependency arrows without saying which is which.
  Say whether converging arrows mean "all" or "any".
- Gloss abbreviations and project terms.
- Use no dummy nodes, such as a "Choose one" box or a mode drawn as a sink state.
- Mark proposed, optional, and failure paths in words and in style.
- Use a subgraph only for a real mode, owner, or phase. Never require start and end
  nodes or subgraphs.
- Quote every label. Never use `end` as a node id. `accTitle` and `accDescr` are
  optional; they are invisible in the PDF.

**Fit the page:**

- Use the soft target, hard floor, and fit formula above.
- Tighten `flowchart.rankSpacing`, `nodeSpacing`, and `wrappingWidth: 260` before
  cutting content.
- Prefer hexagons to diamonds, which are about 24% taller.
- Split into two figures rather than shrinking text.
- Try the default layout first, and `layout: dagre` if the render shows jogs. Let the
  look step choose; never hard-code either.
- Draw edges node to node when the arrowhead matters.

**Style with a small semantic palette.** Use at most three colour classes per figure,
pasted from this fixed set. Each colour class sets `fill`, `stroke`, and `color`, so
text stays readable in GitHub dark mode. `proposed` is a dash modifier stacked with a
second `class` line:

```text
classDef focus    fill:#E8F1FB,stroke:#3B78C2,color:#0F2E52
classDef decision fill:#FFF7E6,stroke:#C77C02,color:#4A3000
classDef outcome  fill:#E7F5EC,stroke:#2E7D4F,color:#123D24
classDef risk     fill:#FDECEC,stroke:#C0392B,color:#5F1414
classDef muted    fill:#F3F4F6,stroke:#6B7280,color:#1F2937
classDef proposed stroke-dasharray:5 3
```

- Give subgraphs `fill:none` and the stroke of their class through `style`.
- Use `-.->` for proposed or optional paths and at most one `==>` main path.
- Never let colour carry meaning alone.
- Use no theme directives, `click`, HTML other than `<br/>`, icons, emoji, or
  index-based `linkStyle`.
- Verify that every class renders under the mirror config, and that stacking two `class`
  lines applies both.

**Render, look, revise.** Give the exact recipe:

1. Make a short scratch dir (`mktemp -d /tmp/mmd.XXXXXX`) and export `TMPDIR=<scratch>`.
   Chrome fails on long socket paths.
2. Write `pdf.json`, the mirror config, with a heredoc.
3. Pick the CLI once. Probe the pinned `npx -y -p @mermaid-js/mermaid-cli@12.0.0 mmdc`
   first, which matches the PDF's Mermaid 12.2.0, then fall back to a global `mmdc`. Use
   the smoke render (`printf 'flowchart TD\n  A["ok"]\n'`) and keep the existing
   `--no-sandbox` puppeteer retry. Record `<mmdc> --version`.
4. Render each figure to SVG and PNG with `-c <scratch>/pdf.json`. For the PNG, add
   `-s 1.5 -b white`, bound the run with `timeout`, and retry once on a Puppeteer
   `ConnectionClosedError`.
5. Measure the root `<svg>` `width` and `height`, falling back to the `viewBox` size,
   and print `W×H px, fit N%`. Ship one small, verified command for this, such as a
   `python3 -` heredoc. Never grep `max-width`. Check the command against real mmdc
   12.0.0 output for a flowchart, a state diagram, and a sequence diagram.
6. Open the PNG with your image-viewing tool and answer the reader question from the
   image alone. Then check for:
   - clipped or overlapping labels;
   - hidden arrowheads;
   - crossed edges;
   - a single reading direction;
   - the measured size.

   Trace every box and arrow to a source sentence. If your runtime cannot view images,
   say so and rely on the measured and source checks.

7. Revise at most twice, then drop the figure.

Treat a browser start failure as an environment problem, not a bad diagram.

### Worked examples

Close the skill with two or three compact worked examples, each followed by a one-line
note of its measured size and fit scale. Render every example with the skill's own
recipe; this also tests the recipe. Every example must print at 100% if possible, and at
75% at the very least. Use generic, public-safe wording.

- **Decision rule.** A hexagon routing decision, adapted from cld's task-routing redraw:
  `research:202610/research_swarm_mermaid_diagram_craft/research_swarm_mermaid_diagram_craft__cld.md`.
- **Modes.** cld's lock redraw from the consolidated report. Fix the dashed edge whose
  arrowhead hides against the AUTO border, for example by ending edges on nodes.
- **Optional third.** A today-versus-proposed pair, or a short sequence diagram, if it
  stays within the line budget.

### Point the lead at the skill

In `xprompts/research_swarm.md`, add one sentence to the lead segment's report-writing
step, for example: "If the report draws or carries over a Mermaid diagram, draw it with
your `/mermaid_diagrams` skill."

Add a fallback for runtimes where the skill is not yet deployed: a bare-name command
that prints the skill. Verify that `sase macro show skill/mermaid_diagrams --format raw`
or `sase macro show /mermaid_diagrams --format raw` works, and use whichever does. Never
write `#skill/…` in this sentence. After whitespace or a quote it would inline the whole
skill into the lead prompt, which Adj 7 rules out.

> [!decision] lead_reach = all Also add the same one-sentence pointer, with its
> fallback, to `xprompts/research.md`, so swarm researchers and standalone `#research`
> reports get it. Test that `#research` contains the pointer and not the skill body.

### Tests and docs

- In `tests/test_macro_loading.py`, or a new `tests/test_skill_loading.py`:
  - `skill/mermaid_diagrams` loads through sase's plugin skill loader
    (`load_skills_from_plugins`), with `skill_name == "mermaid_diagrams"`. Add a compat
    import to `tests/sase_macro_compat.py` if it is needed.
  - Pin the key contract phrases: the reader question, at most three colour classes, the
    0.75 fit floor, `-c <scratch>/pdf.json`, the pinned
    `@mermaid-js/mermaid-cli@12.0.0`, two revision rounds, and no theme directives.
  - Add the expansion and generation round-trip tests from the authoring gotchas.
  - The lead segment contains the pointer and not the skill body. Run the segment-count
    test.
- In `tests/test_wheel_contract.py`, require the wheel and sdist to contain
  `skills/mermaid_diagrams.md`. Extend `tests/wheel_smoke.py` if it enumerates loaded
  resources.
- In the README, add a short "Skills" entry for `/mermaid_diagrams`
  (`#skill/mermaid_diagrams`). Mention `skills/*.md` in `docs/architecture.md`'s
  `sase_macros` row and in the AGENTS.md architecture bullet. Note there that a
  home-scope copy can override the skill.

### Verification

- `sase tool run check` passes.
- Every example is rendered, viewed, and measured with the recipe, and its notes match
  the measurements.
- The no-bob test passes.

## Phase `diagrams`: rewire research/diagrams around reader questions and the skill

Edit `src/sase_research_artifacts/xprompts/research_diagrams.md`. Keep everything the
research says still holds:

- the scope and source rules;
- the fidelity rules;
- the no-table-redraw rule;
- the 3/5 caps and their fourth/fifth-diagram section rule;
- the anchor rules;
- the manifest location, label, and no-overwrite rules;
- the `diagrams` variable contract;
- fail-open behaviour;
- the final-response verdict list.

### Changes by section

- **Description and intro.** Keep the `Propose validated Mermaid diagrams` prefix that
  the test pins. Describe figures that answer named reader questions, not drill-downs
  into panels.
- **§1.** The infographic still comes first when requested. Its panels are candidate
  hints, not the frame.
- **§2 Find candidates.** Keep the rule that the candidate list comes first. Cover each
  `##` section, and each infographic panel when one exists. For each candidate, record:
  - the reader question a picture answers faster than prose;
  - the source passages;
  - the type that fits, per the guide's type section.

  Pictures earn their place by showing order, branching, state, ownership, or before
  versus after.

- **§3 Decide.** Rank passing candidates in this order:
  1. what the bottom line depends on;
  2. today versus proposed;
  3. failure and fallback paths;
  4. drill-downs.

  Reject:
  - anything one sentence already explains;
  - short linear chains;
  - refresh or recompute fan-ins;
  - repeats of an existing diagram, table, or list.

  Keep "Never turn a table into a diagram" and "Never reach zero just because the
  infographic already covers a topic". Say that zero for a survey is a success. Keep the
  rule that an over-large candidate is scoped to its most decision-relevant part.

- **§4 Draw.** Replace the bans list with "draw every diagram by the Mermaid diagram
  guide below". Labels may gloss report terms in plain words but add no facts, and
  proposed designs stay marked proposed. Keep the no-invented-structure sentence and
  give it teeth: trace every box and edge to a sentence.
- **§5 Validate.**
  1. Select the CLI, set the short `TMPDIR`, and write `pdf.json` as the guide's render
     section says.
  2. Render, look, and revise each figure as the guide says: two rounds, then drop.
  3. Run the batch check
     `<mmdc> -c <scratch>/pdf.json -i <scratch>/check.md -o <scratch>/check_render.md`.
     Keep the `Found N mermaid charts in Markdown input` and N-SVG conditions and the
     five-round parse-fix cap, which the test pins. These parse-fix rounds are distinct
     from the guide's two look rounds.
  4. Measure each `check_render-K.svg` with the guide's command. Fail any figure below
     0.75 fit scale; revise it within the look rounds or drop it.

  Delete the `max-width` grep and the 1300 px rule. If no CLI form is usable, the result
  is still `status: failed`.

- **§6 Anchor.** "Anchor a drill-down in the passage that holds its detail" becomes
  "anchor each figure in the passage whose question it answers, not in the bottom line
  or a summary that condenses it".
- **§7 Manifest.**
  - Redraw the `publication-order` example in the new style: title, glossed labels, at
    most three classes, and a takeaway caption. Render it with the mirror config and
    keep it at 100% fit. Keep the id `publication-order`, because the linker's
    reference-sentence example uses it.
  - Each diagram's YAML gains `question:` (the reader question) and `review:`, for
    example `"652x834 px, fit 100%; question answered from the PNG; 2 rounds"`.
  - `caption` becomes the takeaway assertion, at most about 25 words.
  - `check` also records that the PDF-mirroring config was used.
  - Update the field bullets to match.
- **§8.** `considered` counts candidates judged, not "drill-down candidates". The
  final-response list gives each candidate's reader question and verdict.
- **Guide.** Inline `#skill/mermaid_diagrams` exactly once, after step 8, behind a
  one-line lead-in, for example "The Mermaid diagram guide below governs steps 2–5:".
  Steps cite the guide's sections by name.

### Linker

In the linker segment of `xprompts/research_swarm.md`, add one clause where it reads the
manifest: `question` and `review` are the image agent's audit notes, and the linker
never copies them into the report. Change nothing else in the linker. Placement,
anchors, the reference sentence, and the byte-identical fence rules all still hold.

### Tests

- In `tests/test_macro_loading.py`, drop or rewrite the pins on retired wording:
  - `drill-downs`;
  - `adds detail its panel lacks`;
  - `not a repeat of the infographic`, if the wording goes;
  - the four-word label rule;
  - the diamond ban;
  - `max-width: [0-9.]*px`;
  - the unpinned `npx -y -p @mermaid-js/mermaid-cli mmdc`.
- Add pins for:
  - the reader-question candidate list;
  - the ranking order;
  - the rejection rules;
  - `#skill/mermaid_diagrams` appearing once;
  - `-c <scratch>/pdf.json` in the batch check;
  - the 0.75 fail rule;
  - `question:`/`review:` in the manifest example;
  - the removal of the "No init directives, `click`, `style`, `classDef`" ban.
- Assert that the rendered image segment with diagrams on contains a distinctive skill
  sentence and the palette, and that it contains neither with `diagrams=false`.
- Keep `test_standalone_research_image_never_mentions_diagrams` and the nine-segment
  test.
- Assert the linker's new audit-note clause appears only when diagrams run.

### Docs

- In `docs/macros.md`, update the `#research/diagrams` section: selection, the
  validation loop, the manifest fields (`question`, `review`, the 25-word `caption`),
  and the caption-length line. Update the "Diagram handoff" section to note that the
  linker ignores the audit fields.
- In the README, update the `#research/diagrams` bullet so it no longer says "drill into
  the infographic's panels".
- In AGENTS.md, keep the `mmdc`/`npx` fail-open note accurate.

### Verification

- `sase tool run check` passes.
- The redrawn manifest example renders under the mirror config at 100% fit and was
  viewed.
- An expanded diagrams-on swarm prompt shows the guide exactly once and no leftover
  Jinja or raw tags. Check with the tests or `sase macro expand`.

## Out of scope, by design

- **Splitting diagrams into their own `research.{N}.diagrams` segment or `@diagrams`
  alias** (Adj 8). Measure first.
- **A house theme or a vault-free renderer-parity check in the PDF renderer.** Both are
  separate repo decisions.
- **A deterministic Mermaid lint module.**
- **Widening the type list**, beyond `quadrantChart` for real two-axis content.

Record the proposed follow-ups below as `PROPOSED FOLLOW-UP:` notes on the phase bead;
phase workers never create task beads.

## Rollout and follow-ups

### Rollout

- After both phases land, the plugin release goes through release-please and PyPI. The
  user reinstalls the plugin, because global installs are human-only.
- Run `sase skill init` to deploy `/mermaid_diagrams` to every provider. Agents preview
  only, with `--dry-run` or `--diff`.
- Until deployment, `#research/diagrams` still gets the full guide inline, and the
  lead's fallback command covers the gap.

### Follow-ups for phase `diagrams` to record

- **Evaluate after 5–10 swarms with diagrams.** Use the research's criteria:
  - every figure has a title and a recorded `question`;
  - at least 80% print at 100% and none print below 75%;
  - blind ratings on the three reader questions beat the four pre-change diagrams;
  - the linker omission rate does not rise;
  - not every figure is a `flowchart TD`;
  - render rounds and elapsed time stay modest.

  Reopen the segment split only if the model matters, or if the image agent becomes the
  linker's long pole.

- **Optional deterministic lint**, covering node count, longest label, classes missing a
  text colour, banned directives, and measured size, if the look loop proves noisy.
- **Optional PDF-renderer house theme and renderer-parity check**, in the renderer's own
  repo.
