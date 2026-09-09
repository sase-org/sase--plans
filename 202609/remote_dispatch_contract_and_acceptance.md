---
tier: epic
title: Finish Fleet contracts, reliable dispatch, and Apollo acceptance
goal:
  Remote Fleet and Focus consume Rust-owned contracts, preserve successful hosts under
  faults, recover delayed launch receipts, and pass the complete Athena-to-Apollo
  workflow.
parent_bead: sase-xe.16.11
phases:
  - id: core-fleet-contract
    title: Share Fleet request, projection, freshness, and count policy in Rust
    depends_on: []
    description:
      "core-fleet-contract: add transport-independent typed request and response
      normalization plus authoritative Fleet and followed-set Focus counting in
      sase-core, with versioned bindings and regression tests."
    size: medium
  - id: worker-trust-and-pages
    title: Honor TLS trust and isolate catalog continuation by host
    depends_on:
      - core-fleet-contract
    description:
      "worker-trust-and-pages: apply validated connection-plan trust settings, provide
      host-specific bounded catalog continuation, and prove a real authenticated healthy
      gateway survives beside an actual hung host; capture and reject a genuinely
      replaced instance."
    size: medium
  - id: launch-receipt-recovery
    title: Recover delayed launch receipts and newly launched remote identities
    depends_on:
      - worker-trust-and-pages
    description:
      "launch-receipt-recovery: make slow target launches converge through the existing
      durable operation key without duplicate agents, preserve bounded reads, expose
      safe bridge errors, and refresh/resolve newly launched identities for follow,
      output, and exact-instance stop."
    size: medium
  - id: fleet-frontend-integration
    title: Consume published contracts and implement honest Fleet navigation
    depends_on:
      - launch-receipt-recovery
    description:
      "fleet-frontend-integration: ratchet the published core surface, replace Python
      backend policy with thin adapters, implement accessible per-host continuation and
      accurate counts/freshness, and integrate selective machine-section updates without
      blocking navigation."
    size: medium
  - id: real-contract-regressions
    title: Drive ACE requests through real worker envelopes and refresh the fixtures
    depends_on:
      - fleet-frontend-integration
    description:
      "real-contract-regressions: exercise ACE-generated requests through actual
      serialized worker/gateway contracts, replace synthetic Fleet fixtures and affected
      goldens, and prove fault transitions, hidden-Fleet laziness, and the navigation
      budget."
    size: medium
  - id: live-apollo-acceptance
    title: Complete and record the same-session Athena-to-Apollo workflow
    depends_on:
      - real-contract-regressions
    description:
      "live-apollo-acceptance: deploy matching published builds, verify durable
      enrollment, launch authorized observation agents, show recent DONE and new active
      rows, follow/output/stop from Athena ACE, and prove same-session recovery after
      the Apollo gateway restarts."
    size: medium
proposed_by: bbugyi200.athena.sase-xe.16.11.land
bead_id: sase-xe.16.11.6
create_time: 2026-09-09 19:52:43
status: wip
---

- **PROMPT:**
  [prompts/202609/remote_dispatch_contract_and_acceptance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/remote_dispatch_contract_and_acceptance.md)
- **PARENT:**
  [202609/remote_dispatch_landing_remaining.md](remote_dispatch_landing_remaining.md)
- **BEAD:**
  [sase-xe.16.11.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.16.11.6.md)

# Finish Fleet contracts, reliable dispatch, and Apollo acceptance

## Scope and verified starting point

This is the remaining-work handoff from the interrupted landing of
`plan:202609/remote_dispatch_landing_remaining.md`, bead `sase-xe.16.11`. Its parent is
`sase-xe.16`, and that plan's parent is `sase-xe`. Preserve the architecture and
acceptance requirements of `plan:202609/remote_dispatch_fleet.md`. Read these artifacts
and `research:202609/apollo_fleet_contract_repair/apollo_fleet_contract_repair.md` with
`sase artifact read`. Open the research sidecar with `/sase_repo` first if the audited
read reports a missing document root; the report exists.

The landing audit reviewed the epic's four original notes, all five children and all
sixteen original child notes, the linked plan, actual source, and commits in both
repositories. Confirmed implementation commits are SASE `d015f48cb`, `20c7b9804`,
`b7c6bc006`, and core `0318b31`, `06025ba`, `a6d40ba`. The audit tree and freshly
fetched `origin/master` were both `b7c6bc006`; the opened core tree was `3baa689`
(0.32.55).

