---
tier: tale
title: Opt-in critique agent for
goal:
  "#research_swarm accepts critique (default false) and critique_model (default
  @xlarge); when critique is set, a fresh-context critique agent runs after the lead
  researcher, stress-tests and improves the consolidated report, and writes a companion
  <name>/<name>__critique.md beside it, while the default swarm renders byte-identically
  to today."
size: medium
proposed_by: bbugyi200.athena.0ot
create_time: 2026-09-21 16:27:27
status: wip
---

# Plan: Add an opt-in critique agent to `#research_swarm`

## Goal

Add two inputs to the `#research_swarm` xprompt swarm:

- `critique` (bool, default `false`)
- `critique_model` (word, default `"@xlarge"`)

When `critique` is true, the swarm launches one more agent, `research.{@1}.critique`. It
runs after the lead researcher (`research.{@1}.final`) finishes, stress-tests the lead's
consolidated report, improves on it, and writes a **new** companion report named like
the lead's report plus a `__critique` suffix, in the same directory:
`<YYYYMM>/<name>/<name>__critique.md`. The lead's report is never modified. The result
is two complementary artifacts:

- the lead's synthesis, `<name>.md`
- an adversarial but fair review of it, `<name>__critique.md`, that corrects what is
  wrong, fills what is missing, marks what can be trusted, and gives a revised,
  actionable bottom line

A human or agent can read the two together to judge the research question and act on it.

When `critique` is false (the default), the rendered swarm must be **byte-identical** to
today's output.

## Where the work lives

All changes are in the `sase-research-artifacts` linked repo, **not** the sase repo.
Before reading or editing it, open it with `/sase_repo`
(`sase repo open sase-research-artifacts -r "<reason>"`) and work only in the path it
prints. Its `AGENTS.md` lists the build commands (`just check` = lint + test). Files:

- `src/sase_research_artifacts/xprompts/research_swarm.md`: the swarm, a single Jinja
  template whose top-level `---` lines split it into agent segments
- `tests/test_xprompt_loading.py`: swarm rendering, dependency-graph, and queue tests
- `docs/xprompts.md`, `README.md`, `docs/configuration.md`, `AGENTS.md`,
  `src/sase_research_artifacts/default_config.yml`: documentation and descriptions

Do not edit `CHANGELOG.md`; release-please manages it.

## Current contract (what the implementer must preserve)

- **Researchers.** `cdx`, `cld`, `grk`, `mus`, and `gem` each write `<stem>__<short>.md`
  via `#research(suffix=<short>)`. Each registers its report with
  `sase artifact create -p ... -l "research:<repo-relative-path>"`.
- **Lead.** The lead (`%id:research.{@1}.final`) waits on every surviving researcher. It
  finds their reports through a raw-protected `wait.artifacts` loop, reads them with
  `sase artifact read`, and moves them to `<name>/<name>__<short>.md`. It then writes
  the consolidated report to `<name>/<name>.md`. **Today the lead does not register its
  consolidated report as an artifact.**
- **Image agent.** The optional image segment (`should_generate_image`) waits on the
  lead and forks it (`#fork:research.{@1}.final`).
- **Queue directive.** Every segment authors exactly one
  `%q(w=0.25{% if runners is not none %}, capacity=...{% endif %}{% if priority is not none %}, priority=...{% endif %})`.
- **Optional segments.** Each optional segment is gated by
  `%if(should_run={{ <bool> }})`. When the value is false, the segment disappears from
  the expanded swarm. When it is true, the `%if(...)` is stripped.
- **`wait.artifacts`.** At agent runtime, `wait.artifacts` lists the non-chat artifacts
  registered by the **exact** waited producers, in dependency order. Fields:
  `wait_name`, `agent_name`, `ref`, `kind`, `label`, `path`, `source_path`, `vcs_*`. The
  lookup is implemented in `sase.axe.run_agent_refs.WaitRuntimeNamespace`.

## Design decisions

1. **The critique agent does NOT fork the lead.** It starts in a fresh context and waits
   with `%wait:research.{@1}.final`. Why not `#fork`, which the image agent uses:
   - A forked critic inherits the lead's reasoning and anchors on it, which defeats an
     independent review.
   - `critique_model` may name a different provider than the lead.
   - The lead's context is already huge.

   This matches the swarm's existing rule that agents never read predecessor chat
   transcripts.

