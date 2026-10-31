---
tier: epic
title: Stop recurring agent_publication error notifications
goal: "Approving a plan and running research agents no longer produce
  `agent_publication` error notifications. Any refusal that remains states its real
  reason and retires instead of repeating. Rows already stuck by these bugs clear
  through verified acknowledgment, not by being dropped.

  "
decisions:
  cleanup_apply:
    ask:
      Should the cleanup phase re-scope and retry the stuck rows itself instead of
      asking you first?
    choices:
      gate:
        Phase stops at a confirmation gate; nothing changes until you approve the
        commands
      direct:
        Phase applies the audited re-scope and per-key retries itself; rows clear
        unattended
    default: gate
    why: Applying edits the live outbox and pushes to the sase--agents remote
    answer: direct
phases:
  - id: planner-scope
    title: Planner prompt-archive rows use linked (page-only) scope
    depends_on: []
    size: medium
    description:
      "planner-scope: planner approvals enqueue their prompt-archive row with linked
      scope, and enqueue merges keep revision scope when either side has it, so the
      revision gate stops refusing rows for commits the planner never made."
  - id: honest-errors
    title: Report the real refusal reason and retire repeated refusals
    depends_on:
      - planner-scope
    size: medium
    description:
      'honest-errors: gate refusals record the Rust completion decision''s reason
      instead of "did not materialize", the dead retire-on-repeat path works again, and
      the stale-revision doctor hint stops recommending a repair that cannot apply.'
  - id: hood-scan-reuse
    title: Stop re-scanning already-published bytes on every hood publish
    depends_on: []
    size: medium
    description:
      "hood-scan-reuse: targeted hood publication skips the protection scan for bytes
      identical to the previously published file reference, which brings the research
      hood back under the 120s drain budget."
  - id: planner-restore
    title: Deferred restore rebuilds the plan-named planner archive (sase-1c0)
    depends_on:
      - planner-scope
    size: medium
    description:
      "planner-restore: persist the planner archive inputs in the planner's artifacts
      directory so a deferred restore rebuilds the same plan-named archive with its PLAN
      link, then close sase-1c0."
  - id: footer-brackets
    title: Parse bracketed footer values that lack a link definition
    depends_on: []
    size: small
    description:
      "footer-brackets: the sase-core commit-footer parser unwraps `SASE_BEAD=[id]`
      values that have no reference definition, and sase rebuilds the cached stitch-rule
      rows, so the `bead:[...]` warning stops appearing in digests."
  - id: planner-repair
    title: Report-first re-scope repair for proven planner rows
    depends_on:
      - honest-errors
    size: medium
    description:
      "planner-repair: add a report-first `sase agent sync --repair-planner-scopes` mode
      that re-scopes only evidence-proven planner rows to linked through the outbox
      mutation API, with an audit record."
  - id: stuck-row-cleanup
    title: Clear the already-stuck planner and research rows
    depends_on:
      - hood-scan-reuse
      - planner-repair
    size: small
    description:
      "stuck-row-cleanup: once the fixes are live, re-scope proven planner rows and
      retry each targeted row by key, then verify that every one cleared through
      acknowledgment with its pages and archives intact."
proposed_by: bbugyi200.athena.research.5l.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-31 12:01:46
status: wip
---

- **PROMPT:**
  [prompts/202610/agent_publication_notifications.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/agent_publication_notifications.md)

# Plan: Stop recurring agent_publication error notifications

## Background

The `publication / agent_publication` axe job raises `check_error` (one digest entry and
one notification) whenever a drain tick records an error
(`src/sase/scripts/sase_chop_agent_publication.py`). The Oct 30–31 digests under
`~/.sase/axe/error_digests/` contain two recurring error classes. They have different
root causes.

### Class 1: "published agent page for '…' did not materialize during commit drain"

Diagnosed in the research report
`research:202610/agent_publication_planner_scope_mismatch/agent_publication_planner_scope_mismatch.md`
(read it with `sase artifact read`). The user accepted every requirement in it.