Do not repeat the completed setup work: Rust Tailnet classification and enrollment
reconciliation, Rust singleton-to-family follow promotion, their PyO3 bindings and thin
Python adapters, detailed discovery diagnostics, shared init/add/repair activation,
delayed retirement of repair credentials, tracked-only chezmoi apply, and canonical
setup guidance are implemented. Real bootstrap issuance/enrollment/ hello, replay,
expiry, and wrong-pin tests exist. Fault benchmarks now overlap real refresh sequences
with measured navigation. Preserve those improvements.

The audit ran 61 focused setup, machine service, Tailnet, Fleet projection/laziness, and
queue tests on installed core 0.32.55: all passed in 19.88 seconds. This does not
establish the missing behavior or the complete landing gate. Additional non-mutating
probes exposed the failures below despite that green run. The four real
bootstrap/gateway tests also passed without skips in 33.75 seconds, covering
issue/enroll/hello, replay, expiry and wrong pin. Retain that completed evidence; the
missing authenticated healthy-host fan-out proof is a separate test.

## Confirmed gaps

1. Core `federation_worker.rs::RemoteHost::new` builds a default reqwest client and
   never applies validated `plan.tls`. The phase's deadline test uses one hanging TCP
   listener and another closed port; it explicitly asserts the fast host is **not**
   `ok`. This is useful failure-isolation evidence, but cannot prove that a successful
   authenticated host remains usable beside a hung host.
2. The real mutation route rejects stale revision and a fabricated `other-run` locator
   without changing the current fixture. Complete the intended instance reuse proof by
   capturing an old locator, replacing that actual instance, then submitting the
   captured locator against the replacement.
3. `b7c6bc006` fixes basic live catalog/batch traversal, legal page size and some
   metadata, but puts shared normalization/counting policy in Python. The Rust counting
   call still receives raw worker hosts and fails with `unknown field alias`;
   `_fleet_agents_counts.py` swallows the exception. Gateway batch response counts are
   host-wide (`fleet_reads.rs::batch_lookup` copies snapshot counts). A single followed
   DONE summary with host-wide `running=9` therefore projects `focus_remote=9`, although
   none of the followed agents is running.
4. A payload with `freshness.partial=True` still projects `partial=False`, and summary
   `observed_at_unix=1800000000` becomes row observation time `None`. Unknown/stale
   hosts and count scope must be represented explicitly, including when catalog reads
   fail while summary reads succeed.
5. `_fetch_fleet_catalog` broadcasts the first host's cursor to all hosts and stops
   after three pages with no user-accessible continuation. An audit input with host
   cursors `off:100` and `off:200` selects only `off:100`. Per-host snapshot/cursor
   state is absent. Catalog page totals and visible row counts cannot determine
   authoritative running counts.
6. `tests/ace/tui/fleet_fixture.py` and much projection/facade/visual coverage still
   invent `hosts[].summaries`, mapping liveness, and content-as-agent-metadata. New
   hand-written worker-shaped dictionaries are not sufficient proof of production
   serialization or ACE request correctness.
7. Phase `sase-xe.16.11.5` note #4 reports a target agent starting after Athena's reply
   deadline, with no controller receipt and no newly launched Fleet row. TUI output and
   stop were not completed. The gateway route synchronously calls
   `agent_bridge.launch_text` before returning the settled receipt, while the controller
   uses the generic short request timeout. The bridge suppresses useful error details
   into `agent_bridge:launch-text`. Treat the report's noisy project alias logs as a
   lead, not a proven sole cause.

The audit reopened `sase-xe.16.11.3` and `.5`. The latter closed automatically on
commit, and its automatic-close note explicitly implies no verification. Original live
phase `sase-xe.16.10` also remains open. Their existing successful evidence is retained,
but the missing gates must be satisfied before normal closes.

## Constraints

- Open core, chezmoi, research, or any other repo with `/sase_repo`; never assume a
  sibling path. Read applicable reference memory through `/sase_memory_read`.
- Shared domain behavior and pure policy belong in `sase-core`. Keep the core
  independent of the gateway crate. Python may perform platform I/O and Textual
  presentation, but must not retain a fallback implementation of core behavior.
- Reuse existing operation keys, receipts, follow records/tombstones, locators,
  revisions, count semantics, and request contracts. Do not create a competing launch
  journal or restart/retry by generating a new operation key.
- Preserve installation pinning, verified HTTPS, exact-instance mutation fencing,
  bounded transport reads, provider import isolation, and durable operations for
  user-triggered work. No local-host fallback for a failed remote launch.
