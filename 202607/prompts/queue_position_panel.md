- **PLAN:** [../202607/queue_position_panel.md](../queue_position_panel.md)

 %xprompts_enabled:false
# Previous Conversations

You are forking from 1 prior source. Source sections are independent parents, and section order carries no priority. Members inside an agent family section are sequential: each member continued the previous member's work. Carry forward relevant goals, constraints, decisions, and unfinished work with attribution when it matters. The New Query is the active request and takes precedence over conflicting source instructions.

## Source 1 of 1 — agent family `ku`

- **Members shown:** 2 of 2 (sequential chain, oldest first)

Family members ran as one sequential chain: each member continued the previous member's work, and the last member reflects the family's final state. These are transcripts of prior agents' conversations, not your own — attribute decisions to the named member when it matters.

### Member 1 of 2 — agent `ku--plan`

- **Outcome:** `completed` · **Model:** `claude/opus` · **Launch:** `20260725110900`
- **Transcript:** `~/.sase/chats/202607/gh_sase_org__sase-ace_run-ku__plan-260725_110900.md`

**User:**

For agents that are queued, I'd like to start using a new `QUEUED` agent status that replaces the current `WAITING` status that we use for queued agents. This better matches the agent status counts that we use. For example, in , we should replace `WAITING ▶10/10` with `QUEUED`. Make sure we give `QUEUED` a distinct agent status color. I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.

**Assistant:**

*Plan submitted for review.*

**Plan file:** `/home/bryan/.sase/plans/202607/queued_agent_status.md`

> # Plan: A dedicated QUEUED agent status for runner-slot waiters
> ## Context
> ACE already counts queued agents separately from waiting ones. The Agents header reads
> `71 [10/10 running · 1 queued · 19 waiting · 41 done]`, tribe and clan chips render `[Q1 W1]`, and
> `RunnerCapacitySnapshot.global_cap_queue_count` feeds the runner-limit indicator. All of that is derived from
> `agent_is_globally_queued`.
> The row itself contradicts those counts. A queued agent still renders `WAITING ▶10/10` in amethyst, identical in kind to
> an agent blocked on a dependency, a bead, or a time floor. The two are operationally opposite: a dependency wait may
> never resolve and can warrant intervention, while a queued agent is fully unblocked and will start on its own the moment
> capacity frees. Reading the list, there is no way to tell them apart without decoding a dim glyph suffix.

*See full plan file for details.*

### Member 2 of 2 — agent `ku--code`

- **Outcome:** `completed` · **Model:** `codex/gpt-5.6-sol` · **Launch:** `20260725114509`
- **Transcript:** `~/.sase/chats/202607/gh_sase_org__sase-ace_run-ku__code-260725_110900.md`

**User:**

@sase/repos/plans/202607/queued_agent_status.md

The above plan has been reviewed and approved. Implement it now.

**Assistant:**

I’ll implement the approved plan, first checking the plan’s exact scope and the current worktree so I preserve any existing changes. After the edits, I’ll install the workspace dependencies as required and run the full `just check` validation.

The plan isn’t in this checkout, so I’m using the `sase_repo` skill to resolve the plans repository through SASE before reading it. This keeps the implementation tied to the approved canonical plan.

The plan affects ACE refresh behavior, generated agent-status skill text, and xprompt documentation, so the repository instructions require audited reads of the `tui_perf`, `generated_skills`, and `xprompts` memories before I change those areas. I’m loading those constraints now, then I’ll map the existing status pipeline and tests.

The core design maps cleanly onto the existing refresh boundary: one shared WAITING↔QUEUED helper, applied during the existing post-merge runner-context pass and the integrations projection pass. Queue counts will then come from the `Queued` bucket, while admission ranking will include all live slot waiters in priority/FIFO order.

The first implementation pass is in place across the shared status policy, TUI/integrations projections, aggregation, actions, and presentation maps. I’m installing this workspace now, then I’ll run the focused queue/status suites to catch behavioral mismatches before updating the broader fixtures and docs.

