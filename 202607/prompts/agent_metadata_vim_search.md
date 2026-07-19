---
plan: 202607/agent_metadata_vim_search.md
---
 Can you help me add excellent, vim-inspired text search to the agent
metadata panel on the "Agents" tab of the `sase ace` TUI?

- This should be triggered using the `/` and `?` keymaps. The existing keymaps
  for these should be re-mapped to `,/` and `,?` (make sure to re-map `,?` for
  all tabs).
- While "search-mode" is enabled (after using one of these keymaps, entering in
  a search query, and then pressing `<enter>`), the `n`/`N` keymap should work
  as you'd expect.
- The user should be able to press `<esc>` or `q` at any time to exit
  search-mode.
- Make it easy to select and copy free-form text once a search query match has
  been found and jumped to.
- Make sure this search functionality is available on all versions of the agent
  metadata panel (e.g. clans/families/tribes summaries should be supported too).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 