- **Producer.** Plan approval calls `publish_planner_prompt_archive()`
  (`src/sase/axe/run_agent_exec_plan_accept_sdd.py`). That function records the
  workspace's `git rev-parse HEAD` as `primary_revision`. The commit at that `HEAD` is
  usually another agent's landing commit.
- **Scope.** `_publish_prompt_archive()`
  (`src/sase/agents_sync/prompt_archive/publish.py`) enqueues the row with
  `scope=fulfillment_scope_for_commit_cwd(commit_cwd)`, which returns `revision` for any
  primary checkout.
- **Gate.** sase-core's `decide_publication_request_completion` acknowledges a revision
  row only when the SHA is one of the agent's own run commits. A planner never owns that
  SHA, so the decision is always `fulfilled=false`, with the reason "required page
  exists without the requested primary revision".
- **Message.** Python discards that reason and records "did not materialize". The row
  charges three attempts (three notifications) and is then quarantined.
- **Evidence.** Eight of eight plan approvals since the background publisher went live
  failed this way. Replaying the production policy with only the scope changed to
  `linked` acknowledges every one of them.

### Class 2: "agent-hood publication did not complete within 120s"

New in this plan; the research report covers only the historical backlog of this class.
It fires for research agents (outbox rows `research.5j.grk`, `research.5j.gem`,
`research.5l.image` and `research.5l.linker`; four errors in
`digest_20261031_112418.txt`).

- **The hood.** Every research agent belongs to the single top-level `research` hood:
  about 1,220 runs and about 9 MB of chat. `publish_agent_hood` re-renders the
  committing agent's whole top-level hood.
- **The regression.** `~/.sase/logs/agent_publication_drains.jsonl` shows
  `hood_publish:research` taking 9–11 s from Oct 24 to 26, then 50–75 s from Oct 28.
  `inventory_build` adds another 26–50 s, so the drain crosses the 120 s alarm.
- **Cause.** The jump lines up with the transcript-protection commits `67edb9677d`
  (Oct 26) and `0f76e05de4` (Oct 27). Since those landed,
  `src/sase/agents_sync/_publication_snapshot_gating.py::gate_run_files` does the
  following for every run in the hood and every file kind (meta, state, commits, prompt,
  chat, embedded_workflows, prompt_steps):
  - a blocked-record ledger lookup;
  - `scan_publication_bytes`, which writes a private temp file and calls the Rust
    attachment scanner with a full environment capture;
  - `record_publication_bytes_scan`, which takes the ledger lock and reads it.
- **Measurement.** A read-only micro-benchmark scanned 300 sampled research-hood files
  in 3.4 s (about 11 ms per file). The ledger lookups took 0.07 s. The research hood has
  about 8.5k published files, so one publish spends about 60–90 s on scans.
- **Knock-on failure.** When the alarm fires mid-drain, cleanup can also fail on a live
  `index.lock` in the agents clone (`digest_20261031_112418.txt`, error 4).

### Constraints for every phase

- **When fixes go live.** axe's tool environment installs `sase` editable from the
  primary checkout (`~/projects/github/sase-org/sase`), and the job runs as a per-tick
  subprocess. A Python fix is live as soon as that checkout contains it. A Rust change
  also needs the `sase_core_rs` binding rebuilt. Global install is human-only, so never
  run `just install*`.
- **Keep the gate strict.** Keep the completion policy in Rust and do not relax it.
  Foreign-SHA or zero-commit revision rows must stay refused.
- **Keep the errors visible.** Do not downgrade this error class to a warning, and do
  not stop counting it in `check_error`. Do not mute digests or raise the 120 s drain
  budget.
- **Leave the backlog alone.** Never run an unbounded
  `sase agent sync --retry-quarantined`, which would revive about 3,500 historical rows.
  Never purge rows by age.
- **Verification.** Read the `lint_and_test.md` memory before finishing. `just check`
  (through `sase tool run`) is the verification recipe.
- **Docs.** Read the `docs_writing.md` memory before editing anything under `docs/`.
- **CLI changes.** Read the `cli_rules.md` memory before adding a CLI option.

## Phase planner-scope: Planner prompt-archive rows use linked (page-only) scope