The workspace install is compiling the linked Rust binding on this fresh checkout; it’s progressing normally. While that completes, the key invariant is now structural in code: promotion creates the `Queued` bucket, and both summary counts and the capacity snapshot count that bucket rather than independently re-running the queue predicate.

The focused suites now pass with the new admission semantics. I’ve also converted direct aggregation fixtures to assert the structural rule—only rows already displaying `QUEUED` contribute to Q—and added explicit promotion/demotion/idempotence coverage plus `status:queued`/`status:waiting` separation.

The implementation and tests now cover the visible row contract, all-waiter ranking, and cross-surface rank parity. I’m updating the user-facing contract next: ACE status/bucket/query docs, runner-slot troubleshooting, CLI/integrations field semantics, xprompt/config wording, and the generated `sase_agents_status` skill source.

The core QUEUED behavior and consumer updates are in place, and the focused status/ranking suites are passing. I’m finishing the aggregation audit, regenerating the affected visual baselines, then I’ll run the generated-skill refresh and the full required checks.

One aggregation edge case surfaced in the audit: clan containers are projected before slot status is derived, so an all-cap-blocked clan could retain a stale WAITING aggregate for one refresh. I’m correcting that in the same in-memory pass and adding a regression for first-refresh behavior.

The first-refresh clan regression now passes, along with the queue, integration, summary, and tribe suites. Next I’m updating the intentional PNG baselines and will visually inspect the changed queue views before running the full snapshot suite.

The targeted queue visuals look correct, including the new QUEUED bucket and the rank/request-age detail line. The full visual suite is queued behind another repository test run’s shared worker pool; while it waits, I’m reviewing the final diff and status-consumer coverage.

The generated-skills workflow now requires its deployment step because the source `sase_agents_status` skill changed. I’m regenerating managed skills with `sase skill init --force`, then applying the chezmoi state as prescribed by that skill’s instructions.

The first regeneration used the globally installed CLI and did not pick up this editable workspace’s skill source. I verified the deployed `sase_agents_status` content, caught the mismatch, and am rerunning the same workflow through this workspace’s installed CLI so the intended QUEUED guidance is actually generated.

The full snapshot suite found exactly one additional intentional visual change: the existing queued-clan count view reflects the new cornflower Q-chip color. All other 347 visual cases passed. I’m inspecting and accepting that one baseline, then I’ll rerun the visual suite.

The implementation and generated skill are now synchronized, formatting is clean, and the focused regressions pass. I’m starting the repository-mandated `just check`; it covers formatting, lint/type checks, validation, and the full fast test lane. After that I’ll rerun the dedicated visual lane against the accepted baselines.

`just check` passed formatting, ruff, mypy, and script lint, then Symvision flagged the now-internal queue predicate as an unused public symbol. The Symvision memory applies here, so I’m reading its required guidance before making that narrowly scoped visibility fix.

The Symvision guidance confirmed the correct fix: the predicate had become file-local after the counting refactor, so I made it private and shifted its test to observable derivation behavior. The exact Symvision lane and focused queue/integration tests now pass; I’m rerunning the complete check.

The complete check has cleared every formatting, lint, type, size, and validation stage. Its full parallel test phase is running now; the wrapper stays quiet unless it finishes or finds a failure.

The full test lane is still active across 12 workers (not stalled); several workers are in CPU/I/O-heavy cases, and no failure output has appeared. I’m letting it finish rather than interrupting the repository-wide verification.

The full suite completed 21,982 passes but reported four failures. Three are unrelated timing/cache/daemon tests; one scan-wire test touches waiting metadata and may expose a real expectation mismatch. I’m rerunning all four in isolation with full failure output to separate regressions from parallel-suite flakiness before changing anything.

Isolation confirms two failures persist and are outside this change: the cache tests pass alone, while the daemon test counts workspace-resolution subprocesses and the scan-wire test is coming from the linked Rust binding. I’m checking the sanctioned workspace-matched `sase-core` checkout before deciding whether a rebuild resolves the wire mismatch; I will not modify Rust scan behavior for this QUEUED feature.

