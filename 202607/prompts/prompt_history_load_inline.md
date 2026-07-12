---
plan: 202607/prompt_history_load_inline.md
---
 When I trigger prompt history from one of multiple prompt input widgets, find the prompt I want, and then press `<ctrl+i>` to load the prompt in the prompt input widget, it seems that the prompts in the other prompt input widgets are completely lost (their prompt input widgets disappear and I am only left with the new prompt that I just loaded from history). This is not correct.

- We should just load the prompt inline into the current prompt input widget, if the selected prompt is not a multi-agent prompt.
- If the selected prompt is a multi-agent prompt, then we should load the first prompt into the current prompt input widget and then, for each subsequent prompt, create a new prompt input widget (added below the current or last added prompt input widget) and then inline that prompt in that prompt input widget.
- The only time we might need to overwrite data from the TUI's current prompt is if the xprompt property panel already exists and has properties set. The selected prompt that we want to load from history also has xprompt properties. In this case, we should prompt the user (y/n) whether they want to proceed with loading this prompt or not (let them know their current xprompt properties will be lost).

Can you help me fix this? Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  