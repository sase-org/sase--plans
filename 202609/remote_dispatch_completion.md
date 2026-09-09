---
tier: epic
title:
  Complete remote dispatch - target bootstrap, tailnet discovery, canonical machine
  init, and the live Apollo proof
parent_bead: sase-xe
goal: "Finish the remote dispatch feature epic sase-xe shipped incompletely, ending with
  a live proof: a normally installed Apollo that Athena discovers over the tailnet,
  explicitly enrolls through `sase machine init` after a target-local `sase machine
  bootstrap`, and immediately manages - launching 1-3 remote agents with
  %dispatch:apollo and driving them from Athena's TUI. Also lands the
  acceptance-hardening remainder (provider isolation, offline fleet fixture, PNG
  snapshots, fleet benches) reconciled from the sase-xe landing's stashed child plan
  draft.

  "
phases:
  - id: core-fleet-surface
    title: Package the gateway, bind bootstrap issuance, advertise fleet protocol
    depends_on: []
    size: large
    description:
      "core-fleet-surface: in sase-core, make a normally installed wheel a usable
      dispatch target. Extract the gateway binary's startup into a reusable
      `run_gateway_cli(args)` library function and export a `sase_gateway` console
      script from sase_core_py using the exact `sase_federation_worker` packaging
      pattern (Python shim, PyO3 `py.allow_threads` wrapper, registered pyfunction),
      with argument handling, Tokio lifecycle, and clean shutdown preserved. Add a
      narrow dict-in/dict-out PyO3 binding over `FleetCredentialStore::issue_bootstrap`
      that keeps the default 600-second single-use TTL, installation-pin, and scope
      semantics, and never logs or echoes the secret. Extend the public health response
      with a `fleet` object advertising `supported_protocol_versions` derived from
      `FLEET_PROTOCOL_VERSION`, derive the contract snapshot's
      `protocol_negotiation.supported_versions` from the same constant, regenerate both
      committed contract snapshots, and update every full-body health assertion.
      Installed-wheel smoke tests for both console scripts."
  - id: core-pin-and-floor
    title: Ratchet the core pin and dependency floor past the new surface
    depends_on:
      - core-fleet-surface
    size: small
    description:
      "core-pin-and-floor: in the sase repo, once sase-core's remote HEAD contains
      core-fleet-surface, run `just ratchet-core-revision` to advance
      sase-core-revision.txt (this also finally ratchets past the sase-xe flag-removal
      commit that the sase-y9 five-node directive skew is waiting on), bump the
      `sase-core-rs` floor in pyproject.toml to the release that carries the new surface
      once it is published, and keep `tools/check_sase_core_rs_bindings` /
      `tools/validate_sase_core_rs` green. Wait for the release with /sase_monitor; if
      the release-plz release PR needs a human merge, raise it with /sase_questions
      instead of guessing."
  - id: machine-bootstrap-cli
    title: Target-local `sase machine bootstrap` and packaged-command resolution
    depends_on:
      - core-fleet-surface
    size: medium
    description:
      "machine-bootstrap-cli: add `sase machine bootstrap [-e|--expires SECONDS]
      [-j|--json] [-s|--scope SCOPE ...]`, a target-local thin wrapper over the new
      issue_bootstrap binding that prints an enrollment bundle in exactly the format
      `MachineService._parse_enrollment_bundle` accepts, using the same OS user and SASE
      home as the gateway; the secret appears once on stdout and is never logged, echoed
      to prompts, or accepted as an argv value. Fix packaged command resolution:
      `resolve_federation_worker_command` (and a sibling resolver for the new
      `sase_gateway` console script) must also check `Path(sys.executable).parent`,
      because uv-tool installs put dependency console scripts inside the tool venv
      without exposing them on PATH. Add a non-deep doctor check that the federation
      worker command resolves whenever machines are configured (mobile-gateway
      precedent). Update the pinned machine-parser subcommand-set test."
  - id: tailnet-discovery
    title: Real builtin tailnet discovery with bounded probes and honest defaults
    depends_on: []
    size: medium
    description:
      "tailnet-discovery: implement `BuiltinDispatchProviders.dispatch_discover` for
      real. Run `tailscale status --json` without a shell under an enforced process
      deadline and output-size cap; parse the map-shaped Peer payload defensively
      (fixtures for missing/extra fields, self-exclusion, trailing-dot DNS, offline
      peers, missing CLI, malformed and oversized output); form default
      `https://<node-dns>` endpoints from validated MagicDNS names; probe candidates
      independently with per-peer and overall bounds, classifying compatibility from the
      public health response (its new `fleet` advertisement when present, `unknown` when
      absent, `incompatible` for unrelated services). Online state and OS are advisory
      hints with reasons, never exclusions. Return candidates plus structured
      diagnostics so a broken provider or absent binary never renders as an empty
      successful discovery, and stop swallowing provider exceptions silently. Enforce a
      real wall-clock bound on provider hook execution. Make the builtin@tailnet spec's
      supports_discovery claim true. Flip defaults: `builtin@tailnet.enabled: true` and
      `discovery.enabled_providers: [builtin@tailnet]`, honoring explicit disablement
      and explaining (not silently skipping) a disabled selection."
  - id: provider-isolation
    title: Third-party provider imports follow the finalizers trust model
    depends_on: []
    size: medium
    description:
      "provider-isolation: today collect_dispatch_providers and helpers eagerly
      ep.load() third-party entry points in-process, which the parent plan's
      dispatch-plugins phase explicitly forbade. Inventory entry points as metadata
      in-process; import provider code only inside a bounded, separately supervised
      helper subprocess with a deadline and cancellation, and only for the selected
      provider, following the finalizers loading model. A provider exception or timeout
      affects only its machines. Decide the dispatch_connection_plan hook: add it for
      third-party providers or record an explicit deferral decision (builtin providers
      derive plans from enrolled machine records and do not need it)."
  - id: machine-init
    title: Canonical `sase machine init` with real activation and honest outcomes
    depends_on:
      - tailnet-discovery
    size: large
    description:
      "machine-init: add canonical `sase machine init` and refactor `sase init machine`
      and the init-registry machine spec to delegate to one shared machine-owned
      planner/apply service. Checks and previews stay offline; discovery runs only
      during explicit apply, after local identity setup. Existing enrolled machines must
      not suppress an explicit rescan, while an all-enrolled or zero-machine registry
      must not leave `sase init --check` permanently red. Read enrollment bundles with a
      hidden prompt (getpass), file, or stdin - never bare input(). Reuse `sase machine
      add`'s EnrollmentResult handling so quarantined or failed enrollment is reported
      truthfully instead of the current unconditional 'Enrolled'. After
      write_machine_record on a chezmoi-enabled controller, deploy the scoped change
      through the established apply_chezmoi mechanism honoring its tracked-proc
      contract, reload the merged config, resolve the credential, and run an
      authenticated hello before declaring success; apply failure or partial enrollment
      leaves actionable recovery state since the target may have consumed the bootstrap.
      Skip already-enrolled identities, never overwrite a pin during rescan, and route
      changed identity to `sase machine repair`. Fix stale docs/init.md wording and add
      behavioral tests for the interactive flow."
  - id: fleet-fixture
    title: Offline fleet fixture and hidden-Fleet laziness regression tests
    depends_on: []
    size: medium
    description:
      "fleet-fixture: build a reusable offline test substrate that synthesizes resolved
      remote rows, followed-batch and attention responses, counts, and diagnostics
      through a fake federation facade - no network, no Rust worker - so TUI unit tests,
      PNG snapshot tests, and benches can exercise Fleet/Focus states. Using it, add the
      missing TUI-level laziness regression tests: a hidden Fleet subtab performs zero
      catalog hydration in _run_agents_fleet_refresh, and a zero-machine config performs
      zero remote work on the refresh path."
  - id: fleet-visuals
    title: PNG snapshot coverage for Fleet and Focus states
    depends_on:
      - fleet-fixture
    size: medium
    description:
      "fleet-visuals: add the PNG snapshot coverage the fleet-ui phase specified but
      never landed: followed row (filled star plus accent rail), partial running-count
      chips, offline host with cached-age presentation, and the empty, loading,
      unavailable, and loaded-but-zero-results Fleet states - plus keyboard-only,
      narrow-terminal, and no-color review of those surfaces."
  - id: fleet-perf-faults
    title: Fleet benches under faults and the remaining failure-table tests
    depends_on:
      - fleet-fixture
    size: large
    description:
      "fleet-perf-faults: extend the j/k agents bench suite with fleet scenarios - a
      hung host, a reconnect storm, and an event burst - assert the p95 < 16 ms
      performance contract, and document the capture recipe in the perf runbook. Add the
      two failure-table fault tests still missing after landing: a host hang hits the
      facade deadline and leaves other hosts unaffected, and name/PID reuse is rejected
      by exact-instance fencing on remote rows. Wire or deliberately defer
      explicit-follow family promotion at reconciliation time (the follow store's
      reconcile path accepts promotions, but no production caller computes them from
      followed-batch family identity)."
  - id: live-apollo-proof
    title: Runbook plus live Athena-to-Apollo end-to-end proof
    depends_on:
      - core-pin-and-floor
      - machine-bootstrap-cli
      - machine-init
    size: medium
    description:
      "live-apollo-proof: write the target-preparation runbook (docs/) covering
      supported Linux/macOS installation, gateway supervision and restart, loopback bind
      behind node-specific Tailscale Serve, bootstrap issuance, and enrollment recovery
      - then execute it live. Update the installed sase (and sase-core-rs) on athena and
      apollo, restart `sase axe` on both. On apollo over SSH: confirm the packaged
      `sase_gateway` resolves, run it supervised and loopback-bound at 127.0.0.1:7629,
      configure node-specific `tailscale serve` (HTTPS; guide through cert readiness if
      unmet), and issue `sase machine bootstrap --json`. On athena: run `sase machine
      init` end to end (tailnet discovery finds apollo as compatible, enrollment
      consumes the bundle, chezmoi change deploys, authenticated hello verifies), then
      verify `sase machine list`, `sase machine status apollo`, and deep doctor. Launch
      1-3 xsmall remote agents with %dispatch:apollo on an eligible project, then drive
      them from athena using `sase ace --tmux` plus tmux send-keys / capture-pane: Fleet
      sub-view shows apollo with counts, follow a remote agent into Focus, and exercise
      remote management (view output, stop one launched agent). Verify a gateway restart
      on apollo does not break the enrollment. Record captured evidence on the phase
      bead. A Mac pass is best-effort only, never an acceptance gate."