The sanctioned `sase-core` checkout was stale; current upstream already contains the missing `wait_priority_explicit` scanner support and its parity test. I’ve updated only that linked checkout through `sase repo open`; now I’m rebuilding the workspace binding with `just install`, which should resolve the scan-wire failure without any Rust source edits.

The Rust binding rebuild is progressing normally against `sase-core` v0.9.1; the core crate is compiled and the Python extension crate is now building. No source files in either repository were edited during this repair.

The Rust rebuild fixed the scan-wire failure exactly as expected. The remaining persistent failure is an unrelated existing test bug: in numbered workspaces it patches the global `subprocess.Popen`, so two canonical-workspace resolution subprocesses are counted as daemon spawns. I’m leaving that out-of-scope test/source untouched and running the required dedicated visual suite now; the two transient cache failures already pass in isolation.

The dedicated visual run is clean through 41% so far, including the new mixed queued/waiting frame and the updated queued-clan Q-chip baseline. It has only four shared workers available, so this pass is slower than the earlier 17-worker run.

All 348 visual cases have executed without a failure marker. The pytest coordinator is in final filesystem teardown under heavy shared I/O, so I’m waiting for its exit summary before declaring the visual lane complete.

The dedicated visual lane is fully green: 348 passed, 1 skipped. I’m doing the final worktree/diff audit now—checking for whitespace issues, accidental unrelated edits, and confirming the linked core checkout is clean—then I’ll hand back the implementation with the exact validation caveat from the unrelated daemon test.

Implemented the approved QUEUED status plan.

- Added reversible, in-memory `WAITING ↔ QUEUED` derivation shared by TUI and integrations.
- Added the Queued bucket, `…` glyph, cornflower `#5F87FF` styling, aggregation/count parity, and priority/FIFO ranks across all live waiters.
- Updated rows, detail metadata, actions, filters, CLI JSON, documentation, generated skill, tests, and three PNG goldens.
- `just test-visual`: **348 passed, 1 skipped**.
- Formatting, ruff, mypy, Symvision, size checks, and SASE validation passed.

`just check`’s full run reached **21,982 passed, 7 skipped**. After refreshing the stale linked Rust binding, the scan-wire failure passed; two cache failures also passed in isolation. One unrelated numbered-workspace daemon test remains: it expects one `Popen` call but its broad mock also counts two canonical-workspace resolution subprocesses. I left that out-of-scope source untouched.

Main-repository changes remain uncommitted. The required skill regeneration was deployed through chezmoi’s automatic commit/push workflow.

These are the implementation changes from this session, so the finalizer requires a commit. I’m using the `sase_git_commit` skill now and will commit exactly the listed files with the mandated create-commit workflow.

The audit matches the approved change set: 66 files, including one new visual golden, with no whitespace errors or unrelated paths. I’m committing it as a user-facing feature that surfaces runner-cap queue state and admission order.

The commit workflow is now staging only the finalizer’s file list, committing locally, rebasing onto the six newer upstream commits, and pushing through SASE’s guarded flow.

Upstream added editable and explicit wait-priority behavior in two files touched by this feature, so the guarded rebase paused with two real conflicts. I’m preserving both upstream priority controls and the new QUEUED display semantics, then I’ll resume the existing commit workflow rather than creating a second commit.

The combined behavior exposed one upstream test expectation that still says “eligible”; the approved semantics rank all live waiters, so the UI correctly says “queue.” I’m updating that newly introduced assertion to the merged contract and checking for any other stale eligibility wording before continuing.

The conflicts are resolved with both behaviors intact, and the focused wait/status suite passes (95 tests). The original commit has replayed successfully on current upstream; I’m resuming SASE’s deferred push and ChangeSpec bookkeeping now.

Committed and pushed as `899a257f2` (`feat: surface runner-cap queued agent status`).

Resolved upstream wait-priority conflicts while preserving QUEUED semantics. Focused validation: **95 passed**. Worktree is clean and synced with `origin/master`.

Agent-hood publication hit a busy lock and will retry automatically.

---

%xprompts_enabled:true
# New Query

 Can you now help me start showing which spot in the queue the currently selected QUEUED agent has in the agent metadata panel somehow? I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
