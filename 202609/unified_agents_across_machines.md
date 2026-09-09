---
tier: epic
title: One Agents experience across machines
goal: "ACE presents one trustworthy Agents list across the local machine and every
  enrolled machine — with durable fleet-wide attention, explicit launch targets, and a
  Machines home — built on repaired Rust-owned fleet contracts and proven by live
  Athena-to-Apollo acceptance.

  "
parent_bead: sase-xe.16.11
phases:
  - id: fleet-wire
    title: Rust fleet read contract - normalization, counts, freshness, continuation
    depends_on: []
    size: medium
    description:
      "fleet-wire: normalize real federation envelopes, own counts and freshness in
      sase_core, add per-host continuation types, and export the bindings."
  - id: attention-wire
    title: Fleet-wide pending-attention inventory contract
    depends_on: []
    size: medium
    description:
      "attention-wire: add an owner-side bounded pending-attention inventory with
      continuation and coverage so every authorized decision is discoverable."
  - id: trust-fencing
    title: Worker TLS trust, per-host continuation, and real fault proofs
    depends_on:
      - fleet-wire
    size: medium
    description:
      "trust-fencing: apply validated TLS trust in the federation worker, thread
      per-host cursors through fan-out, and prove healthy-host survival and
      exact-instance fencing on real paths."
  - id: launch-durability
    title: Durable launch settlement, reconciliation, and visibility
    depends_on: []
    size: medium
    description:
      "launch-durability: settle slow accepted launches durably, reconcile lost replies
      by operation key, surface newly launched rows, and return safe structured launch
      errors."
  - id: unified-projection
    title: Rebuild ACE's remote projection on published core contracts
    depends_on:
      - fleet-wire
      - attention-wire
      - trust-fencing
      - launch-durability
    size: medium
    description:
      "unified-projection: ratchet to the published core, replace Python normalization
      and counting with bindings, merge remote rows into the pure shared pipeline, and
      add the machine query field and grouping."
  - id: one-list
    title: One Agents list across machines
    depends_on:
      - unified-projection
    size: medium
    description:
      "one-list: behind a new beta flag, replace Focus/Fleet with a single list carrying
      machine chips, machine grouping, scoped counts, honest staleness, and bounded
      paged history."
  - id: attention-inbox
    title: Durable cross-fleet attention inbox
    depends_on:
      - unified-projection
    size: medium
    description:
      "attention-inbox: surface every authorized pending decision in the durable
      notification inbox with a global needs-you count, snooze, dedupe, and
      already-resolved handling."
  - id: unified-actions
    title: Familiar operations with explicit ownership
    depends_on:
      - one-list
    size: medium
    description:
      "unified-actions: give local and remote rows the same verbs with bound keys,
      owner-explicit stop/fork/retry semantics, and per-target results for mixed-machine
      group actions."
  - id: launch-experience
    title: Launch target picker, source context, and outcome tracking
    depends_on:
      - one-list
    size: medium
    description:
      "launch-experience: add a target selector synchronized with %dispatch, show
      resolved source context and unmet prerequisites before submission, and track
      launches through a provisional row with check-outcome reconciliation."
  - id: machines-home
    title: Admin Center Machines pane and persistent Connect flow
    depends_on:
      - unified-projection
    size: medium
    description:
      "machines-home: give machines one administration home with health, capacity, and
      repair actions, plus a resumable Connect flow replacing toast-only setup."
  - id: real-contract-proof
    title: Real wire fixtures, contract regression, and performance evidence
    depends_on:
      - one-list
      - attention-inbox
      - unified-actions
      - launch-experience
      - machines-home
    size: medium
    description:
      "real-contract-proof: replace synthetic fixtures with serialized real envelopes,
      drive ACE-built requests through actual worker and gateway paths, and re-prove
      visual and performance budgets."
  - id: cutover
    title: Flag removal, follow migration, and documentation cutover
    depends_on:
      - real-contract-proof
    size: medium
    description:
      "cutover: make the unified experience unconditional, delete the legacy Focus/Fleet
      branch, migrate follow data out of membership control, and update config, help,
      and docs."
  - id: live-acceptance
    title: Live Athena-to-Apollo acceptance of the unified experience
    depends_on:
      - cutover
    size: medium
    description:
      "live-acceptance: execute the research validation scenarios live between Athena
      and Apollo, record pane evidence, and capture the close evidence for the reopened
      parent phases."