proposed_by: bbugyi200.athena.08c
bead_id: sase-xe.16
create_time: 2026-09-09 19:52:42
status: wip
---

- **PROMPT:**
  [prompts/202609/remote_dispatch_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/remote_dispatch_completion.md)
- **BEAD:**
  [sase-xe.16](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.16.md)

# Plan: Complete remote dispatch - target bootstrap, tailnet discovery, canonical machine init, and the live Apollo proof

## Context

Epic sase-xe shipped remote dispatch and the Focus/Fleet agents experience, but the
setup path cannot actually be completed today, and the landing stashed an
acceptance-hardening child plan that was never proposed. This epic finishes both.

Verified state of the tree (sase master `a0ac015e0`, sase-core master `be8f552` /
v0.32.44):

1. **The tailnet discoverer is a stub.** `BuiltinDispatchProviders.dispatch_discover`
   (`src/sase/dispatch/providers.py:72-82`) unconditionally returns `()`; no `tailscale`
   invocation exists anywhere in `src/`. The builtin@tailnet spec claims
   `supports_discovery: True` while implementing nothing. `_safe_discover`
   (`providers.py:290-311`) passes `timeout_seconds` through as an advisory kwarg with
   no wall-clock enforcement and swallows provider exceptions into an empty result.
2. **Defaults double-gate discovery off.** `src/sase/default_config.yml:31-39` ships
   `builtin@tailnet: {enabled: false}` and `discovery.enabled_providers: []`, so
   `discover_dispatch_candidates` short-circuits before touching any provider, and
   `plan_init_machine` returns its "no discovery providers configured" no-action branch
   on every fresh machine.
