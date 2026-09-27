---
tier: tale
title: Epic land agents prefer tale plans for remaining work
goal:
  Epic land agents that find unfinished work prefer a self-closing tale plan over a
  child epic whenever one coding agent can finish the remaining work.
size: small
proposed_by: bbugyi200.athena.0ta
create_time: 2026-09-27 17:44:09
status: wip
---

# Plan: Epic Land Agents Prefer Tale Plans For Remaining Work

## Problem

The epic land agent prompt (`bd/land_epic` in `src/sase/default_config.yml`) tells a
lander that finds remaining work to "use your /sase_plan skill to plan it", with no tier
preference. The rest of the paragraph is written only for a child epic ("Do not include
this epic's close, symvision pass, or plan-file status update as a child phase; the
child epic's `parent_bead` link is the handoff that lets its land agent resume this
interrupted landing"). In practice, landers often author child epics for work that a
single coding agent could finish. That spends an extra round of phase workers plus a
second land agent.

We want landers to prefer a `tale` plan whenever the remaining work fits one coding
agent, and to fall back to a child epic only when it genuinely needs phases or multiple
agents.

### Why the tale path needs explicit closeout instructions

A tale does not get the child-epic resume path. Current behavior (verified in the code):

- The land segment runs with bare `%auto` (`src/sase/bead/work_prompt.py`), so a
  proposed tale auto-approves. The coder continues in the same process via
  `handle_accepted_plan` / `continue_as_successor`
  (`src/sase/axe/run_agent_exec_plan_accept.py`).
- The coder's prompt is only `@<plan>` + "The above plan has been reviewed and approved.
  Implement it now." It says nothing about beads, epics, or landing.
- `sase plan propose` stamps `bead: <epic_id>` on a tale proposed by the lander
  (`src/sase/main/plan_propose_handler.py`). The coder inherits `SASE_BEAD_ID=<epic_id>`
  as its assigned bead.
- `SASE_PLAN` points at the tale plan. The coder's commit hook therefore marks only the
  tale plan `status: done`, never the epic's plan file.
- No bead is created and no land agent is spawned for a tale. Nothing reruns the
  lander's step 3 (follow-up triage, `--epic-symbol` cleanup, epic close,
  `just symvision`, epic plan-file `status: done`). Nothing reruns its `parent_bead`
  ancestor walk either. Unless the tale plan itself contains those steps, the epic stays
  `in_progress`.

So preferring tales is only safe if the prompt also says that a lander-authored tale
must carry the rest of the landing itself.

## Changes

### 1. Rewrite the remaining-work paragraph of `bd/land_epic` (`src/sase/default_config.yml`)

Replace the paragraph that begins "If steps 1-2 uncover remaining work, ..." with
instructions that do the following. Keep the prompt's existing voice and line wrapping
(~120 columns, `{{ bead_id }}` templating):

1. Keep: use `/sase_plan`, complete its tier-aware validate/revalidate/propose loop, and
   "Plan only the remaining work."
2. **Tier preference (new).** Prefer `tier: tale`. Choose a tale whenever one coding
   agent can finish the remaining work directly, meaning a tale size of `xsmall`,
   `small`, or `medium` per the SASE size guidance. Author a child `epic` only when the
   remaining work genuinely needs multiple agents or phases, or is too large
   (`large`/`xlarge`) for one agent to implement directly.
3. **Tale closeout (new).** Say plainly that a tale has no land agent of its own and
   nothing resumes this landing after its coder finishes, so a lander-authored tale must
   finish the landing itself:
   - Before proposing the tale, the lander finishes the step-3 follow-up triage itself
     (`/sase_new_task` for each distinct follow-up not caused by the epic). It records
     every outcome, including declined proposals, with
     `sase bead note {{ bead_id }} "..."`. This work doesn't depend on the remaining
     changes, and the lander already holds the collected `PROPOSED FOLLOW-UP:` entries.
   - The tale's final step must be this epic's closeout, written out concretely so the
     coder needs no other context:
     - resolve or re-key every `sase bead epic-symbols {{ bead_id }}` entry;
     - close the epic with `sase bead close {{ bead_id }} --note "<verification>"`;
     - run `just symvision`;
     - set `status: done` in the epic's plan file, naming the PLAN path shown by
       `sase bead read`;
     - when the epic has a `parent_bead`, handle that parent as described in the
       prompt's final paragraph, using the concrete parent ID.
   - The step-3 rules still apply in the tale: never `--force` merely to make a close
     succeed, and never use it to advance a successful nested landing.