1. **`src/sase/agents_sync/prompt_archive/publish.py`.**
   - Add a keyword-only `fulfillment_scope: str | None = None` to
     `publish_prompt_archive` and `_publish_prompt_archive`.
   - Enqueue with
     `scope=fulfillment_scope or fulfillment_scope_for_commit_cwd(commit_cwd)`.
   - Reject values other than the existing `PUBLICATION_SCOPE_LINKED` and
     `PUBLICATION_SCOPE_REVISION` constants.
2. **`publish_planner_prompt_archive`.** Pass
   `fulfillment_scope=PUBLICATION_SCOPE_LINKED`. Add a comment explaining why:
   - the pre-plan `HEAD` is only the row's anchor and key;
   - the planner does not own that commit;
   - the planner's obligation is the session page plus the prompt archive, and
     `acknowledge_completed` already gates the archive.
3. **`enqueue_agent_publication`**
   (`src/sase/agents_sync/publication_outbox_operations.py`). Merge scope so that
   `revision` wins when either the existing row or the incoming row has it. This mirrors
   `_publication_revision_repair_apply.py`. A planner enqueue then cannot weaken a
   commit row that has the same `(global_agent, SHA)` key.
4. **Leave the commit path alone** (`workflow_publication.py`). It publishes the agent's
   own post-commit SHA, and the revision gate is correct there.
5. **Docs.** Update `docs/reference/agents/hood-sync.md`. Its outbox paragraph says
   linked scope applies only to linked-repo commits; add that approved-planner archive
   rows also use page-only (`linked`) fulfillment.

Tests:

- The planner path enqueues `linked`, and the commit-path prompt archive still enqueues
  `revision`.
- **Epic planner:** a drain acknowledges the planner row on its first attempt when the
  session page and prompt archive exist and the hood has no run commits.
- **Tale planner:** same as the epic case, after a coder commits a SHA that descends
  from the planner's `HEAD`.
- **Merge:** enqueuing `linked` on top of an existing `revision` row keeps `revision`.
- **Still refused:** a real commit row naming the wrong SHA, a missing page, missing
  prompt bytes, and a protection failure.

Record a `PROPOSED FOLLOW-UP:` note on this phase bead for an explicit `prompt` scope in
sase-core. That change needs a scope in `complete.rs` and `stale.rs::normalize_scope`,
an outbox schema bump with migration, the binding, and a `sase-core-revision.txt` pin.
Do not implement it here: one row with an unknown scope makes the whole outbox
unreadable to older readers.

## Phase honest-errors: Report the real refusal reason and retire repeated refusals

1. **Report the real reason.**
   - Have the completion check return the full `PublicationCompletionDecision` (`kind`,
     `required_page`, `fulfilled`, `reason`) instead of a bool. Keep a thin bool wrapper
     only if other callers need one.
   - In `acknowledge_completed` (`_commit_publication_transaction_steps.py`) and the
     full-sync site in `git_sync.py`, record a message such as:
     `agent publication for 'X' not acknowledged: <reason> (page <required_page>, scope <scope>, revision <sha12>)`.
   - Still charge the attempt, and still count the message in the tick's errors.
   - Update the tests that assert the old wording:
     - `tests/agents_sync/test_commit_publication_drain_accounting.py`
     - `tests/agents_sync/test_git_sync_outbox/test_sync.py`
     - `tests/agents_sync/test_git_sync_outbox/test_retry_retired.py`
   - Error classes do not match on this text (it classifies as `other`). Confirm that
     before changing it.
2. **Repair the dead retire-on-repeat path.**
   - The bug: `prepare_publications` writes `error=None` to every prepared row before
     `acknowledge_completed` charges the refusal. The
     `item.last_error == terminal_reason` check in `update_agent_publications` therefore
     never matches, so deterministic refusals quarantine after three attempts instead of
     retiring after two.
   - The fix: compare against the error the row held before this drain cleared it. The
     in-memory request carries it, or you can stop clearing the error until
     acknowledgment. Pick whichever keeps the hood-digest update and status projection
     correct.
   - Required behavior: an identical deterministic refusal on two consecutive attempts
     retires the row. Timeouts, lock-busy, withheld runs, and other uncharged failures
     never retire. The full-sync path behaves the same way.
