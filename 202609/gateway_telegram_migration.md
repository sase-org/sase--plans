---
tier: tale
title: Gateway builtin and Telegram plugin migration
goal:
  The service host can own the mobile gateway and Telegram receiver without duplicate
  supervisors or restart-created pairing challenges.
size: medium
proposed_by: bbugyi200.athena.sase-11y.6
bead: sase-11y.6
create_time: 2026-09-18 05:43:18
status: wip
---

- **PARENT:**
  [202609/service_host_1.md](https://github.com/sase-org/sase--plans/blob/main/202609/service_host_1.md)
- **BEAD:**
  [sase-11y.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11y/sase-11y.6.md)

# Gateway builtin and Telegram plugin migration

## Goal

Complete phase `sase-11y.6` by making the service host the sole prospective owner of the
mobile gateway and Telegram receiver while preserving an explicit compatibility path
whenever the `service_host` beta gate is disabled or an older SASE installation lacks
service-proc APIs. The gateway must launch its binary directly without creating pairing
challenges during restarts, pairing must become an explicit CLI action, and the Telegram
plugin must declare a disabled-by-default daemon service proc without creating a second
`getUpdates` consumer.

## Constraints and decisions

- Reuse `_prepare_mobile_gateway_launch` for the gateway builtin, but pass canonical
  agent/helper bridge commands and exec the returned gateway argv directly. Never route
  the builtin through `sase mobile gateway start`.
- Keep `service.procs.gateway.enabled: false` in core defaults and respect effective
  machine enablement and boot-scoped stops.
- When the existing `service_host` beta gate is enabled and the configured gateway is
  effectively enabled, `sase mobile gateway start` delegates ownership to the host
  instead of binding port 7629 itself. Otherwise it retains the foreground legacy
  behavior.
- Add `sase mobile gateway pair` as the explicit request that POSTs a fresh challenge to
  the already-running configured gateway and prints the challenge fields.
- The phase-launch instruction forbids creating beads, while a new SASE sunset flag can
  only be created through `sase flag new` and therefore creates a flag bead. Use the
  already-approved `service_host` beta gate as the compatibility switch: the Telegram
  tick no-ops only when that gate is on and the `telegram_receiver` service proc is
  configured; gate-off, missing-API, and composition-failure paths preserve legacy rearm
  behavior.
- Do not change Telegram's long-poll receiver loop or offset storage, and do not edit
  athena/apollo overlays; live enablement belongs to the rollout phase.
- Do not change `sase-core`; the existing service config/status contracts and gateway
  CLI provide the required primitives.

## Implementation

1. Extend the gateway integration and public CLI in the SASE repository.
   - Add an explicit pairing request/renderer and a sorted `pair` parser entry.
   - Detect effective host ownership for `start`, clear a gateway stop, start/nudge the
     host as needed, and return with guidance to run `pair` instead of spawning a
     competing foreground gateway.
   - Resolve the `gateway` builtin in the service host to direct gateway argv prepared
     from `mobile_gateway.*`, including canonical agent/helper bridge commands; make
     builtin preparation failures become service-proc failures rather than crashing a
     reconcile cycle.
   - Ensure the foreground service host excludes project-local config before resolving
     gateway settings.
   - Add focused parser, pairing, host-delegation, direct-argv, failure, and
     default-disabled regression tests.

2. Remove the Telegram-specific proc-origin exception from SASE core UI restart logic.
   - Treat any surviving legacy receiver row like ordinary background work; service proc
     rows are host-owned and no longer need the origin-string exemption.
   - Rewrite the focused restart tests to prove only monitor shells retain their
     durable-process exemption.

3. Migrate `sase-telegram` to declarative service ownership.
   - Register a `sase_config` entry point and package `default_config.yml` declaring
     `telegram_receiver` as a disabled daemon running `sase_job_tg_inbound --receiver`,
     with `restart: on-failure` and exit code 0 accepted as clean.
   - Make `ensure_receiver_running` return without submitting a legacy proc when the
     `service_host` beta gate is on and that service proc is configured. Preserve the
     old implementation for gate-off, older-SASE, and config-error compatibility.
   - Add tests for plugin resource discovery/config shape, both compatibility branches,
     and the absence of a legacy submit when service ownership is active.

4. Verify both repositories and finish only the assigned phase.
   - Run targeted SASE tests while iterating, then follow the required SASE lint/test
     memory and run its prescribed repository check.
   - Run `just check` in `sase-telegram`.
   - Review both git diffs for unrelated edits and record any genuinely out-of-scope
     discovery only as a `PROPOSED FOLLOW-UP:` note on `sase-11y.6`.
   - Run `sase bead epic-symbols sase-11y.6`, resolve or re-key any reported symbol,
     then close only `sase-11y.6` with a note naming the checks and ownership behavior
     verified.

## Acceptance criteria

- A host restart launches the gateway binary directly and does not POST a pairing
  challenge or include `sase mobile gateway start` in its child argv.
- `sase mobile gateway pair` is the sole new CLI path that asks an already-running
  gateway for a challenge, while legacy foreground `start` retains current pairing
  behavior only when host ownership is inactive.
- Core defaults keep the gateway disabled, and an enabled host-owned gateway cannot be
  raced by the legacy foreground start command.
- Installing `sase-telegram` contributes a disabled `telegram_receiver` service proc;
  active service-host ownership suppresses the five-second rearm submission, while the
  compatibility branch still works when the gate or API is absent.
- SASE no longer contains a hardcoded `telegram-receiver` origin exception.
- Required checks pass in both changed repositories, no phase epic symbols remain, and
  only `sase-11y.6` is closed.
