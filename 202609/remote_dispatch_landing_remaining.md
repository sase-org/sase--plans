---
tier: epic
title: Finish remote dispatch setup correctness and live acceptance
goal: "Remote setup preserves discovery failures, activates enrollments through durable
  operations, uses Rust-owned shared policy, and has real fault and Athena-to-Apollo
  evidence sufficient to resume the interrupted sase-xe.16 landing.

  "
parent_bead: sase-xe.16
phases:
  - id: core-setup-policy
    title: Put discovery and enrollment reconciliation policy in Rust
    depends_on: []
    size: medium
    description: "core-setup-policy: port the newly added pure Tailnet parsing, endpoint
      and health classification, and enrollment reconciliation decisions into sase-core
      with versioned wire bindings and regression coverage; distinguish an explicitly
      unrelated healthy service from an older SASE gateway without a fleet field.

      "
  - id: core-follow-policy
    title: Share followed-family promotion decisions across frontends
    depends_on:
      - core-setup-policy
    size: medium
    description: "core-follow-policy: move the followed-batch singleton-to-family
      promotion derivation added by sase-xe.16.9 into sase-core, preserving identity
      matching, ambiguous-family refusal, explicit-follow scope, and unfollow
      tombstones.

      "
  - id: real-fault-proofs
    title: Exercise actual deadlines, instance fencing, and bootstrap enrollment
    depends_on:
      - core-follow-policy
    size: medium
    description: "real-fault-proofs: add real worker/gateway fault tests for a hung host
      beside a healthy host and rejection of a replaced exact instance, plus a real
      binding-to-gateway bootstrap round trip; strengthen the Fleet fault benchmark to
      exercise refresh transitions and measure navigation while faults are active.

      "
  - id: setup-integration
    title: Integrate shared policy, honest discovery, and durable activation
    depends_on:
      - real-fault-proofs
    size: medium
    description: "setup-integration: consume the published core surface through thin
      adapters, preserve discovery diagnostics in all init entry points, share verified
      activation with machine add and repair, remove untracked chezmoi fallback, update
      stale Fleet setup guidance, and ratchet the core pin and published floor.

      "
  - id: live-apollo-acceptance
    title: Complete the real Athena-to-Apollo workflow
    depends_on:
      - setup-integration
    size: medium
    description:
      "live-apollo-acceptance: finish the reopened sase-xe.16.10 acceptance using
      node-specific Tailscale Serve, fresh installed builds, canonical init,
      authenticated status, remote launches, TUI follow/output/stop, and gateway restart
      recovery; retain redacted command and pane evidence."
proposed_by: bbugyi200.athena.sase-xe.16.land--1
bead_id: sase-xe.16.11
create_time: 2026-09-09 19:52:45
status: wip
---

- **PROMPT:**
  [prompts/202609/remote_dispatch_landing_remaining.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/remote_dispatch_landing_remaining.md)
- **BEAD:**
  [sase-xe.16.11](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.16.11.md)

# Plan: Finish remote dispatch setup correctness and live acceptance

## Why this is remaining epic work

This is the child handoff from the interrupted landing of
`plan:202609/remote_dispatch_completion.md` (sase-xe.16). It does not repeat gateway
packaging, initial discovery implementation, the offline Fleet fixture, or the six
already committed Fleet PNG goldens. The architecture and acceptance contracts in
`plan:202609/remote_dispatch_fleet.md` remain binding.

The landing audit inspected all ten child beads and all their notes, the epic's own
record, implementation commits in both repositories, and intervening master commits. At
audit time the sase tree and freshly fetched origin/master both were `890660e25`; the
opened core tree was `f79d58b` (0.32.50). No base-branch commits were outstanding.

Implemented surfaces were confirmed in core commit `9adb209`, Python commits
`5015d76e9`, `18b0a91a2`, `ace9e2cd4`, `f081f2303`, `338e3b349`, `8c4f8fd22`,
`6ae983ddc`, `7ee2e5177`, and documentation commit `890660e25`. The intervening queue
directive commits `c235300c6` and `0770357cd` already add dispatch/queue rejection
coverage; retain that contract. Model-alias, usage, leader-key, artifact-link, sidecar,
and stitch-recovery changes were reviewed for overlapping surfaces. Their changes do not
replace the missing setup and acceptance work described below.

The following are confirmed gaps, not optional enhancements:

1. `MachineInitService.apply()` calls `MachineService.discover()`, which discards
   `DiscoveryResult.diagnostics`. Injecting `tailnet_status_unavailable` into the real
   candidates-only adapter yields exit code 0, no errors, and `nothing_to_enroll=True`.
   `machine discover` already uses `discover_detailed()`; canonical init must preserve
   the same truth.