3. **Doctor hint.** `stale_requested_revision_rows` and its hint in
   `publication_completion.py` currently point every stale row at
   `--repair-revisions --apply`.
   - Stop recommending `--apply` for rows the classifier cannot repair. For example,
     point at the report-only `--repair-revisions` pass instead.
   - `--check` and doctor must stay cheap: do not build the hood inventory there.
4. **Docs.** Make `docs/reference/agents/hood-sync.md` match the new message, and the
   retire-after-one-confirming-retry wording that is now true again.

## Phase hood-scan-reuse: Stop re-scanning already-published bytes on every hood publish

1. **Measure first.** Profile the research hood read-only with
   `tools/agents_publication_profile --project gh_sase-org__sase`, or with a read-only
   script that times `plan_targeted_hoods` and `build_hood_snapshot` for the `research`
   hood. Do not apply, commit, or push to the agents clone. Confirm that
   `gate_run_files` scans dominate.
2. **Reuse prior verdicts for unchanged bytes.**
   - Have `build_hood_snapshot` (`_publication_snapshot_hood.py`) pass each run's
     previous published file references into `gate_run_files`. `previous_by_global`
     already holds them.
   - For each candidate:
     - Run `_prior_content_blocked` first, and keep it first: a durable `blocked` record
       always withholds.
     - If `content_digest(content)` equals the digest the previous snapshot records for
       the same run and destination, accept the bytes without `scan_publication_bytes`
       and without a ledger write.
     - Scan everything else exactly as today: new files, changed files, and runs that
       have no previous snapshot.
   - Apply the same rule wherever prior bytes are rescanned on the merge path
     (`_prior_bytes_releasable` and `merge_richer_run_files`), if profiling shows it
     matters.
3. **Why this is safe.** Bytes that match a previous published reference already sit in
   the published sidecar. Rescanning cannot unpublish them, and rewriting identical
   bytes is a no-op. Only new or changed bytes add exposure, and they are still scanned.
   The one behavior change is that pre-protection bytes (published before Oct 26) are no
   longer rescanned on every publish. They are already in sidecar history either way.
   State this in a code comment and in the commit message.
4. **Tests.**
   - Republishing an unchanged hood performs zero scans (use a scan spy).
   - A changed chat file is scanned.
   - A new run is scanned.
   - A digest with a durable `blocked` record is withheld even when it matches the
     previous reference.
   - A scanner hit on changed bytes still withholds them.
5. **Acceptance.**
   - Re-measure on athena. The research-hood planning time must fall back near its
     pre-regression level of about 10–15 s.
   - A full research-hood drain must fit comfortably inside the 120 s budget.
   - Put the before and after numbers on this phase bead.
6. **Follow-ups.** If `inventory_build` still pushes the research drain past about 60%
   of the budget, record a `PROPOSED FOLLOW-UP:`. The `research` hood grows without
   bound because every `research.<id>.*` agent shares it. Record another
   `PROPOSED FOLLOW-UP:` if the post-timeout cleanup failure on a live `index.lock` (an
   interrupted git child that still holds the lock) reproduces.

## Phase planner-restore: Deferred restore rebuilds the plan-named planner archive (sase-1c0)

This phase delivers task bead `sase-1c0`. Read it first with `sase bead read sase-1c0`.

- **The gap.** Commit `66b15496f9` already lets a deferred restore find the planner
  source through `artifacts_dir`. But `prompt_archive/deferred.py` calls
  `prepare_prompt_archive` without `plan_ref`, `prompt_name`, or `yyyymm`. A planner
  archive restored after lock contention therefore lands under the default name, without
  its PLAN link, and possibly with a body other than the plan snapshot the inline path
  publishes.
- **Persist the inputs.** In `publish_planner_prompt_archive`, before enqueueing, write
  a small versioned JSON record into the planner's artifacts directory (the row already
  stores `artifacts_dir`). The record holds the archive inputs: `plan_ref`,
  `prompt_name`, `yyyymm`, and the exact planner `prompt_content` (or a file beside it).
  Do not widen the outbox schema: a schema bump makes the outbox unreadable to older
  readers.
