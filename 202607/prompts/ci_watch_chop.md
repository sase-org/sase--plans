- **PLAN:** [../202607/ci_watch_chop.md](../ci_watch_chop.md)

 
%xprompts_enabled:false
# Previous Conversation

**User:**

You are the lead researcher: two independent researchers have reported on the request below, and you will add your own
research and merge all three perspectives into one consolidated report.

Research request:

I want to write a lumberjack chop that uses the act stat command line utility to check the most recent GitHub CI actions workflow run for all SASE repos. It either launches an agent to fix the CI failures or merges all release-please and release-plz release PRs, which should automatically trigger a release of the appropriate packages. Can you do some research to help me think about the best way to implement this? End your analysis with a recommended solution.

The researchers' chat transcripts:

Month directory (create it if missing):

$(sase repo path research --ensure)/$(date +%Y%m)

Steps:

1. Read both transcripts to learn which report file each researcher wrote (`research.@.cdx` -> `__a`, `research.@.cld`
-> `__b`), then read both reports. Never assign `__a`/`__b` from filesystem order.
2. Research the request yourself, prioritizing gaps, weak evidence, and disagreements between the two reports.
3. Pick a descriptive stem `<name>` that collides with nothing in the month directory (do NOT end the name with
`_consolidated` or `_<YYYYmmdd>` or anything similar unless it relates to the research topic), create
`<month-dir>/<name>/`, and move the two reports to `<name>__a.md` and `<name>__b.md` inside it. Preserve both files
and never overwrite: on any collision, pick a different stem first.
4. Write the consolidated report to `<name>/<name>.md`: merge the strongest findings from both reports and your own
research, resolve conflicts, cut duplication, and add missing critical context without unnecessary length.

Final layout:

```text
<month-dir>/<name>/
├── <name>__a.md
├── <name>__b.md
└── <name>.md
```

**Assistant:**

I'll start by reading the two researcher transcripts to find their report files, and check the month directory.

Now writing the consolidated report.

Consolidation complete. The final report is at
`sase/repos/research/202607/actstat_ci_release_chop/actstat_ci_release_chop.md`, with the codex researcher's report preserved as `__a.md` and the claude researcher's as `__b.md` beside it. (Only research-sidecar markdown changed, so `just check` doesn't apply.)

**The consolidated recommendation:** one script chop, `bugyi_chop_ci_watch`, in `bugyi-chops`, configured as a 5-minute lumberjack lane in chezmoi's `sase_athena.yml`. It runs one `actstat -f jsonl` sweep per tick against a chop-owned actstat config, then decides per repo: an actionably-red default branch gets one deduplicated `#pr(...)` fix-agent proposal; a green branch with a fully green release PR gets a deterministic `gh pr merge --squash --match-head-commit`. Everything unknown fails closed. Ship with `merge_enabled: false`, soak, then enable merges with a 1-per-tick cap in dependency order (core → plugins → sase).

The reports agreed on most of the architecture; I verified every load-bearing claim myself and resolved the four real disagreements:

- **Global vs per-repo gating** (the biggest split): researcher A wanted any red repo to block all merges; researcher B wanted per-repo either/or. Per-repo wins on hard evidence — `sase`'s `published-core-minimum-smoke` job depends on the *published* `sase-core` package, so A's global rule can deadlock (red `sase` blocks the very core release that would turn it green), while B's rule self-heals. I kept A's fail-closed rule only for process-level failures.
- **Dry-run safety**: I confirmed A's finding that `ChopScriptContext` has no `dry_run`/`source` field, so `--dry-run` today would not stop a script's direct `gh pr merge`. Resolution: use B's `merge_enabled` var immediately *and* land A's SASE contract change before ever enabling merges.
- **actstat config**: B's dedicated-config approach wins, and I settled the one factual discrepancy — `ACTSTAT_CONFIG` isn't in `--help`, but I confirmed via the binary's strings that it is supported.
- **The central safety finding** (B's): no SASE repo has branch protection — I re-verified all four return 404 today — so the chop is the only gate between a tick and a PyPI publish. That justifies the redundant merge guards and the recommendation to add branch protection regardless.

The report ends with four open questions for you: whether `sase-google`/`sase-gchat` are in scope, whether you want a `sase gate` confirmation on the first merges, what model/effort fix agents should get, and whether red release PRs themselves ever warrant action.

The finalizer flagged changes I made this session (the consolidated report plus the two moved researcher reports), so I'll commit them in the research repo using the commit skill.

The primary commit succeeded but post-commit publication queueing failed; the tool says to resume.

Committed and pushed. Summary of what I did:

- **Commit**: `40b3c5c docs(research): consolidate actstat CI-watch chop research` in the research repo (both finalizer entries — linked repo `research` and sidecar `sase-org/sase--research` — point to the same checkout, so one commit covers both). It stages the moves of the two original researcher reports out of `202607/` and the new `202607/actstat_ci_release_chop/` directory with `__a.md`, `__b.md`, and the consolidated `actstat_ci_release_chop.md`.
- **Verified clean**: `git status --short --branch` shows `main` in sync with `origin/main` with an empty working tree, and `git diff --stat origin/main HEAD` is empty, so the commit is pushed.

One residual issue worth flagging: the primary commit and push succeeded, but sase's post-commit "agent publication" step failed repeatedly with `project 'sase--research' was not found`, and `sase commit --resume` / `sase_git_commit --resume` both hit the same error. There is a `~/.sase/projects/sase--research/` directory, but it contains only `skill_uses.jsonl` — no ProjectSpec — and `sase project list -s all` doesn't list it. It looks like the publication queue resolves the sidecar repo name as a project key, but sidecars are repos, not projects (per the glossary), so the lookup can never succeed. This smells like a sase bug; happy to file a bead for it if you want.

The research deliverable itself is unchanged from before: the consolidated report at `sase/repos/research/202607/actstat_ci_release_chop/actstat_ci_release_chop.md` recommends a single `bugyi_chop_ci_watch` script chop in `bugyi-chops` on a 5-minute lumberjack lane — per-repo either/or gating (fix-agent proposal on red default branch, deterministic `gh pr merge --squash --match-head-commit` on green release PRs), fail-closed on anything unknown, shipped with `merge_enabled: false` until soaked.

---

%xprompts_enabled:true
# New Query

 Can you now use this research as inspiration to help me implement this new chop? One change that we should make to the recommended solution is that agents that this chop launches should only run once there are zero other agents running. To make sure that there aren't, for example, epics that are currently being worked on, in which case the epic lander agent might fix the CI failures. Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 