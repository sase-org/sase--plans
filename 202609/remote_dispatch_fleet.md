---
tier: epic
title: Remote dispatch and the Focus/Fleet agents experience
goal: "A user can enroll remote machines (Tailnet by default, plain HTTPS without
  Tailscale), launch agents on them with %dispatch:<machine>, browse every enrolled
  machine's agents in a new Fleet sub-view of the Agents tab, follow remote agents into
  the Focus sub-view, and manage followed remote agents (view, kill, retry, fork, answer
  questions, approve gates) with the same action vocabulary as local agents - all with
  zero added cost when no machines are configured and no event-loop stalls when a
  machine hangs.

  "
phases:
  - id: local-reads
    title: Bounded index-backed local listing reads
    depends_on: []
    size: medium
    description:
      "local-reads: remove the redundant child-summary scan from the CLI agent listing,
      route the CLI and mobile-bridge listings through one bounded index-backed summary
      query with owner-side liveness, and add scan/candidate/decode instrumentation so
      later remote summary endpoints have a fast, measurable local substrate."
  - id: identity-contracts
    title: Portable identity, resolved records, and operation contracts in sase-core
    depends_on: []
    size: large
    description:
      "identity-contracts: define per-user installation identity,
      origin/agent/family/run locators, versioned resolved summary and detail record
      types, capability sets, the single running-count definition, the
      store-generation-plus-sequence cursor, and serializable connection-plan and
      operation-key types in the transport-free sase_core domain crate, exposed through
      the existing PyO3 binding."
  - id: viewer-purity
    title: Split owner resolution from pure presentation in ACE loaders
    depends_on:
      - local-reads
      - identity-contracts
    size: medium
    description:
      "viewer-purity: extract the local side effects concentrated in the ACE
      running-agent loaders (stale-claim release, running-marker unlink, artifact-index
      mutation, PID liveness) into an owner-resolution step so rendering consumes
      resolved records and opaque handles only, with effect-failing tests proving
      remote-origin records can never trigger local process or path effects."
  - id: gateway-auth
    title: Authenticated enrollment and hardened gateway access
    depends_on:
      - identity-contracts
    size: large
    description:
      "gateway-auth: add a fleet API with target-authorized bootstrap enrollment,
      short-lived single-use enrollment secrets, scoped revocable bearer credentials, a
      hello/capabilities/version-negotiation handshake, installation-identity pinning
      with quarantine on mismatch, rate limits, and fixes for the hot-path auth costs,
      replacing reliance on the self-authorizing loopback pairing flow for anything
      remotely reachable."
  - id: gateway-reads
    title: Bounded remote read protocol with recoverable events
    depends_on:
      - gateway-auth
    size: large
    description:
      "gateway-reads: serve maintained resolved summaries, bounded catalog pages, batch
      followed-ID lookups, lazy detail, and content handles over versioned HTTP/JSON by
      calling the core scan/index in-process, and make the SSE stream actually deliver
      post-connect invalidations through a broadcast channel with a
      generation-plus-sequence cursor and an explicit resync path."
  - id: op-journal
    title: Durable mutation journal and launch admission recovery
    depends_on:
      - gateway-auth
    size: large
    description:
      "op-journal: add operation keys with payload fingerprints, receipts and tombstones
      through a documented retry window, atomic run reservation before spawn, crash
      recovery that finds the admitted run instead of relaunching, and exact-instance
      fencing so a reused name or PID is never targeted."
  - id: dispatch-plugins
    title: Dispatch provider plugin hooks, built-in providers, and config schema
    depends_on:
      - identity-contracts
    size: large
    description:
      "dispatch-plugins: add the sase_dispatch entry-point group with
      dispatch_provider_spec, dispatch_discover, and dispatch_connection_plan hooks
      following the metadata-only finalizers loading model, built-in tailnet and https
      providers, the dispatch: config section with schema and layer-aware parsing,
      mode-restricted credential storage, and the remote_dispatch beta feature flag."
  - id: machine-cli
    title: sase machine CLI group and sase init enrollment
    depends_on:
      - gateway-auth
      - dispatch-plugins
    size: large
    description:
      "machine-cli: add the sase machine command group (add, discover, list, remove,
      rename, repair, status) wired through the parser registry with the default-list
      convention, an init-registry spec that offers remote-machine enrollment during
      sase init, and doctor checks for enrolled machine records."
  - id: follow-store
    title: Durable follow subscriptions with family continuity
    depends_on:
      - identity-contracts
    size: medium
    description:
      "follow-store: persist viewer-local follow records keyed by origin and logical
      agent/family identity, promote singleton follows to the family identity when a
      family forms, make explicit unfollow tombstones win over automatic re-creation,
      and implement the shared Focus/Fleet running-count calculations over authoritative
      summaries with unknown-host propagation."
  - id: federation-worker
    title: Local federation worker and Python remote facade
    depends_on:
      - gateway-reads
      - dispatch-plugins
    size: large
    description:
      "federation-worker: build the on-demand per-user Rust worker that owns remote
      connections, cached projections, per-host deadlines and backoff, and
      subscription-driven reconciliation behind a permission-restricted local IPC
      endpoint, plus the Python facade, binary packaging, and process supervision that
      ACE and the CLI call."
  - id: fleet-ui
    title: Focus and Fleet sub-views of the Agents tab
    depends_on:
      - viewer-purity
      - follow-store
      - federation-worker
    size: large
    description:
      "fleet-ui: add an agents-header strip with Focus and Fleet modes over the existing
      agent list widgets, running-count chips with partial and freshness states, machine
      section headers, follow stars and accent rails, bounded intent previews, per-mode
      selection/fold/scroll state, new named configurable keybindings, and demand-tiered
      hydration that does no remote work when no machine is enrolled."
  - id: dispatch-launch
    title: The %dispatch directive and reliable remote launch
    depends_on:
      - op-journal
      - machine-cli
      - federation-worker
    size: large
    description:
      "dispatch-launch: add %dispatch:<machine> to the dual-sourced directive
      vocabulary, intercept routing in the shared launch trunk before any target-side
      allocation, persist source intent with an operation key, send a portable launch
      request validated and admitted on the target, bind the receipt, auto-follow the
      resulting family, and reconcile lost replies without ever falling back to another
      machine."
  - id: remote-actions
    title: Remote lifecycle management parity
    depends_on:
      - fleet-ui
      - dispatch-launch
    size: large
    description:
      "remote-actions: execute kill/stop, retry, and fork-on-target for followed remote
      agents through the mutation journal with exact instance identity, stream bounded
      output/diff/chat content through opaque handles, and partition bulk actions per
      origin with honest per-target results."
  - id: attention-parity
    title: Remote questions, gates, and notification deduplication
    depends_on:
      - remote-actions
    size: large
    description:
      "attention-parity: surface remote question and gate attention in Focus promptly,
      answer questions and approve gates through consume-once mutation semantics with an
      already-answered result for a losing controller, and deduplicate notifications by
      origin plus request identity across reconnects."
  - id: acceptance
    title: Fleet-wide acceptance, flag removal, and polish
    depends_on:
      - attention-parity
    size: medium
    description:
      "acceptance: run the performance and failure acceptance matrix (local-first paint,
      j/k p95 under hung hosts, laziness tiers, fault recovery), finish
      keyboard-only/narrow/no-color UX review with updated PNG snapshots, remove the
      remote_dispatch beta flag by making the On branch unconditional, and record
      proposed follow-ups."