- **Restore from them.** Teach the deferred restore to load that record when present and
  pass the inputs through. Rows without the record keep today's behavior.
- **Regression test.** A planner request whose `primary_revision` is not one of the
  run's commits:
  - fails its inline publish on lock contention;
  - is restored by a later drain to `prompts/<yyyymm>/<plan_name>.md`, with the PLAN
    link and the plan-snapshot body;
  - is acknowledged under the linked scope from phase planner-scope.
- **Close the bead.** After verifying, close it with
  `sase bead close sase-1c0 --note "<what you verified>"`.

## Phase footer-brackets: Parse bracketed footer values that lack a link definition

- **The defect.** Two historical commits carry `SASE_BEAD=[id]` with no reference
  definition: `sase` `8c8c47f720` and `bob-cli` `cc9bd5f5`. The parser keeps the
  brackets, the stitch projection emits the ref `bead:[id]`, and every drain and
  backfill logs `dropping projected edge with invalid ref: rule=stitch-bead ...`. That
  line pads every digest.
- **sase-core.** Open the repo with `/sase_repo`, read its `AGENTS.md`, and verify with
  `sase tool run check`. In `crates/sase_core/src/commit_footer.rs`:
  - treat a value of the exact form `[label]` (a Markdown shortcut reference with no
    `][`) as `label`, with no destination;
  - treat `[label][id]` whose `id` has no definition the same way.
  - Keep `raw_value` unchanged, so `update_commit_footer` round-trips stay
    byte-identical.
  - Add parser tests for both forms, plus a round-trip test.
- **sase.** The stitch rule cache (`src/sase/artifact_links/projection/`) has already
  stored the bad rows, so the warning would persist after the parser fix. Invalidate
  only the `stitch-bead` and `stitch-agent` cache entries, for example with a rule-level
  version in their cache signature, so they rebuild once. Do not bump the document-wide
  `_CACHE_SCHEMA_VERSION` if that would also force unrelated rules to recompute. If a
  sase test depends on the new parser behavior, move the pin with
  `just ratchet-core-revision`.
- **Going live.** The change takes effect on athena only after the binding is rebuilt.
  Note that on the phase bead; do not run an install.

## Phase planner-repair: Report-first re-scope repair for proven planner rows

1. **The command.** Add `sase agent sync --repair-planner-scopes` (`-P`), modeled on
   `--repair-revisions` (`publication_revision_repair.py` and its classification, apply,
   and models modules).
   - Process one project per call, under the agents lock.
   - Report only by default. Write only with `--apply`.
   - `--json` output is stable.
   - Reject combining it with `--check`, `--drop-retired`, retry flags, or other repair
     modes.
   - Keep the options sorted, and write help text in the style of the existing flags.
2. **Candidates.** Only `revision`-scope rows that are not acknowledged.
   - Classify cheaply. Never build the full hood inventory or run ancestry checks for
     every row: the full `--repair-revisions -p sase` pass did not finish within 280 s.
   - The whole `sase` outbox (about 3.5k rows) must report in well under a minute.
3. **Proof.** A row is a proven planner row when either:
   - (a) the `agent_meta.json` in its `artifacts_dir` names the planner `--plan` turn,
     with `plan_approved=true` and `workspace_head` equal to the row's
     `primary_revision`; or
   - (b) the agents sidecar has an `archive prompt for <local agent>--plan` commit
     within a small window of the row's `created_at`.

   Check both rules against the live rows listed in the research report before fixing
   the matching details. Rows without proof are reported as `unproven` and never
   changed.

4. **Apply.**
   - Change only `scope` to `linked`, through the outbox mutation API.
   - Keep the key, attempts, errors, and quarantine or retirement state. The operator
     retries rows explicitly.
   - Append one audit JSONL line per change, recording the key, old and new scope, the
     evidence kind, and the time.
   - A second apply is a no-op.