3. **Targets cannot be prepared from a normal install.** The `sase_gateway` binary
   exists only as a Rust bin target; the published `sase-core-rs` wheel exports just one
   console script (`sase_federation_worker`). `FleetCredentialStore::issue_bootstrap`
   (`crates/sase_gateway/src/fleet_auth.rs:89`) has no HTTP route, no CLI, and no PyO3
   binding - its only callers are test helpers. There is no supported way to mint an
   enrollment bundle on a target.
4. **Enrollment outcomes are lost by init.** `run_init_machine`
   (`src/sase/main/init_machine_handler.py`) discards the `EnrollmentResult` and prints
   `Enrolled {alias}` unconditionally (a quarantined enrollment reads as success), reads
   the bundle with echoing `input()` while `sase machine add` uses `getpass`, and
   `plan_init_machine` early-returns whenever any machine is already configured, so
   rescan after the first enrollment is impossible.
5. **Registry writes never deploy.** `_edit_machine_mapping`
   (`src/sase/dispatch/config.py:245`) writes the chezmoi _source_ file and clears
   caches, but nothing in `src/sase/dispatch/` calls `apply_chezmoi()`
   (`src/sase/config/targets.py:112`), so on a chezmoi-enabled controller
   (`use_chezmoi: true` is live) the applied config the runtime loads stays stale.
