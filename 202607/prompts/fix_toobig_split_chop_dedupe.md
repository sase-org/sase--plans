---
plan: 202607/fix_toobig_split_chop_dedupe.md
---
 Why does the toobig_split sase chop not launch agents for these files
(see the output below)? We recently performed a major sase chop upgrade /
migration (see the sase-6v epic bead for details) and this hasn't worked quite
right since then. Here is how it should work:

- Once an hour, we check if there are any existing waiting or running sase
  agents in the `split_file` hood and abort if any are found; otherwise, we run
  `just toobig` in this repo.
- If any files match, we then launch one agent per file using the `split_file`
  hood.

Can you help me diagnose the root cause of this issue and fix it? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 