2. **Handoff goes through `wait.artifacts`, as researcher → lead does today.** When
   `critique` is true, the lead gets one extra step: register its consolidated report
   with
   `sase artifact create -p "<absolute-report-path>" -l "research:<repo-relative-report-path>"`
   (no `--move`). The critique segment uses the same raw-protected loop to find it.
   - The step renders only when `critique` is true, so the default path is unchanged.
   - Do not use `#fork` or clock-derived paths for the handoff.
3. **The critique agent derives its output directory from the lead's artifact label, not
   from `$(date +%Y%m)`.** A swarm can cross a month boundary, and `<name>` is only
   known at runtime.
   - For this reason the critique segment must **not** append `#research(...)`. That
     xprompt computes the month from the clock.
   - Instead, the segment spells out the same write-without-overwrite and register
     contract inline.
4. **Segment placement.** Append the critique segment as the **last** authored segment,
   after the image segment. Existing rendered-segment tests that treat the image as the
   last segment then stay valid while critique is off. Image and critique both depend
   only on the lead, so they run in parallel.
5. **The critique report is a companion, not a rewrite.** It references the lead's
   report by section heading instead of restating it. It still includes a self-contained
   verdict and revised bottom line that a reader can act on without opening anything
   else. This keeps the two artifacts complementary rather than near-duplicates.
6. **Input order.** Append `critique` and `critique_model` at the **end** of the input
   list, after `should_generate_image`, so positional callers are unaffected.

## Changes

### 1. `research_swarm.md` frontmatter

- **Description.** Update the `description:` to mention the optional critique, for
  example: "Launch independent per-provider research agents, then have a lead researcher
  extend and consolidate their findings. Optionally critique the consolidated report and
  generate an infographic."
- **New inputs.** Append after `should_generate_image`:

  ```yaml
  - name: critique
    type: bool
    default: false
    description:
      Run a critique agent after the lead researcher. It stress-tests and improves the
      consolidated report and writes `<name>__critique.md` beside it without modifying
      the lead's report.
  - name: critique_model
    type: word
    default: "@xlarge"
    description:
      Model alias or provider model for the optional `<clan>.critique` agent. Choosing a
      different provider than `lead_model` reduces shared blind spots.
  ```

### 2. Lead segment: conditional registration step

In **both** lead branches (the researchers branch and the solo branch), add a step 5
after step 4 ("Write the consolidated report to `<name>/<name>.md` ..."). It renders
only when `critique` is true:

```text
5. After the write succeeds, register the consolidated report as a durable snapshot so
   the critique agent, `research.{@1}.critique`, can find it:

   sase artifact create -p "<absolute-report-path>" -l "research:<repo-relative-report-path>"

   Use the consolidated report's actual absolute path and its path relative to the
   research repo root, for example `research:202609/<name>/<name>.md`. Register only the
   consolidated report, and do not pass `--move`. If registration fails, report that
   failure; do not report the task as fully complete.
```

Rules for this step:

- Use Jinja whitespace control (`{%- if critique %}` / `{%- endif %}` as appropriate) so
  that with `critique` false the lead segment renders exactly as it does today.
- Do not change the lead's other steps or its "Final layout" block.
- Do not tell the lead anything else about the critique. It should not write the
  critique itself or hedge because a reviewer is coming.

### 3. New critique segment (appended after the image segment)

**Header computation.** Next to the existing `layout_lines` computation at the top of
the template, build a `critique_layout_body`: the drafts (`├── <name>__<short>.md` per
surviving researcher), then `├── <name>.md`, then `└── <name>__critique.md`, rooted at
`<month-dir>/<name>/`. With no researchers it is just the last two lines.

**Segment text.** Author the segment below. Keep the prose wording, adjusting only for
Jinja correctness and whitespace:

````text
---

%if(should_run={{ critique }}) %id(critique, clan=research.{@1}) %m:{{ critique_model }}
%wait:research.{@1}.final %q(w=0.25{% if runners is not none %}, capacity={{ runners }}{% endif %}{% if priority is not none %}, priority={{ priority }}{% endif %})

You are the critique agent for a research swarm. The lead researcher,
`research.{@1}.final`, has written a consolidated report on the request below. Your job
is to stress-test that report and improve on it in a new companion report. People
deciding what to believe, and agents implementing a solution, will read the lead's report
and yours side by side, so yours must make the pair more trustworthy and more actionable
than the lead's report alone. Critique without improvement is half the job: wherever you
find a problem, do the research needed to fix it.

SASE derives your plan's links from the artifacts you read this turn; use
`sase artifact read` for context you actually used.

Research request:

