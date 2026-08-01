---
tier: tale
title: Show QUEUED for queued-and-waiting agent clans
goal: Agent clans with queued work and no higher-priority member state display QUEUED
  while retaining distinct queued and waiting counts.
create_time: 2026-07-26 07:52:23
status: done
---

- **PROMPT:** [prompts/202607/queued_clan_status.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/queued_clan_status.md)
- **AGENTS:**
  - [bbugyi200.athena.ld](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.ld/README.md)
  - [bbugyi200.athena.ld--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ld.md#member-code)
- **COMMITS:**
  - [30f3a22](https://github.com/sase-org/sase/commit/30f3a22c86d445f7be5560bc7e9a966286c1bd60) — fix(ace): prioritize queued clan status

# Plan: Show QUEUED for queued-and-waiting agent clans

## Goal

When an agent clan has at least one member whose derived display status is `QUEUED` and no member in a higher-priority
live/error/input state, show the clan's aggregate status as `QUEUED` even when other members are `WAITING`. This makes a
clan such as `[Q3 W6]` read as queued in the Agents row, status grouping, and detail panel while preserving the separate
queued and waiting counts.

## Context and semantics

`QUEUED` is a reversible, in-memory display status derived after runner-slot analysis; persisted agent records and the
Rust scan wire intentionally remain `WAITING`. The original queued-status implementation made `WAITING` outrank `QUEUED`
in `aggregate_clan_status`, which produces the reported mismatch after member promotion: the clan counts show queued
work, but its aggregate row remains in the Waiting bucket.

Change only the aggregate priority:

- `QUESTION` / `WAITING INPUT`, pending plan review, failure/killed, and running/starting keep their existing
  precedence.
- If at least one remaining member is `QUEUED`, return `QUEUED` before considering `WAITING`.
- A clan containing only `WAITING` members remains `WAITING`.
- Completed members remain neutral when live work exists, preserving the existing `QUEUED` plus `DONE` behavior; a
  queued/waiting clan does not flip back to `WAITING` merely because an earlier member is done.
- An all-done clan remains `DONE`, and empty/unknown fallbacks remain unchanged.

The rule applies to the shared aggregate policy already used for clan projection, runner-slot refresh, wait/completion
targets, tribe summaries, cleanup previews, and legacy parallel-family compatibility. It does not merge the `Q` and `W`
counts or relabel an individual explicit/dependency/time waiter as queued.

## Implementation

1. Put the pure agent-group aggregate ladder beside `status_bucket_for_values` in `src/sase/agent/status_buckets.py`,
   with the queued-before-waiting rule above. Keep `src/sase/ace/tui/models/_agent_clan.py`'s `aggregate_clan_status`
   entry point as a thin compatibility delegate so existing TUI, completion, cleanup, tribe, and parallel-family callers
   continue to use one policy without import churn.
2. Replace the duplicate aggregate ladder in `src/sase/integrations/_editor_helper_agents.py` with the same shared
   helper. This keeps the editor clan catalog consistent whenever its snapshot contains derived `QUEUED` members and
   prevents the two priority ladders from drifting again.
3. Rely on the existing `QUEUED` bucket, row color, detail styling, and grouping machinery. Do not add filesystem work,
   asynchronous work, another member traversal, scan-wire fields, or persisted status changes. Update the concise clan
   status documentation in `docs/ace.md` so the aggregate rule and the independent `[Qn Wn]` counts are explicit.

## Tests

- Add a table-driven aggregate-policy test covering empty input, all waiting, all queued, queued plus waiting, queued
  plus waiting plus done, queued plus done, waiting plus done, all done, and every higher-priority override. Keep a
  compatibility assertion through `aggregate_clan_status` so the TUI-facing delegate cannot diverge.
- Extend runner-slot projection coverage with a clan whose members are promoted to one `QUEUED` and one explicit
  `WAITING` on the first refresh; assert the synthetic container becomes `QUEUED`, the member statuses remain distinct,
  and repeated refreshes are idempotent.
- Update clan detail, tribe-summary, wait-target/completion, legacy parallel-family, and editor-catalog assertions that
  exercise mixed queued/waiting aggregates. Confirm the displayed status/bucket becomes `QUEUED` while the status chip
  remains `[Qn Wn]`.
- Update the existing queued-clan visual test expectations and regenerate only its affected PNG golden. Inspect the
  rendered image to verify the clan row and detail header use the existing cornflower `QUEUED` treatment, the member
  rows retain their individual queued/waiting treatments, the counts remain visible, and by-status/attention placement
  follows the new aggregate bucket.

## Verification

Run the focused status-bucket, clan, runner-slot, clan/tribe detail, completion/wait-target, editor-catalog,
parallel-family, and queued-clan visual tests. Regenerate the intentional queued-clan PNG with
`--sase-update-visual-snapshots`, rerun that visual test without the update flag, and inspect the golden/diff artifacts.
Then run `just install` followed by the repository-required `just check`. Review the final diff and both repository
statuses to confirm no `sase-core`, memory, generated provider-instruction, or unrelated snapshot files changed.

## Risks and safeguards

- Moving the aggregate ladder can affect every group consumer at once. The precedence matrix and focused downstream
  assertions make the intended propagation explicit and protect question, plan-review, failure, running, waiting-only,
  and done-only behavior.
- A simple global swap is correct only because terminal `DONE` members are neutral once live work exists. Tests must pin
  mixed queued/waiting/done behavior so the user-visible rule is not accidentally narrowed to one exact status set or
  broadened over higher-priority states.
- The aggregate status and member counts answer different questions. Keep counts derived from concrete member buckets so
  `QUEUED [Q3 W6]` remains truthful instead of hiding the six authored/dependency waiters.
