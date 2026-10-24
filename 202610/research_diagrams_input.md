---
tier: tale
title: Add a diagrams input to the
goal: '#research(diagrams=true) has the writer add skill-guided Mermaid diagrams;
  the default stays unchanged.'
size: small
decisions:
  minimum:
    ask: Should `diagrams=true` require at least one diagram in every report?
    choices:
      at_least_one: Every report gets a figure; only a Mermaid CLI that will not render
        excuses none
      allow_zero: Surveys and other mechanism-free reports may ship with no figure,
        and the final response says why
    default: at_least_one
    why: The caller asked for one or more; the skill's zero-figure rule targets the
      swarm companion
    answer: at_least_one
proposed_by: bbugyi200.athena.13p
proposed_with: claude/opus
decided_by: auto
status: done
---

# Add a `diagrams=true|false` input to `#research`

## Goal

`#research` gains a `diagrams` boolean input that defaults to `false`. When it is
`true`, the macro tells the writing agent to use its `/mermaid_diagrams` skill to add
one or more useful Mermaid diagrams to the research report it creates, placed where they
help. When it is `false` or omitted, the expansion is byte-identical to today's.

## Where the code lives

Everything changes in the linked **`sase-research-artifacts`** plugin repo. Open it with
`sase repo open sase-research-artifacts -r "<reason>"`, work only in the printed path,
and read that repo's `AGENTS.md` first. Nothing in the primary `sase` repo (code, docs,
tests, or memory) references `#research`'s inputs, so the primary repo needs no change.

Relevant files in the plugin repo:

- `src/sase_research_artifacts/xprompts/research.md`: the `#research` macro. It has two
  inputs today (`report_target: path`, `suffix: word`). The body has a three-way
  location branch (`report_target`, then `suffix`, then the default), followed by the
  "After the write succeeds, register the report…" paragraph.
- `src/sase_research_artifacts/skills/mermaid_diagrams.md`: the shipped skill, exposed
  as `/mermaid_diagrams` and as the inlineable `#skill/mermaid_diagrams`. It already
  covers figure selection, type choice, standalone labels, the PDF palette, and the
  "Render, look, and revise" recipe (pinned `mmdc`, PDF-mirroring config, 75% fit floor,
  two visual revisions).