- Preserve hidden-Fleet and zero-machine laziness, off-thread processing, pump-free
  callbacks and coalesced refreshes. No eager catalog discovery or unbounded paging.
- Release-plz owns core versions. Publish through normal host finalizers and the release
  process; never hand-pin an unpublished version. Honor live instructions for local
  build isolation. Core verification includes PyO3 via `just check` or
  `scripts/check.sh`, not only `cargo test -p sase_core`.
- Read CLI, TUI performance, Symvision and verification memory when applicable. Update
  default config/help/schema for any changed named control. Do not change feature flags,
  relax test budgets, or add flake allowances to obtain closure.
- Phase workers record outside-scope discoveries as `PROPOSED FOLLOW-UP:` notes. They
  must leave genuinely unmet work open; a commit's automatic close is not evidence. Use
  the applicable mid-flight no-close workflow if needed.

## Core Fleet contract

Create versioned transport-independent wire inputs and results in `sase_core`, reusing
the actual catalog, batch, summary and resolved-agent types. Export narrow
dict-in/dict-out PyO3 bindings for request validation, envelope normalization,
freshness/diagnostics, and count aggregation. A normalized result must retain host
identity, per-host continuation/snapshot state, authoritative counts, observation time,
partial/unknown state, and complete row metadata/locators/revisions/capabilities.

Normalize each distinct envelope: catalog `payload.page.rows`, followed batch
`payload.entries[].summary` including unresolved entries, and summary
`payload.counts`/freshness. Followed requests use logical keys. Maintain strict
validation with actionable diagnostics for malformed envelopes. Do not blindly accept
the old invented fixture forms as an undocumented compatibility contract.

Fleet running totals use authoritative host summaries independently of catalog
page/search filters. Focus remote totals count only the complete bounded followed set
using existing logical-agent/family semantics; batch host totals are not followed
totals. Missing or stale hosts must not become authoritative zeroes. Keep counts,
occupied slots, catalog totals and rendered rows distinct. Test DONE follows, duplicate
family members, local/remote separation, unknown hosts, failed catalog plus successful
summary, freshness and observation propagation.

Record the precise schema and exported names for downstream phases. Tests must validate
the PyO3 envelopes and ensure another frontend receives the same decisions.

## Worker trust, pagination and actual fault proofs

Resolve and apply the existing `TlsTrustPlanWire` modes and references at the proper
transport/config boundary. Validate what `ca_ref` and `server_name_ref` mean before
implementing them; do not guess that an opaque reference is a raw path or hostname.
Reuse established reference resolution where available. Fail closed with useful
diagnostics if required trust material cannot be resolved or configured. Preserve
default system roots. Do not disable certificate/hostname checks or fall back to a
default client after a requested trust configuration fails.

Provide an isolated loopback HTTPS fixture trusted through this production path, backed
by an actual gateway with authenticated hello and real read payloads. Prove successful
reads beside an actually reached hung host, bounded overall completion, per-host
deadline diagnostics, and successful reads again on subsequent requests. Also test trust
mismatch/failure and preserve the existing fast-refusal test.

Make catalog continuation host-specific end to end. The current facade/worker fan-out
accepts one query for every host; extend the supported request contract so each host can
retain and consume its own cursor, independently finish, fail, restart or invalidate its
snapshot. A minimal targeted-host or per-host-query surface is preferable to a second
worker architecture. Reject invalid cursors without losing healthy host pages; do not
forward one host's cursor to another. Provide production contract tests with unequal
host histories/cursors and more than three pages.

Complete exact-instance proof through the actual mutation route: capture the first
instance's locator and revision, replace it under the same logical name with reused
PID/name identity, reject the old locator/revision, and verify zero effects on the
replacement. Use an isolated controllable bridge that records side effects; do not stub
the decision under test. Then verify the replacement's own locator can act.

## Delayed launch recovery and identity visibility

Trace the request across `dispatch/launch.py`, federation IPC/worker, gateway
`fleet_launch`, `FleetLaunchStore`, host bridge and mobile project resolution. Measure
admission/bridge/reply timing using bounded redacted diagnostics. Keep the shared short
read deadline intact; fix launch admission/observation through the existing durable
operation model. A longer wait alone is not proof of recovery.

Ensure slow accepted launches settle durably even if the controller disconnects or times
out. Repeating/reconciling the same operation key must retrieve the original receipt and
bind the target's authoritative logical/exact identity without another launch. Account
for failure before launch, success followed by lost reply, and controller/worker/gateway
restart. Preserve acceptance-window rules. Bound bridge execution and move blocking
bridge waits off the async runtime where necessary. Expose safe structured launch errors
with bounded, redacted detail, never raw credential-bearing stderr or filesystem
secrets.

