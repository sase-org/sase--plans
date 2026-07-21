---
plan: 202607/shared_word_completion_min_length.md
---
 We currently complete current words in the prompt input widget via the `<ctrl+t>` keymap, but it seems like there is no cap on the number of characters in those words. Can you help me start only completing words that are five characters or longer? We have an existing sase config field which controls this for the common words completion. Let's start using this same config field (that already defaults to 5), which you should probably rename and update docs for, for the words in the current buffer as well. Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
