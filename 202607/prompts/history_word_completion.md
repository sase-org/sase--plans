---
plan: 202607/history_word_completion.md
---
 We recently added support to the prompt input widget keymap for
completing words that already exist in the prompt input widget when there are no
other available completions. Can you help me start triggering a different
completion menu when this fails, which uses a set of commonly used words?

- These commonly used words should not be static but instead should be derived
  from historical user prompts by saving the last `<N>` >=`<M>` character unique
  words that were used in previous prompts.
- Let's default to 1000 for `<N>` and 5 for `<M>` but make these configurable
  via new sase config field that users can override.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 