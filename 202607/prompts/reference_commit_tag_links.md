---
plan: 202607/reference_commit_tag_links.md
---
 Can you help me add support for using reference-style markdown links as
values for sase commit tags (the references will go after the sase commit tags,
after a blank line) and, as our first use-case, start using a link to the plan
file in the corresponding sase project's `plans` sidecar repo on GitHub? For
example, assume a commit for this project (sase) with the following sase commit tags:

```
SASE_MACHINE=athena
SASE_PLAN=202607/amd_agents_template.md
```

After this change, a commit like this would instead write something like the following for its sase commit tags:

```
SASE_MACHINE=athena
SASE_PLAN=[202607/amd_agents_template.md][1]

[1]: https://github.com/sase-org/sase--plans/blob/main/202607/amd_agents_template.md
```

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 