proposed_by: bbugyi200.athena.0hv
bead_id: sase-xe.16.11.7
create_time: 2026-09-09 19:52:51
status: wip
---

- **PROMPT:**
  [prompts/202609/unified_agents_across_machines.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/unified_agents_across_machines.md)
- **BEAD:**
  [sase-xe.16.11.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.16.11.7.md)

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                              | Why                                                                                                                                  |
| ------------ | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| derives-from | [research:202609/agents_across_machines/agents_across_machines.md][1] | The consolidated UX research this epic implements.                                                                                   |
| related      | file:explicit:edfec327db7e0bf48738d238                                | Recovered uncommitted Rust fleet-contract diff from the failed sase-xe.16.11.6.1 agent; reusable prior art for the fleet-wire phase. |
| supersedes   | [plan:202609/remote_dispatch_contract_and_acceptance.md][2]           | Canceled sase-xe.16.11.6 plan whose repair scope this epic absorbs.                                                                  |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202609/agents_across_machines/agents_across_machines.md
[2]:
  https://github.com/sase-org/sase--plans/blob/main/202609/remote_dispatch_contract_and_acceptance.md

<!-- sase:links:end -->

# Plan: One Agents experience across machines

## Scope, lineage, and recovered work

This epic is the successor to canceled `sase-xe.16.11.6`. It carries two obligations at
once: implement **all** UX changes recommended by
`research:202609/agents_across_machines/agents_across_machines.md`, and absorb every
confirmed repair from `plan:202609/remote_dispatch_contract_and_acceptance.md` that was
lost when that epic was canceled. Read both with `sase artifact read` before
implementing any phase, along with
`research:202609/apollo_fleet_contract_repair/apollo_fleet_contract_repair.md` for the
live reproduction evidence. The parent epic is `sase-xe.16.11`; its reopened phases
`sase-xe.16.11.3` and `sase-xe.16.11.5` stay open until this epic supplies their missing
evidence.

Two pieces of the failed first attempt are recoverable and must not be re-invented from
scratch:

- SASE commit `8c8dfc3f6` ("feat(fleet): consume Rust federation counts", reverted by
  `3feb02311`) shows the intended Python consumption of Rust count aggregation,
  including binding-collection tool updates and its regression tests. Recover it from
  git history as prior art; it may only re-land once the core bindings it consumes are
  actually published.
- `file:explicit:edfec327db7e0bf48738d238` (read with `sase artifact read`) preserves
  the failed agent's uncommitted sase-core diff: `FleetFederationNormalizeRequestWire`,
  `FleetNormalizedHostWire`, `FleetCatalogContinuationWire`,
  `FleetEnvelopeDiagnosticWire`, `normalize_fleet_federation_response`,
  `count_focus_and_fleet_from_federation`, and two PyO3 exports. It is written against
  core v0.32.55 (`3baa689`); current master is ≥ v0.32.60. Rebase and adapt it to this
  plan's revised contract — do not apply it blindly, and drop its Focus/Fleet count
  split where this plan's scope model replaces it.

## Design vision (binding for every UI phase)

The product model: **your agents are one body of work; machines are where they run.**
Machine identity is an attribute, a filter, and an optional grouping — never a separate
place. Three commitments follow:

1. **One list.** The Agents tab shows local and remote agents through the same pure
   filter/fold/group pipeline. Projects and families remain the working structure.
   Focused/Unfocused and Focus/Fleet disappear as user-facing concepts.
2. **Attention is a contract.** Every authorized pending question or gate anywhere in
   the fleet is durably discoverable until its owner reports settlement. Browsing
   choices (queries, folds, pages, tabs, follows) never hide an unresolved obligation.
3. **Explicit consequences.** Every action names its target machine. Every launch shows
   what will run and where before submission. Four facts stay independent in
   presentation: agent lifecycle, connection health, observation age, and operation
   certainty. Stale data is never styled as a live guarantee.

Illustrative layout (not a literal keymap or style spec):

```text
Agents | Artifacts | AXE            here: athena   Machines !1   Needs you 2
Machine: All v   project:sase        3 running; mac unknown   Group: project

sase
  auth-fix       QUESTION       apollo    Approve policy
  docs-refresh   RUNNING        here      Update docs
  ci-watch       WAS RUNNING    mac       last seen 12m ago

Target: Here v   Prompt: ____________________________________________
```