The recent canonical project alias change in `_mobile_agent_context.py` is part of this
integration: verify portable project IDs resolve through the supported project registry
without creating shadow project files or symlinks as the solution. Preserve ordinary
mobile launch behavior and both local and remote project identities.

Investigate the missing newly launched row through owner snapshot acquisition,
invalidation/cache keys, receipt mapping and catalog projection. Fix the demonstrated
cause and integrate launch settlement with existing refresh/follow paths. Management
must resolve an authoritative exact instance even when the row was outside the current
UI page or the launch reply was lost. Use the durable operation/receipt or supported
target lookup; never blindly kill by a guessed name or PID. Include delayed bridge and
lost-reply tests that observe exactly one target launch, eventual receipt, visible row,
readable output and an exact-instance stop.

## Frontend and intervening-change integration

Wait for the new core surfaces to land and publish, then ratchet revision and dependency
floor with the supported commands and run `just install`. Update binding
collection/contract validation. Preserve the unconditional queue directive floor and
dispatch/queue rejection, star/explicit-model shortcut and ACE/LSP parity, checkpoint
recovery, and required-plugin behavior that landed during the original epic. The floor
must include required worker behavior as well as bindings: the current declared 0.32.54
minimum precedes `a6d40ba`'s outer-deadline fix, which is in 0.32.55. Verify the
eventual published wheel actually contains all required changes; testing only a newer
local checkout is not evidence for a lower published minimum.

Replace Python normalization/counting decisions in `_fleet_agents_payload.py`,
`_fleet_agents_counts.py`, and relevant projection/refresh code with thin binding calls.
Delete the exception-swallowing row-count fallback. Preserve all actual
`ResolvedAgentSummaryWire` metadata; content is content metadata. Feed authoritative
summary counts even when catalog pages are filtered or fail. Present unknown/stale
counts and host diagnostics clearly, including zero-row hosts.

Use the worker continuation surface for bounded on-demand pages. Keep each host's cursor
independently, retain accessible continuation beyond the initial budget, reset correctly
on refresh/filter changes, and merge updated rows by real identity without keeping stale
earlier revisions. Keep selection/fold/scroll state stable.

Investigate the merged panel proposal against `agent_panels`,
`_display_panel_widgets.py` and row diff semantics. Fulfill the existing machine section
presentation contract and update only affected hosts/rows where possible. Do not fake
local tribe metadata merely to obtain a panel key. Changes to revision, freshness,
health or attention that alter visible state must be detectable even if existing fleet
dataclass fields are `compare=False`. Verify unaffected host widgets stay stable during
another host's row-count change, while changed rows really repaint.

Review source drift since the audit baseline in both repos before finalizing these
callers. The initial audit inspected all SASE commits since epic creation and core
commits since `0318b31`. Unrelated usage/pager/sidecar changes need no new abstraction;
preserve them and the `pytest_plugins` benchmark fixture registration from `f4ca78c0f`.

## Real contract regression and performance evidence

Drive requests built by the actual ACE refresh/continuation code through a real
federation worker and gateway, then project their serialized responses. Reuse the
trusted fixtures; no actual production credentials. Include two unequal hosts, healthy
plus failed/hung, active and recent DONE, a followed DONE outside page one, different
independent cursors, more than three pages, malformed requests, host recovery, counts
independent of pages/filters, and new launches with late receipts.

Replace synthetic fixture schemas across Fleet model/facade tests, visual snapshots and
fault benchmarks with validated real wire shapes. Correct bad assertions such as
counting all displayed rows as running; do not merely preserve existing goldens. Keep
test construction offline except for the explicitly isolated integration suite. Any
unavailable real gateway/worker fixture must be an explicit unmet verification
requirement, not a skipped test presented as successful acceptance.

Run the changed code's required checks, affected visual tests, and the three Fleet fault
benchmarks. Require fault sequences to overlap measured navigation, healthy rows to
remain navigable, p95 below 16 ms, and no stalls. Measure host contention honestly; do
not reduce meaningful scenario coverage or relax the budget. If the panel proposal is
already satisfied by the final design, record the concrete evidence instead of adding an
unnecessary parallel rendering mechanism. No remote work while Fleet is hidden beyond
its documented demand tier, and none with zero machines.

## Live Apollo acceptance