{{ prompt }}

The lead researcher's registered reports:

<EXACTLY the same {% raw %}...{% endraw %} wait.artifacts loop block the lead segment uses>

Steps:

1. Before you read the lead's report, list for yourself the questions a complete answer
   to the request must settle and the evidence that would settle each. Use that list as
   your checklist so the report's framing does not quietly become yours.
2. From the registered reports above, identify the lead's consolidated report: exactly
   one entry with `wait_name` `research.{@1}.final` whose label has the form
   `research:<YYYYMM>/<name>/<name>.md` (its filename stem equals its parent directory
   name and carries no `__<suffix>`). Ignore every other entry. If there is not exactly
   one such entry, stop and report the missing or ambiguous input instead of guessing.
   Open the research repo with `/sase_repo`, then read the report through its canonical
   research reference (or the `ref` field's `file:<id>` reference if the original has
   moved) using `sase artifact read`. Do not read the lead's or any researcher's chat
   transcript. Never modify, move, or rename the lead's report or any researcher draft.
{% if researchers -%}
3. The lead consolidated {{ researchers | length }} independent researcher {{ "draft" if researchers | length == 1 else "drafts" }}, which now {{ "sits" if researchers | length == 1 else "sit" }} beside its report as {% for r in researchers %}`<name>__{{ r.short }}.md`{% if not loop.last %}, {% endif %}{% endfor %}.
   After forming your own view of the lead's report, read each draft with
   `sase artifact read` through its `research:<YYYYMM>/<name>/<name>__<suffix>.md`
   reference to check synthesis fidelity: findings or caveats the lead dropped,
   disagreements it settled without evidence, and consensus it overstated. Treat the
   drafts as evidence to weigh, not as authorities. A missing draft is worth noting but is
   not a reason to stop.
{% else -%}
3. No independent researchers ran for this dispatch, so the lead's report is a single
   researcher's work with no drafts to cross-check. That makes independent verification
   in the next steps more important.
{% endif -%}
4. Review the lead's report against your checklist, most important questions first:
   - Does it answer the request that was actually asked, including its implicit
     constraints, rather than a neighboring, easier question?
   - Do the load-bearing claims (the ones its conclusions or recommendation depend on)
     hold up? Verify each against primary sources such as official documentation,
     specifications, source code, papers, or data. Check that every cited source says
     what the report says it does, and flag stale versions, numbers, and dates.
   - Is the reasoning sound? Look for unsupported leaps, overgeneralization, false
     dichotomies, cherry-picked evidence, and correlation read as causation.
   - What is the strongest case against its recommendation? Steelman the best
     alternative and decide whether it should win.
   - What is missing: credible alternatives or prior art, costs, risks and failure
     modes, constraints, edge cases, and migration or rollback concerns?
   - Could someone act on it? Check that the recommendation is specific, gives decision
     criteria, states confidence that matches the evidence, and names its open questions.
5. Improve on what you find. Research each significant problem until you can confirm or
   correct it, fill each important gap, and evaluate any alternative the report missed.
   Where you cannot settle something, say exactly which evidence, experiment, or decision
   would settle it. Say which of your own claims you verified against a primary source,
   which you inferred, and how confident you are. Record the load-bearing claims you
   checked and found sound: knowing what to trust is as useful as knowing what not to.
   Do not manufacture criticism or pad with style nitpicks. If the report holds up, say
   so plainly and keep your report short.
6. Write your report to `<name>__critique.md` in the lead report's directory, that is
   `<YYYYMM>/<name>/<name>__critique.md`, taking `<YYYYMM>/<name>/` from the lead
   report's label, never from the current date. Create it without overwrite: if the file
   already exists, stop and report the collision visibly instead of replacing it or
   choosing another name. Follow the research repo's `README.md` conventions when
   present, and mirror the lead report's frontmatter conventions (such as `create_time`,
   `updated_time`, `status`, and `tags`) when it has them. Use this structure, omitting a
   section only when it would be empty:
   - A title naming the lead report's topic, and a relative link to `<name>.md` near the
     top.
   - Verdict: two to five sentences saying whether the lead's main conclusion holds
     (holds, holds with corrections, or does not hold), how far the report can be
     trusted, and the single most important correction.
   - Revised bottom line: the corrected, self-contained answer or recommendation a
     reader or implementer should act on, stating what stands and what changes from the
     lead's report.
   - Findings, ordered by severity: critical (changes the conclusion or
     recommendation), major (materially affects a decision or an implementation), then
     minor (worth fixing). For each, give where it occurs in the lead's report (section
     heading), the problem, the evidence with sources, the fix, and its effect on the
     conclusion.
   - Verified claims: the load-bearing claims you checked and confirmed, with sources.
{% if researchers %}   - Synthesis check: researcher findings the lead dropped or misrepresented, and
     disagreements it left unresolved.
{% endif %}   - Open questions: what neither report settles, and what would settle each.

   Reference the lead's report by section instead of restating it: yours is a companion,
   not a rewrite. Even so, a reader must be able to act on your verdict and revised
   bottom line without opening anything else.