6. **Packaged commands do not resolve on installed machines.**
   `resolve_federation_worker_command`
   (`src/sase/dispatch/federation/_supervisor.py:26`) tries `shutil.which` then
   linked-core dev builds. On this uv-tool install, `sase_federation_worker` exists in
   the tool venv bin but not on PATH, so a normally installed machine with no dev
   checkout resolves nothing; no doctor check catches this before a `%dispatch` launch
   fails.
7. **The stashed acceptance-hardening draft is unproposed.** sase-xe note 7 records
   `sase_plan_remote_dispatch_acceptance_hardening.md` in the land agent's artifacts
   dir. Its five phases (tailnet-discovery, provider-isolation, fleet-fixture,
   fleet-visuals, fleet-perf-faults) are reconciled INTO this epic: tailnet-discovery is
   updated with the third investigation's hardening requirements, and the other four are
   carried substantially verbatim. This epic supersedes that draft; do not propose it
   separately.
8. **The core pin ratchet is outstanding.** `sase-core-revision.txt` pins `3fa0577`
   (v0.32.42) while sase-core master contains the sase-xe flag-removal commit `65203fc`
   (released in v0.32.44). Until the pin ratchets, CI's pinned build still fails the
   five sase-y9 directive contract/parity nodes; those failures are known skew, not new
   regressions.

Consolidated research:
`research:202609/tailnet_dispatch_setup/tailnet_dispatch_setup.md` (read it via
`sase artifact read` before any phase). Parent plan
`plan:202609/remote_dispatch_fleet.md` remains binding for architecture, the
performance/laziness demand tiers, and the failure-behavior table.

## Decisions this plan adopts

- **`sase machine bootstrap` is target-local and never an init spec.** Bundle minting is
  non-idempotent (each call mints a new single-use secret), so it can never satisfy the
  init registry's convergent plan/apply contract - a bootstrap spec would either keep
  `sase init --check` permanently red or be unreachable from bare init. Minting stays in
  Rust (`issue_bootstrap`) behind a narrow local binding; discovery never gains an
  unauthenticated secret-minting endpoint.
- **`sase machine init` is canonical.** `sase init machine` and the init-registry
  machine spec delegate to the same machine-owned planner/apply service. Target
  _readiness_ is convergent and may live in init planning; the mint is not.
- **Compatibility comes from the fleet protocol advertisement, not version equality.**
  Public health gains `fleet.supported_protocol_versions` derived from
  `FLEET_PROTOCOL_VERSION`. An older gateway without the field is "compatibility
  unknown" (manual bundle-authorized enrollment still works via the existing
  enrollment-time protocol check). A public health response never supplies the trusted
  installation pin. Package versions are diagnostics only.
- **Online state and OS are advisory.** Show offline or unsupported-looking peers with
  reasons, permit explicit probing, and let the gateway protocol decide readiness. Never
  derive SASE's machine selector from a Tailscale `HostName`.
- **Trust rules are unchanged.** Tailnet membership never implies SASE authorization;
  bearer credentials are the baseline. Never overwrite an installation pin during
  rescan; changed identity routes to deliberate `sase machine repair`. Gateways stay
  loopback-bound behind node-specific Tailscale Serve (never Funnel).
- **No feature flag.** An empty machine registry keeps every surface inert - the same
  rationale the parent epic used when removing `remote_dispatch`. Flipping the discovery
  defaults only affects explicit setup surfaces; it must not add runtime work in the
  zero-machine state (the demand-tier table still binds).

