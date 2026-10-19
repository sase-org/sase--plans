---
tier: tale
title: Let the research swarm image agent propose up to five Mermaid diagrams
goal:
  The research swarm image agent may propose up to five validated Mermaid diagrams for
  complex research reports that warrant them, while ordinary reports keep the
  three-diagram ceiling.
size: small
decisions:
  cap_rule:
    ask:
      Should diagrams four and five require a complex report, a distinct subject, and
      their own `##` section?
    choices:
      gated:
        Three stays the norm; a 4th or 5th needs a complex report, a new subject, and
        its own section
      flat:
        One flat cap of five under the existing pass tests; ordinary reports may drift
        above three
    default: gated
    why:
      You asked for five only on complicated reports; a flat cap invites quota-filling.
    answer: gated
  linker_cap:
    ask: Should the swarm linker also enforce the five-diagram ceiling?
    choices:
      enforce: The linker omits every manifest diagram after the fifth as over the cap
      trust:
        The linker places every valid proposal; only the image agent's prompt holds the
        cap
    default: enforce
    why: The linker is the only publisher and already gates every proposal.
    answer: enforce
proposed_by: bbugyi200.athena.125
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-19 16:11:04
status: wip
---

# Let the `#research_swarm` image agent propose up to five Mermaid diagrams

## Goal

Today the swarm's image agent (`<clan>.image`, which runs `#research/diagrams` after
`#research/image`) may propose **at most three** Mermaid drill-down diagrams. Raise the
ceiling to **five**, but only for complex research reports that warrant it. Ordinary
reports keep today's behavior: zero to three diagrams, with zero still the usual result
for surveys and recommendation lists.

## Where the work lives

Every change is in the linked **`sase-research-artifacts`** repo. Open it first with
`sase repo open sase-research-artifacts -r "<reason>"`, work only in the path it prints,
and read that repo's `AGENTS.md` before editing. Nothing changes in the primary `sase`
repo, in `sase-core`, or in any memory note. No memory note mentions the diagram cap;
the only places that state it are listed below.

Files to change (paths relative to the linked repo root):

- `src/sase_research_artifacts/xprompts/research_diagrams.md`: the image agent's diagram
  instructions, where the cap lives.
- `src/sase_research_artifacts/xprompts/research_swarm.md`: the linker segment
  (`linker_cap` only).
- `docs/macros.md` and `README.md`: the user-facing descriptions of the cap.
- `tests/test_macro_loading.py`: the prompt-content contract tests.

Do **not** edit `CHANGELOG.md`; release-please generates it from the conventional commit
(`feat(research-diagrams): ...`).

## Non-goals

- No new macro input. `#research/diagrams` keeps taking no inputs
  (`test_research_diagrams_takes_no_inputs` must still pass), and `#research_swarm` gets
  no `max_diagrams` knob. The ceiling depends on the agent judging the report, not on a
  launch argument.
- No change to the `diagrams` status variable schema (`count` may now reach 5), the
  manifest format, the anchor format, the drawing and size rules in step 4, the swarm
  topology, or any agent count.
- No change to standalone `#research/image`, which must still never mention diagrams.

## Current text being replaced

In `research_diagrams.md`, step **3. Decide** opens with:

```text
Judge every candidate, then propose each one that passes, best first. Propose at most three.
Zero is right only when no candidate passes. ...
```

Step **5. Validate** ends its fix loop with
`Fix, simplify, or drop that diagram and rerun; stop after three rounds.` Because `mmdc`
stops at the first failing diagram, each round surfaces at most one parse error. A
three-round budget could therefore drop valid fourth and fifth diagrams just because the
loop ran out of rounds.

Step **8. Register and report** ends with
`In the final response, list every candidate with its verdict: proposed, or rejected with a reason.`

## Changes

### 1. `research_diagrams.md`: step 3, Decide

> [!decision] cap_rule = gated

Change the first sentence to
`Judge every candidate, then propose each one that passes, best first, within the cap below.`
and delete `Propose at most three.` from that line. Keep the rest of that paragraph
("Zero is right only when..."), the pass-test bullets, the ranking paragraph, and the
"Add no actors..." paragraph exactly as they are.

Between the ranking paragraph ("Rank passing candidates ... Do not drop the candidate.")
and the "Add no actors, steps, numbers..." paragraph, insert this cap paragraph:

