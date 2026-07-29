- **PLAN:** [../202607/placeholder_prompt_local_precedence.md](../placeholder_prompt_local_precedence.md)

 
%xprompts_enabled:false
# Previous Conversation

**User:**

Is placeholder completion for the placeholders in the current prompt working correctly? If not, use your /sase_plan skill to plan the appropriate changes.
I'm worred that our recent support for "common placeholders" has overridden the old functionality (which was not the intention).

**Assistant:**

I'll investigate how placeholder completion works and whether "common placeholders" support broke the existing behavior.

Short answer: **prompt-local placeholder completion is not overridden — it still works, and it still ranks first.** But one narrow piece of the old workflow did change, and I think that's what you're feeling.

## What I verified

**Core merge order** (`sase-core/crates/sase_core/src/editor/placeholder.rs:255`): document candidates are collected first, common tags appended after, and dedup keeps the prompt-local entry when both sides have the same text. The Python filter (`src/sase/ace/tui/widgets/placeholder_completion.py:67`) only ever drops rows whose source is `common` — it can't drop a prompt row.

**Property check against your real store** (`~/.sase/prompt_placeholders.json`, 98 entries): across 160 prompt/prefix/trigger combinations, the prompt-source candidate list, its order, and the replacement range were byte-identical with an empty store vs. your warm store, and prompt rows were always the leading rows. Zero violations. The 46 existing placeholder tests pass.

Concretely, with your store loaded: typing `<ctrl+e` after `Rebind <ctrl+enter>` gives `<> ctrl+enter` (prompt) first, then `◆ ctrl+e`, `◆ ctrl+e/y` — and the saved duplicate of `ctrl+enter` is deduped away.

## What actually changed

1. **`Ctrl+T` lost its one-key insert.** `_try_file_completion_tab` still auto-accepts when there's exactly one candidate (`src/sase/ace/tui/widgets/_file_completion_tab.py:74`), but the saved list now almost always contributes extra matches, so the count is rarely 1. Before: `Ctrl+T` on `<ctrl+e` inserted `<ctrl+enter>` outright. Now: a 3-row menu you have to confirm with Enter. I reproduced both. This one was called out as intended in the epic plan (D5, `sase/repos/plans/202607/common_placeholder_tags.md:172`) — but that reasoning assumed the shortcut would fire for saved tags, not that it would stop firing for your own prompt's tags.

2. **Placeholders inside inline code or fences are no longer offered at all.** That's from `aedbb6b07` (sase-9q.2), not the common-placeholder work: ``Fix `<the widget>` and update <t`` now yields no prompt candidate. If you write keybinding-style prompts with backticks, this is the change most likely to read as "my prompt's placeholders disappeared."

3. Typing `<x` in prose with no prompt-local tags now pops a saved-tag menu (a bare `<` still stays silent). Intended per D5.

## Recommendation

Item 1 is worth fixing and is a small change: count only `source == "prompt"` candidates when deciding the `Ctrl+T` auto-accept, so a lone prompt-local match still inserts outright while the saved group stays visible in the menu. Item 2 is a design call — offering literal-zone tags as *completion* candidates is defensible even though excluding them from highlighting and substitution is right.

Want me to run `/sase_plan` for item 1 (and item 2 if you want that reverted for the completion path only)?

---

%xprompts_enabled:true
# New Query

 Can you help me fix 1 and 2? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 