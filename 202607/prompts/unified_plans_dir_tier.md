---
plan: 202607/unified_plans_dir_tier.md
---
 We currently store epic plan files in the sdd repo's epics/ directory and plan files in its tales/ directory. Can you help me start only using a new plans/ directory instead and storing all tales and epics in that directory?

- We should be able to use the `tier` frontmatter field to determine which plan files are tales and which are epics.
- Make sure the `sase plan list` command has great support for displaying and filtering by tier.
- The `sase sdd init` command should handle this migration for us and set the `tier: tale` field for any tale plan files that don't have a `tier` property (among other fixes--you determine what's necessary).
- We will want to start making sure that this tier property is always added to plan files in the future.
- There are likely a lot of edge cases to cover here so make sure you consider them all.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  