- **PLAN:** [../202608/fix_pyscripts_closer_package.md](../fix_pyscripts_closer_package.md)

 ---
IMPORTANT: You should make the necessary file changes, but should NOT create a commit, branch, or PR yourself.
Exception: If a post-completion finalizer instructs you to commit, you MUST follow those instructions and commit.


Can you complete the work for task bead sase-de? The bead is already reserved for you and assigned to your
agent name: it was set to status=in_progress by the launch that started you; do not set the status by hand. Run
`sase bead show sase-de`, read the description and notes, do the work, and close the bead with
`sase bead close sase-de --note "<what you verified>"`. If you discover genuinely distinct follow-up work,
do not expand this bead's scope: file a new task bead (`sase bead create -T task ...`), refine it while it is
`open`, and mark it ready to triage with `sase bead update <id> -s ready`.