- `src/sase_research_artifacts/xprompts/research_swarm.md`: the lead segment already
  carries an on-demand skill pointer (the paragraph starting "When you author or carry
  over Mermaid diagrams, load `/mermaid_diagrams` before drafting…"). Reuse its fallback
  wording. The swarm's researchers call `#research(suffix=<short>)`, and the swarm has
  its own diagrams pipeline (`#research/diagrams` plus the linker), so researchers must
  keep the default `diagrams=false`. **Do not edit `research_swarm.md`.**
- `tests/test_macro_loading.py`: macro contract tests (`_research_macros()`,
  `expand_single_macro`, `test_research_prompt_declares_typed_input`,
  `test_research_registers_report_in_every_branch`,
  `test_research_prompt_suffix_branch_renders_without_artifacts`).

## Step 1: Add the input and the gated instruction block (`research.md`)

### Input

Append after `suffix`, so existing input positions are unchanged:

```yaml
- name: diagrams
  type: bool
  default: false
  description:
    Add one or more useful Mermaid diagrams to the report, chosen, drawn, and
    render-checked with the `/mermaid_diagrams` skill. Off by default.
```

SASE's `bool` coercion already accepts `true/false/yes/no/on/off/1/0`, so the template
can test `{% if diagrams %}` directly. The `suffix != 'false'` workaround is not needed
here.

### Body block

Insert a `{% if diagrams %} … {% endif %}` block **between the location branch's closing
`{% endif %}` and the "After the write succeeds, register…" paragraph**. That placement
puts the diagram work before registration, so the registered snapshot already contains
the figures. Use Jinja whitespace control (`{%-` / `-%}`) so that the expansion with
`diagrams` omitted or `false` is byte-identical to today's output in all three location
branches. Step 3 verifies this.

Target wording (adjust lightly for flow, but keep every rule):

```markdown
Add one or more useful Mermaid diagrams to this report. Load your `/mermaid_diagrams`
skill before drafting any figure, and follow it to choose, draw, style, and render-check
every diagram. If that skill is not deployed on this host, read the same guide with
`sase macro show skill/mermaid_diagrams --format raw` (or
`sase macro show /mermaid_diagrams --format raw`) and use whichever form works.

- Choose figures by the skill's rules: name the reader question each figure answers,
  prefer what the report's conclusion depends on, and draw only actors, steps, and
  relations the report itself states. <MINIMUM SENTENCE: see the decision below>
- Place each figure directly after the paragraph, or after the whole list, whose
  question it answers. Never put a figure directly under a heading or inside a list or
  blockquote. Follow each fence with its one-sentence caption as one italic line.
- Keep render scratch files (`.mmd`, SVG, PNG, and configs) outside the research
  checkout; only the report belongs there. Each fence in the report must be
  byte-identical to the source that passed the skill's render check.
- Finish the diagrams before you register the report below, so the registered snapshot
  includes them.
- Diagram trouble never blocks the report. If no Mermaid CLI renders on this host, write
  and register the report without diagrams and say so in your final response.
- In your final response, list each diagram you added with the reader question it
  answers.
```

> [!decision] minimum = at_least_one MINIMUM SENTENCE: "Add at least one figure, usually
> one to three and never more than five. Even a survey gets one: draw the strongest
> candidate the report supports, such as its decision rule or comparison structure,
> scoped to the part the conclusion depends on."

> [!decision] minimum = allow_zero MINIMUM SENTENCE: "Add a figure only where it passes
> the skill's tests, usually one to three and never more than five. When no reader
> question needs a picture, add none and say why in your final response." With this
> choice, the fail-open bullet also covers "no candidate passed".

Hard rules for this block:

- **Never write the literal `#skill/mermaid_diagrams`** (or any other `#skill/…`
  reference) in the body. Macro expansion would inline the ~350-line guide into every
  `diagrams=true` prompt. The user asked for the on-demand `/mermaid_diagrams` skill,
  matching the swarm lead's pointer.
- Keep the block free of swarm vocabulary (manifest, linker, `figure-<id>` anchors).
  Standalone `#research` writes figures straight into its own report.
- The literal `$(…)` and `{{ … }}` already in the body stay as they are.

## Step 2: Tests (`tests/test_macro_loading.py`)

1. Update `test_research_prompt_declares_typed_input` so `#research`'s inputs are
   `[("report_target", "path"), ("suffix", "word"), ("diagrams", "bool")]`, with
   `research.inputs[2].default is False`.
2. Add `test_research_diagrams_off_by_default`, parametrized over the three branch arg
   sets `{}`, `{"suffix": "a"}`, and `{"report_target": "x.md"}`:
   - the expansion with no `diagrams` equals the expansion with `"diagrams": "false"`;
   - its lowercased text contains neither `mermaid` nor `diagram`.
3. Add `test_research_diagrams_true_points_to_the_skill`, parametrized over the same
   three arg sets with `"diagrams": "true"` added:
   - it contains `/mermaid_diagrams`,
     `sase macro show skill/mermaid_diagrams --format raw`, and
     `sase macro show /mermaid_diagrams --format raw`;
   - it contains neither `#skill/mermaid_diagrams` nor `## Mermaid diagram craft` (so
     the guide is not inlined), and no `{%` or `{{ diagrams`;
   - the registration command
     `sase artifact create -p "<absolute-report-path>" -l "research:<repo-relative-report-path>"`
     is still present, and it appears **after** the `/mermaid_diagrams` pointer (compare
     `str.index`);
   - branch text survives: `x.md` for `report_target`, `<stem>__a.md` for `suffix`, and
     "new markdown file under" for the default;
   - it asserts the chosen minimum wording: "at least one" under `at_least_one`, or "add
     none" under `allow_zero`.
4. Leave the swarm tests alone. Their existing exact assertions
   (`"some topic #research(suffix=cdx)" in cdx`, and so on) and
   `test_research_swarm_points_only_the_lead_to_the_skill`, which checks that researcher
   segments never mention `/mermaid_diagrams`, already guard that swarm researchers keep
   the default.

## Step 3: Docs (plugin repo)

- `docs/macros.md`, `#research` section: add a `diagrams` row to the Input table
  (`bool`; "Default `false`. When `true`, adds render-checked Mermaid diagrams to the
  report via the `/mermaid_diagrams` skill"). After the registration paragraph, add a
  short `### Diagrams` subsection covering:
  - what `diagrams=true` asks for (the skill pointer and its `sase macro show`
    fallback);
  - the chosen minimum, and the cap of three, or five for complex reports;
  - placement after the passage a figure answers, with an italic caption;
  - scratch kept outside the checkout;
  - fail-open when no Mermaid CLI renders;
  - an example: `#research(diagrams=true)`;
  - a note that `#research_swarm` never forwards it (the swarm's own `diagrams=true`
    runs `#research/diagrams` plus the linker instead).
- `README.md`:
  - append a clause to the `## Macros` bullet for `#research` (for example,
    "`diagrams=true` adds render-checked Mermaid diagrams via `/mermaid_diagrams`");
  - in `## Skill`, name `#research(diagrams=true)` as a second on-demand consumer
    alongside the swarm lead.
- `docs/architecture.md`, the paragraph that says the lead "receives an on-demand
  pointer": add `#research(diagrams=true)` as another on-demand consumer.
- `AGENTS.md` in the plugin repo, the `skills/mermaid_diagrams.md` architecture bullet:
  add the same consumer. This file is hand-maintained plugin documentation. The plugin
  has no `sase/memory/`, so it is not generated SASE memory.
- Do **not** edit `CHANGELOG.md`. release-please owns it, and the conventional commit
  subject below feeds it.

## Step 4: Verify

1. If the plugin checkout has no `.venv`, run `just install-venv` there first. It builds
   a local venv routed to the linked sase checkout; it is not a global install.
2. **Byte-identity check:** before editing `research.md`, use the venv's python with
   `expand_single_macro` (imported via `tests/sase_macro_compat.py` or
   `sase.macro.processor`) to capture `#research` expansions for `{}`,
   `{"suffix": "a"}`, and `{"report_target": "x.md"}`. After editing, confirm that each
   matches the new expansion both with no `diagrams` and with `diagrams="false"`. Then
   print the three `diagrams="true"` expansions and read them for flow and whitespace
   (no doubled blank lines around the block).
3. Run `sase tool run check` in the plugin checkout. `check` is guarded, so never run
   bare `just check`. It must pass: ruff, mypy, and pytest.
4. `just test-wheel` is not needed, because no packaged resource files are added or
   removed.

## Out of scope

- `#research_swarm` and its own `diagrams` flag and pipeline; `#research/diagrams`;
  `#research/image`.
- A `diagrams` passthrough on `#research/prompt`, which embeds a bare `#research`.
- Any change to the `/mermaid_diagrams` skill itself.
- Releasing or reinstalling the plugin. The change reaches the installed SASE on the
  next plugin release or upgrade, which the user runs.

## Commit

One plugin-repo change set, declared through the normal host-owned finalizer, with the
subject `feat(research): add diagrams input to #research`.
