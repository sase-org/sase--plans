---
plan: 202607/clickable_sdd_frontmatter_links.md
---
 I would like to start using a markdown link for the `prompt` (used in
the frontmatter of plan files) and `plan` (used in the frontmatter of prompt
files) properties. Can you help me implement this?

- The goal is for me to be able to jump from one page to another when viewing
  these files on GitHub.
- For example, consider the shared_word_completion_min_length.md plan file
  hosted at
  https://github.com/sase-org/sase--plans/blob/main/202607/shared_word_completion_min_length.md
  (#sshot shows what this page looks like on GitHub). After this change, I would
  like the user to be able to click on
  `202607/prompts/shared_word_completion_min_length.md` from that page to
  navigate to the
  https://github.com/sase-org/sase--plans/blob/main/202607/prompts/shared_word_completion_min_length.md
  page. They should then be able to click on
  `../202607/shared_word_completion_min_length.md` from that page (note that we
  don't include the `../` currently, but you should add this) to navigate back
  to the plan file page on GitHub (i.e. the page we were on before clicking on
  `202607/prompts/shared_word_completion_min_length.md`).

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