```text
Propose at most three diagrams for most reports. A complex report may warrant up to five:
one that explains four or more distinct mechanisms in separate `##` sections, such as an
architecture, a protocol, a migration, or a multi-phase plan. Propose a fourth or fifth
diagram only when it passes every test above, drills into a panel or mechanism that no higher-ranked proposal covers,
and anchors in a `##` section that holds no other proposal (step 6).
Five is a ceiling, not a target: never lower the bar, split one diagram into two, or add one to reach a count.
A passing candidate the cap leaves out is reported as over the cap (step 8).
```

> [!decision] cap_rule = flat

Replace `Propose at most three.` with `Propose at most five.` on the same line. After
the ranking paragraph, insert one line:
`Five is a ceiling, not a target: never lower the bar, split one diagram into two, or add one to reach a count.`
followed by
`A passing candidate the cap leaves out is reported as over the cap (step 8).` Skip the
step-6 rule in change 3.

**Line-wrapping rule (both branches):** the contract tests use plain substring checks,
so every phrase a test asserts must stay on **one source line**, even if that line runs
long. The file already does this; see the long line in step 2. That covers
`Propose at most three diagrams for most reports`, `four or more distinct mechanisms`,
`no higher-ranked proposal covers`, `Five is a ceiling, not a target`, `over the cap`,
and `stop after five rounds`.

### 2. `research_diagrams.md`: step 5, Validate

Change `stop after three rounds.` to `stop after five rounds.` so the fix budget grows
with the new maximum.

### 3. `research_diagrams.md`: step 6, Anchor (`cap_rule = gated` only)

After the existing "Anchor a drill-down in the passage that holds its detail..."
sentence, add:

```text
If a fourth or fifth diagram's anchor shares a `##` section with another proposal's anchor, drop that diagram (step 3).
```

### 4. `research_diagrams.md`: step 8, Register and report

Change the final-response sentence to
`In the final response, list every candidate with its verdict: proposed, rejected with a reason, or over the cap.`
Keep the following `none`-reason sentence unchanged.

### 5. `research_swarm.md`: linker step 2, Inventory (`linker_cap = enforce` only)

Step 2 of the linker segment contains this text, rendered only when `run_diagrams`:
`For each manifest diagram, record its ID against the one lead block ... Mark a diagram whose anchor matches zero or several blocks as omitted.`
Inside that same `{%- if run_diagrams %} ... {%- endif %}` block, append:

```text
Mark every diagram after the fifth in manifest order as omitted (over the cap).
```

Keep it on one source line. The linker's final response already lists omitted IDs with
reasons, and its step-6 check ("Every manifest diagram appears exactly once,
byte-identical, or is named as omitted") already covers the omitted diagrams, so no
other linker text changes. Do not add diagram text outside `run_diagrams` blocks:
`test_research_swarm_linker_diagram_rules_present_when_diagrams_off` and the
diagrams-off branches of the matrix test must keep passing.

With `linker_cap = trust`, skip this change and its tests.

### 6. Docs

`docs/macros.md`, in the `#research/diagrams` section:

- Gated: replace `then proposes every candidate that passes, at most three, best first.`
  with wording that says: it proposes every passing candidate, best first; at most three
  for most reports and up to five for a complex report that explains four or more
  distinct mechanisms in separate `##` sections; each fourth or fifth diagram must drill
  into a panel or mechanism that no higher-ranked proposal covers and must anchor in a
  `##` section that holds no other proposal; five is a ceiling, not a target. Flat: say
  "at most five, best first" and keep the ceiling-not-target sentence.
- Validation subsection: change `with a three-round cap` to `with a five-round cap`.
- Status-variable subsection: change
  `The final response lists every candidate with its verdict.` so it names the three
  verdicts: proposed, rejected with a reason, or over the cap.
- `### Diagram handoff`: change `proposes zero to three validated Mermaid diagrams` to
  `zero to five` (gated: add "more than three only for a complex report"). With
  `linker_cap = enforce`, add one sentence: the linker places at most five and omits any
  later manifest entry as over the cap.

`README.md`, in the `#research/diagrams` bullet under `## Macros`: add a short
parenthetical with the count. Gated: "(at most three, or up to five for a complex
report)". Flat: "(at most five)".

Reflow the edited `docs/macros.md` and `README.md` paragraphs to those files' existing
width (about 88 columns). The macro prompt files follow the one-line rule from change 1
instead.

### 7. Tests (`tests/test_macro_loading.py`)

- In `test_research_diagrams_restraint_check_and_anchor_rules`, remove the
  `assert "Propose at most three" in content` line. The new test below owns the cap.
- Add `test_research_diagrams_cap_scales_to_five_for_complex_reports` asserting on
  `_research_macros()["research/diagrams"].content`:
  - gated: `"Propose at most three diagrams for most reports"`,
    `"may warrant up to five"`, `"four or more distinct mechanisms"`,
    `"no higher-ranked proposal covers"`, and
    ``"shares a `##` section with another proposal's anchor"``. Flat:
    `"Propose at most five."`;
  - both: `"Five is a ceiling, not a target"`, `"over the cap"`,
    `"stop after five rounds"`, `"stop after three rounds" not in content`, and
    `"rejected with a reason, or over the cap"`;
  - ordering: the cap text's index is greater than
    `content.index("Judge every candidate")`. This keeps the candidate list, then the
    judging, then the cap, in that order.
- `linker_cap = enforce` only:
  - In `test_research_swarm_diagrams_image_matrix`, assert
    `"Mark every diagram after the fifth in manifest order as omitted (over the cap)." in linker_seg`
    in the `run_diagrams` branch, and that `"after the fifth in manifest order"` is
    **not** in `linker_seg` in the diagrams-off branch.
  - In `test_research_swarm_linker_diagram_rules_present_when_diagrams_off`, assert
    `"after the fifth in manifest order" not in linker`.
- Leave every other test unchanged. In particular,
  `test_research_diagrams_takes_no_inputs` and
  `test_standalone_research_image_never_mentions_diagrams` must still pass unchanged.

## Verification

From the linked repo root:

1. `sase tool run check`. This is lint plus tests; `check` is guarded, so do not run
   bare `just check`. Fix anything it reports. Run `just fmt` first if ruff format flags
   the test file.
2. Reread the final step 3 of `research_diagrams.md` from start to finish, as the image
   agent would. Confirm that it still says zero is the normal answer for surveys, that
   three is the stated default, that the condition for going above three can be checked
   from the candidate list the agent already wrote in step 2, and that no sentence reads
   like a target the agent should fill.
3. `git diff` should touch only the five files listed above.

## Acceptance

- The image agent's instructions allow up to five diagrams, and (gated) only for complex
  reports where each extra diagram covers a new panel or mechanism in its own `##`
  section. Ordinary reports still stop at three.
- The validation fix loop allows five rounds.
- The final response labels passing candidates that the cap left out as over the cap.
- (`linker_cap = enforce`) The linker never publishes more than five proposed diagrams.
- Docs and README state the new ceiling. `sase tool run check` passes in
  `sase-research-artifacts`.