## Contracts declared by this plan (build against these, not against each other's WIP)

- **Health `fleet` object**: `GET /api/v1/health` gains
  `"fleet": {"supported_protocol_versions": [1]}` derived from `FLEET_PROTOCOL_VERSION`
  (`crates/sase_gateway/src/wire.rs:48`). Consumers treat an absent `fleet` object as
  compatibility-unknown, never as an error. This lets `tailnet-discovery` proceed in
  parallel with `core-fleet-surface`.
- **Enrollment bundle format**: `sase machine bootstrap` emits exactly what
  `MachineService._parse_enrollment_bundle` (`src/sase/dispatch/machine_service.py:330`)
  accepts: JSON (raw or base64url) with `bootstrap_id`, `bootstrap_secret`,
  `pinned_installation_id`, `protocol_versions`, `requested_scopes`.
- **Gateway console script**: the packaged entry point is named `sase_gateway`,
  defaulting to `127.0.0.1:7629`, honoring the existing `--bind/-b`, `--sase-home/-H`
  and loopback-only-by-default semantics of the current Rust bin.

## Constraints and obligations for every phase

- Read `research:202609/tailnet_dispatch_setup/tailnet_dispatch_setup.md` via
  `sase artifact read` before starting; read the parent plan
  `plan:202609/remote_dispatch_fleet.md` for any phase touching its contract areas.
- Read the lint/test memory before finishing; run `just check` after changes (via
  `/sase_monitor` when slow); `just check-full` through `/sase_monitor` only. Read the
  CLI rules memory for any phase adding or changing CLI surfaces (sorted subcommands,
  short aliases for every public long option, options never required). Read the TUI perf
  memory before any phase touching the Agents tab refresh or bench paths (fleet-fixture,
  fleet-visuals, fleet-perf-faults).
- Open sase-core only with the `/sase_repo` skill. Two-repo changes keep
  `tools/validate_sase_core_rs` green. Never hand-edit sase-core versions or path-dep
  pins (release-plz owns them; see sase-core `AGENTS.md`).
- Provider hook argument names/kinds are a cross-repo compatibility boundary:
  positional-or-keyword without defaults (`providers.py:39-41`); pluggy invokes
  hookimpls positionally, and a keyword-only signature already silently broke discover
  once. Keep the regression coverage green.
- Secrets hygiene everywhere: a bootstrap secret or bundle never appears in argv, logs,
  echoed prompts, JSON diagnostics, or bead notes. Interactive bundle entry uses a
  hidden prompt; automation uses `-B/--bootstrap-file` or stdin.
- The demand-tier table from the parent plan binds: discovery never runs at launch,
  completion, or ordinary refresh; empty `dispatch.machines` means no provider imports,
  no worker, no network, no remote timers.
- Phase workers never create beads: record `PROPOSED FOLLOW-UP:` notes on your own phase
  bead. Do not edit files under `sase/memory/`.
- Do not weaken directive parity assertions or add flake allowances for the sase-y9
  pinned-core skew; the fix is the pin ratchet in `core-pin-and-floor`.

## Phase details

### Phase `core-fleet-surface` (sase-core)

- Extract the gateway startup from `crates/sase_gateway/src/main.rs` into a public
  `run_gateway_cli(args)` (mirroring `run_federation_worker_cli`), keeping the bin a
  thin shim. Preserve arg parsing, contract-out modes, loopback-by-default
  (`server.rs:70-77`), Tokio lifecycle, and clean shutdown.
- In `crates/sase_core_py`: add console script
  `sase_gateway = "sase_core_rs.gateway:main"` beside the federation worker one, with
  the same Python-shim → `#[pyfunction]` → `py.allow_threads` pattern
  (`lib.rs:12396-12404` is the model). Installed-wheel smoke test that both scripts
  resolve and answer `--help`.
- Narrow PyO3 binding for bootstrap issuance (suggested:
  `fleet_issue_bootstrap(sase_home, request: dict) -> dict`), delegating to
  `FleetCredentialStore::issue_bootstrap` with its 600-second single-use default TTL,
  strict-future expiry, optional `installation_pin` equality check, scope normalization,
  and 0700/0600 store modes intact. The binding adds no policy and never logs the
  secret.