proposed_by: bbugyi200.athena.0gq
bead_id: sase-xe
create_time: 2026-09-09 19:52:44
status: wip
---

- **PROMPT:**
  [prompts/202609/remote_dispatch_fleet.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/remote_dispatch_fleet.md)
- **BEAD:**
  [sase-xe](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/README.md)

# Plan: Remote dispatch and the Focus/Fleet agents experience

## Context and sources

This epic implements remote machine dispatch for sase: enrolling remote machines,
browsing and following their agents from the local TUI, and launching agents on them
with a new `%dispatch:<machine>` directive.

The design follows the consolidated research report
`research:202609/remote_dispatch_and_fleet_focus/remote_dispatch_and_fleet_focus.md`
(read it with `sase artifact read` before implementing any phase) and the user's
accepted annotations on it:

- The two Agents sub-views are named **Focus** and **Fleet** (not Local/Remote), and
  subscribing is called **Follow** in all UI copy.
- A new **`sase machine`** CLI group is added for explicit machine operations.
- `%dispatch:<machine>` is the accepted directive spelling; dispatched agents are
  auto-followed by the launching machine.

Decisions this plan treats as settled (do not re-litigate in phases):

1. **One SASE protocol, pluggable machine access.** Remote access providers are plugins
   that produce discovery candidates and serializable connection plans; every provider
   fronts the same versioned HTTP/JSON + SSE fleet protocol served by the Rust gateway.
   Tailscale is never a hard dependency: the built-in `tailnet` provider is enabled by
   default for explicit discovery, and the built-in `https` provider is the
   non-Tailscale baseline.
2. **Focus and Fleet are two modes over the existing Agents list machinery**, not a
   second list implementation and not per-machine sub-sub-tabs. Focus = local agents
   plus followed remote agents; Fleet = the union of enrolled hosts' authoritative
   catalogs, deduplicated by stable origin identity. Each host exports only agents it
   owns - no transitive re-export of a host's own follows (prevents
   A-follows-B-follows-A loops).
3. **Follow is durable viewer-local intent**, separate from caches and dispatch
   receipts. It tracks the logical agent/family across sequential agent, monitor, and
   gate shells. Explicit unfollow overrides automatic membership and must not be
   resurrected by reconciliation.
4. **Owner resolution is separated from presentation.** Only the owning host resolves
   process liveness, lifecycle state, and content availability. Viewers receive
   versioned resolved records and opaque content handles - never actionable PIDs or
   filesystem paths.
5. **Discovery never runs at launch, completion, or ordinary refresh.** It runs only
   during explicit setup (`sase init`, `sase machine discover`). Empty
   `dispatch.machines` means no provider imports, no federation worker, no network
   traffic, and no remote timers.
6. **Mutations are journaled and recoverable.** Every remote mutation carries an
   operation key + payload fingerprint + exact target identity + preconditions. Launches
   atomically reserve run identity before spawn. Never silently fall back to local or to
   another machine.
7. **Provider is routing metadata, not identity.** Canonical agent identity is per-user
   installation ID + project + logical agent/family + shell/run instance. Switching a
   machine from tailnet to https must not duplicate rows or invalidate follows. An
   endpoint presenting a different installation identity is quarantined until
   deliberately repaired.
8. **Live remote reads do not build on `agents_sync`.** The git-sidecar publication
   pipeline is publish-only by decision (`agents-sync-publish-only`); the fleet protocol
   is a separate live read/mutate path.

Out of scope for this epic (record as `PROPOSED FOLLOW-UP:` notes if demand appears):
cross-user process control, automatic placement/pools/scheduling, cross-host `%wait` or
clan coordination, "fork here" (cross-host context migration), provider-session
migration, replicated agent history, transitive federation, an SSH tunnel provider,
always-on notification delivery while ACE is closed, and remote terminal/editor/tmux
integration.

## Architecture

```text
 ACE / CLI (Python)
   ├─ local reads and actions ──────────── existing direct path (unchanged)
   └─ remote facade / thin IPC adapter
           │
     local Rust federation worker (started on demand, per user)
       follows · cached projections · operation recovery
       per-host deadlines, connection pools, invalidations
           │
     versioned HTTP/JSON + SSE fleet API - one protocol for every provider
           │
     target Rust gateway → sase_core resolution/admission/lifecycle

 Python pluggy registry (sase_dispatch group) → validated machine records
       tailnet (built-in, default)   https (built-in)   later: ssh tunnel
```

Placement rules (from the Rust core backend boundary and verified crate layout):

- `crates/sase_core` stays transport-free (it has no tokio/reqwest/axum today - keep it
  that way). New domain types and rules (identity, resolved records, counts, cursors,
  operation keys, admission) go there and are exposed through `crates/sase_core_py`
  (`sase_core_rs`, dict-in/dict-out with `py.allow_threads`).
- Server-side fleet API extends `crates/sase_gateway` (already axum + tokio + reqwest).
  The federation worker lives in the gateway crate family - either a second binary in
  `sase_gateway` or a sibling crate depending on `sase_core`; the existing unused
  `crates/sase_gateway/src/daemon.rs` scaffolding (host identity, per-host run root,
  computed-but-never-bound unix socket path) is the starting skeleton. The worker never
  imports Python and contains no Textual policy.
- The gateway's mobile API v1 keeps working unchanged. The fleet API is a new versioned
  route namespace with its own contract snapshot generated through the existing
  `crates/sase_gateway/src/contract.rs` machinery.
- Two-repo changes must bump the pinned core revision (`sase-core-revision.txt`) and
  keep `tools/validate_sase_core_rs` green; open sase-core with the `/sase_repo` skill.
- Provider hook execution is never on a hot path: hot reads use enrolled machine records
  only. Provider code follows the finalizers trust model: inventoried as metadata
  in-process, imported only inside a bounded, separately supervised helper subprocess
  with cancellation (in-process pluggy calls cannot be timed out).
- Cross-machine agent naming reuses the existing machine-hood scheme
  (`{username}.{machine}.{bare}`, `crates/sase_core/src/machine_hood.rs`,
  `src/sase/core/agent_identity_facade.py`); enrolled aliases default to the target's
  authoritative `id.machine_name` read from the authenticated hello, so fleet names,
  `%dispatch` targets, and the existing name-lookup candidates stay one vocabulary.
  Provider device names and DNS names are never sanitized into identity.