2. `_run_scoped_chezmoi_apply()` catches submit/wait exceptions and directly invokes
   `apply_chezmoi()`. A mocked submit failure proves that untracked apply runs and can
   return success. The helper's contract explicitly requires a tracked proc, and a wait
   failure must not launch a second apply while the original may still run.
3. Activation is used only by init. `_handle_add()` and `_handle_repair()` report the
   enrollment result without applying the chezmoi source, reloading, or validating
   authenticated hello. Repair is the recovery command the new init instructs users to
   run, making this an integration requirement. Phase sase-xe.16.6 note #1 proposed add
   activation; absorb it here rather than creating unrelated follow-up debt.
4. `action_setup_agent_machine()` still teaches `machine discover` followed by
   `machine add`, without canonical init or target bootstrap guidance.
5. New shared policy lives in Python: Tailnet status/health/identity classification in
   `dispatch/tailnet_discovery.py`, reconciliation in `dispatch/machine_init.py`, and
   follow-family derivation in `ace/tui/models/fleet_agents.py`. The Rust backend
   boundary and the consolidated setup research explicitly put these decisions in core.
   Existing Rust follow reconciliation applies supplied promotions; it does not derive
   them from followed-batch observations.
6. `_classify_tailnet_health_payload({'status': 'ok', 'service': 'unrelated'}, ...)`
   returns `unknown`. The binding contract requires unrelated services to be
   distinguished from an older SASE gateway that lacks the fleet advertisement.
7. `test_facade_read_deadline_preserves_healthy_partial_host` supplies a mocked
   successful response containing a deadline diagnostic. It never hangs a host or
   expires a production deadline. Likewise,
   `test_replaced_same_logical_agent_rejects_old_exact_instance` supplies a mocked
   `precondition_mismatch`; it proves request forwarding, not actual fencing. Keep those
   adapter tests but supply the missing enforcement evidence.
8. The three Fleet fault benchmark names currently vary a static response and share one
   delayed catalog response. Real reconnect/event transitions during navigation need
   coverage before claiming the complete stress contract.
9. The operational proof did not finish. Sase-xe.16.10 note #7 says Serve, enrollment,
   and remote dispatch remain blocked; note #8 is an automatic commit close that
   explicitly implies no verification. The land audit reopened sase-xe.16.10, which also
   reopened ancestor sase-xe. A fresh SSH read confirmed Apollo's
   `sase-gateway-proof.service` is active and loopback health is OK with protocol 1,
   while `tailscale serve status` still says `No serve config`.

## Constraints

- Read plans and research with `sase artifact read`, and reference memory through
  `sase memory read`. Both consolidated research reports exist: use
  `sase repo open research` if their sidecar has not been materialized, then retry the
  audited read. Do not interpret an unopened sidecar as deleted reports.
- Open core and any other repository with `/sase_repo`. Shared backend decisions belong
  in Rust; Python retains UI rendering, prompts, platform I/O, and adapters.
- Preserve existing public behavior except for the identified correctness fixes. Do not
  introduce a feature flag, a Python fallback backend, eager discovery, or remote work
  when the machine registry is empty.
- Keep third-party provider import isolation, hard helper deadlines, output limits, and
  positional-or-keyword hook argument contracts intact.
- Do not derive trusted installation pins or SASE machine selectors from unauthenticated
  health or Tailscale host names. Keep HTTPS verification, node-specific Serve, explicit
  enrollment, protected local credentials, and exact-instance mutations.
- Bootstrap bytes never enter command argv, logs, agent prompts, captures, or bead
  notes. Use hidden entry or protected file/stdin transport and clean up temporary
  bundles. Never edit a credential store by hand.
- Core release versions are release-plz-owned. Publish through the normal host finalizer
  and release process. Run core's complete `just check` or `scripts/check.sh`, including
  PyO3; never substitute only `cargo test -p sase_core`.
- Read the verification, CLI, TUI performance, and Symvision memories as applicable. Run
  `just install` before dependent Python checks; use `just check` for changed sase
  files. `just check-full` is exclusively a `/sase_monitor` landing gate, with
  TESTING/TESTED labels. Do not weaken tests or flake allowances to achieve closure.
- Phase workers record new outside-scope findings as `PROPOSED FOLLOW-UP:` notes. They
  do not create task beads, close an epic, or mark a parent plan done.

## Core setup policy

Port the pure operations from `tailnet_discovery.py` and
`MachineInitService.reconcile()` into an appropriately scoped Rust module. Expose
versioned request/result wire structs and narrow dict-in/dict-out PyO3 bindings. Keep
subprocess management and HTTP transport out of the pure classifier; callers provide
status/health observations and receive candidates, diagnostics, and reconciliation
decisions. Python will continue enforcing platform I/O deadlines.

