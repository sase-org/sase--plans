---
create_time: 2026-07-13 11:39:20
status: wip
prompt: 202607/prompts/land_epic_xprompt_integration.md
tier: tale
---
# Improve the `#bd/land_epic` xprompt: integration duty + tighter prompt

## Context

`#bd/land_epic` is the built-in xprompt (defined in `src/sase/default_config.yml` under `xprompts:`) rendered for the
final "land agent" that `sase bead work <epic_id>` launches after every phase agent finishes. Launch mechanics that
shape the prompt design:

- The land agent is named `<epic_id>`, waits (`%w`) on all phase agents, runs with `%auto:tale`, and receives
  `SASE_BEAD_ID=<epic_id>` (`src/sase/bead/work.py::render_multi_prompt`).
- Because of `%auto:tale`, any plan the land agent submits via `/sase_plan` is auto-approved as an SDD tale and a
  **follow-up coder agent** executes it — the land agent itself terminates after proposing. Whatever landing steps are
  not written into that plan never happen.
- `sase bead show <epic_id>` prints the epic's CHILDREN (with status icons) and the linked PLAN file path, so the prompt
  does not need a filesystem hunt for the plan file (which today assumes an in-tree `sdd/plans/` layout and is wrong for
  sidecar SDD repos).
- Symvision's `--epic-symbol <bead_id>(<symbol>)` Justfile whitelist entries expire once the epic bead closes — the
  concrete reason `just symvision` must run AFTER `sase bead close`.

The prompt content lives only in `src/sase/default_config.yml` (the `bd/land_epic` entry, currently lines 453–473).
Tests (`tests/test_bead_xprompt_tags.py`, `tests/test_bead/test_work_rendering.py`,
`tests/test_bead/test_cli_work_epic_launch.py`) assert only the xprompt name and `land_epic` tag, never the content, so
this is a contained content change. Docs reference the xprompt by one-line description only (`docs/xprompt.md` tag table
and built-ins table).

## Problems with the current prompt

1. **Missing integration duty (the main request).** Work committed by other agents while the epic was in flight could
   not integrate with the epic's feature because the feature was incomplete. Nobody is currently assigned that
   reconciliation; the land agent is the natural owner since it runs last.
2. **The `%auto:tale` gap.** The prompt says a remaining-work plan must include "closing the bead as the final step",
   but the symvision run and the plan-file `status: done` update sit outside that instruction — so in the
   plan-remaining-work path, the follow-up coder never performs them.
3. **Fragile plan-file discovery.** "find the plan file ... in the sdd/plans/ directory ... in a YYYYMM subdirectory"
   assumes an in-tree layout and ignores that `sase bead show` already prints the linked plan path directly.
4. **Diffuse wording.** The verify/close/plan branches are interleaved prose; a numbered checklist is cheaper in tokens
   and harder to partially skip.

## Design decisions

- **Keep**: the `tags: land_epic` tag, the `input: bead_id: word` signature, the distrust-and-verify stance ("read the
  actual source, don't trust completion claims"), the child-bead notes check, the close command, the AFTER-close
  symvision ordering with its rationale, the plan-file `status: done` step, and the `/sase_plan` branch. All of these
  are load-bearing.
- **Add** an explicit "Integrate" step with the _why_ (concurrent work couldn't adopt an incomplete feature) and one
  concrete recipe (`git log` since the first commit mentioning the bead ID, excluding the epic's own commits; in a PR
  workflow also review the base branch). Child bead IDs are prefixed with the epic ID, so one grep covers the family.
  Keep it goal-oriented rather than prescribing exact git flags — the agent runs in both same-branch (`#commit`) and
  PR-branch (ChangeSpec) contexts.
- **Restructure** as a three-step checklist (Verify → Integrate → Land) plus one branching rule at the end, and state
  that a remaining-work plan must carry the entire Land step (close + symvision + plan-file status) as its final phase,
  closing the `%auto:tale` gap.
- **Sharpen** plan-file discovery to "the PLAN path shown by `sase bead show`", and make the symvision note actionable
  (stale epic-symbol whitelist entries for this bead must be removed).
- **Do not** grow scope: no changes to `bd/work_phase_bead`, `bd/new_epic`, tag resolution, or launch code.

## Phase 1: Rewrite the `bd/land_epic` content in `src/sase/default_config.yml`

Replace only the `content:` block of the `bd/land_epic` entry (keep `tags`/`input` unchanged, keep-sorted block
untouched) with:

```
You are the land agent for epic bead {{ bead_id }}: verify the epic is truly complete, integrate it with changes
that landed since it started, then close it out.

1. Verify. Run `sase bead show {{ bead_id }}` (children, linked plan file), then `sase bead show` on each child
   bead. Confirm every bead note was addressed, and read the actual source code and the epic's commits (bead IDs
   appear in commit messages) to confirm the work previous agents reported complete really is.

2. Integrate. Changes committed since this epic started could not integrate with this epic's feature while it was
   incomplete. Find them (e.g. `git log` since the first commit mentioning {{ bead_id }}, excluding the epic's own
   commits; in a PR workflow also review commits on the base branch) and update anything that should now use what
   this epic added or that duplicates or conflicts with it. This integration is part of the epic's work.

3. Land. Close the epic with `sase bead close {{ bead_id }}`. AFTER closing, run `just symvision` if available
   (epic-symbol whitelist entries for {{ bead_id }} expire at close) and remove the stale entries and unused code
   it reports. Finally, set `status: done` in the frontmatter of the epic's plan file (the PLAN path shown by
   `sase bead show`).

If steps 1-2 uncover remaining work, use your /sase_plan skill to plan it, with step 3 as the plan's final phase
(close, run symvision, mark the plan file done) so the agent that executes the plan finishes the landing.
Otherwise do step 3 now.
```

Also update the two one-line descriptions of `bd/land_epic` in `docs/xprompt.md` (the `land_epic` tag table row and the
`#bd/land_epic` built-ins table row) to reflect the widened duty, e.g. "Final land agent prompt used by
`sase bead work`: verifies, integrates, and closes the epic".

## Verification

- `just install` (fresh ephemeral workspace), then `just check` (lint + mypy + tests; the touched YAML must still parse
  and the tag-resolution tests must still pass).
- Sanity-render the xprompt to confirm Jinja substitution and no directive/marker misparse, e.g.
  `sase run --dry-run '#bd/land_epic:sase-99'` or the equivalent xprompt explain/preview command available in the CLI.