- Health advertisement per the declared contract; also derive `contract.rs:949-954`'s
  `protocol_negotiation.supported_versions` from `FLEET_PROTOCOL_VERSION` instead of the
  hard-coded literal. Regenerate both committed contract snapshots; update
  `routes.rs:6999` (full-body health assert), `server.rs:180-208` smoke asserts, and the
  README route list.
- Verification: `scripts/check.sh` (fmt, clippy -D warnings, workspace tests) plus a
  maturin wheel build with import/console-script smoke, per the repo's CI shape.

### Phase `core-pin-and-floor` (sase)

- Ratchet `sase-core-revision.txt` with `just ratchet-core-revision` once sase-core's
  remote HEAD contains core-fleet-surface. This also carries the pin past `65203fc`,
  clearing the sase-y9 deterministic skew; confirm those five nodes pass at the
  ratcheted pin and record the evidence on sase-y9 (a +1 or note), not by touching
  baselines.
- Bump the `pyproject.toml` `sase-core-rs` floor to the published release carrying the
  new surface. The floor bump requires the PyPI wheel to exist; monitor the release with
  `/sase_monitor`, and if the release-plz release PR needs a human merge, ask via
  `/sase_questions`. If the release is still unpublished after a reasonable wait, land
  the SHA ratchet alone and record the floor bump as an explicit `PROPOSED FOLLOW-UP:`
  rather than pinning to an unpublished version.
- Keep `tools/check_sase_core_rs_bindings` and `tools/validate_sase_core_rs` green; run
  `just check`.

### Phase `machine-bootstrap-cli` (sase)