## Product specification

### Focus / Fleet UI

The Agents tab gains a header strip (new `Horizontal(id="agents-header")` in the Agents
view, reusing `PanelTabStrip` exactly like the Artifacts tab's `#artifacts-subtabs`)
rendered only when at least one machine is enrolled; before that, the tab is visually
unchanged and a "Connect a machine" command-menu action opens setup guidance.

```text
 Agents
 ┌─ Focus · 7 running ─────── Fleet · 12 running · partial ──┐
 │ ● fix-auth       apollo    RUNNING   04:12   codex        │
 │ ? review-cache   here      QUESTION  00:38   claude       │
 ├───────────────────────────────────────────────────────────┤
 │ fix-auth · apollo · Updated now                           │
 │ ... existing detail layout, hydrated on demand ...        │
 └───────────────────────────────────────────────────────────┘
```

- **One list machinery, two modes.** Mode switch swaps query, layout density, selection,
  scroll position, and detail state; it paints cached state immediately and may schedule
  missing pages in the background. No second list implementation and no
  `ContentSwitcher` pane duplication of agent lists.
- **Focus** preserves today's local experience and adds followed remote rows with an
  origin label. **Fleet** is one scrollable catalog with light machine section headers
  (origin, connectivity, freshness, running count), restrained columns (follow state,
  agent/family name, project, status, elapsed, short intent summary; model only at wider
  widths), and origin repeated on rows when a header is offscreen or filtering mixes
  hosts.
- **Followed rows are visually distinct**: filled vs outlined star plus a subtle accent
  rail; color reinforces but never solely carries the distinction (no-color terminals
  distinguish via the star glyph).
- **Counts in the sub-tab titles** use one core definition: distinct logical agents
  whose current agent shell satisfies the existing running predicate
  (`sase_agent_status_counts` semantics - running bucket excluding dismissable, plus
  container-projected starting); tree headers, historical shells, gates, and monitors
  are not counted; transient family handoffs are deduplicated. Focus counts local
  running + running followed remote; Fleet counts running agents owned by enrolled
  hosts. The two overlap and are never summed. A never-contacted host contributes
  unknown - the title shows `partial`, and cached totals show last-seen age. Counts come
  from authoritative summaries, never from whichever page happens to be loaded.
- **Follow/unfollow** acts immediately and locally: following adds the row to Focus
  without switching modes or moving focus (footer offers "View in Focus"); unfollowing
  removes only this viewer's follow and offers Undo. Selection shows a bounded intent
  preview, never a full transcript. A host going offline keeps cached rows with age and
  disabled actions. Empty, loading, unavailable, and loaded-but-zero-results states must
  be visually distinct.
- **Keybindings**: `[`/`]` are taken on the Agents tab (`toggle_thinking` /
  `toggle_thinking_reverse`) - do not displace them. Add named, configurable actions for
  mode cycling and follow/unfollow. Known-free options at app scope: uppercase
  `B`/`I`/`P`; digits `1`-`9` are effectively free on the Agents tab (bound app-wide to
  `show_artifacts_digit` but availability-gated to the Artifacts tab), which matches
  `PanelTabStrip(show_numbers=True)` convention for direct mode selection. Update
  `src/sase/default_config.yml` (`ace.keymaps.app`), `AppKeymaps` (fields have no
  defaults - config entry is mandatory), `_BINDING_META`, availability gating, help
  modal, and command palette metadata together; mouse and command-menu paths work
  regardless of key choice.

### Fleet catalog semantics

"All agents shown in remote TUIs" means the same underlying eligible agent catalog
(active + attention-needed + recent first; older history reachable through paging and
explicit filters), not a mirror of each remote viewer's query/folds/dismissals.
Hidden/history eligibility is declared by the fleet API and deliberately matched to
existing local semantics. Each host exports only agents it owns.

### Follow semantics

- Persisted viewer-locally, keyed by origin installation ID + logical agent/family
  identity + created-by (explicit follow vs dispatch); `%dispatch` prewrites the follow
  before launch submission so a crash cannot lose it.
- Follows the family across agent/monitor/gate shell handoffs (agents are single-turn;
  continuation is mechanical - a follow that dies with the first shell would miss the
  very approval the user is waiting for). Singleton follows are promoted to the family
  identity when a family forms. Siblings/descendants are not auto-followed; a fork/retry
  launched by this viewer gets its own follow.
- Completed followed agents obey normal recent/history rules in Focus. Viewer dismissal
  never archives or deletes the remote authority's record. Unfollow is never overridden
  by reconciliation or dispatch-receipt recovery.

### Configuration

New top-level `dispatch:` section. It must be added to
`src/sase/config/sase.schema.json` (the root is `additionalProperties: false`, so an
undeclared section fails every layer that uses it; a correct declaration also earns a
Config Center editor row via the Rust `config_field_model` flattener) and to
`src/sase/default_config.yml` with the repo's comments-as-docs style. Illustrative
shape:

```yaml
dispatch:
  discovery: # global discovery defaults; NO global execution provider
    - use: builtin@tailnet
  machines:
    apollo:
      use: builtin@tailnet # access provider for this machine (routing, not identity)
      endpoint: https://apollo.example-tailnet.ts.net
      installation_id: install_opaque_id
      credential_ref: apollo # key into the credential store; never a secret in YAML
```

- The per-machine provider selector field is named `use:` with `<plugin>@<id>` values
  (`builtin@tailnet`, `builtin@https`) - the generic config walk in
  `src/sase/plugins/required.py` collects `use:` keys and enforces `plugins.required`
  declarations for third-party prefixes automatically.
- Config parsing copies the layer-aware, provenance-tracking, diagnostic-collecting
  pattern of `src/sase/finalizers/config.py` (frozen dataclasses, key allowlists,
  per-layer merge records) - not the untyped-dict pattern.
- Credentials are never stored in YAML. Bearer tokens minted at enrollment go in a
  mode-0600 store under the sase home directory (following the
  `mobile_gateway/devices.json` precedent), keyed by `credential_ref`; display paths
  redact via the `_looks_sensitive` convention. Env-var indirection remains available
  for externally managed secrets.
- Empty/absent `dispatch:` is the default and loads nothing at runtime.
- The tailnet provider path keeps the gateway loopback-bound behind Tailscale Serve (the
  sanctioned pattern in the mobile runbook; never Funnel) - but tailnet membership and
  identity headers are hints, never SASE authorization. SASE bearer credentials remain
  the baseline on every transport.

### `sase machine` CLI group

Follows the CLI rules memory: bare `sase machine` delegates to `list` via the central
default-list wiring; subcommands and options sorted alphabetically; every public long
option has a short alias; options are never required; colored, scannable output;
excellent `-h`. Note `machine` is an already-occupied word (`id.machine_name`, the
`~/.sase/machine_name` selector): `sase machine list` embraces that by showing the local
machine's own identity row first, then enrolled remote machines - one machine
vocabulary, not two.

- `sase machine add <name>` - enroll: resolve `<name>` against discovery candidates (or
  take `-e/--endpoint` + `-u/--use <plugin>@<id>` for manual/https setup), run the
  authenticated enrollment handshake (operator enters the short-lived bootstrap secret
  generated on the target), pin the installation identity, store the credential, write
  the machine record. A newly discovered device is never implicitly trusted.
- `sase machine discover` - run enabled providers' discovery explicitly; print
  candidates with provider labels and reachability hints; enrolls nothing.
- `sase machine list` - local identity row + enrolled machines from config + cached
  status (alias, provider, endpoint, pinned identity, last-seen, cached counts). No
  network.
- `sase machine remove <name>` - unenroll, revoke/delete the stored credential, keep
  follow records (their rows show unavailable until re-enrollment).
- `sase machine rename <name> <new-name>` - relabel the alias; identity pin unchanged.
- `sase machine repair <name>` - deliberate resolution of a quarantined identity
  mismatch (confirm reinstall/migration, re-pin, rotate credentials).
- `sase machine status [<name> ...]` - explicit network: authenticated hello per machine
  showing reachability, negotiated protocol version, capabilities, and authoritative
  running counts.

`sase init` gains an init-registry spec (plan-then-apply like the existing
config/memory/repo/skills specs, placed after `config` since it needs `id.machine_name`)
that offers remote-machine setup: enumerate candidates, show provider/reachability
labels, enroll selected machines through the handshake. Existing explicit choices
authorize the steps - no per-step re-confirmation, no silent service exposure, no
tailnet policy changes. Doctor checks validate enrolled records without network;
live-reachability checks are `deep=True`.

### `%dispatch` directive

```text
#gh:sase %dispatch:apollo Investigate the cache regression.
```

- No directive means local; `local` is reserved as the explicit escape. Standard xprompt
  arg grammar (`%dispatch:apollo` / `%dispatch(apollo)`); no short alias initially.
- The directive vocabulary is dual-sourced: the Python table in
  `src/sase/xprompt/_directive_types.py` AND the Rust
  `sase_core_rs.directive_contract()`, with exact-set parity tests. Adding `%dispatch`
  therefore includes the sase-core contract bump (the `typed_launch_units` /
  `%if`/`%proc` change is the reference footprint). The contract's `feature_flag` field
  gates completion visibility while the beta flag is on.
- Completion reads enrolled aliases from cached config and cached eligibility only - it
  never loads a provider, opens a connection, or resolves anything side-effectful
  (keystroke paths are read-only and prompt-free per the TUI perf memory). Stale
  eligibility informs the picker but never permanently disables a healthy target;
  authoritative validation happens on submission.
- v1 restrictions, rejected with clear errors: `%dispatch` only at the outer launch
  level (no per-branch targets in `%{...}` fan-out, no hidden or late target changes
  after expansion), no cross-host `%wait`/`%clan` composition. Same-target family
  continuations, plans, gates, monitors, and host finalizers stay on the execution host;
  the launching machine follows the family.
- Side-effectful xprompt expansion, command substitutions, workspace setup, and
  finalizers execute on the target; the source parses routing syntax only.
- `@` references and attachments are validated explicitly: portable published references
  resolve at the target; machine-local files, dirty worktrees, and unpublished artifacts
  are rejected with a useful preparation message - never a silent launch from the
  target's default branch.

Launch sequence (each step's failure semantics are part of the contract):

1. Parse routing side-effect-free in the shared launch trunk
   (`src/sase/main/query_handler/_launch.py` `launch_query()`, where ACE's durable
   `sase run` and the CLI converge) before `launch_agents_from_cwd` performs any
   target-side name/timestamp/workspace allocation. The ACE-side workflow timestamp
   remains a local dedup key only.
2. Persist source intent: target installation ID, operation key, reconstructible launch
   input/attachment references, payload digest, `follow_requested=true`. Show a pending
   dispatch row immediately (not counted as running).
3. Prepare a portable request: project identified by provider/repository identity plus
   intended revision or Patch context - never a local checkout path.
4. Validate and admit on the target: project registration, repo/revision availability,
   provider/model/plugins/directives/finalizers, capacity. The target reserves run
   identity and allocates its own workspace; full capacity may produce a durable
   target-side queued run.
5. Bind the receipt: persist the authoritative agent/family/run locator, replace the
   pending row without losing selection, activate the prewritten follow. A lost reply
   shows "Checking dispatch outcome" and reconciles by operation key - never "Failed;
   launch again", never a fresh operation key, never local fallback. A source timeout is
   not evidence of nonexecution.

### Remote action parity

Same action vocabulary as local agents, executed by the target authority with exact
instance identity and journal protection: kill/stop, retry, child creation, fork
(default is fork-on-target, following the result), view summary/detail/output/diff via
bounded opaque handles with range reads and digest caches, question answers and gate
approvals as consume-once mutations ("Already answered on mac" for the losing
controller). Feedback names the host ("Stop requested on apollo"). Remote location alone
adds no extra confirmations; existing confirmation policy applies with the target
explicit. Capability checks combine host + resource + authorization + freshness;
target-side revalidation is mandatory even after a fresh row fetch. Bulk actions
partition per origin and report per-target results.

## Performance and laziness contract

These demand tiers gate every phase that touches them (read the TUI perf memory before
any TUI change; all I/O and decoding stays off the Textual event loop AND the serial
message pump via `spawn_pump_free_task`, user mutations run as tracked procs, selection
is re-read after every await):

| Demand                       | Work permitted                                                                                               |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------ |
| No configured machines       | Existing behavior exactly; no provider imports, worker, discovery, or remote timers                          |
| First paint                  | Local data + bounded cached remote working set; no network/provider wait                                     |
| Agents visible, Fleet hidden | Cheap summary reconciliation for title chips; batch followed-state lookups; no unfollowed row/detail fetches |
| Fleet visible                | First/visible catalog pages per host, compact previews; cancel obsolete requests on query change             |
| Explicit content open        | Bounded stream/range fetch with cache validation                                                             |
| Agents tab not displayed     | Slow/suspend decorative count refresh; preserve follow attention + operation recovery                        |

Reconciliation starts at jittered 30-60s summaries while visible, event-driven
invalidation when connected, exponential backoff for unavailable hosts - all tunable.
Acceptance (measured, not assumed - see the `acceptance` phase): local-only first paint
within baseline noise; `j`/`k` p95 < 16 ms with a hung host, reconnect storms, and event
bursts; hot reads bounded by relevant candidates (instrument scan count, candidate
count, liveness checks, decode time, bytes); followed IDs visible even when absent from
the first catalog page; hidden Fleet never triggers full catalog hydration; counts stay
honest under partial failure.

## Failure behavior (all phases)

| Failure                                | User-visible result                                                                         |
| -------------------------------------- | ------------------------------------------------------------------------------------------- |
| Missing Tailscale/provider             | That route unavailable; local work and other providers unaffected                           |
| Host hangs / laptop sleeps             | Cached rows with age; other hosts update independently; resume without restarting ACE       |
| Unauthorized/incompatible host         | Specific host state and remedy; never an empty "successful" catalog or weaker-auth fallback |
| Endpoint presents another installation | Quarantine; preserve follows; require deliberate `sase machine repair`                      |
| Stream replay gap / store rebuild      | Generation mismatch → bounded authoritative resync                                          |
| Launch reply lost / source crash       | Recover the same operation and follow intent; no fresh launch key                           |
| Name/PID reuse                         | Exact-instance fencing conflict; never target the replacement                               |
| Gate answered elsewhere                | Show settled result; never execute twice                                                    |
| Unfollow during pending dispatch       | Reconcile the operation while honoring the unfollow                                         |

## Feature flag

The `dispatch-plugins` phase creates a `beta` flag `remote_dispatch` with
`sase flag new` (never hand-registered; the registry validates at import time). While
the flag is off: no `dispatch:` config activation, no agents-header strip, `%dispatch`
is absent from the directive contract's visible set and rejects at parse, and
`sase machine` explains the beta gate. Every phase keeps both states tested and the Off
branch trivially deletable. The `acceptance` phase removes the flag: delete the Off
branch, make On unconditional, remove the registry entry, close the flag bead in the
same change. (An empty machine registry already keeps the released feature inert, so no
permanent config toggle replaces the flag.)

## Phase details

Common obligations for every phase: read
`research:202609/remote_dispatch_and_fleet_focus/remote_dispatch_and_fleet_focus.md` via
`sase artifact read`; read the memory notes named in the phase; open sase-core via
`/sase_repo` when the phase touches Rust, and bump `sase-core-revision.txt` / keep
`tools/validate_sase_core_rs` green for two-repo changes; run `just check` (via
`/sase_monitor` when slow) after changes; do not create beads - record
`PROPOSED FOLLOW-UP:` notes on your own bead; do not edit files under `sase/memory/` in
any phase of this epic.

### Phase `local-reads`: Bounded index-backed local listing reads

Standalone local win that later remote summaries build on. Read the TUI perf memory
first.

- `src/sase/integrations/agent_list_entries.py`: `agent_list_entries()` takes a listing
  snapshot and then calls `_children_by_parent_timestamp()`, which runs a **second**
  `scan_agent_artifacts` with its own scan options. The listing snapshot is already
  carried on `_RunningAgentListing.artifact_snapshot`
  (`src/sase/agent/running_listing.py`) - derive the child summaries from that single
  snapshot instead. Check `src/sase/core/agent_cleanup_python.py`'s sibling helper for
  the same pattern.
- Route the CLI listing and `src/sase/integrations/_mobile_agent_summary.py` through a
  bounded, index-backed summary query modeled on ACE's loader
  (`query_agent_artifact_index` via `src/sase/ace/tui/models/agent_loader.py` and
  `_agent_loader_artifacts.py`: active/recent tiers with explicit limits,
  `candidate_filter` pushdown, cached/revalidate freshness, bounded fallback scan with a
  repair reason). Owner-side liveness (`_record_is_live`) and visibility/family rules
  must still be applied - raw stored `active` status wildly over-counts.
- Add instrumentation counters (scan count, candidate count, liveness checks, decode
  time, bytes) to the listing paths so the acceptance phase can measure; follow the
  existing `SASE_TUI_TRACE` span conventions where applicable.
- Tests: listing parity against the old path on fixture archives; a regression test that
  `agent_list_entries()` performs at most one scan; bench note in the phase summary.

### Phase `identity-contracts`: Portable identity, resolved records, operation contracts

All in the sase-core linked repo's `crates/sase_core` (transport-free - no new runtime
deps), exposed through `crates/sase_core_py`; Python-side smoke tests via
`require_rust_binding`.

- Installation identity: per-user SASE installation ID (created/persisted locally;
  rotation/migration hooks for reinstall/clone), distinct from machine network identity,
  from `id.machine_name`, and from display aliases.
- Locators: origin (installation) + project + logical agent/family + exact shell/run
  instance IDs, composing with the existing machine-hood name scheme (`machine_hood.rs`)
  rather than replacing it. Human names are labels; name-addressed conveniences must
  resolve to an exact instance and verify before acting.
- Versioned resolved summary/detail record types: origin, stable IDs, display fields,
  owner-resolved liveness, observation time, row revision, capability set, opaque
  content handles. Never a PID or path as actionable identity. Separate lifecycle status
  vs process liveness vs connection health vs observation freshness; "was running; last
  seen ..." is representable. Build on the existing wire types (`AgentMetaWire`,
  `AgentArtifactRecordWire`) rather than duplicating them; the resolved records are a
  projection, not a second source of truth.
