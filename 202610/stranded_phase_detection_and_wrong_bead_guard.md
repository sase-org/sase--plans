---
tier: epic
title: Catch phases that end without closing their bead, and stop agents from working
  the wrong bead
goal: A phase agent that misreads its own bead ID is warned or refused at the CLI.
  A single-turn phase that ends with its bead still open is classified stranded and
  raises one wait alert. A follow-on epic proposed after its bead closed no longer
  nests under it.
decisions:
  near_miss_mutation:
    ask: Should bead close/open refuse a target that is a one-segment near-miss of
      the agent's assigned bead?
    choices:
      refuse: Close and open exit non-zero before writing; read, note, update, and
        epic-symbols warn
      warn: Every command only warns; a miscopied close still goes through
    default: refuse
    why: No workflow legitimately mutates such a bead, and a wrong close is worse
      than a stall
    answer: refuse
phases:
- id: stranded-root-members
  title: Let the stranded-work collector see single-turn sessions
  size: medium
  depends_on: []
  description: 'stranded-root-members: add an agent_name candidate-filter field to
    the sase-core artifact index and query agent_session OR agent_name from the stranded
    collector, so a session''s root member (agent_session null) is no longer reported
    unknown/no_records.'
- id: stranded-bead-wait-alert
  title: Raise one wait alert per stranded bead a live waiter is blocked on
  size: medium
  depends_on:
  - stranded-root-members
  description: 'stranded-bead-wait-alert: make wait_checks classify the unclosed beads
    that live waiters block on through a targeted, TTL-cached stranded-work collection,
    and upsert one deduplicated, self-dismissing notification per stranded bead with
    its cause and relaunch command.'
- id: assigned-bead-near-miss
  title: Warn or refuse when an agent targets a one-segment near-miss of its assigned
    bead
  size: medium
  depends_on: []
  description: 'assigned-bead-near-miss: detect, inside SASE agents, bead IDs that
    differ from the assigned bead by one inserted or deleted non-trailing segment;
    banner the read output, refuse close/open per the near_miss_mutation decision,
    and tighten the phase and task work prompts.'
- id: closed-parent-stamp
  title: Stop plan propose from parenting new epics under an already-closed bead
  size: small
  depends_on: []
  description: 'closed-parent-stamp: skip the automatic parent_bead stamp in sase
    plan propose when the proposing agent''s active bead is already closed, so follow-on
    campaign epics are top-level instead of growing the ID one segment per batch.'
proposed_by: bbugyi200.athena.14p
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-26 18:10:47
status: wip
bead_id: sase-1ok
---

- **PROMPT:** [prompts/202610/stranded_phase_detection_and_wrong_bead_guard.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/stranded_phase_detection_and_wrong_bead_guard.md)
- **BEAD:** [sase-1ok](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ok/README.md)

# Catch phases that end without closing their bead, and stop agents from working the wrong bead

## Incident

Epic `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6` (bob-cli backlog batch 15) stalled on
2026-10-26. Phases 3, 4, and 5 closed. Phase 1's worker (codex, single turn, 15:54 to
16:03 EDT) was asked to work `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.1`. It ran
`sase bead read bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.1` instead, dropping one interior
`.6`. That bead is batch 14's phase 1, which was already closed. The worker concluded
"the bead is already marked closed" and verified batch 14's work. It ran `epic-symbols`
on the wrong ID too, then ended with outcome `completed` without touching its own bead.

Phase 1's bead is still `in_progress`. Phase 2 and the land agent are live `WAITING`
runners blocked on it, and nothing will ever release them:

1. **Wrong-ID misread.** The ID has 16 segments, 12 of them `.6`. Nothing in the bead
   CLI noticed that the agent asked for a bead one segment away from its assigned
   `SASE_BEAD_ID`. The phase prompt's "if the host notice says the bead was already
   closed" branch was taken without any host notice.
2. **Stranded detection is blind to single-turn sessions.**
   `src/sase/bead/stranded_facts.py::_query_session_records` filters the artifact index
   on `agent_session == assignee`. A session's root member keeps
   `agent_meta.agent_session: null` until it gets a successor. So every one-turn worker,
   and every live waiter, gets zero records and the verdict `unknown/no_records`. On
   2026-10-26 that verdict covered 9 of bob-cli's and 143 of sase's in-progress
   candidates, including all three live beads of this epic. ACE's ⚐ STRANDED badge and
   `sase doctor`'s `project.beads` advisory never fired.
3. **No alert for bead waits.** `wait_checks` only raises "Wait dependency can never
   self-resolve" when a named dependency ends _unsuccessfully_
   (`src/sase/scripts/_chop_wait_checks_terminal.py::terminal_blockers`). A bead wait
   whose assignee finished successfully without closing the bead is never flagged. The
   old EpicResume scan was removed in `5938b6dced` because its unbounded five-minute
   scan broke AXE health.
4. **Ever-deepening IDs.** `src/sase/main/plan_propose_handler.py` always stamps
   `parent_bead = active_bead` on epic plans proposed from bead work. Each campaign
   lander closes its epic and then proposes the next batch, so every batch nests one
   level deeper under an already-closed epic. About 120 tasks remain at five per batch,
   so this chain would reach roughly 40 segments.