Cover defensive map-shaped Peer parsing, missing and extra fields, self exclusion,
trailing-dot DNS normalization, endpoint overrides, offline/OS advisory reasons, no
inferred SASE selector/pin, and recognized legacy SASE versus unrelated service
classification. Reconciliation preserves enrolled pins, skips matching identities, and
directs positively identified changes to deliberate repair. An untrusted discovery hint
must not overwrite enrollment identity. Keep protocol compatibility derived from the
fleet protocol constant and reject malformed version types.

Port the existing pure-policy tests, add the reproduced unrelated-service case, and test
the binding envelopes and malformed inputs. Document the precise exported binding names
and wire schema in the phase note so the adapter phase consumes a committed surface.

## Core follow policy

Move `followed_batch_family_promotions()` and its identity/matching rules from the TUI
model to core. Reuse existing fleet logical locator and follow reconciliation types.
Inputs are active follows and followed-batch observations; output is the validated
promotions consumed by `reconcile_follow_records()`.

Retain same-origin/project/agent matching, explicit-singleton eligibility, refusal when
observations match multiple families, no redundant promotion, and unfollow tombstones
winning at reconciliation. Tests should prove that non-TUI consumers get the same
decisions. TUI projection and off-thread persistence remain Python glue.

## Real fault proofs

Use isolated, loopback-only fixtures and the actual federation worker/gateway paths:

- One host never completes a read while another responds. Enforce the production
  deadline, assert a bounded return and a per-host failure, and prove the healthy host
  remains usable on the same and subsequent requests. Use handshakes/events to establish
  the hang instead of host-speed assumptions. Inspect and fix production code if the
  test reveals that a deadline discards healthy results.
- Replace an instance under the same logical name and reuse the relevant PID/name
  identity. Submit the old exact locator/revision through the real mutation enforcement
  path and assert rejection plus zero lifecycle side effects on the replacement. Do not
  stub the final decision being asserted.
- Issue a bootstrap through the actual exported Rust binding into an isolated SASE home,
  pass its CLI-format bundle through the production Python parser and gateway enrollment
  path, and prove authenticated hello. Retain/extend expiry, replay, wrong-pin, and
  secret-storage assertions without printing the secret.

Strengthen `bench_tui_jk_fleet.py`/the reusable fixture so reconnect churn and event
bursts are sequences applied during measured navigation, and the hung-host scenario
keeps a healthy row source available. Assert faults/refreshes actually overlap the
samples. Preserve p95 < 16 ms and no-stall requirements, and record scenario results.
Maintain hidden-Fleet and zero-machine no-remote-work tests.

## Setup integration

Once the core phases have landed and their release is published, ratchet
`sase-core-revision.txt` with the supported command and bump the dependency floor to the
first published release carrying every consumed binding. Never hand-pin an unpublished
release. Verify core binding collection, adapter contract checks, and ACE/LSP directive
parity against the installed extension and LSP. Existing queue directive migration must
remain intact.

Replace Python policy implementations with thin adapters to the new core functions.
Remove now-dead helpers and relocate pure tests to core while retaining Python
integration tests. Keep Fleet promotion reconciliation off the event loop and before
attention/projection.

Have all init entry points consume detailed discovery. Preserve candidates from working
providers alongside diagnostics from failed providers. Human and JSON results
distinguish unavailable/disabled/failed discovery from a legitimately empty tailnet;
missing tooling cannot return a successful empty setup. Offline planning and optional
zero-machine onboarding remain pure and convergent.

Factor activation so init, direct add, and repair share deploy/reload/credential/hello
verification. Preserve full enrollment/quarantine results and report partial activation
honestly. Ensure repair does not retire the only credential needed by the still-applied
record before replacement activation succeeds. A source-write/apply failure must retain
a usable recovery route without silently consuming a new bundle on retry or overwriting
pins during rescan.

Run scoped chezmoi apply only through a durable tracked operation. If submission fails,
return actionable failure without applying. If observation/wait fails after submission,
retain the proc identity and report uncertain/in-progress state without starting another
apply. Preserve noninteractive behavior and the helper's non-raising result contract.
Add tests for both failure boundaries and for source-versus-applied config, quarantine,
hello failure, direct add, and repair.

Update the Fleet setup command/help and relevant tests to teach target bootstrap and
canonical `sase machine init`; allow the explicit canonical rescan guidance when
machines already exist. Update runbook recovery text for the final behavior. Keep setup
guidance presentational and free of discovery work on the UI thread.

## Live Apollo acceptance

Read `tailnet.md` and the live-proof phase's notes. The previous gateway service is
`sase-gateway-proof.service`; inspect current host state before replacing anything.
Upgrade both normal installations to builds carrying this child's implementation, verify
packaged scripts, and restart AXE on both hosts as the parent plan requires.