- The single running-count definition (shared by chips, summaries, CLI): distinct
  logical agents whose current agent shell satisfies the running predicate, family
  handoffs deduplicated; waiting/attention and occupied-runner-slot counts are separate
  fields (`agent_runtime.rs` already distinguishes slot occupancy - reuse it).
- Cursor type: store generation + sequence/revision, with `resync_required` semantics
  and tombstone expectations documented. Keep feed position separate from mutation
  preconditions.
- Operation-key types: key scoped to the authenticated controller, payload fingerprint,
  acceptance window, receipt/conflict results.
- Serializable connection-plan schema (provider ref, endpoint URL, credential reference,
  pinned installation identity, TLS/trust settings) - data only, no live streams or raw
  auth headers.
- Elapsed/staleness rules: owner-relative durations for runtime display; viewer
  monotonic clock for cache age; conservative staleness after viewer restart.

### Phase `viewer-purity`: Split owner resolution from pure presentation

- The local side effects are concentrated in
  `src/sase/ace/tui/models/_loaders/_running_loaders.py`: PID liveness checks
  (`_claim_pid_is_live`), stale-claim release via `release_workspace`, `running.json`
  unlink, and `update_agent_artifact_index_for_marker_mutation`. Extract these into an
  owner-resolution step that runs only for locally owned records; the presentation path
  (row building + enrichment; note `_meta_enrichment_common.py` is already side-effect
  free) consumes resolved records and opaque handles from `identity-contracts`.
