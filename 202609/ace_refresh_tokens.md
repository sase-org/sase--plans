---
tier: tale
title: Gate ACE refresh surfaces with stat-only change tokens
goal:
  ACE auto-refresh skips unchanged agents, Axe, notification, Patch, and proc-store
  reads on watcherless and watcher-active hosts while retaining bounded sanity
  reconciles, overlap protection, and an immediate sunset-flag fallback to the legacy
  unconditional path.
size: medium
proposed_by: bbugyi200.apollo.sase-wn.5
bead: sase-wn.5
---

- **PARENT:** [202609/sase_idle_cpu_diet.md](sase_idle_cpu_diet.md)
- **BEAD:**
  [sase-wn.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-wn/sase-wn.5.md)
- **AGENTS:**
  - [bbugyi200.apollo.sase-wn.5](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.apollo.sase-wn.5.md)
- **COMMITS:**
  - [2eb1335](https://github.com/sase-org/sase/commit/2eb13350f991a84b340f4d6619334b9311bd7f9c)
    — feat(ace): gate auto-refresh on per-surface change tokens

# Plan: Gate ACE Refresh Surfaces With Stat-Only Change Tokens

## Context and invariants

The current ACE refresh spine already coalesces timer ticks, runs slow work from a
pump-free task, honors watcher dirty flags on Linux, and performs a hard-coded 60-second
full sanity refresh. Its remaining expensive fallback is
`not watcher_active => refresh every surface`: this is permanent on macOS and is also
costly when Linux filesystem churn repeatedly dirties broad surfaces. `ProcObserver`
similarly reparses the complete durable proc store at 2 Hz before suppressing only an
unchanged delivery.

Keep the public refresh cadence, watcher event handling, tab/activity gates, selected
live-file refresh, loader results, and subprocess behavior unchanged. Token probes must
perform metadata-only filesystem work, run off the Textual event loop, fail open on an
unreadable or indeterminate input, and never suppress a surface beyond the sanity
interval. A surface's accepted baseline is the token associated with its last
successfully completed load, not merely its last probe.

## Implementation

1. Create the required default-on sunset flag with
   `sase flag new ace_refresh_tokens -k sunset`. Author the generated flag bead so the
   enabled state uses token-gated ACE/proc refreshes, the disabled state preserves
   today's watcher-aware but watcherless-unconditional behavior, and removal requires a
   soak on both Linux and macOS that meets the idle CPU and freshness targets. Paste
   only the command-generated registry scaffold into
   `src/sase/feature_flags/registry.py`; do not hand-create any other bead. Add the
   normal `current_flags().enabled(...)` consumer at the refresh boundary and test both
   flag states.

2. Add a focused ACE surface-token module under
   `src/sase/ace/tui/actions/event_refresh/`. Represent file metadata with stable
   `(path, exists, mtime_ns, size)`-style values and construct one immutable token per
   surface without opening file contents:
   - Agents: the projects-root membership plus each project's `artifacts` root and
     `.ace_refresh_pulse`, which the agent/shell/monitor settlement paths already touch.
   - Axe: Axe/lumberjack membership plus the bounded per-lumberjack status, metrics,
     timestamp, and log metadata needed to notice status/run changes without walking
     immutable chop history.
   - Notifications: the canonical `notifications.jsonl` metadata.
   - Patches: project membership and direct ProjectSpec/archive metadata plus the
     effective bead store's compact projection/manifest metadata, so project-file and
     bead mutations invalidate the Patch surface without parsing either store.
   - Procs: the canonical `procs.jsonl` metadata, owned by `ProcObserver` rather than
     the ACE auto-refresh mixin.

   Treat a genuinely absent path as a stable token component, include parent/membership
   metadata so later creation is observable, and return an explicit indeterminate result
   for permission/stat/scandir failures so callers refresh rather than assume clean.
   Keep directory enumeration shallow and bounded by projects/lumberjacks; never recurse
   through agent or chop archives.

3. Extend ACE initialization with the per-surface last-completed token state and replace
   `_should_refresh()` in `_auto_refresh.py` with a decision that combines: the sunset
   flag, sanity due, watcher dirty state, and token drift. Collect all current surface
   tokens in one `asyncio.to_thread()` call from the existing pump-free refresh body.
   With the flag enabled, token drift must work whether or not the event watcher is
   active; watcher events continue to set precise dirty flags and enqueue exact agent
   deltas. With the flag disabled, preserve the current unconditional watcherless branch
   exactly.

   Advance a surface baseline only after its corresponding refresh succeeds. Preserve
   dirty state and the old baseline when a load is deferred by navigation/input, off-tab
   gating, debounce, or an in-flight worker. A successfully consumed exact watcher agent
   delta may accept the probed agent token only when there is no fallback reason; broad
   and sanity loads accept it on completion. Keep the existing scheduled/running/pending
   guards for agents, Axe, and Patches so token churn cannot create overlapping heavy
   workers, and continue refreshing the selected live agent file on otherwise-clean
   Agents ticks.

4. Make the safety reconcile interval configurable on `sase ace` with an alphabetized
   public `-s/--sanity-refresh-interval` option, defaulting to 300 seconds and rejecting
   non-positive values so missed changes remain bounded. Thread it through the ACE
   handler, `AceApp`, and state initialization instead of relying on the current
   60-second module constant; update parser/help and constructor tests. Sanity-due
   refreshes bypass tokens, dirty flags, tab deferral, and debounce just as the current
   safety floor does, and update the sanity timestamp only after the full pass.

5. Gate `ProcObserver`'s expensive `read_procs()` parse with the proc-store token while
   retaining its 0.5-second stat cadence. Cache decoded store rows for an unchanged
   token, then continue composing pending placeholders, watches, completion delivery,
   liveness, and selected-detail log tails from those cached rows so UI semantics do not
   go stale. `request_poll()` must force the next full read, a changed or indeterminate
   token must read immediately, and the sunset flag's disabled state must parse on every
   poll as today. Add a bounded sanity reread so same-signature/coarse filesystem
   anomalies cannot persist indefinitely.

## Tests and evidence

- Add pure token tests using temporary project/Axe/notification/bead/proc trees. Assert
  stable tokens for unchanged metadata, isolation when each surface changes, detection
  of creation/removal and refresh-pulse updates, shallow/non-recursive behavior, and
  fail-open results for probe errors.
- Extend the event-refresh tests for watcherless clean ticks, one-surface token drift,
  watcher-plus-token cooperation, first-load baselining, off-tab/debounced baseline
  retention, exact agent deltas, sanity expiry, token-probe failure, overlap coalescing,
  and enabled/disabled flag behavior. Update test doubles to expose explicit token
  snapshots rather than touching the developer's live SASE stores.
- Extend proc-observer tests to count `read_procs()` calls across unchanged, changed,
  forced, sanity-due, and stat-error polls; retain tests for pending/submitted procs,
  exactly-once completions, session liveness, and growing selected logs under cached
  store rows.
- Run the targeted ACE refresh, proc-observer, parser, and feature-flag suites, then run
  `just check` as the required whole-repository verification. If the scoped lane
  escalates to full verification, use the SASE monitor workflow rather than blocking
  inline.
- Capture before/after idle ACE process CPU, RSS trajectory, surface reload counts, and
  stall-watchdog records during a 30-minute watcherless session; target less than 10% of
  one core and approximately zero idle stalls without freshness loss. Record the
  commands and measurements in the phase bead notes. If RSS still grows after reloads
  stop, append a `PROPOSED FOLLOW-UP:` note to `sase-wn.5` with the evidence rather than
  creating a task bead.
- Before closure, run `sase bead epic-symbols sase-wn.5`; resolve every remaining symbol
  or re-key its Justfile entry to the parent epic or a still-open later phase. Close
  only `sase-wn.5` with
  `sase bead close sase-wn.5 --note "<verified tests and performance evidence>"`; do not
  close `sase-wn` or any ancestor.