7. After the write succeeds, register your report as a durable snapshot:

   sase artifact create -p "<absolute-report-path>" -l "research:<repo-relative-report-path>"

   Use the report's actual absolute path and its path relative to the research repo
   root, for example `research:202609/<name>/<name>__critique.md`. Do not pass `--move`.
   If registration fails, report that failure; do not report the task as fully complete.

Final layout:

{{ "```text\n" ~ critique_layout_body ~ "\n```" }}
````

**Authoring constraints for the segment.**

- **Reuse the lead's `wait.artifacts` loop verbatim.** The raw-protected block must be
  byte-identical to the lead's so that the tests' `_WAIT_ARTIFACTS_LOOP` /
  `_without_wait_artifacts_loop` helper strips it from both segments. The lead's
  `research:` label filter is correct here too, because the critic waits only on the
  lead.
- **Keep the prose free of accidental expansion tokens.**
  - No `#word` token anywhere: the xprompt processor treats `#name` after whitespace,
    backticks, or quotes as an xprompt reference.
  - No `%word` directive, no `@research:` pointer, and no `$(` shell substitution.
  - No line that is exactly `---`.
  - `{@1}` is intentional.
- **Directive form.** Use `%m:` (as the researchers and lead do), not `%model:`.

### 4. Tests: `tests/test_xprompt_loading.py`

- **Helper.** Give `_swarm_segments` a `critique: bool = False` keyword, mirroring
  `image`, that sets `critique="true"`.
- **Update `test_research_swarm_declares_typed_input`.** Add `("critique", "bool")`
  (default `False`) and `("critique_model", "word")` (default `"@xlarge"`) at indices 16
  and 17.
- **Rename and update the segment-count test.** Rename
  `test_research_swarm_has_seven_top_level_segments` to `..._eight_...` and assert
  `len(segments) == 8`.
- **Update `test_research_swarm_dependency_graph_preserved`.** Unpack eight segments and
  assert all of the following on the critique segment:
  - it contains `%if(should_run={{ critique }})`, `%id(critique, clan=research.{@1})`,
    `%m:{{ critique_model }}`, and `%wait:research.{@1}.final`
  - it contains no `#fork:` and no `%clan(`
  - it has exactly one `%q(`
  - it contains `_WEIGHTED_QUEUE_TEMPLATE` and `priority is not none`

  Add the critique segment to the existing all-segments tuples.

- **Fix unpacking.** Update every
  `*_researchers, final, _image = _authored_swarm_segments()` to also unpack the
  trailing critique segment.
- **Update `test_research_swarm_defaults_to_three_expanded_agents`.** Also assert that
  no segment contains `%id(critique`. Also assert that the default lead segment contains
  neither `sase artifact create` nor `critique`, which guards the byte-identical
  default.
