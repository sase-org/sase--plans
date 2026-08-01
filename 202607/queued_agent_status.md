---
tier: tale
title: Give runner-slot waiters a dedicated QUEUED agent status
goal: 'Agents holding only for a free runner slot under the global cap display a dedicated
  QUEUED status with its own bucket, glyph, and color, exactly matching the Q count
  already shown in the Agents header and status chips.

  '
create_time: 2026-07-25 11:45:03
status: done
---

- **PROMPT:** [prompts/202607/queued_agent_status.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/queued_agent_status.md)
- **AGENTS:**
  - [bbugyi200.athena.ku](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.ku/README.md)
  - [bbugyi200.athena.ku--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ku.md#member-code)
- **COMMITS:**
  - [899a257](https://github.com/sase-org/sase/commit/899a257f22b0a36225485f8c81faaf72cef4fdf9) — feat: surface runner-cap queued agent status

# Plan: A dedicated QUEUED agent status for runner-slot waiters

## Context

ACE already counts queued agents separately from waiting ones. The Agents header reads
`71 [10/10 running · 1 queued · 19 waiting · 41 done]`, tribe and clan chips render `[Q1 W1]`, and
`RunnerCapacitySnapshot.global_cap_queue_count` feeds the runner-limit indicator. All of that is derived from
`agent_is_globally_queued`.

The row itself contradicts those counts. A queued agent still renders `WAITING ▶10/10` in amethyst, identical in kind to
an agent blocked on a dependency, a bead, or a time floor. The two are operationally opposite: a dependency wait may
never resolve and can warrant intervention, while a queued agent is fully unblocked and will start on its own the moment
capacity frees. Reading the list, there is no way to tell them apart without decoding a dim glyph suffix.

Introduce `QUEUED` as a first-class display status so the row agrees with the counts that already exist.

## Semantics

**`QUEUED` means: this agent has cleared every dependency, bead, and time wait, and is holding only for a free runner
slot under the global `max_running_agents` cap.**

The membership rule is exactly today's `agent_is_globally_queued` — a live, slot-participating row whose `waiting.json`
carries `slot_requested_at` with `wait_runners_explicit` false. Nothing about which agents qualify changes; only how
they are labeled, bucketed, colored, and laid out.

Treat this as a hard invariant and cover it with a test: **the set of rows displaying `QUEUED` is identical to the `Q`
count in every chip, to `N queued` in the Agents header, and to `RunnerCapacitySnapshot.global_cap_queue_count`.** The
counting code must stop calling the queued predicate independently and derive `queued` from the status bucket instead,
so the invariant is structural rather than a coincidence maintained in several places.

Agents parked by an authored `%wait(runners=N)` threshold keep `WAITING`. They are held by a user-declared condition
rather than by ambient global capacity, their `▶N→M` suffix is already row-specific and meaningful, and folding them
into `QUEUED` would break the parity with the `Q` count that motivates this change. This is the one judgment call in the
design worth confirming; if it should go the other way, both the status rule and the `Q` count must move together.

`QUEUED` is derived, never persisted. Do not change `waiting.json`, the Rust `sase-core` agent-scan wire, or the
snapshot-level base status (`active_status_for_record` and the record-status helper in the integrations entry builder
keep returning `WAITING`). Promotion happens in memory at the single point where each surface already derives
runner-slot context after its merge:

- the TUI loader path, in the runner-slot context refresh that already attaches live counts and queue positions;
- the presentation-neutral integrations path, in the helper that attaches the same context to agent list entries.

Both surfaces must apply the same rule, and the WAITING↔QUEUED transition itself belongs in one shared helper next to
the other status-bucket policy so the two implementations cannot drift.

The pass runs on every full and selective refresh, so it must be idempotent and reversible: promote a qualifying
`WAITING` row to `QUEUED`, and demote a `QUEUED` row back to `WAITING` when it no longer qualifies. Demotion is a real
path — the wait modal can attach an explicit runners threshold to a parked agent, which must return the row to `WAITING`
on the next refresh. The structural waiter predicate that gates the queue must therefore accept both `WAITING` and
`QUEUED` so a second pass over already-promoted rows classifies and orders them identically.

## Bucket and grouping

Add a `Queued` status bucket ordered between `Running` and `Waiting`, matching the order the header already uses. Queued
rows are closer to running than blocked ones, and `by status` grouping should read the same way the header does.

Give the bucket the `…` banner glyph. It reads as "pending, not yet started", and unlike most alternatives it exists in
the Fira Code build pinned by the visual snapshot suite. (Note for context, not for action: the existing `⏳` Waiting
and `✗` Failed glyphs are _absent_ from that font and render as tofu in PNG goldens. Real terminals fall back to a
system font, so this is a snapshot-only artifact and is out of scope here — do not fix it in this change.)

Bucketing queued rows separately also drops them from the tribe panel's NEEDS ATTENTION list, which currently selects
`Failed`, `Stopped`, and `Waiting`. That is the correct outcome and should be kept deliberately: a queued agent needs no
attention, and today it pollutes the triage list with a stale "waiting for `<name>`" line describing a dependency it has
already satisfied.

For clan and family aggregation, rank `Queued` between the running and waiting ranks, and extend the aggregate-status
ladder so `WAITING` still outranks `QUEUED` — a unit with any genuinely blocked member reads as `WAITING`, while a unit
whose only non-terminal members are queued reads as `QUEUED`.

## Color

Use `#5F87FF` (cornflower) for `QUEUED`. It is unused in the codebase today, sits around 5.3:1 contrast on the ACE
background, and reads as cool and patient — clearly apart from `RUNNING` gold `#FFD700`, `WAITING` amethyst `#AF87FF`,
`STARTING` sky `#87D7FF`, `ANSWERED` azure `#5FD7FF`, `STOPPED` `#8787AF`, and the pink/orchid plan-review family. It
stays in the blue-violet neighborhood as `WAITING`, so the two read as related states, while the hue and weight
separation keeps them unmistakable at a glance.

Apply the same color to the status label, the `Queued` bucket banner glyph, and the `Q` count chip. The chip is
currently `#FF87D7`, which collides with both the workflow-step pink and the `WAITING INPUT` pink; retargeting it
removes that collision and makes the chip, the header count, and the row agree on one color. Verify the chosen value
against the PNG goldens rather than by reasoning alone, and adjust within the same cool-blue family if it renders too
close to a neighbor.

## Row and detail layout

Render a queued row as `(QUEUED #3/12)`: the label bold in the queued color, the queue position in the same color
without bold, so the label leads and the position reads as metadata.

Drop the `▶{in_use}/{cap}` suffix on queued rows. It is byte-identical on every queued row in the list and the same
numbers already lead the Agents header as `10/10 running`, so repeating it per row is noise. Keep
`▶{in_use}→{threshold}` on explicit-threshold `WAITING` rows, where the threshold genuinely varies per row.

Also suppress the dependency, bead, time-remaining, `until`, and missing-wait-target `?` suffixes on queued rows. A
queued agent has satisfied those waits by definition, but the marker retains the original `waiting_for` names, so
rendering them would assert a blocker that no longer exists. A queued row shows its label and its position, nothing
else.

The queue position needs a fix to be meaningful. Both surfaces currently populate `runner_slot_queue_position` and
`runner_slot_queue_size` only for waiters whose threshold is _already_ satisfied by the live running count. When the
pool is full — precisely when rows are queued — no row qualifies and both fields are empty. Drop that eligibility filter
so rank and size cover every live slot waiter in true admission order: `wait_priority` first, then `slot_requested_at`
FIFO, then the existing tiebreakers. This matches the order the admission gate actually uses, is never empty while
anything is queued, and is a small change applied identically in both surfaces. Update the field docstrings, the CLI
JSON documentation, and the `sase_agents_status` skill description to match the new meaning, and relabel the detail
panel's `eligible #N of M` to `queue #N of M`.

In the agent detail and prompt metadata panels, present the queued state in the queued color and lead with the position
and the time since the slot was requested; the cap and threshold detail stay available but should no longer be the first
thing read.

## Consumer parity

`WAITING` is enumerated in roughly two dozen frozensets, predicates, and style maps. Every one of them must be audited,
because a missed site silently strips behavior from queued agents rather than failing loudly. Search the tree for the
literal and for the `"Waiting"` bucket name, and resolve each hit explicitly. The known clusters:

- Liveness and capability sets: auto-approve eligibility, the agent detail and zoom-panel active-status sets, the
  live-diff source statuses, the mobile killable-status set, the slow-tool-call active predicate, and the footer
  keybinding availability checks that gate on `STARTING`/`WAITING`/`RUNNING`.
- Aggregation: the clan aggregate-status ladder and member sort priority, the summary and lane status counters, the
  editor-helper catalog aggregate, the tribe attention selector, and the agent ordering rules that keep queued children
  grouped.
- Presentation maps that switch on a status string: the agent list row renderer, the neighbor modal, the jump-all modal,
  the revive and saved-group revival renderers, the completion-model status style, and the status-bucket glyph table.
- Filters and actions: the custom cleanup modal's `waiting` status filter, and the wait modal's status gate — pressing
  `w` on a queued row must still open the modal and apply a runners-only edit live.

Ambiguous cases should follow one rule: anywhere `WAITING` meant "this agent is alive and pre-run", `QUEUED` belongs
alongside it. Only where `WAITING` specifically meant "blocked on a named dependency or a clock" should it stay alone.

The Agents query language needs no new syntax — `status:` is a case-insensitive substring match, so `status:queued`
starts working and `status:waiting` correctly stops matching queued rows. Confirm that behavior with a test and mention
the new value in query help and documentation.

## Verification

- Unit-test the promotion pass directly: promotion, non-promotion of explicit-threshold waiters, demotion when a row
  stops qualifying, and idempotence across repeated refreshes over the same rows.
- Add the invariant test tying displayed `QUEUED` rows to the `Q` chip count, the header count, and the capacity
  snapshot's queue count, so the two can never drift again.
- Cover the queue-rank change: with the pool full and several waiters parked, every queued row has a distinct
  `#rank/size` in priority-then-FIFO order, and the ranks agree between the TUI model and the integrations projection.
- Update the existing tests that assert `status == "WAITING"` for fixtures that are in fact globally queued — the loader
  dedup, apply-boundary, runner-slot, summary-count, and agents-snapshot suites all contain such fixtures. Keep
  assertions that cover dependency, bead, and time waits on `WAITING`.
- Extend the runner-slot visual fixtures so the goldens show a queued row, an explicit-threshold waiting row, and a
  dependency-blocked waiting row side by side, plus a `by status` frame showing the new bucket banner next to the
  existing ones. Regenerate the affected PNG goldens in the same change and inspect them; the point of this feature is
  how it looks, so the goldens are the acceptance evidence, not a formality.
- Update the documentation that describes the current behavior: the ACE active-status table, the Waiting-bucket and
  row-glyph sections, the runner-slot troubleshooting page (including its `[R/L · Q queued]` explanation), the
  integrations bucket table, the configuration reference's runner-slot wording, the xprompt notes that describe a parked
  launch as a `WAITING` row, and the `sase_agents_status` skill description.
- Run the focused agent-status, runner-slot, loader, summary-count, and query suites, then `just install` followed by
  `just check`, and `just test-visual` for the goldens.

## Risks and constraints

- Keep the promotion pure and in-memory. It runs inside the existing post-merge context refresh on the Agents-tab load
  path; it must add no filesystem work, no awaits, and no second pass over rows, per the TUI performance contract.
- Do not push `QUEUED` into `waiting.json`, the Rust scan wire, or the artifact index. The scan layer has no cross-agent
  view and cannot decide queue membership; keeping the status derived means old artifacts and a stale binding continue
  to work unchanged.
- Retargeting the `Q` chip color is a deliberate, visible change beyond the row itself. It is what makes the chip, the
  header, and the row read as one system; call it out in the change description so it is reviewed rather than assumed.
- The queue-position redefinition changes a field already exposed through `sase agent list --json` and the integrations
  projection. The field names and types are unchanged and the new meaning is a superset of the old, but the docstrings,
  docs, and skill description must be updated together so no consumer is left reading the old contract.
