---
plan: 202607/model_alias_buckets.md
---
 Can you help me add support for model "alias buckets" in the "Models" panel of the TUI (triggered via the `,m` keymap)?

- We should be able to use these buckets to configure a set of aliases that are all grouped together on that panel and shown in the same row using the bucket name (show other useful information on this line, but keep it concise so it all fits on one line).
- If the user selects that row they can then use the `l` keymap to navigate into that bucket and see all of the model aliases that are contained within it. The `h` keymap should take them back to the main model alias list.
- As a first use-case, move the `research` model aliases (defined in a sase.yml file in my chezmoi repo) into a single `research` model alias bucket, rename the existing `research` alias to `research_a`, rename the `research_assist` alias to `research_b`, and add a new model alias `research_c` that maps to the `codex/gpt-5.6-sol` model.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  