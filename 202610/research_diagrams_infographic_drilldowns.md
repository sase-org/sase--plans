---
tier: tale
title: Research image agent drills into its infographic with Mermaid
goal:
  The swarm image agent lists drill-down candidates panel by panel and proposes every
  Mermaid diagram that reveals report detail its infographic panel could not show.
size: medium
decisions:
  cap:
    ask: Should the per-report diagram cap rise from three to four?
    choices:
      three:
        Keep at most three diagrams; the PDF length budget stays as agreed in the
        original design
      four:
        Allow up to four diagrams; panel-rich reports get one more drill-down, but the
        PDF gets longer
    default: three
    why:
      The failure is zero diagrams, not too few slots. Raise the cap only after the
      pilot shows demand.
    answer: three
  considered_count:
    ask:
      Should the `diagrams` output variable record how many drill-down candidates were
      considered?
    choices:
      record:
        Add a `considered` count to `ready`/`none` variables so the pilot can measure it
      response_only:
        Keep the variable schema; candidates and verdicts appear only in the final
        response
    default: record
    why:
      Additive and free; lets the sase-1mo pilot tell a considered zero from a reflexive
      one
    answer: record
proposed_by: bbugyi200.athena.120
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 12:31:43
status: wip
---

# Plan: Make the research image agent drill into its infographic with Mermaid

## Problem

In a `#research_swarm` run with `image=true`, the `<clan>.image` agent runs
`#research/image` and then `#research/diagrams`. The diagrams step almost always
proposes nothing. Five of five recent image agents I inspected (`research.4v` through
`research.4z`, 2026-10-19) set `diagrams.status: none`. Their reasons were the same each
time: diagrams "would duplicate the infographic", or the flows are "already concise
lists or tables".

That reflex is wrong. Take `research:202610/hermes_agent_sase_provider/...` as an
example. Its infographic panel "Same CLI. Different execution." shows two tiny flows.
The report's section "What the gateway path does to a naive adapter" holds the full
story:

- the client spawns a detached daemon, which inherits its environment;
- later sessions get only an allowlisted environment;
- Ctrl-C detaches the client but the work continues;
- six SASE mechanisms each break in their own way.

The panel "Run the Phase 0 trial first" compresses the report's seven-step spike and its
explicit go/no-go decision rule into two boxes. Each panel is the overview. The report
holds the detail that a Mermaid drill-down could draw. The agent declined both as
duplicates.

## Root cause

All of this is in `src/sase_research_artifacts/xprompts/research_diagrams.md`, in the
linked `sase-research-artifacts` repo:

- **Step 1** ends with "If this prompt also asked for an infographic, finish it first
  and do not repeat what it shows."
- **Step 2** opens with "Zero is a normal outcome, and the usual one for surveys and
  recommendation lists". It also forbids anything that would "repeat the infographic ...
  a table, or a short list."
- **No stated relationship.** Nothing tells the agent how the infographic and the
  diagrams relate. The accepted design report
  (`research:202610/research_swarm_mermaid_handoff_to_linker/research_swarm_mermaid_handoff_to_linker.md`,
  "What is right about the plan") intended it this way: "the infographic gives the
  overview, and each diagram explains one mechanism." That framing never made it into
  the macro.
- **Result.** The agent treats any overlap in topic as repetition. It starts from zero,
  and it never lists what each panel left out.

## Goal

The diagrams step should treat the infographic as an index of drill-down opportunities:

- Enumerate candidates panel by panel.
- Judge each one on whether it reveals report detail the panel could not show.
- Propose every candidate that passes, up to the cap.

All of these stay unchanged:

- fidelity: no invented actors, steps, numbers, orderings, or causal arrows;
- the size and validation rules;
- anchoring;
- the manifest format;
- the linker contract;
- fail-open behavior.

Zero is still a valid outcome, for example for a survey with no mechanism to draw. It
must now be reached by rejecting named candidates, not by default.

## Where the work happens

All edits are in the linked `sase-research-artifacts` repo:

- Open it with `sase repo open sase-research-artifacts -r "<why>"`.
- Use the printed path for every read and write.
- Read its `AGENTS.md` first.
- Verify with `sase tool run check` there, never bare `just check`.