Read `tailnet.md`, `/sase_run`, the original plans, and phases `sase-xe.16.11.5` and
`sase-xe.16.10`. The preceding phase already proved node-specific HTTPS Serve, canonical
init, durable chezmoi source/applied alias `apollo`, authenticated hello and dispatch
doctor. Its managed service is now `sase-gateway.service`, not the old transient proof
service. Inspect current state before making operational changes.

Install matching published SASE/core builds on Athena and Apollo and refresh gateway,
federation worker, AXE and ACE. Verify packaged executables and actual running versions.
Use the existing enrollment when still valid and demonstrate canonical init
rescan/status/doctor convergence; only repair/re-enroll if required. Any bootstrap
bundle goes through protected file/stdin transport and is deleted afterward. Never put
it in argv, prompts, captures or notes. Do not hand-edit credential stores.

Launch 1-3 xsmall observation agents using `%dispatch:apollo` and `/sase_run` from an
eligible clean published project. This is already authorized by the parent plan. Choose
a bounded observation prompt that remains alive long enough for management and does no
unrelated work. Require the original receipt/operation key to settle on Athena and
correlate it to the actual Apollo process and logical/exact identity.

Use `sase ace --tmux`, `send-keys` and `capture-pane` for live evidence of:

- A known recent completed Apollo agent with readable name/provider and correct state,
  and the newly launched active agent appearing in the same session.
- Correct authoritative running counts independent of visible catalog pages; an actual
  second page/continuation check. Local multi-host regression proves the
  more-than-three-page case if Apollo has too few records.
- Follow into Focus and actual nonempty remote output/content for the launched
  observation agent. A toast saying content is unavailable does not pass.
- Exact-instance stop from Athena's TUI, independently confirmed on Apollo, without
  affecting another agent. Clean up only the test agents launched for this proof.
- Honest host failure presentation and recovery in the **same ACE session** after
  restarting Apollo's managed gateway, with installation/enrollment identity intact.

Capture redacted commands, exit outcomes, versions, timestamps, agent/operation
identities and pane evidence on this phase and the reopened original live phases. Do not
claim arbitrary archive history from the bounded recent catalog (currently active plus
up to 512 recent records). Mac is best-effort and does not block this acceptance. If any
item fails, preserve the exact unmet gate and leave the work open.

## Proposal dispositions and landing handoff

Preserve all five original `PROPOSED FOLLOW-UP:` outcomes in the eventual parent close
note:

| Proposing bead/note  | Outcome                                                                                                                                                                           |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sase-xe.16.11.3` #1 | TLS omission is absorbed into the required successful healthy-host fault proof.                                                                                                   |
| `sase-xe.16.11.3` #2 | Merged Fleet panel cost is absorbed into machine-section/selective refresh and realistic navigation evidence; avoid unnecessary changes if final measurements prove the contract. |
| `sase-xe.16.11.5` #5 | Rust normalization/count extraction is mandatory shared-backend repair.                                                                                                           |
| `sase-xe.16.11.5` #6 | Delayed receipt convergence and safe launch errors are required dispatch acceptance work.                                                                                         |
| `sase-xe.16.11.5` #7 | Missing launch visibility and authoritative management identity are required dispatch acceptance work.                                                                            |

The landing audit used `/sase_new_task`, searched bug and all task types across all
statuses, swept every task from the last week, and inspected active epic plans and
related Fleet/launch phases. No duplicate repair task was found. `sase-ya` is
documentation-memory work, not a repair duplicate. No new standalone task was created:
all proposals have causal ownership in this active epic. None was discarded.

The child epic's `parent_bead` link is the handoff to `sase-xe.16.11` landing. Do
**not** add epic close, post-close Symvision, or linked-plan status updates as child
phases. The child land agent must verify the completed evidence, then close reopened
phases `sase-xe.16.11.3` and `.5` normally with specific verification before attempting
the parent plan's readiness checks. Never force a successful nested close.

Rerun all descendant/linked-plan readiness checks and intervening source drift. The
complete combined-tree `just check-full` must run via `/sase_monitor` with
TESTING/TESTED status before landing; no such gate has passed in this audit.
`sase bead epic-symbols sase-xe.16.11` was empty at audit time, but recheck every epic
and resolve or deliberately re-key exemptions before each actual normal close. After
each close, run `just symvision` and mark that epic's linked plan done.

When resuming `sase-xe.16`, verify and close original live phase `sase-xe.16.10`
normally on the complete evidence, preserve its previous landing-note dispositions, and
recheck all descendants, linked plans and post-child drift. Continue through directly
parented plan ancestors only while each is fully complete. An ambiguous or incomplete
ancestor gets a specific blocker note and remains open.