Recovering the incident epic is an operator action and is out of scope here. The command
is `sase bead work bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6 --yes-to-all`; its dry run
kills the two waiters and relaunches phase 1, phase 2, and land.

## Phase stranded-root-members: let the collector see single-turn sessions

Follow the Rust core backend boundary core memory and read `docs/rust_backend.md` first.
Open the linked core repo with `sase repo open sase-core`.

1. In sase-core `crates/sase_core/src/agent_scan/index/index_wire.rs`, add an
   `AgentName` variant to `AgentArtifactCandidateFieldWire` (serde `agent_name`). It
   reads the existing indexed `agent_name` column. Update the doc comment beside
   `AgentSession`.
2. Carry `agent_name` on `IndexedCandidateRow` (`index/selection.rs`, loaded in
   `index/candidates.rs`) and return it from `scalar_values` for `AgentName`. Matching
   is case-insensitive like every other `Equals` field. Add a Rust test beside
   `index/tests/agent_session_filter.rs` with a root member (`name: lane-a`, no
   `agent_session`) and its successor (`name: lane-a--plan`, `agent_session: lane-a`).
   `Any[Equals(agent_session, lane-a), Equals(agent_name, lane-a)]` must return both
   rows and no `lane-b` rows. Run sase-core's checks.
3. Move sase's `sase-core-revision.txt` pin past the new sase-core commit, as
   `docs/rust_backend.md` describes. Update any Python-side literal or type that
   enumerates candidate-filter fields; grep `agent_scan_wire` and its conversion module.
4. In `src/sase/bead/stranded_facts.py::_query_session_records`, query with
   `{"kind": "any", "filters": [<agent_session equals>, <agent_name equals>]}`. Keep the
   exact `_record_matches_assignee` recheck and dedupe records by `artifact_dir`.
5. Regression tests (`tests/` beside the existing stranded-facts and reducer tests):
   - A single-turn phase whose root member has `agent_session: null`, a done marker with
     outcome `completed`, and an `in_progress` bead is classified `stranded`/`unclosed`
     once past the 120s settle grace, not `unknown`/`no_records`.
   - Its parent epic is `blocked`.
   - A live single-turn `WAITING` member is `owned`.
   - Multi-member sessions still classify exactly as before.
6. If `docs/reference/work-units/bead-lifecycles.md` "Stranded beads" says anything that
   is now wrong about `unknown`, correct it.

## Phase stranded-bead-wait-alert: one alert per stranded bead with a live waiter

`wait_checks` ticks every 2 seconds under a 2-minute job timeout
(`src/sase/default_config.yml`), and the removed EpicResume scan failed precisely
because it was unbounded. Every step below must stay targeted and cached.

1. Add a targeted entry point next to `collect_stranded_work_input`, for example
   `collect_stranded_work_input(project, *, bead_ids=...)`. It builds candidates only
   for the named in-progress beads plus their context child plan beads, and makes one
   index query per assignee. Keep `load_stranded_report` behavior unchanged for ACE and
   doctor.
2. In `src/sase/scripts/_chop_wait_checks_run_waiters.py::process_one_waiter`, take the
   unresolved branch. For blocked labels that come from `wait_for_beads` (not closed per
   `closed_bead_ids`), record them on the tick, grouped by the waiter's project. After
   all waiters are processed, run one targeted collection and reduction per project.
   Cache it on a per-project TTL (60s), keyed by the sorted bead-ID set plus the
   bead-store fingerprint if one is available. Never let a collector failure park or
   release a waiter: log, skip, and fail open, the way the existing `noqa: BLE001`
   blocks do.
3. For each verdict with `state == "stranded"`, upsert one notification from the
   `wait_checks` sender in `src/sase/scripts/_chop_wait_checks_terminal.py`:
   - dedup key `wait_checks:stranded-bead:<project>:<bead_id>`, with every blocked
     waiter listed and later waiters added as `+1` notes;
   - notes holding the bead ID and title, the cause via
     `sase.bead.stranded_presentation.stranded_cause_sentence`, the last turn and its
     age (`stranded_age_label`), and the waiters it blocks;
   - the remedy: for a phase bead, `sase bead work <parent epic id> --yes-to-all`
     ("relaunches every non-closed phase and the land agent"); for a task bead,
     `sase bead work <task id> --yes`; plus closing it with
     `sase bead close <id> --note` when the work is actually done;
   - `action="JumpToAgent"` toward the last turn when `last_turn` and its artifact dir
     are known, mirroring `upsert_epic_follow_blocked_notification`;
   - red color, tags `wait`, `blocked`, `stranded`.

   Do not also raise the per-waiter `terminal-blocked` entry for the same bead.

4. Reconcile like `reconcile_epic_follow_notifications`. Dismiss a `stranded-bead` row
   once its bead is closed, owned, delegated, or settling for a project evaluated this
   tick. Rows for projects whose collection failed this tick stay put.