No change is needed in `sase` or `sase-core`. Do not edit `CHANGELOG.md`; release-please
owns it. Do not change `research_image.md`: standalone `#research/image` stays PNG-only,
and `test_standalone_research_image_never_mentions_diagrams` must keep passing. The
swarm's segment wiring in `research_swarm.md` and the linker steps need no change.
Drill-down diagrams use the same manifest, anchors, and placement rules as before.

## Changes to `research_diagrams.md`

Keep the file's tight, imperative style. The wording below is a draft: tighten it as you
like, but keep every rule and every test-pinned phrase listed under Tests. Renumber the
steps and update any internal "(step N)" references.

### Frontmatter `description`

Add the drill-down framing, for example: "Propose validated Mermaid diagrams that drill
into the infographic's panels for a swarm lead's research report, in a companion
manifest that the swarm linker places." Keep the substring
`Propose validated Mermaid diagrams`, which a test pins.

### Opening paragraph

Say that the agent first lists drill-down candidates and then proposes the ones that
help a reader. Keep "The linker is the only publisher: you propose, it places or omits."

### Step 1, Scope and source

Replace the last bullet with: "If this prompt also asked for an infographic, finish it
first. It is the overview, and your diagrams are its drill-downs (step 2)." Leave the
other bullets unchanged.

### New step 2, Find candidates (before Decide)

Draft text:

> The infographic is the overview, and your diagrams are its drill-downs. Each one opens
> a single panel and draws the detail the report gives for it but the panel had no room
> for: the steps, branches, conditions, actors, states, failure or fallback paths, and
> gates behind it. Expanding a panel this way is the purpose of this step, not a repeat
> of the infographic.
>
> Before deciding anything, write a candidate list in your scratch directory:
>
> - For each infographic panel, taken from your generation prompt and the image you
>   checked, name the report passages that hold its detail and what the panel left out.
>   With no infographic in this prompt, use each report section that describes a
>   mechanism instead.
> - Add any mechanism the report explains at length that the infographic left out
>   entirely.
> - Note the diagram type that would fit each candidate (step 4).

### Decide (now step 3)