4. **Child epic (kept, reworded as the fallback).** Keep the existing rule for the epic
   path. Do not include this epic's close, symvision pass, or plan-file status update as
   a child phase. The child epic's `parent_bead` link is the handoff that lets its land
   agent resume this interrupted landing after the child lands.

Do not change steps 1–3 or the final `parent_bead` paragraph except for the minimal
cross-reference in item 3.

Keep these exact substrings, which `tests/test_bead_xprompt_tags.py` asserts:

- "Plan only the remaining work"
- "Do not include this epic's close, symvision pass"
- "as a child phase"
- "child epic's `parent_bead` link is the handoff"
- "use `/sase_new_task`"
- "sase bead epic-symbols {{ bead_id }}"
- "never use `--force` to advance a successful nested landing"

Also keep every assertion in `test_builtin_land_prompt_resumes_nested_parent_handoffs`.

The prompt must not gain any `%` wait or queue directives.
`test_bead_worker_builtin_xprompts_do_not_author_wait_directives` and
`test_builtin_land_prompt_does_not_author_queue_weight` guard this.

### 2. Tests (`tests/test_bead_xprompt_tags.py`)

Extend `test_builtin_land_prompt_plans_remaining_work_only`, or add a sibling
`test_builtin_land_prompt_prefers_tale_plans` using `_builtin_prompt_body` /
`_single_spaced`. Assert that the land body:

- states the tale preference (e.g. contains "Prefer" together with `tier: tale` wording,
  and the `xsmall`/`small`/`medium` sizing condition);
- says a tale has no land agent of its own and nothing resumes the landing;
- requires the tale's final step to carry the epic closeout:
  `sase bead close {{ bead_id }}`, `just symvision`, and the plan-file `status: done`;
- requires the lander to finish follow-up triage and record outcomes with
  `sase bead note {{ bead_id }}` before proposing a tale.

Pick the asserted substrings from the final prompt text so the tests pin the behavior,
not incidental wording.

### 3. Docs (`docs/beads.md`)

In the `sase bead work` epic-launch walkthrough, step 6 currently ends with "An agent
may author a tale or an epic as needed; the plan's authored `tier` selects the
corresponding automatic follow-up path." Add one or two sentences there saying:

- the land agent prefers a tale for remaining work that one agent can finish;
- because nothing resumes the landing after a tale's coder finishes, the land agent
  triages follow-ups first and writes the epic's closeout into the tale as its final
  step;
- a child epic instead hands the resumed landing to its own land agent through
  `parent_bead`.

Do not otherwise restructure the docs.

## Non-Goals

- No runtime or code changes to plan approval, tale coder prompt construction, bead
  stamping, or finalizers. This is an instruction-only change. A runtime path that
  automatically resumes the landing after a lander-authored tale is out of scope.
- No change to `bd/work_phase_bead`, `bd/work_task`, the `sase_plan` skill template, or
  SASE memory notes.
- No change to the lander's `%auto` directive in `src/sase/bead/work_prompt.py`.

## Verification

- `just check` passes. Run it through the project's guarded-recipe path as the lint/test
  memory note requires.
- Targeted: `pytest tests/test_bead_xprompt_tags.py` passes, including the new/extended
  assertions.
- Render the prompt, e.g. `sase xprompt show bd/land_epic` or the equivalent built-in
  prompt listing, and read it once end-to-end. The tier preference, the tale-closeout
  requirement, and the unchanged child-epic handoff should read as one coherent
  paragraph sequence without contradicting step 3 or the final `parent_bead` paragraph.