Establish node-specific Tailscale Serve to Apollo's existing loopback gateway. If Serve
still requires an admin change, use `/sase_questions` with the actual current enablement
URL and explain the observed prerequisite. A user's acknowledgment is not proof that the
admin setting changed: retry the bounded command and require a real Serve configuration
plus HTTPS health with protocol 1. Do not substitute Funnel, insecure TLS, a public
gateway bind, or a fabricated enrollment.

Issue a fresh short-lived bundle using `sase machine bootstrap --json` on Apollo into
protected transport storage. On Athena, use canonical `sase machine init -B <file>`
through its actual supported interaction: discovery must identify Apollo compatible, the
intended alias must be explicitly selected, chezmoi activation must complete, and
authenticated hello must succeed. Verify list, status, and deep dispatch doctor.

Launch 1-3 xsmall observation agents with `%dispatch:apollo` on an eligible project
using `/sase_run`. This launch is explicitly authorized by the parent acceptance plan.
Use `sase ace --tmux`, tmux send-keys, and capture-pane to show Apollo's Fleet rows and
counts, follow one into Focus, view output, and stop one of those test agents. Confirm
the stop acts on Apollo. Restart the gateway and prove status and the same ACE session
recover with the enrolled identity intact. Remove temporary bootstrap files. Mac testing
is best-effort and cannot hold acceptance open.

Record redacted commands, exit outcomes, versions, agent identities, and pane evidence
on this phase and on the reopened original phase sase-xe.16.10. The runbook commit alone
must never count as operational acceptance. Leave work open on failure and preserve the
exact unmet gate.

## Audit verification and proposal dispositions to preserve at landing

The original land audit ran `just install`, then 74 focused tests covering keymaps,
machine init, Tailnet discovery, dispatch directives, Fleet laziness, federation, and
mutations; all passed. A second 89-test run covering directive contracts/parity,
provider isolation, bootstrap service, doctor, parser, and Fleet models passed. Separate
injected reproductions confirmed the diagnostics loss and untracked apply fallback
despite the existing suite being green. No complete landing gate is claimed.

Every original `PROPOSED FOLLOW-UP:` entry has this disposition:

| Proposing bead/note                | Disposition                                                                                                                                                                                                          |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sase-xe.16.2 #1, sase-xe.16.7 #1   | Both research reports exist and audited reads succeed after `sase repo open research`. Decline content restoration. Corroborate the missing-clone classification problem on existing task sase-u3.                   |
| sase-xe.16.5 #1                    | Decline a new keymap failure task: the current complete keymap test module passed; ref:plan o/O expectations are empty as intended in the committed module. Retain as historical evidence, not a reproduced failure. |
| sase-xe.16.6 #1                    | Direct-add activation is absorbed into setup integration, together with repair because the new recovery instructions depend on it.                                                                                   |
| sase-xe.16.8 #1, Grok part         | Existing epic sase-y5 note #3 owns the probe race; supplementary proposing-bead evidence recorded there, no duplicate task.                                                                                          |
| sase-xe.16.8 #1, clan-summary part | Corroborated existing task sase-xb with the proposing bead's unchanged-tree rerun evidence. No claim of a fresh land-audit failure.                                                                                  |
| sase-xe.16.10 #6                   | Decline new work: commit 5b8ee98ce already privatized the eligibility schema helper and updated its tests/manifest.                                                                                                  |

Sase-xe.16.9's verification note is not prefixed as a proposal, but its proc-handler and
SIGKILL timeout observations already appear on sase-j7 note #66 and sase-xb's
phase-reporter +1 respectively; retain those existing outcomes. The pin ratchet
requirement's dispatch parity evidence is now supplemented on sase-y9 with this audit's
passing installed-core/LSP run; do not close that task solely from one run.

## Landing handoff, outside the child phases

The child epic's `parent_bead` is sase-xe.16. Do not add parent-epic close, Symvision,
or parent-plan status updates as implementation phases. The child land agent and resumed
parent landing own those lifecycle operations under the original land prompt.

Original phase sase-xe.16.10 is intentionally open because its automatic close was
invalid evidence. Before treating the original parent as ready, verify that the live
phase above completed every original requirement, then close that original phase
normally with the concrete evidence. Never force a successful nested landing.

Rerun descendant/linked-plan readiness and post-child drift checks, run the complete
combined-tree `just check-full` through `/sase_monitor`, and preserve the proposal
dispositions above in the original epic's eventual close note. Both
`sase bead epic-symbols sase-xe.16` and `sase bead epic-symbols sase-xe` had no entries
at audit time; recheck before each actual close. After normal close and post-close
Symvision, mark the linked plan done. Ancestor sase-xe is now open and must be audited
through every descendant/note, its prior landing notes and linked plan, and subsequent
drift before any attempt to close it. Stop and record a blocker on the first ancestor
whose required work is not complete.