5. Tests:
   - The incident shape: phase 1 done/completed with its bead in progress, phase 2 and
     land waiting. Expect exactly one notification naming phase 1 and both waiters, with
     the epic relaunch command.
   - No notification during the settle grace, or while the bead has a live member, a
     pending handoff/monitor/gate, or an open child plan bead.
   - Dismissal after the bead closes.
   - A collector exception leaves waiters untouched.
   - A cache test proving at most one collection per project per TTL across many ticks.
6. Docs: add the new entry to the `wait_checks` sender/event rows in
   `docs/notifications.md`, and to the "Stranded beads" section of
   `docs/reference/work-units/bead-lifecycles.md` ("a live waiter blocked on a stranded
   bead raises one notification").

## Phase assigned-bead-near-miss: catch miscopied IDs at the CLI

1. Add a pure helper, for example `src/sase/bead/assigned_near_miss.py`. Its signature
   is `near_miss_of_assigned(requested: str, assigned: str) -> bool`. It is true when
   the two full IDs share the same root (the text before the first `.`), are unequal,
   and neither is an ancestor or descendant of the other (dot-segment prefix). It also
   requires that inserting or deleting exactly one dot segment turns one into the other.
   A trailing-only insert or delete is an ancestor or descendant and so never counts.
   Unit-test it with the incident IDs:
   - assigned `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.1` and requested
     `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.1` → true;
   - a sibling (`…6.6.2`), the parent epic, a child, a different root (`bob-cli-cj`),
     and a two-segment difference → false.
2. Resolve the assigned bead the way `plan_propose_handler` does: `SASE_PHASE_BEAD_ID`,
   then `SASE_BEAD_ID`, then the agent-meta `phase_bead_id`/`bead_id` association. Act
   only when an assigned bead resolves, which means only inside SASE agent runs. Compare
   full IDs after shorthand resolution. When the requested ID does not resolve at all,
   append "did you mean `<assigned>`?" to the not-found error if it is a near-miss.
3. `sase bead read` (`src/sase/bead/cli_query.py::handle_bead_read`): in text formats,
   print a warning banner at both the top and the bottom of stdout. Agent harnesses
   often truncate long output from one end. The banner reads: "⚠ `<requested>` is not
   your assigned bead `<assigned>`; the two IDs differ by one segment. If you meant your
   own bead, rerun with the exact ID `<assigned>`." In JSON format, leave stdout
   unchanged and print the warning on stderr.
4. `sase bead note`, `sase bead update`, and `sase bead epic-symbols`: print the same
   warning on stderr.

> [!decision] near_miss_mutation = refuse `sase bead close` and `sase bead open`
> (`src/sase/bead/cli_crud_lifecycle.py`) exit non-zero before any write. The error
> names both IDs and says the target differs from the assigned bead by one segment. Add
> no override flag.

> [!decision] near_miss_mutation = warn `close` and `open` print the stderr warning and
> proceed.

5. Prompt templates in `src/sase/default_config.yml`, `bd/work_phase_bead` and
   `bd/work_task`: after the "If the host notice says the bead was already closed"
   sentence, add one sentence. "Only the host notice tells you that; if `sase bead read`
   shows a closed bead and no host notice said so, you copied the wrong ID — re-copy
   `{{ bead_id }}` from this prompt before concluding anything." Update any prompt
   snapshot tests that pin these templates.
6. Tests cover the read banner (text and JSON), the note/update/epic-symbols warnings,
   the close/open branch of the decision, and "no effect" when no assigned bead
   resolves. Add no new CLI options.

## Phase closed-parent-stamp: no parenting under a closed bead

1. In `src/sase/main/plan_propose_handler.py`, before
   `stamps["parent_bead"] = active_bead` for `target_tier == "epic"`, look up
   `active_bead`'s status in the project's bead store. Reuse an existing status read
   such as `src/sase/finalizers/declaration_manifest_commit.py::_assigned_bead_status`
   if it fits, or the bead read facade. If the status is `closed`, do not stamp
   `parent_bead`. Print one line: "Active bead `<id>` is already closed; proposing a
   top-level epic (the PARENT plan section still links the earlier plan)." Keep the
   `upsert_parent_plan_section` provenance. If the status cannot be read, keep today's
   stamping. Leave an author-written `parent_bead` alone in both cases.
2. Tests:
   - A land agent with a closed epic proposes an epic plan, and the archived frontmatter
     has no `parent_bead`.
   - With the active bead in progress, the stamp is unchanged.
   - When the status is unreadable, the stamp is unchanged.
   - A tale proposal is unaffected.
3. Docs: in `docs/reference/work-units/bead-work.md`, extend the "When an epic-tier plan
   is proposed from bead work…" paragraph. It should say the stamp is skipped when the
   active bead is already closed, because there is no interrupted landing for a child
   epic to resume.

## Verification

Each phase runs `just check` in sase and records results on its bead. The
stranded-root-members phase also runs sase-core's checks. Before closing, the land agent
re-runs the stranded collector for `gh_bobs-org__bob-cli` and `gh_sase-org__sase`. It
records how the `unknown/no_records` counts changed, and confirms that any surviving
`unknown` verdicts are beads with genuinely no artifact records.