Replace the opening "Zero is a normal outcome, and the usual one ..." framing and the
"does not repeat" criterion. Keep the fidelity paragraph that follows the criteria ("Add
no actors, steps, numbers, orderings, or causal arrows ..."). Draft text:

> Judge every candidate, then propose each one that passes, best first. Propose at most
> three. Zero is right only when no candidate passes. That is common for surveys and
> recommendation lists, where no panel hides a mechanism. Never reach zero just because
> the infographic already covers a topic.
>
> A candidate passes when all of these hold:
>
> - it draws three or more connected parts: a flow, a state lifecycle, an exchange
>   between actors over time, a dependency structure, or a phased plan;
> - it adds detail its panel lacks: at least two steps, branches, conditions, actors, or
>   states the panel does not show. Redrawing a panel at the same level of detail is a
>   repeat;
> - a reader would grasp that detail faster from the picture than from the prose;
> - every box, arrow, and label restates something the report says;
> - it does not repeat a Mermaid diagram already in the report, a table, or a short list
>   that already shows every step, branch, and condition the diagram would draw. Never
>   turn a table into a diagram.
>
> Rank passing candidates by how much report detail they reveal. Prefer candidates from
> different panels over two from one panel. When a panel's detail is too big for one
> diagram (step 4), draw its most decision-relevant part, such as the go/no-go rule or
> the failure path, and scope the caption to that part. Do not drop the candidate.

> [!decision] cap = four Replace "Propose at most three" with "Propose at most four"
> everywhere it appears in the macro, docs, and tests. Also update the README and docs
> wording that says "at most three".

### Draw (now step 4)

Leave the content unchanged. Drill-downs are bound by the same size rules: about 650 px
column, about half a page, at most four boxes side by side, about five ranks.

### Anchor (now step 6)

Add one sentence: "Anchor a drill-down in the passage that holds its detail, not in the
bottom line or summary the panel condenses." The linker places each diagram after its
anchored block, so a body anchor puts the detail diagram next to the detailed prose.

### Register and report (now step 8)

The `none` reason must name the candidates considered and why each one failed. For
example:
`"reason": "Survey of five naming options; the three panels compare options and hide no flow."`

Reword the bullet that says to mention the outcome in the final response. The final
response must list every candidate with its verdict: proposed, or rejected with a
reason. A diagram failure still never fails this agent.

> [!decision] considered_count = record Add an integer `considered` field to the `ready`
> and `none` examples, for example
> `{"status": "ready", "count": 2, "considered": 4, "manifest": ..., "source_ref": ...}`
> and `{"status": "none", "count": 0, "considered": 3, "reason": ...}`. Keep `status` as
> the first key, so the existing test prefixes still match. `failed` may omit it. The
> linker reads only `status`, so it needs no change.

> [!decision] considered_count = response_only Leave the three `sase var set diagrams`
> examples unchanged. Candidates appear only in the final response.

## Docs

- **`docs/macros.md`, `#research/diagrams` section.** Rewrite the paragraph that begins
  "Zero diagrams is the normal outcome ..." to describe the overview/drill-down
  relationship, the candidate list, the pass criteria including "adds detail its panel
  lacks", and zero only when no candidate passes. Under "Status variable and artifact
  label", update the JSON examples to match the decision on `considered`.
- **`README.md`, `#research/diagrams` bullet.** Say the diagrams drill into the
  infographic's panels. Keep the Mermaid CLI wording.
- **Other docs.** Check `docs/architecture.md`, `AGENTS.md`, and `docs/configuration.md`
  for wording that contradicts the new behavior. As of planning, none describes the
  selection criteria. Change only real contradictions.

## Tests (`tests/test_macro_loading.py`)

- **`test_research_diagrams_restraint_check_and_anchor_rules`.** Drop the
  `"Zero is a normal outcome"` assertion. Pin the new contract instead: `drill-downs`,
  `not a repeat of the infographic`, `Never reach zero just because the infographic`,
  `adds detail its panel lacks`, and `Never turn a table into a diagram`. Keep the
  existing check, anchor, `byte-identical`, and `Propose at most three` assertions (or
  `four`, per the `cap` decision).
- **New test for the candidate step.** Assert that the candidate list comes before the
  decision: the index of the candidate-list instruction is less than the index of
  `Judge every candidate`. Also assert the no-infographic fallback ("each report section
  that describes a mechanism") is present.
- **`considered` test (record branch only).** Assert that `"considered"` appears in the
  `ready` and `none` `sase var set diagrams` examples.
- **Final-response test.** Assert the rule that the final response lists every candidate
  with its verdict.
- **Leave alone.** Keep `test_standalone_research_image_never_mentions_diagrams` and the
  description test (`Propose validated Mermaid diagrams`) unchanged. Both must pass.

## Pilot bead note

Open task `sase-1mo` ("Measure the Mermaid handoff in a 20-swarm post-release pilot")
measures the share of runs with at least one diagram. Its reopen criteria treat
proposals that "duplicate the infographic" as a failure signal.

Read the `sase_beads` reference memory first. Then append a note to `sase-1mo`, without
touching its status, saying three things:

- the selection criteria changed in this commit, from restraint-first to panel
  drill-downs;
- pilot swarms must record which plugin version they ran;
- a diagram that expands an infographic panel with report detail is the intended
  outcome, not a duplicate.

Skip the note if `sase-1mo` is already closed.

## Verification

1. **Tests.** Run `sase tool run check` in the plugin repo: ruff, mypy, and pytest must
   all pass.
2. **Paper check.** Read two published reports with `sase artifact read` and walk the
   new step 2 and step 3 against each. Record the outcome in the final response.
   - `research:202610/hermes_agent_sase_provider/hermes_agent_sase_provider.md`, with
     its `_infographic.png`: at least one drill-down candidate should pass, such as the
     Phase 0 decision rule or the gateway-path exchange.
   - A survey-style report, such as
     `research:202610/sase_next_direction_reading_list/...`: the new criteria should
     still yield zero, with named rejected candidates.
3. **No live swarm.** Do not launch a live swarm, and do not write a manifest into the
   research repo. Live behavior is measured by the `sase-1mo` pilot after release.