5. **Docs and hint.** Extend the stale-revision doctor hint from phase honest-errors to
   mention `-P` for planner rows. Document the mode in `docs/reference/cli/agent.md`,
   `docs/contribute/agents-sidecar-sync.md` (its repair section), and
   `docs/reference/agents/hood-sync.md`.
6. **Tests.**
   - Report mode writes nothing.
   - Apply re-scopes rows proven by (a) and rows proven by (b).
   - Unproven rows, real commit rows, and linked rows are untouched.
   - The audit line is written.
   - A repeated apply is idempotent.
   - The option-conflict errors fire.

## Phase stuck-row-cleanup: Clear the already-stuck planner and research rows

1. **Precondition.** The primary checkout (`~/projects/github/sase-org/sase`) must
   contain the landed commits of phases planner-scope, honest-errors, hood-scan-reuse,
   and planner-repair. Check with `git merge-base --is-ancestor`. If it does not, stop
   and report what is missing.
2. **Baseline.** Record per-project active, quarantined, and retired counts for `sase`
   and `bob-cli` with `sase doctor -C state.agent_publication_outbox -j`.
3. **Report.** Run `sase agent sync -P -p sase -j` and `-p bob-cli`. Expected results:
   - the `sase` planner rows from the research report (`sase-1pj.6`, `sase-1pj.7`,
     `sase-1q2.land`, `sase-1q4.land`, `sase-1q5.land`, and `sase-1pj.5.1.land` if
     sidecar evidence proves it), plus any older rows with evidence;
   - `bob-cli-eg.land` and `bob-cli-er.land`.
4. **Targets.**
   - The proven planner rows.
   - The research-hood timeout rows (`research.5j.grk`, `research.5j.gem`,
     `research.5l.image`, `research.5l.linker`), unless they have already acknowledged
     on their own.
5. **Apply and retry.** Apply the re-scope, then retry each target with a bounded retry
   by key:

   ```sh
   sase agent sync -p <project> --retry-quarantined --retry-limit 1 --retry-key '<global_agent>@<full sha>'
   ```

   Use `--retry-retired` for rows that are retired.

   > [!decision] cleanup_apply = gate End the phase with `/sase_gate`, proposing the
   > exact apply and per-key retry commands followed by the verification in step 6.
   > Nothing changes until the user confirms.

   > [!decision] cleanup_apply = direct Run the apply and the per-key retries yourself,
   > one key at a time, stopping at the first unexpected failure. Then run the
   > verification in step 6.

6. **Verify.**
   - Each targeted row disappeared through verified acknowledgment, not a drop.
   - Its session or agent page and its archived prompt still exist in the agents
     sidecar.
   - Quarantine counts fell by exactly the number of targets.
   - The next digest has no `agent_publication` error.
7. **Leave the rest.** Leave every other quarantined row untouched. Record a
   `PROPOSED FOLLOW-UP:` for a deliberate, batched recovery of the older backlog. It
   includes about 3,500 quarantined `sase` rows, about 1,450 of which describe pages
   that were never published, plus most of `bob-cli`'s 61 unproven rows.

## Acceptance criteria

1. **Planner approvals go quiet.** In a primary workspace whose `HEAD` belongs to
   another agent, approving a plan (epic or tale) produces a planner row that is
   acknowledged on its first drain, with no `agent_publication` error.
2. **The research hood fits the budget.** A research agent's hood publication completes
   within the 120 s drain budget, and the drain timing log shows the scan regression
   gone.
3. **Real failures are still refused.** Commit rows with the wrong SHA, missing pages,
   missing prompt bytes, and protection failures are all still refused. The message now
   names the real reason, and an identical refusal retires after one confirming retry.
4. **Merges and repairs are stable.** Enqueuing a planner row and a commit row with the
   same key keeps `revision`. Repeating the repair or the retry changes nothing further.
5. **Planner archives restore faithfully.** A planner archive restored after lock
   contention lands under its plan name with its PLAN link. `sase-1c0` is closed.
6. **No `bead:[…]` warning.** The digests stop showing it once the core binding is
   rebuilt.
7. **Cleanup is verified.** After cleanup, every targeted row cleared through
   acknowledgment and its session page and archived prompt are still present. Check
   quarantine counts, not just active counts.