- New `sase machine bootstrap` subcommand (alphabetical placement; update the pinned
  9-subcommand set in `tests/main/test_parser_machine.py:16`). Options all optional with
  short aliases: `-e/--expires SECONDS` (default: store's 600s), `-j/--json`,
  `-s/--scope SCOPE` (repeatable; empty means the store's default scope set). Help text
  states plainly that the command prints a live single-use secret and where it must run
  (on the target, as the gateway's OS user, against the gateway's SASE home).
- Output: human mode prints a redacted summary (id, expiry, scopes, pin) to stderr and
  the bundle alone to stdout; `--json` prints the raw bundle JSON to stdout. Round-trip
  test: the emitted bundle parses through `MachineService._parse_enrollment_bundle` and
  enrolls against a store fixture.
- Until `core-pin-and-floor` lands, develop against the linked-core build through a seam
  (injected binding callable) so unit tests need no compiled new binding; an integration
  test marked to require the real binding runs once the pin carries it.
- Packaged-command resolution: extend `resolve_federation_worker_command` to check
  `Path(sys.executable).parent / FEDERATION_WORKER_COMMAND` between `shutil.which` and
  the dev-build fallbacks, and add the equivalent resolver for the `sase_gateway`
  console script where target-side tooling needs it. Regression test with a fake venv
  layout.
- Doctor: non-deep `dispatch.worker` check - when `dispatch.machines` is non-empty, the
  federation worker command must resolve (message mirrors the mobile-gateway precedent);
  orphan/zero-machine state SKIPs.

### Phase `tailnet-discovery` (sase)

Carried from the stashed draft, hardened by the third investigation:

- Execute `tailscale status --json` argv-style (no shell) with an enforced deadline,
  bounded output size, and kill-on-timeout; a missing binary yields a
  provider-unavailable diagnostic while local onboarding and other providers stay
  usable.
- Defensive parsing with committed fixtures: map-shaped `Peer` payload, missing/extra
  fields, self-exclusion (never a candidate), DNS names validated and trailing dot
  stripped to form `https://<node-dns>` default endpoints, explicit endpoint overrides
  allowed, offline peers retained as advisory-flagged candidates with reasons,
  malformed/oversized output as structured failure. Tailscale documents the JSON format
  as automation-supported but changeable - parse accordingly.
- Bounded probing: per-peer and overall deadlines; classify each candidate `compatible`
  / `unknown` / `incompatible` per the health `fleet` contract declared above; record
  per-candidate diagnostics (unreachable, TLS failure, unrelated service, timeout)
  distinctly from a legitimately empty tailnet.
- Provider execution: give `_safe_discover` a real wall-clock bound (bounded worker +
  hard deadline) and surface timeout/exception diagnostics instead of a silent `()`.
  Note in-process threads cannot be force-cancelled - a timed-out provider is reported
  and its result discarded; full subprocess supervision for third-party providers is
  `provider-isolation`'s job. Keep hook signatures untouched; if the result shape needs
  a diagnostics channel, extend it compatibly (builtin callers and the fake-provider
  tests both updated).
- Defaults: `builtin@tailnet.enabled: true` and
  `discovery.enabled_providers: [builtin@tailnet]` in `default_config.yml` and the
  code-level defaults (`dispatch/config.py:163-166`), schema updated, explicit
  disablement honored, and an explicitly selected-but-disabled provider explained to the
  user instead of silently skipped (`providers.py:191-192`). Make the builtin@tailnet
  spec's `supports_discovery` truthful. Zero-machine runtime paths must remain provably
  inert (the existing laziness tests stay green).

### Phase `provider-isolation` (sase)

Carried from the stashed draft unchanged in scope: metadata-only inventory in-process;
provider imports only inside a bounded, separately supervised helper subprocess with
deadline and cancellation, only for the selected provider, following the finalizers
loading model; a provider exception or timeout affects only its machines; decide
`dispatch_connection_plan` (add for third parties or record an explicit deferral
decision). Builtin providers may remain in-process but honor the same deadline contract
established in `tailnet-discovery`.

### Phase `machine-init` (sase)

- One machine-owned planner/apply service consumed by all three entry points: new
  canonical `sase machine init`, the `sase init machine` alias, and the init-registry
  machine spec (`init_registry.py` order stays config-then-machine). Update the pinned
  parser/init tests and the stale `docs/init.md` "all four read-only plans" wording
  (five specs; machine runs second).
- Planner honesty within the init contract: `--check`/`--json`/previews are pure and
  offline (no discovery, no provider execution). A zero-machine or all-enrolled registry
  must not leave bare `sase init --check` permanently `needs_attention` - resolve the
  offer-versus-drift tension explicitly (an interactive TTY-gated offer is acceptable;
  perpetual drift is not). Explicit `sase machine init` always supports rescan: existing
  machines are listed, already-enrolled identities are skipped, and selecting one
  existing machine never suppresses discovery of another
  (`init_machine_handler.py:22-30` early-return removed in favor of the shared service's
  reconcile behavior).
- Enrollment: candidates selected explicitly; bundle via `getpass`-style hidden prompt,
  `-B/--bootstrap-file`, or stdin (the `_init_input_func` injection convention keeps it
  testable); reuse the `machine add` outcome handling so quarantine exits non-zero with
  the quarantine reason and JSON rows carry the full enrollment fields. Never infer
  trust from tailnet membership; never overwrite a pin; changed identity routes to
  `sase machine repair`.
- Activation: on a chezmoi-enabled controller, after `write_machine_record`, deploy the
  scoped change through `apply_chezmoi` respecting its tracked-proc contract
  (`targets.py:123`) and its non-raising return, then reload the merged config and
  verify the enrolled record and credential resolve, then run an authenticated hello
  - only then report success. Apply failure or partial enrollment prints actionable
    recovery state (the target may have consumed the single-use bootstrap; say what to
    do next: retry apply, or re-issue bootstrap and `sase machine repair`). Non-chezmoi
    controllers keep today's direct-write behavior plus the reload/hello verification.
- Tests: behavioral coverage of the interactive loop (fixture gateway), quarantine
  reporting, rescan with one enrolled + one new candidate, chezmoi source-vs-applied
  activation (the `source_has_probe/applied_has_probe` reproduction becomes a regression
  test), and offline check purity.

### Phases `fleet-fixture`, `fleet-visuals`, `fleet-perf-faults` (sase)

Carried from the stashed draft verbatim (see the phase frontmatter descriptions). Read
the TUI perf memory first; fleet-visuals also follows the PNG snapshot suite conventions
from the lint/test memory (`just test-visual`, goldens under
`tests/ace/tui/visual/snapshots/png/`).

### Phase `live-apollo-proof` (sase, live operations)

Preparation:

- Runbook doc (suggested `docs/remote_dispatch.md`, linked from the machine CLI help
  epilog): supported installation on Linux/macOS, packaged gateway supervision (systemd
  user unit example for Linux with restart-on-failure; launchd sketch for macOS),
  loopback bind + node-specific `tailscale serve` setup including HTTPS cert readiness,
  bootstrap issuance, controller-side enrollment, verification commands, and the
  partial-enrollment recovery path.
- Read the home `tailnet.md` memory (`sase memory read tailnet.md`) for SSH aliases and
  users: `ssh apollo` (Linux, user bryan). The Mac is best-effort only.

Live execution (record command output evidence as you go; put transcript/capture
excerpts on the phase bead):

1. Update installs: upgrade sase (and its `sase-core-rs` dependency) on athena and
   apollo through the standard uv-tool path to the release/build containing every
   dependency phase; restart `sase axe` (`stop` then `start`, or `ensure`) on both.
2. Apollo over SSH: verify `sase_gateway` resolves from the installed venv (this is what
   the venv-sibling resolution fix enables); start it supervised, loopback at
   `127.0.0.1:7629`, with the gateway's SASE home = apollo's real `~/.sase`; configure
   node-specific `tailscale serve` to that loopback port; confirm
   `https://apollo.tail297af1.ts.net/api/v1/health` answers with the `fleet`
   advertisement.
3. Apollo: `sase machine bootstrap --json` as the same user/SASE home; carry the bundle
   to athena over the SSH session into a mode-0600 temp file; it is single-use and
   expires in 600s, so sequence steps 3-4 tightly.
4. Athena: `sase machine init` - discovery must list apollo as `compatible`; enroll with
   `-B <bundle-file>`; confirm the chezmoi-managed config deployed (applied file, not
   just source), credential resolves, hello verified. Then `sase machine list`,
   `sase machine status apollo`, and the deep doctor lane; all green.
5. Launch 1-3 xsmall remote agents with `%dispatch:apollo` on an eligible configured
   project (trivial observation prompts; xsmall exists exactly for launching agents to
   observe while testing agent features).
6. Athena TUI: `sase ace --tmux` prints the tmux target; drive it with `tmux send-keys`
   and observe with `tmux capture-pane`: Agents tab shows the Fleet strip with apollo
   and counts; open Fleet, see the launched agents; follow one into Focus; view its
   output; stop one of the launched agents from the TUI and confirm the stop lands on
   apollo. Capture panes as evidence.
7. Resilience: restart the gateway process on apollo; confirm the enrollment survives,
   `sase machine status apollo` recovers, and the TUI resumes without restarting ACE.
   Delete the bundle temp file.
8. Best-effort (never a gate): if the Mac is online, repeat target preparation there and
   verify a sleeping Mac does not stall the apollo rows.

If a step fails, fix-forward only within this epic's own surfaces; a defect outside them
is a `PROPOSED FOLLOW-UP:` note plus, where possible, a downgraded-but-honest proof of
the remaining steps. Do not hand-edit credential stores or bootstrap files as a
workaround; the Rust-issued bootstrap contract is the only supported setup path.

## Acceptance gates

| Gate                            | Required evidence                                                                                                                                                                                                                |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Installed target                | A supported install resolves and runs `sase_gateway` and `sase_federation_worker` with no source checkout; bootstrap uses the same identity/store.                                                                               |
| Real discoverer                 | Fixture coverage for missing/extra fields, self, trailing-dot DNS, offline peers, missing CLI, malformed/oversized output, and deadline expiry; live apollo classified compatible.                                               |
| Compatibility and authorization | Unrelated HTTPS service and unknown/incompatible protocol get distinct states; expired/replayed bootstrap and wrong pin fail safely; no secret is ever echoed or logged.                                                         |
| Shared init behavior            | All three init entry points use one flow; explicit rescan finds a new peer beside an enrolled one; checks/previews do no discovery; zero-machine `sase init --check` is not permanently red; empty-registry runtime stays inert. |
| Configuration activation        | On the chezmoi-enabled controller the applied overlay reloads immediately and resolves its credential; apply failure and partial enrollment have a tested recovery path.                                                         |
| Operational completion          | Authenticated hello, `%dispatch:apollo` launches, and TUI-driven follow/manage/stop all succeed live; a gateway restart does not break the enrollment; evidence captured on the phase bead.                                      |
| Acceptance hardening            | Offline fleet fixture powers laziness, PNG, bench, and fault coverage; p95 < 16 ms holds under hung-host/reconnect/burst scenarios; the two missing failure-table tests exist.                                                   |