- Remove or guard remaining string-conversion/path-construction escape hatches in the
  presentation layer; a type without `__fspath__` is not a complete proof.
- Effect-failing tests: feed synthetic remote-origin records through the full render
  pipeline with local process checks, `release_workspace`, marker unlink, index
  mutation, and path opens instrumented to fail the test if invoked.
- No behavior change for local rows (the gate-shell nuance in
  `_stale_claim_is_releasable` - a pending gate keeps its dead creator's PID - must
  survive the refactor); existing snapshot/bench expectations hold.

### Phase `gateway-auth`: Authenticated enrollment and hardened gateway access

In sase-core's gateway crate, as a new fleet API namespace beside mobile v1. The current
pairing flow is self-authorizing (an unauthenticated `pair_start` returns the 6-digit
code in its own response body; security rests entirely on loopback binding) and must
never be the remote path.

- Target-authorized enrollment bootstrap: enrolling requires a short-lived, single-use
  enrollment secret generated on the target by an authorized local action (surfaced
  through `sase machine add`'s handshake in `machine-cli`). Result: scoped, revocable
  per-controller bearer credentials with expiry and rotation; a revocation route (the
  store's revoke function currently has no route).
- Hello/capabilities endpoint: protocol version negotiation (replace the exact-match
  `validate_schema` model with a negotiated min/max for the fleet namespace), a real
  capability set (the mobile session capability list is static and unenforced - fleet
  capabilities must be checked per handler), authoritative installation identity and
  `id.machine_name`. Enrollment pins the installation identity; any later mismatch
  quarantines the connection (no automatic re-trust), preserving follows.
- Hot-path hygiene while adding auth: constant-time token/code comparison, an in-memory
  token cache so authentication stops rewriting `devices.json` (and fsyncing an audit
  line) on every request, cross-process safety for the credential stores, rate limits on
  unauthenticated surfaces, request body size limits.
- Keep the first release single-owner: credentials authorize one user's installations;
  no cross-user sharing.
- Extend the contract-snapshot machinery (`contract.rs`) to cover the fleet namespace;
  commit the snapshot like the mobile one.
- Fault tests: replayed/expired enrollment secrets, revoked tokens, identity-mismatch
  quarantine, version-negotiation failures, rate-limit and body-limit behavior.

### Phase `gateway-reads`: Bounded remote read protocol with recoverable events

In sase-core's gateway crate, building on `local-reads`' bounded summary substrate and
`identity-contracts`' record types.

- Serve fleet reads by calling `sase_core` scan/index functions in-process
  (`query_agent_artifact_index` with its `candidate_filter` AST and freshness modes) -
  not by shelling out to the Python mobile bridge (the existing
  blocking-subprocess-in-async-handler bridge patterns must not be copied; where
  subprocess work is unavoidable, use `spawn_blocking` with deadlines).
- Endpoints: authoritative summary (counts + revision), bounded catalog pages
  (independent query contract from counts), batch lookups for followed IDs, lazy
  per-agent detail, content handles (size limits, digests, range/offset reads for
  growing output), project eligibility. Summaries come from maintained resolved state
  with owner-side liveness - a small JSON body backed by an unbounded archive scan is
  still expensive.
- Events: the current SSE stream delivers only heartbeats after the initial replay
  (there is no subscriber channel - connected clients never see mutations, and
  per-connection heartbeats pollute the shared 128-slot replay ring). Add a broadcast
  path so post-connect invalidations are actually delivered; stop buffering heartbeats
  in the replay ring; cover all mutation sites including local launches, lifecycle
  changes, and process exits. Cursor = store generation + sequence; a stale cursor or
  unavailable deletion history yields `resync_required` plus a bounded authoritative
  snapshot. Periodic bounded reconciliation repairs missed events without serializing
  history.
- Fault tests: replay gaps, ring rollover, store rebuild/restore (generation change),
  snapshot-vs-cursor consistency, growing-output range reads, multiple concurrent SSE
  consumers.

### Phase `op-journal`: Durable mutation journal and launch admission recovery

In sase-core (domain rules) + the gateway crate (enforcement). Today `request_id` is
pure passthrough correlation with no dedupe anywhere - this phase adds the real
machinery, for the fleet namespace.

- Every mutation: authorization, exact target instance, durable operation key, payload
  fingerprint, preconditions (resource/action revision or pending-request ID), bounded
  acceptance policy. Same key + same payload → the original operation's receipt; same
  key + different payload → conflict; expired key → rejection, never a silent new
  launch.
- Launch admission: atomically reserve run identity + admission record before spawn;
  serialize workers with durable ownership/fencing; recovery finds the existing
  process/run for a reservation (the proc store's reserve/claim/finish lifecycle is
  prior art). Guarantee: one admitted run per launch intent through the documented
  recovery window.
- Receipts/tombstones retained through the retry window.
- Fault tests: crash between reservation/spawn/run-record/reply (each boundary),
  duplicate submissions, two competing controllers, receipt expiry, name/PID reuse
  fencing.

### Phase `dispatch-plugins`: Provider hooks, built-in providers, config schema

Python side. Read the CLI rules memory (config conventions) and the flags memory first.

- New entry-point group `sase_dispatch` added to `ENTRY_POINT_GROUPS` in
  `src/sase/plugins/inventory.py` as a **provider** group (metadata-only inventory;
  `SASE_DISABLE_PLUGIN_DISPATCH` comes free from the group-name convention). Follow the
  plurality convention deliberately: entry-point group name vs pluggy project name (the
  task-types precedent is group `sase_task_types` / project `sase_task_type`).
- Three hooks (typed, versioned request/result dataclasses, keyword arguments; hook
  argument names are a cross-repo compatibility boundary - positional-or-keyword without
  defaults per the workspace-provider hookspec notes):
  - `dispatch_provider_spec()` - stable provider ID, schema version, config schema,
    supported connection kinds, diagnostic metadata. Static and cheap.
  - `dispatch_discover(request)` - read-only candidate enumeration for explicit
    setup/rediscovery with deadline + cancellation; never enrolls or opens a control
    channel.
  - `dispatch_connection_plan(request)` - for the selected provider, the serializable
    plan to reach the standard fleet endpoint. No agent semantics. Core owns enrollment,
    credentials, channel lifetime, retries, caching, listing, launch, and control; there
    are deliberately no per-action plugin hooks.
- Loading follows the **finalizers trust model**, not the eager `ep.load()` styles:
  inventory entry points as metadata in-process; import provider code only inside the
  bounded, separately supervised helper subprocess (with cancellation), and only for the
  selected provider. A provider exception or timeout affects only its machines.
  Registration order never selects anything.
- Built-in `tailnet` provider: discovery via `tailscale status --json` parsed
  defensively with fixtures for missing/extra fields; peer visibility and Tailscale
  online status are hints only; absent binary → provider unavailable, everything else
  unaffected. Built-in `https` provider: static endpoint entry, validated TLS. Register
  both under the `builtin@` prefix in `pyproject.toml` beside the existing builtin entry
  points.
- `dispatch:` config section per the product spec: schema addition to
  `sase.schema.json`, documented defaults in `default_config.yml`, layer-aware typed
  parsing (finalizers-config pattern), `use:` selector fields, credential store (0600,
  under sase home, redacted display).
- Create the `remote_dispatch` beta flag with `sase flag new` (three authored
  sentences); gate the config-activation surface and add the
  `current_flags().enabled(...)` branch points.
- Tests: hook contract tests with a fake provider package, tailnet fixture parsing,
  lazy-loading proof (importing sase never imports a provider module; asserted, not
  assumed), config-layer provenance and unknown-key diagnostics, both flag states.

### Phase `machine-cli`: `sase machine` group and `sase init` enrollment

Read the CLI rules memory first.

- Wire the group through all five registration points: `_COMMAND_REGISTRARS` in
  `src/sase/main/parser_registry.py` (string-lazy), the static import + tuple in
  `parser_full_registrars.py`, a new `src/sase/main/parser_machine.py`
  (`register_machine_parser`, modeled on `parser_flag.py`: RawDescriptionHelpFormatter,
  examples epilog, sorted subcommands, short aliases like `-j/--json`), an alphabetical
  lazy-import branch in `entry.py`, and (optionally) the compact root-help list. A
  `machine_handler.py` shim delegates to a new `src/sase/dispatch/cli*.py` package.
  Naming a subcommand exactly `list` gets bare-group delegation and the runtime notice
  for free; the guard test asserting registry/parser parity must pass.
- Subcommands per the product spec: `add`, `discover`, `list` (default), `remove`,
  `rename`, `repair`, `status`. `add` drives the `gateway-auth` enrollment handshake end
  to end (bootstrap secret entry, identity pinning, credential storage, machine record
  write; interactive prompts follow the `_init_input_func` injection convention so they
  are testable). `status` and `discover` are the only other network-touching
  subcommands.
- `sase init`: add an `InitCommandSpec` to `init_registry.py` after `config` (order is
  load-bearing - it needs `id.machine_name`), with the plan-then-apply model
  (`InitPlan`/`InitAction`, `--check`/`--diff`/`--json` behavior) and TTY-gated
  interactivity like the existing onboarding flow.
- Doctor: new `src/sase/doctor/checks_dispatch.py` with `dispatch_check_specs(context)`
  registered lazily in `doctor/runner.py` - config/credential/pin coherence checks by
  default, live-reachability checks marked `deep=True`.
- Both flag states tested; help output and delegation-notice coverage per existing CLI
  test conventions.

### Phase `follow-store`: Durable follow subscriptions with family continuity

Python persistence + sase-core shared rules (family promotion and count definitions are
domain logic - rules in sase-core, persistence and wiring in Python).

- Durable viewer-local follow records (origin installation ID + logical agent/family
  identity + created-by: explicit follow vs dispatch), persisted under the sase state
  directory, separate from caches and receipts, written atomically.
  Prewritten-then-activated lifecycle for dispatch follows.
- Family promotion: singleton follow promotes to family identity on family formation;
  gate/monitor/agent shell handoffs stay followed; forks/retries launched by this viewer
  get their own follow; no auto-follow of siblings.
- Explicit unfollow tombstones override automatic re-creation (dispatch receipts,
  reconciliation, ACE restart).
- Implement the shared count calculations (Focus and Fleet definitions from the product
  spec) over authoritative summaries, reusing the `sase_agent_status_counts` running
  semantics, with unknown-host propagation (`partial`) and per-host observation times.
- Tests: promotion across shell handoffs, unfollow-wins reconciliation, count honesty
  with unknown hosts and overlapping Focus/Fleet membership.

### Phase `federation-worker`: Local federation worker and Python remote facade

Rust (gateway crate family; start from the unused `daemon.rs` scaffolding - host
identity, per-host run root, and the computed-but-never-bound unix socket path) + Python
facade.

- On-demand per-user worker: started when remote features are first used (never at ACE
  startup with an empty machine registry), shared by local consumers over a
  permission-restricted unix-socket IPC endpoint (user-only); shipped and supervised
  alongside the existing gateway binary distribution (packaging and the
  `sase mobile gateway`-style resolve/spawn/health-poll supervision pattern are in
  scope). Clean shutdown/idle exit; ACE functions fully without it.
- Owns: connection pools with per-host deadlines and exponential backoff, cached
  projections (summaries, catalog pages, batch follow lookups), SSE subscriptions with
  cursor tracking and resync, follow/dispatch-intent persistence read at startup
  (persisted by Python before worker involvement, so worker crashes lose nothing),
  bounded concurrency, prioritization of followed/visible work, byte/page/cache limits.
- Receives core-validated machine records + connection plans from Python; never imports
  Python; contains no Textual policy.
- Python facade: thin IPC adapter with methods ACE/CLI call; worker restart
  transparency; every call carries a deadline; cached remote reads decode off the event
  loop.
- Fault tests: worker crash/restart mid-subscription, host hang (deadline fires, other
  hosts unaffected), backoff behavior, IPC permission checks, cursor resync end to end
  against a real gateway in tests.

### Phase `fleet-ui`: Focus and Fleet sub-views of the Agents tab

Read the TUI perf memory first; every rule there applies. Verified integration points:

- Mode state: a `current_agents_subtab` reactive beside the existing
  `current_artifacts_subtab` precedent in `app.py`, with a watcher modeled on
  `watch_current_artifacts_subtab` in `_app_watchers.py`.
- Strip: new `Horizontal(id="agents-header")` in `_app_layout.py`'s `#agents-view`
  (there is none today), hosting a `PanelTabStrip(show_numbers=True, ...)` like the
  Artifacts header. Counts go into the tab labels (the plugins-browser `set_tabs`
  refresh pattern); if styled chips are wanted, extend `PanelTab`/`_build_content` to
  accept Rich-`Text` labels and reuse `format_agent_count_chip`. Strip hidden until a
  machine is enrolled.
- Modes over existing machinery: Focus/Fleet is a partition/query axis over the existing
  per-panel `AgentList` widgets and loader pipeline - per-mode query, selection
  (`_agents_last_idx`/`_agents_last_identity` pattern generalized per mode), fold
  registries, and explicit scroll save/restore (`OptionList` scroll is not app-tracked
  today). Teach the incremental display gates
  (`_try_refresh_agents_display_incremental`'s fallback-reason list treats panel
  membership changes as rebuilds) about the mode axis. Route refreshes through
  `_refilter_agents()` + `_schedule_agents_async_refresh()`; selective `patch_row()` /
  `try_remove_rows()` updates on reconciliation, never full rebuilds (banner chips heal
  on the next full refresh - same caveat applies to mode chips and is acceptable).
- Fleet rendering: machine section headers, restrained columns, follow star/accent rail,
  origin repetition rules, bounded intent preview in the detail area (through the
  existing `DetailPanelDebouncer`; highlight paints immediately).
- Remote rows enter the pipeline as resolved records via the federation facade with
  origin-qualified stable IDs on `AceRowHandle` (the `provider_contract.py` vocabulary -
  pages, counts, deltas - is the shape remote snapshots plug into; `AgentsViewport`
  drives per-host page sizes).
- Follow/unfollow actions (immediate local effect + toast "Following fix-auth on
  apollo"; Undo for unfollow), machine filter/picker, "View in Focus" footer affordance,
  "Connect a machine" command-menu action.
- Focus merge: followed remote rows interleaved with local rows, origin label visible,
  stale/offline presentation with age.
- Demand tiers wired to visibility per the performance contract; all fetches through the
  federation facade with cancellation on query/selection change; all slow work via
  `spawn_pump_free_task` with teardown cancellation; selection re-read after every
  await.
- Keybindings per the product spec (new `AppKeymaps` fields + `_BINDING_META`
  - availability gating + `default_config.yml` + help modal + command palette; `[`/`]`
    stay with `toggle_thinking`).
- PNG snapshots for new states (followed row, partial counts, offline host,
  empty/loading/unavailable/zero-result Fleet); `j`/`k` bench unchanged on Focus with
  remote rows present; both flag states.

### Phase `dispatch-launch`: `%dispatch` and reliable remote launch

Read the xprompts memory first. Spans Python (directive parsing, launch trunk, ACE
pending rows) and sase-core/gateway (portable launch request, admission via
`op-journal`).

- Directive vocabulary: add `dispatch` to `_KNOWN_DIRECTIVES` + `PromptDirectives` (new
  `dispatch: str | None` field) in `src/sase/xprompt/_directive_types.py` with
  collect/extract/scan updates, AND to the Rust `directive_contract()` with
  `feature_flag: remote_dispatch` so completion visibility is gated; satisfy the
  contract-parity and completion-parity tests, `tools/validate_sase_core_rs`, and the
  core revision pin. The `%if`/`%proc` commit is the reference footprint
  (types/collect/extract/scan, completion candidates, LSP, shell completion catalog,
  docs directive tables, history prompt-metadata gate, literal zones, xprompt inspect,
  doctor xprompt checks).
- Routing interception: in `src/sase/main/query_handler/_launch.py` `launch_query()`
  before `launch_agents_from_cwd` - both ACE (durable `sase run` via the RUN_LAUNCH
  sidecar payload) and the CLI converge there. ACE additionally records the target in
  the launch-proc payload (`_launch_submission.py`) for pending-row display; the ACE
  workflow timestamp stays a local dedup key. `AgentNameForeignMachineError` in
  `src/sase/agent/launch_validation.py` currently hard-refuses foreign-machine names -
  dispatch routing supersedes that check for the dispatched branch while keeping it for
  local launches.
- Enforce the v1 restrictions (outer level only, no per-branch targets in fan-out, no
  cross-host `%wait`/`%clan`, conflicting targets rejected). Completion reads enrolled
  aliases from cached config only.
- Implement launch steps 1-5 from the product spec: persisted source intent + operation
  key, pending dispatch row through the tracked-proc launch lineage
  (`_submit_launch_proc` / launch records - a proc completing means submitted/settled,
  not remotely finished), portable project identity (provider/repo + revision/Patch
  context), target-side validation/admission/reservation through the fleet API, receipt
  binding, automatic follow activation, operation-key reconciliation after
  reconnect/restart, explicit unsent/uncertain/accepted states. Never fall back to local
  or another machine.
- Target-side execution: xprompt expansion, substitutions, workspace setup, and
  finalizers run on the target through its normal launch machinery; reject local-only
  context (attachments, dirty state) with preparation guidance.
- Fault tests: lost reply, source crash + restart reconciliation, capacity queueing,
  unreachable machine, duplicate submission, unfollow-during-pending, flag-off rejection
  (directive absent from visible contract and rejected at parse).

### Phase `remote-actions`: Remote lifecycle management parity

- Kill/stop, retry, and child-creation for followed remote agents through the existing
  TUI action mixins (`src/sase/ace/tui/actions/agents/` kill/fork flows), routed by
  origin: local rows keep the existing path; remote rows submit journal-protected
  mutations with exact instance identity via the federation facade (tracked procs;
  optimistic UI; the optimistic-UI → sync-worker → on-complete shape from the TUI perf
  memory).
- Fork defaults to fork-on-target for a remote row ("Fork on apollo"), following the
  result. "Fork here" is out of scope (record a follow-up if requested).
- Content viewing: chat/output/diff/artifact through bounded opaque handles with range
  reads for growing output and digest caches; origin + age always visible; following
  never downloads full history.
- Bulk actions partition by origin, report per-target results, never pretend to be
  atomic fleet transactions. Stale selections revalidate in the background; offline
  destructive work is not queued for later execution.
- Fault tests: kill of an already-exited remote instance (fencing), action against a
  stale row revision, host death mid-action, capability-missing surfaces a clear reason.

### Phase `attention-parity`: Remote questions, gates, notifications

- Remote question/attention states render in Focus with the same visual language as
  local attention; a followed family's question/gate surfaces promptly on
  connect/reconnect without repeated toasts (dedupe by origin + request/event identity).
- Question answering and gate approval as consume-once mutations against the exact
  pending request + revision; the second controller gets "Already answered on <host>"
  with the settled result; decision/command details are readable before approval; a
  gate's follow-up launch happens on the execution host under the journal (a gate runs
  commands - it must not bypass the journal because its button looks like notification
  UI).
- Fault tests: two controllers racing one gate, reconnect toast dedupe, answer against a
  superseded request revision.

### Phase `acceptance`: Fleet-wide acceptance, flag removal, polish

- Run the acceptance matrix from the performance contract and record results: local-only
  first-paint baseline comparison (empty machine registry), `j`/`k` p95 < 16 ms with a
  hung host/reconnect storm/event burst (bench suite + `SASE_TUI_PERF=1`), laziness-tier
  audits (hidden Fleet does no catalog hydration; no-machine config does zero remote
  work - assert via instrumentation from `local-reads`), count-honesty scenarios, and
  the failure table end to end.
- Full-tree verification with `just check-full` through `/sase_monitor` before landing;
  refresh remaining PNG snapshots (`just test-visual`); keyboard-only, narrow-terminal,
  and no-color review of Focus/Fleet/Follow surfaces.
- Remove the `remote_dispatch` flag: delete the Off branch, make On unconditional,
  remove the registry entry, close the flag bead in this change.
- Record `PROPOSED FOLLOW-UP:` notes for: a reference memory note documenting remote
  dispatch operations (memory is deliberately untouched by this epic), the stale
  plugin-docs count in `docs/plugins.md` if still unfixed, an SSH tunnel provider, "fork
  here", always-on notification delivery, remote terminal/tmux integration, and any
  deferred benchmark-driven tuning.