Visual language: machine chips are short lowercase alias labels (`here` for local),
rendered in a dim secondary style, present on every row of a mixed list and suppressed
only under an unambiguous machine group header; details and operations always name the
owner. Status vocabulary leads each row at narrow widths — name, actionable status,
origin, and stale/unknown meaning come before model or intent at 82 columns. Healthy
rows carry no repeated "online · fresh" chrome. Glyphs and text carry meaning; color
only reinforces (monochrome-safe). Counts always state their scope ("3 running; mac
unknown", "showing 20 of 65 matches"), and endpoint errors coalesce into one incident
per machine/cause. With zero machines enrolled the compact local experience is visually
unchanged and a Machines/Connect entry stays discoverable.

Decisions settled by the research and its accepted predecessors — do not re-litigate in
phases: one unified list (no Local/Remote tabs, no per-machine tabs); a compact machine
control that edits existing query state (advanced expressions display as "Custom");
machine administration in Admin Center with only a small health shortcut in top chrome;
attention fetched for all authorized pending decisions independent of scope, pages, and
active tab; no new Watch axis now; capacity shown as observed data, with
`%dispatch:auto`, queue composition, and automatic placement deferred;
`%dispatch:<alias>` stays the portable spelling; owner authority, stable identities,
bounded reads, journaled recoverable operations, publish-only agents-sync, and
discovery-only-at-setup all remain binding from `plan:202609/remote_dispatch_fleet.md`.

## Constraints

- Open sase-core, chezmoi, research, or any other repository with `/sase_repo`; never
  assume sibling paths. Read plans and research with `sase artifact read` and reference
  memory with `/sase_memory_read`: `tui_perf.md` before any TUI phase, `sase_flags.md`
  in the flag-creating and cutover phases, `lint_and_test.md` before finishing any sase
  change, `symvision.md` for lint failures, `cli_rules.md` if a phase adds CLI surface,
  and `xprompts.md` before directive work.
- Shared domain behavior and pure policy belong in `sase-core`; keep `sase_core`
  transport-free and independent of the gateway crate. Python keeps platform I/O,
  Textual presentation, and thin adapters, with no fallback implementation of core
  behavior and no exception-swallowing fallbacks.
- Reuse existing operation keys, receipts, follow records/tombstones, locators,
  revisions, and request contracts. Do not create a competing launch journal, restart by
  minting a new operation key, or recreate the retired agents-sync import pipeline.
- Preserve installation pinning, verified HTTPS, exact-instance mutation fencing,
  bounded transport reads, provider import isolation, quarantine semantics, and durable
  operations for user-triggered work. No local-host fallback for a failed remote launch.
  Bootstrap bytes never enter argv, logs, prompts, captures, or notes.
- Preserve laziness and performance: zero enrolled machines start no federation worker,
  provider import, remote timer, or network request; hidden or unenrolled surfaces do no
  remote work; keystroke paths stay read-only and prompt-free; j/k p95 stays under 16 ms
  on every tab; selection is preserved by stable identity across every async return.
  Route refreshes through the existing fast path and selective-update APIs.
- Exactly one new feature flag is authorized: the `one-list` phase creates a `beta` flag
  with `sase flag new` gating the unified user-facing surface, and the `cutover` phase
  removes it by deleting the legacy branch. Both states need tests while the flag
  exists. Do not change other flags, relax test budgets, or add flake allowances.
- Release-plz owns core versions. Publish through normal host finalizers; never hand-pin
  an unpublished version. Core verification runs the complete `just check` or
  `scripts/check.sh` including PyO3, never only `cargo test -p sase_core`. Python phases
  run `just install` after ratcheting and verify the published wheel actually contains
  the required changes.
- Any changed named control updates `src/sase/default_config.yml`, the keybinding
  footer, and the help modal together (per the ACE conventions and the keymap gotcha).
- Phase workers never create beads; record outside-scope discoveries as
  `PROPOSED FOLLOW-UP:` notes on their own phase bead and leave genuinely unmet work
  open. A commit's automatic close is not evidence.

## fleet-wire — Rust fleet read contract

In `sase_core::fleet_contract`, make the shared read pipeline own what Python currently
improvises. Rebase the recovered diff (`file:explicit:edfec327db7e0bf48738d238`) onto
current master and finish it:

- **Envelope normalization.** Accept the real federation fan-out shape
  (`FederationReadResponseWire` host rows carrying `alias`, `status`, `cached`,
  `payload`, `error`) and normalize each distinct payload: catalog `payload.page.rows`,
  followed batch `payload.entries[].summary` including unresolved entries, and summary
  `payload.counts`/freshness. Strict validation with actionable per-envelope
  diagnostics; malformed envelopes degrade that host, never the response. The current
  `FleetHostCountInputWire` rejects federation rows outright (`deny_unknown_fields` on
  `alias`) — normalization is the ingestion path that fixes the "unknown field alias"
  failure for good.
- **Counts and scope.** Count aggregation consumes normalized hosts and authoritative
  `payload.counts`, independent of catalog page and search filters. Followed/local
  totals use existing logical-agent and family semantics; a batch host's snapshot-wide
  totals are never a followed total (the live defect projected a host-wide `running=9`
  onto one followed DONE row). Logical agents and occupied runner slots stay distinct.
  Missing, stale, or errored hosts are represented explicitly (the counts contract
  currently has no errored-host state) and never become authoritative zeroes.
- **Freshness.** `freshness.partial=True` and summary `observed_at_unix` must survive
  projection (both were dropped live). Unknown coverage propagates as unknown.
- **Continuation.** Add per-host continuation state to the request/result contract: each
  host retains its own opaque cursor bound to its snapshot generation
  (`StoreCursorWire.store_generation`), can independently finish, fail, restart, or
  invalidate, and an invalid cursor resyncs that host without losing healthy hosts'
  pages. No cursor is ever forwarded to another host.
- **Bindings.** Export narrow dict-in/dict-out PyO3 functions for request validation,
  normalization, freshness/diagnostics, and count aggregation, plus the currently
  missing validators Python needs (catalog query/cursor validation at minimum). Fix
  `sase_core_py`'s stale hardcoded `sase_gateway` version pin while touching the crate.
  Record the exported names and precise schema in the phase note so downstream phases
  consume a committed surface.

Tests validate the PyO3 envelopes with real serialized worker shapes (not the invented
`hosts[].summaries` fixtures) and prove another frontend would receive identical
decisions: DONE follows, duplicate family members, local/remote separation, unknown and
errored hosts, failed catalog beside successful summary, freshness and observation
propagation, unequal per-host cursors.

## attention-wire — fleet-wide pending-attention inventory

The current gateway attention route accepts a bounded list of already-known logical keys
and deliberately reads no store for an empty list, so removing client-side Follow checks
can never create fleet-wide discovery. Build the missing contract:

- An owner-side **pending-attention inventory**: a versioned request/response pair that
  enumerates all authorized pending questions and gates on that host — including
  decisions whose agents have never appeared in any viewer's loaded catalog — with
  bounded pages, continuation, and per-host coverage/freshness. An unavailable machine
  yields unknown coverage, not zero entries.
- Stable **request identity**: origin installation, request key, and revision travel
  with each entry (the current label-only correlation is too weak to deduplicate or to
  detect a changed approval preview). Settlement, snooze, and answer decisions key off
  this identity; a revision change invalidates a stale preview.
- Gateway route, federation worker fan-out op, and PyO3 bindings following the same
  fan-out, deadline, cache, and degradation model as other reads. Preserve the existing
  bounded-keys route for row-scoped hydration. Zero enrolled machines keep starting no
  worker and no timers.

Tests: empty-fleet laziness, an entry for an agent absent from every catalog page,
continuation across more entries than one page, coverage under a hung and an errored
host, dedupe identity stability across reconnect, and binding envelope validation.

## trust-fencing — worker TLS trust, per-host fan-out, real fault proofs

- `RemoteHost` currently builds a default reqwest client and silently ignores validated
  `plan.tls` (`PinnedCa`/`PinnedServerName` behave as `SystemRoots`). Resolve and apply
  `TlsTrustSettingsWire` at client construction: validate what `ca_ref` and
  `server_name_ref` mean before implementing them (reuse established reference
  resolution; do not guess that an opaque reference is a raw path or hostname), fail
  closed with useful diagnostics when trust material cannot be resolved, preserve
  default system roots for `SystemRoots`, and never fall back to a default client after
  a requested trust configuration fails.
- Provide an isolated loopback HTTPS fixture trusted through this production path,
  backed by a real gateway with authenticated hello and real read payloads. Prove:
  successful authenticated reads beside an actually reached hung host (the existing
  deadline test asserts the fast host is _not_ ok — that is failure isolation, not the
  required proof), bounded overall completion, per-host deadline diagnostics, recovery
  on subsequent requests, trust mismatch failure, and the existing fast-refusal case.
- Thread the fleet-wire per-host continuation contract through the worker fan-out:
  per-host queries and cursors in the IPC request, independent completion and resync,
  invalid cursors rejected without losing healthy pages, production contract tests with
  unequal host histories and more than three pages.
- Complete the exact-instance proof through the actual mutation route: capture a first
  instance's locator and revision, replace it under the same logical name reusing
  PID/name identity, submit the captured locator against the replacement, assert
  rejection plus zero side effects on the replacement, then prove the replacement's own
  locator acts — with an isolated controllable bridge recording side effects, not a stub
  of the decision under test.

## launch-durability — settlement, reconciliation, visibility

Trace the request across `src/sase/dispatch/launch.py`, federation IPC, the gateway
launch route, `FleetLaunchStore`, the host bridge, and mobile project resolution, with
bounded redacted timing diagnostics. Keep the shared short read deadline; a longer wait
is not recovery. Confirmed leads from source inspection:

- The gateway launch route synchronously awaits `agent_bridge.launch_text` between
  `reserve` and `settle`; there is no recovery path for a record stranded between the
  two, and `FleetReadService::reconcile` — the invalidation diff that would make a new
  row appear — has zero callers. Bound bridge execution, move blocking waits off the
  async runtime, and make slow accepted launches settle durably even when the controller
  disconnects or times out (recover stranded reservations instead of letting them
  silently expire).
- Repeating or reconciling the same operation key must retrieve the original receipt and
  bind the target's authoritative logical/exact identity without a second launch,
  covering: failure before launch, success with a lost reply, and
  controller/worker/gateway restart. Preserve acceptance-window rules. Keep the
  controller-side durable intent ledger (`unsent` → `acceptance_uncertain` →
  `accepted`/`settled`) as the single source the TUI will later project.
- Fix the demonstrated missing-new-row cause through owner snapshot acquisition,
  invalidation/cache keys, and receipt mapping; management must resolve an authoritative
  exact instance even when the row is outside the current page or the reply was lost —
  never kill by guessed name or PID.
- Expose safe structured launch errors with bounded redacted detail (the bridge
  currently collapses everything into an opaque `agent_bridge:launch-text`); never leak
  credential-bearing stderr or filesystem secrets.
- Verify portable project IDs resolve through the supported project registry (the
  canonical alias change in `_mobile_agent_context.py`), and make the live workflow
  durably create/restore the `apollo` machine alias in dispatch config — parent epic
  note #2 reproduced `machine status apollo` failing because `dispatch.machines` was
  empty. If journal growth (no tombstoner or compaction exists) needs more than a
  bounded touch here, record it as a `PROPOSED FOLLOW-UP:`.

Tests observe exactly one target launch with a delayed bridge and a lost reply: eventual
receipt, visible row, readable output, and an exact-instance stop.

## unified-projection — ACE on published core contracts

Wait for the four core phases to land and publish; ratchet the core revision and
dependency floor with the supported commands, run `just install`, and verify the
published wheel actually contains the new surfaces before consuming them. Then:

- Replace the Python normalization/count policy in
  `src/sase/ace/tui/models/_fleet_agents_payload.py`, `_fleet_agents_counts.py`, and
  `src/sase/dispatch/counts.py` callers with thin binding calls; delete the
  exception-swallowing row-count fallback. Recover the reverted `8c8dfc3f6` consumption
  work where it still fits. Preserve all real `ResolvedAgentSummaryWire` metadata —
  content describes content, never agent metadata.
- **Un-smuggle machine identity.** Remote rows currently spend the project field on the
  host alias (`project_display_name=alias`, `project_file="/fleet/<alias>/…"`). Give
  rows their true project from the wire and carry machine identity only in the dedicated
  fleet-origin fields, so "project sase across machines" is a real view.
- **Merge remote rows into the pure shared pipeline.** Owner-resolved remote projections
  enter before the shared filter/fold/group boundary (`_loading_apply.py` currently
  merges after it, and remote refresh finalizes on a second path). Remote records never
  pass through local PID checks, filesystem hydration, or cleanup side effects; mutating
  local actions keep treating them as read-only.
- **`machine:` query field.** Add `machine` as a substring property across the
  tokenizer, evaluator, pushdown, highlighting, seed helper, and help-modal syntax
  table, with `machine:here` matching local rows (decide and document the bare
  `machine:` form using the existing bare-`tribe:` carve-out as precedent). Add a
  `BY_MACHINE` grouping mode to the grouping cycle with label, fold registry,
  persistence, and a widened selective-update gate so machine grouping is not excluded
  from the row-removal fast path.
- Project the four-fact state model (lifecycle, connection health, observation age,
  operation certainty) into row state, and keep the render-cache key honest for every
  field that can change visible state, including where existing dataclass fields are
  `compare=False`.

This phase changes projection internals and additive query surface only; the legacy
Focus/Fleet presentation still renders until `one-list` supersedes it.

## one-list — one Agents list across machines

First act: create the beta flag with `sase flag new` (key such as `ace_unified_agents`),
gating everything user-visible in this phase; read `sase_flags.md` first. Flag off
preserves today's Focus/Fleet experience unchanged. Flag on:

- The Agents sub-tab strip disappears. One list renders local plus all enrolled origins'
  rows through the shared pipeline, defaulting to all machines: active work, pending
  decisions, unresolved operations, and the same bounded recent-completion policy for
  every origin. Older history is explicit and paged — "More results" and incomplete
  coverage are visible states, with no fixed page ceiling that makes later results
  unreachable.
- Machine chips on every row of a mixed list including `here`, suppressed only under an
  unambiguous `BY_MACHINE` group header. Zero machines enrolled keeps the compact local
  view (no chips) with a discoverable Connect entry.
- The header carries the identity and scope strip from the design vision: current
  machine, coalesced machine-incident shortcut, needs-you count, and honest scoped
  counts ("3 running; mac unknown") sourced from authoritative summary counts, never
  loaded pages. Off-scope attention appears as an explicit global count/link, never as
  rows silently violating the user's query.
- Honest staleness: "WAS RUNNING · last seen 12m ago" presentation, cached output
  retained, only currently-impossible operations disabled, detail shows observation time
  and the current shell of the selected logical agent.
- Selection, folds, and scroll survive refresh, family handoff, paging, and inbox
  round-trips by stable identity; new arrivals update counts without stealing focus.
  82-column priority order and monochrome rendering follow the design vision. Both flag
  states get tests; visual snapshots cover the new layout at 82×28 and wider.

## attention-inbox — durable cross-fleet attention

Unify today's two disconnected systems — the durable local notification inbox and the
toast-plus-dedupe-ledger remote attention path — on the inbox:

- Project fleet pending-attention inventory entries (from attention-wire, via the
  facade) into the durable notification store as typed entries keyed by origin, request
  identity, and revision, gaining tabs, badge counts, read/snooze/mute, and the existing
  modal surface. The owner in the header; reuse local answer/approval presentation
  through the existing remote-attention modal and durable submission path.
- Requests stay discoverable until the owner reports settlement: seeing, dismissing a
  toast, or reading never marks a request answered. Reconnect reconciles the inbox
  instead of replaying toasts. If another controller answered, show "Already resolved"
  and refresh; a changed revision requires renewed review and never applies a stale
  decision. Ownership and permissions come from the protocol, never Follow.
- Policy separation is explicit and tested: query/fold/dismiss changes browsing only;
  mute silences routine updates while decisions stay discoverable; snooze changes
  prominence until a chosen time with a visible snoozed count; answer/approve is an
  owner-directed request. Unfollow never implies muting.
- Run the lightweight attention service while ACE is open on every tab (refresh
  scheduling currently returns early outside Agents), keeping heavy catalog/detail
  hydration demand-driven, honoring tick cadence, coalescing, and zero-machine laziness.
  The needs-you count updates on Artifacts and AXE without focus theft.

## unified-actions — familiar operations, explicit ownership

- Local and remote rows expose the same visible verbs where supported — open
  output/chat/diff, answer, stop, retry, fork — routed to the owner with exact-instance
  preconditions underneath. Preserve familiar local bindings; the fleet-only actions
  that today default to `unbound` are either absorbed into the shared verbs or given
  deliberate bindings. Conditional actions appear in the footer per convention;
  unsupported actions carry a discoverable reason in details or the palette. Do not
  promise remote tmux/editor access or cross-host continuation.
- Retry and fork default to the selected agent's owner; a new unrelated prompt defaults
  to Here; selecting a remote row or filtering to a machine never changes the launch
  target. Terminal dismissal remains a viewer operation and can never become a remote
  stop. "Stop accepted" is distinct from "Stopped"; if no stop was sent because the host
  was unavailable, say so; uncertain delivery stays uncertain — no offline action queue
  that executes later without fresh intent.
- Mixed-machine group actions show an explicit target preview and per-target results:
  one unreachable machine never turns partial completion into generic success, and
  selection changes never retarget an operation already submitted.
- Update the footer keymap surface, help modal, and `default_config.yml` together.

## launch-experience — target, source, outcome

- Add a target selector beside the prompt, synchronized bidirectionally with a typed
  `%dispatch:<alias>` directive; duplicate or conflicting directives produce an inline
  error. The picker names the target with cached health and supported eligibility from
  the existing machine completion catalog; typing never triggers discovery or blocking
  network work.
- Surface source context before submission: the resolved project and revision/Patch
  evidence beside the target, with unmet prerequisites (dirty checkout, no upstream,
  unpublished HEAD, rejected local attachments) explained in place by mapping the
  existing validation chain's refusals. Preserve the draft after refusal; never silently
  drop attachments, substitute an older revision, publish changes, or launch locally.
- An immediate provisional row tracks the launch by projecting the durable intent ledger
  through the research's outcome vocabulary: not submitted; submission outcome unknown
  with a **Check outcome** action that reconciles the same operation key;
  accepted/queued retaining the pending row until the real row appears (launch-
  durability guarantees it will); rejected with the owner's reason and a preserved
  draft; then the owner's observed lifecycle. "Check outcome" never manufactures a new
  operation; "Retry agent" is visibly a new execution.

## machines-home — Admin Center Machines pane

- Register a Machines pane in the Admin Center catalog (modeled on the Projects
  inventory pane family): Here plus every enrolled origin with health, last successful
  observation, eligibility/capabilities, and capacity shown as observed usage and
  configured limit with age — logical running counts are not occupied slots, and stale
  headroom is not an admission promise. Actions: Connect, Check status, Repair, Rename,
  Remove, and Show agents (seeds a `machine:<alias>` query in the Agents list). Wire the
  pane through the standard session-state, keymap-section, footer, and help conventions.
- Connect becomes a persistent resumable flow replacing the twelve-second setup toast:
  explain target preparation, show copyable commands, select a protected bootstrap file
  (never pasted through argv or logs), review the target, activate, verify, and retain
  partial-success state with the appropriate recovery step. Enrollment stays reachable
  with zero remote agents.
- Removal copy explains it removes this controller's enrollment and does not stop work
  on the target. Repair respects changed installation identity — cached rows are never
  rebound to a different machine because an alias matches. The top-chrome health
  shortcut and contextual error links (offline row, launch picker) open this same pane.

## real-contract-proof — fixtures, regression, performance

- Replace the invented fixture schemas (`hosts[].summaries`, mapping liveness,
  content-as-metadata) across Fleet model/facade tests, visual snapshots, and fault
  benchmarks with serialized real wire shapes validated by the core contract. Correct
  bad assertions — counting displayed rows as running, goldens that encode the old
  membership rule — rather than preserving them.
- Drive requests built by the actual ACE refresh/continuation/attention code through a
  real federation worker and gateway on the trusted loopback fixtures (no production
  credentials), projecting their serialized responses: two unequal hosts, healthy beside
  failed and hung, active plus recent DONE, a followed DONE outside page one,
  independent per-host cursors, more than three pages, malformed requests, host
  recovery, counts independent of pages and filters, new launches with late receipts,
  and attention entries for never-cataloged agents. An unavailable fixture is an
  explicit unmet requirement, not a skipped test presented as success.
- Re-prove performance honestly: the three Fleet fault benchmarks with fault sequences
  overlapping measured navigation, healthy rows navigable, p95 under 16 ms, no stalls,
  idle ticks quiet, zero-machine and hidden-surface laziness intact. Keep test
  construction offline outside the isolated integration suite. Both flag states remain
  covered until cutover.

## cutover — unconditional, migrated, documented

- Remove the beta flag per the removal rule: delete the Off branch — the Focus/Fleet
  sub-tab strip, mode reactive and cycling actions, follow-based membership filtering,
  and the split count presentation — make the unified surface unconditional, remove the
  registry entry, and close the flag bead in the same change.
- Migrate follow data: preserve follow records and explicit tombstones as
  migration/rollback input, but nothing user-facing reads them for membership or
  attention anymore. Do not infer notification muting from an old unfollow. Show a
  one-time explanation of the change and retain existing explicit notification
  preferences. If a curated working set proves necessary later, it will extend the
  shared pin/query mechanism — record demand as `PROPOSED FOLLOW-UP:`, do not build it.
- Sweep the vocabulary: `default_config.yml` keymaps (remove obsolete actions, keep the
  new ones), footer, help modal, onboarding/setup copy, and `docs/remote_dispatch.md`
  all describe the unified experience. Run the Symvision gate over the deleted branch
  and re-key or resolve exemptions deliberately.

## live-acceptance — Athena to Apollo, for real

Read `tailnet.md` and the prior live-phase notes first; inspect current state before
operational changes. Install matching published SASE/core builds on Athena and Apollo,
refresh gateway, federation worker, AXE, and ACE, and verify packaged executables and
running versions. Reuse valid enrollment where possible (canonical init
rescan/status/doctor convergence; repair only if required); any bootstrap bundle moves
by protected file/stdin transport and is deleted afterward. Confirm the durable `apollo`
alias survives (parent note #2). Launch 1-3 xsmall observation agents with
`%dispatch:apollo` through the new target picker and `/sase_run` (authorized by the
parent plan), with bounded observation prompts.

Record `sase ace --tmux` pane evidence for the research validation table, executed on
the shipped default:

- A directly launched, unfollowed Apollo agent asks a question while ACE sits on
  Artifacts with a Here-only query → a durable needs-you entry appears without changing
  filters or loading its catalog page; answer it from the inbox.
- Find `project:sase` work across machines, isolate Apollo with `machine:apollo`, return
  and see navigation state restored; a second-page/continuation check with authoritative
  counts unchanged by paging (local multi-host regression covers the
  more-than-three-page case if Apollo is small).
- Identical agent names on two machines stay distinguishable; rename a machine and prove
  selection stability and correct-owner actions.
- A launch attempt from a dirty checkout explains the source constraint in place and
  preserves the draft.
- Kill a reply mid-launch, restart ACE, and reconcile the same operation with Check
  outcome — no duplicate run, no false success; follow into the launched agent's real
  nonempty output (a content-unavailable toast does not pass) and an exact-instance stop
  from Athena, independently confirmed on Apollo, leaving other agents untouched.
- Restart Apollo's managed gateway in the same ACE session: honest failure presentation,
  coalesced incident, recovery with enrollment identity intact.

Clean up only the test agents launched for this proof. Do not claim archive history
beyond the bounded recent catalog. Mac is best-effort and does not block. Capture
redacted commands, exit outcomes, versions, timestamps, operation identities, and pane
evidence on this phase bead and on reopened `sase-xe.16.11.3` and `sase-xe.16.11.5` so
their normal closes are grounded. If any item fails, preserve the exact unmet gate and
leave the work open.

## Dispositions and landing handoff

Preserve the five original `PROPOSED FOLLOW-UP:` dispositions in the eventual parent
close note, remapped to their owning phases here: `.3#1` TLS → trust-fencing; `.3#2`
merged-panel cost → one-list and real-contract-proof evidence; `.5#5` Rust extraction →
fleet-wire and unified-projection; `.5#6` late receipts/diagnostics → launch-durability;
`.5#7` launch visibility/identity → launch-durability. The canceled epic's pager
copy-link flake proposal (from its first phase) passes to this epic's land agent for
triage. Deferred by decision, not omission: Watch, `%dispatch:auto`, cross-machine queue
scheduling, automatic fallback, and always-on delivery while ACE is closed.

Do not add epic-close, post-close Symvision, or linked-plan status phases as children.
The land agent verifies the completed evidence, closes reopened `sase-xe.16.11.3` and
`.5` normally with specific verification, and never force-closes a nested bead. Rerun
descendant/linked-plan readiness and intervening source drift in both repositories; the
complete combined-tree `just check-full` must run via `/sase_monitor` with
TESTING/TESTED status before landing. After each close, run `just symvision`, recheck
`sase bead epic-symbols`, and mark the linked plan done through the normal landing flow.
