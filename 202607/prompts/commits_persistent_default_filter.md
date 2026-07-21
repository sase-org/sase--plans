---
plan: 202607/commits_persistent_default_filter.md
---
 Can you help me make it much clearer what commits we are filtering for
on the the "Commits" sub-tab of the "Artifacts" tab by always showing a filter
at the top?

- The filter should default to `sidecar:false since:24h`.
- You can remove the `Sidecars hidden` text we show at the top of this tab.
- Also, re-work the logic that makes sidecar repos hidden by default to just be
  a consequence of this default query.
- Also let's make the default query for this tab configurable via a new sase
  config field.
- Also, let's try to make the line that shows projects / project commit counts
  which is rendered at the top of this tab more consice by only showing projects
  that have >=1 commit matched by the current filter.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 