- **New tests:**
  - **Opt-in.** `critique=true` yields 4 segments, and the last one:
    - contains `%id(critique, clan=research.{@1})`, `%m:@xlarge`,
      `%wait:research.{@1}.final`, `__critique.md`, `sase artifact read`, and
      `sase artifact create`
    - contains no `%if(` and no `#fork:`

    The lead segment now contains the registration command
    `sase artifact create -p "<absolute-report-path>" -l "research:<repo-relative-report-path>"`
    and `research.{@1}.critique`. Run `_assert_each_segment_has_one_queue` on all four
    segments; it also proves `plan_typed_launch_units` accepts them.

  - **Custom model routing.** `critique_model=@critique_custom` appears only in the
    critique segment. `lead_model` does not leak into it, and it does not leak into the
    lead.
  - **Image plus critique.** Both options on yields 5 segments, ordered
    `..., final, image, critique`. Image and critique each wait on
    `research.{@1}.final`. The critique does not wait on the image.
  - **`wait` arg.** With `wait="research.0f.final"`, the critique segment does not
    contain `%wait:research.0f.final`.
  - **Queue options.** With `runners` and `priority` supplied, the critique segment
    carries them. Extend one existing runners/priority test with `critique=True`.
  - **Draft listing.** The default critique segment lists `<name>__cdx.md` and
    `<name>__cld.md` and includes the synthesis-check bullet. With `grok=true` it also
    lists `<name>__grk.md`. With all researchers off, it says no independent researchers
    ran, lists no `<name>__` drafts, and omits the synthesis-check bullet. In that case
    the lead's solo branch still carries the registration step.
  - **Final layout.** The rendered critique "Final layout" ends with
    `└── <name>__critique.md` and lists `├── <name>.md`.
  - **End-to-end render.** Mirror
    `test_research_swarm_lead_renders_registered_reports_via_wait_artifacts`:
    1. Register a lead report under a `research.m.final` producer dir with label
       `research:202609/topic/topic.md`.
    2. Query with `query_artifact_context` for that single producer group.
    3. Render the critique segment with `bind_runtime_template_vars` and
       `render_template`.
    4. Assert the entry
       `wait_name=research.m.final label=research:202609/topic/topic.md` plus its
       `source_path`, `path`, and `ref`.
    5. Assert no `{{` and no `{%` remain.

### 5. Docs and descriptions

- **`docs/xprompts.md`, `#research_swarm` section:**
  - Add `critique` and `critique_model` rows to the input table.
  - Change "up to seven authored segments" to eight, and add the critique agent to the
    parenthetical.
  - Note that the critique opt-in adds another `0.25` capacity unit.
  - Mention `critique_model` alongside the other per-role model overrides.
  - Add list item 8 for `<clan>.critique`:
    - opt-in via `critique=true`, default model `@xlarge`
    - waits on the lead and does not fork it
    - finds the lead's report through `wait.artifacts`
    - may cross-check the moved researcher drafts
    - writes and registers `<name>/<name>__critique.md` and never modifies the lead's
      report or the drafts
  - Add a short paragraph on the handoff contract: the lead registers its consolidated
    report only when `critique=true`, and the critic derives its directory from that
    label.
  - Note the pairing guidance: pick a `critique_model` on a different provider from
    `lead_model` for a more independent review.
- **`README.md`:**
  - In the `#research_swarm` bullet, describe `critique=true` and `critique_model`
    (default `@xlarge`), with an example like
    `#research_swarm(prompt="A research topic", critique=true)`.
  - In the `research-highlights` paragraph, note that `__critique.md` companions are
    excluded by the same draft glob, so the critique gets no Highlights PDF.
- **`docs/configuration.md`.** Add `critique_model` to the per-invocation overrides
  sentence, and note that the optional critique agent also defaults to `@xlarge`.
- **`AGENTS.md` architecture bullet.** Note that the optional critique agent, like the
  lead, defaults to SASE's built-in `@xlarge` alias.
- **`default_config.yml`.** Add "the optional critique agent" to the `researchers`
  bucket description, keeping the word "infographic", which a test asserts.

## Verification

1. **Capture the baseline.** Before editing, in the opened plugin repo, save the
   rendered default swarm with a short Python snippet that uses the same calls as the
   tests: `load_xprompts_from_plugins()["research_swarm"]`, then
   `expand_single_xprompt(xp, ["some topic"], {}, preserve_segment_separators=True)`.
   Also capture `{"should_generate_image": "true"}` and an all-researchers-off render.
   After the change, re-render all three and diff. They must be identical.
2. **Run `just check`** in the plugin repo (ruff + mypy + pytest) and fix everything it
   reports. `just test-wheel` is not required, because no packaged file list changes.
3. **Eyeball the rendering.** Optionally inspect a `critique=true` render to confirm
   there is no stray Jinja, no stray `#`/`%` tokens in the prose, and correct draft-list
   pluralization for one and several researchers.

## Out of scope / follow-ups

- **No Highlights PDF for the critique.** The `research-highlights` file hook's
  `!20*/*/*__*.md` glob excludes `__critique.md`. This is left as is and documented.
  Opting critiques into the reading queue is a separate policy change.
- **Research lineage.** sase's `_research_lineage.py` still derives `derives-from` rows
  only from the retired `__a`/`__b` suffixes. It is already filed as bug bead
  `sase-15r`, which also notes that a `__critique.md` companion must not become a
  consolidation source. Do not touch the sase repo in this change.
- **Image agent.** The image agent keeps illustrating the lead's report. It does not
  wait for the critique.
