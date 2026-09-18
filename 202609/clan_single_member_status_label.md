---
tier: tale
title: Preserve a sole clan member's custom failure status label
goal:
  Show TESTED on a clan whose sole nonwaiting, nondone member displays TESTED, while
  preserving its failure bucket, styling, counts, and status precedence.
size: small
proposed_by: bbugyi200.athena.0n1
create_time: 2026-09-18 13:48:23
status: wip
---

# Preserve a sole clan member's custom failure status label

## Diagnosis and evidence

The reported screenshot, `~/tmp/screenshots/20260918_133737.png`, shows clan `sase-11l`
as `FAILED [W1 F1 D9]` while its direct family member `sase-11l.10` displays `TESTED`.
The landing agent is waiting and the other nine members are done.

The read-only `sase agent list -a -j` inspection confirmed that `sase-11l.10--mon-1` has
`status: TESTED`, `status_bucket: Failed`, `monitor_state: failed`,
`monitor_exit_code: -15`, and the authored pair `TESTING` / `TESTED`. It started at
`2026-09-18T13:24:49-04:00`. Earlier monitor shells also used this pair for failed or
timed-out checks. A newer `--mon-2` started at 13:39:26, after the screenshot, so
current live status is not a stable regression fixture.

`TESTED` is an authored stop label, not an assertion that verification passed. The
failure bucket is correct. The defect is that the clan loses the member's more
informative display label and monitor presentation metadata.

The decisive path is:

1. Family normalization already mirrors the current monitor's label, effective bucket,
   and presentation fields onto its family row.
2. `project_clan_tree()` in `src/sase/ace/tui/models/_agent_tree.py` calls
   `apply_clan_container_status()` in `src/sase/ace/tui/models/_agent_clan.py`.
3. `aggregate_agent_group_effective_status()` correctly maps `TESTED` with an explicit
   `Failed` bucket to canonical `FAILED` for aggregate precedence.
4. The clan helper restores a member's refined label only when the aggregate is
   `RUNNING` and exactly one member is in `Running`. A failed monitor therefore falls
   through to `FAILED` and its presentation fields are cleared.
5. Both runner-capacity refresh paths call this same helper, so changing only initial
   tree construction would not be sufficient.

A read-only reproduction using `project_clan_tree()` with a `TESTED`/`Failed` member,
one waiter, and nine completed members produced:

```text
member: label=TESTED, bucket=Failed, monitor_state=failed
clan:   label=FAILED, bucket=Failed, monitor_state=None
counts: failed=1, waiting=1, done=9
```

The running-only restriction originated in commit `21cdb658b0`, whose existing tests
cover lone running-member inheritance but not custom failed stop labels.

## Scope and implementation

This is one bounded presentation fix, suitable for one follow-up coding agent. Keep
lifecycle outcomes, family-shell selection, shared aggregate precedence, and persisted
records unchanged. The extension belongs in the existing TUI clan projection: it refines
a label after the canonical bucket is chosen. It requires no new Rust domain rule or
wire field. If implementation reveals a need to change shared backend semantics, follow
the Rust-core boundary rather than adding a Python backend rule.

1. Extend `apply_clan_container_status()` while retaining its identity deduplication and
   canonical effective-status aggregation.
   - Identify direct members outside the `Queued`, `Waiting`, and `Done` buckets. Queued
     rows are waiting work and must not prevent a lone failed or running member from
     supplying the label.
   - Inherit from exactly one such member when its effective bucket is `Running`,
     `Failed`, or `Stopped` and equals the canonical aggregate's bucket. Count
     `Starting` as a competing member but retain the existing lone-`STARTING` result of
     `RUNNING`.
   - Copy the source's exact label, explicit effective bucket, and every field in
     `_SHELL_STATUS_PRESENTATION_FIELDS`. Use the existing copy helper; include its gate
     execution/finalizer fields as well as monitor/gate labels, states, and accent.
   - For zero or multiple eligible members, a bucket mismatch, or empty input, preserve
     existing canonical aggregation/fallback behavior and clear all inherited fields.
     Repeated projection must not retain a previous source's metadata.
   - Do not select a nested historical shell directly. The normalized direct family row
     is the source, and each family continues to count once.

2. Keep tree construction and both runner-slot refresh paths routed through this helper.
   Reuse existing rendering: the inherited monitor state should give `TESTED` the same
   red failure styling as the member. Preserve the existing failed-monitor glyph
   suppression and timeout/lost glyph behavior.

3. Update the helper docstring and the clan-status paragraph in `docs/ace.md`. Explain
   the sole-member rule and the distinction between an authored stop label and its
   outcome bucket. In the reported case the result is `TESTED [W1 F1 D9]` in the Failed
   group, with no change to counts or filtering.

## Regression coverage

Extend `tests/ace/tui/models/test_agent_tree_clan_status.py` and
`tests/ace/tui/test_agent_runner_slots_families.py`, reusing their fixtures. Add a
focused helper test in `tests/test_agent_clan.py` only where needed to exercise in-place
reprojection or duplicate input identities.

- Reproduce the screenshot with one direct family row carrying `TESTED`/`Failed`, one
  waiter, and nine done members. Assert label, effective bucket, monitor pair/state,
  matching rendered style, and `W1 F1 D9` counts. Cover failed, timed-out, and lost
  monitor states without treating any as success.
- Include a family-level fixture with older failed shells and a current shell so
  normalization, direct-member projection, and one-per-family counting are tested
  together. A later running shell must still replace the earlier stop label.
- Cover a custom failed gate label and its presentation metadata; retain existing lone
  running monitor and gate cases. Cover a sole input-paused member so the `Stopped`
  extension preserves its label and effective bucket.
- Confirm a queued or waiting companion does not suppress the lone failure label;
  duplicate identities do not create an artificial second member.
- Confirm multiple nonwaiting, nondone members retain canonical aggregation: two
  failures; failed plus running; and the existing question/plan-over-running precedence
  cases. Preserve lone `STARTING`, all-done custom `TESTED`/`Done`, all-waiting/queued,
  and empty-input behavior.
- Exercise transitions on the same container: `TESTING` to failed `TESTED`, then to a
  later plain running member or all done; adding a second relevant member must restore
  aggregate labeling and clear inherited metadata.
- Verify the failed custom label and bucket survive both `refresh_runner_slot_context()`
  without a capacity limit and the configured capacity-snapshot path, including repeated
  refreshes. Assert grouping and summary counts still classify the failed row by its
  explicit bucket.

Keep the tests focused on externally observable projection, rendering, and refresh
behavior. No new live-state mutation or screenshot harness is needed.

## Verification and completion

The planning environment's existing virtualenv could run the pure projection
reproduction, but full row rendering encountered a missing editable `sase_core_rs`
extension. Before implementation validation, use the documented `just install` setup if
that dependency remains unavailable; do not work around it with a Python fallback.

Run the targeted model, refresh, clan-count, and family-monitor tests, then format the
changed files and run `just check` as required by `lint_and_test.md`. Use
`/sase_monitor` if a verification command becomes long-running; run `just fix` or at
least `just fmt` before handing verification to a monitor. Full verification is only
needed under the repository's normal escalation/landing rules.

Completion requires the screenshot-shaped fixture to display `TESTED` with `Failed`
bucketing and unchanged counts, the refresh regressions to pass, and existing
running-label and precedence tests to remain green. No implementation changes were made
during planning. A long-running TUI must restart to import the eventual fix; live agent
progress after the screenshot is not evidence that the historical reproduction has been
corrected.
