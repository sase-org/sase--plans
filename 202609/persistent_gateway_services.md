---
tier: epic
status: done
title: Persistent SASE gateway services on Athena and Apollo
goal: "Athena and Apollo run their loopback-only SASE gateways from enabled, durable,
  chezmoi-managed user services that survive process failure and host restart without
  disrupting the existing Tailscale Serve endpoints or Apollo enrollment.

  "
phases:
  - id: provision-service
    title: Add the managed gateway unit
    depends_on: []
    size: small
    description:
      "provision-service: add and validate the host-scoped chezmoi source for the
      permanent user unit."
  - id: rollout-services
    title: Deploy and verify both gateways
    depends_on:
      - provision-service
    size: small
    description:
      "rollout-services: apply the landed unit on Athena and Apollo, replace the
      transient proof service, and verify restart-safe health."
proposed_by: bbugyi200.athena.0ar.f0.f1
bead_id: sase-yt
create_time: 2026-09-09 19:52:37
---

- **PROMPT:**
  [prompts/202609/persistent_gateway_services.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/persistent_gateway_services.md)
- **BEAD:**
  [sase-yt](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yt/README.md)

# Plan: Persistent SASE gateway services on Athena and Apollo

## Current state and constraints

- Athena and Apollo are Linux hosts owned by the same `bryan` user, and both user
  managers have lingering enabled. An enabled user unit can therefore start at boot
  without an interactive login.
- Both hosts have a persistent Tailscale Serve rule that terminates tailnet HTTPS and
  proxies `/` to `http://127.0.0.1:7629`. Preserve those rules; this plan changes only
  supervision of the loopback backend.
- Athena has no `sase-gateway.service`, no process listening on port 7629, and therefore
  no healthy backend for its existing Serve rule.
- Apollo currently has a healthy gateway on port 7629, but it is owned by the transient
  `sase-gateway-proof.service`. Its unit has no install target and cannot survive a host
  restart. The process runs
  `/home/bryan/.local/share/uv/tools/sase/bin/sase_gateway --bind 127.0.0.1:7629 --sase-home /home/bryan/.sase`
  with `Restart=on-failure`.
- The stable uv-tool path exists on both hosts. Noninteractive SSH on Apollo does not
  include `uv` or `chezmoi` in `PATH`, so remote rollout must invoke known absolute
  paths (notably `/home/bryan/bin/chezmoi`) or deliberately use its login-shell setup.
- The permanent unit belongs in the linked `chezmoi` repository. A separate rollout
  phase depends on the source change being landed so Apollo consumes committed managed
  state rather than an ad hoc copy from an ephemeral workspace.
- Do not alter enrollment credentials or issue another bootstrap bundle. Athena's
  existing controller record for Apollo is healthy and returns an authenticated
  `hello ok`.

## Phase 1: Add the managed gateway unit

In the linked `chezmoi` repository, add
`home/dot_config/systemd/user/sase-gateway.service` as the durable source for both Linux
hosts. Follow the repository's existing user-service convention while making the
gateway-specific behavior explicit:

- Order after and want `network-online.target`.
- Run as a simple user service from `%h/.local/share/uv/tools/sase/bin/sase_gateway`.
- Pass `--bind 127.0.0.1:7629` so the gateway is never exposed directly on a LAN or
  public interface.
- Pass `--sase-home %h/.sase` so the service and `sase machine bootstrap` use the same
  identity, credential store, and installation pin.
- Use `Restart=on-failure` with a short restart delay, matching the supported remote
  dispatch runbook without creating a tight failure loop.
- Install under `default.target`, which combines with the already-enabled user linger
  setting to make the service boot-persistent.

Extend `home/.chezmoiignore` so this Linux systemd unit is selected for Athena and
Apollo only and is not materialized on the Mac or unrelated hosts. Do not add a second
gateway wrapper, copy credentials into the repository, hard-code a workspace path, or
change the existing Tailscale Serve configuration.

Validate the authored unit before landing it:

1. Confirm the rendered/source unit contains one loopback bind, one `%h/.sase` home, the
   stable uv-tool executable, the intended restart policy, and
   `WantedBy=default.target`.
2. Run `systemd-analyze --user verify` against the unit and resolve every diagnostic
   attributable to it.
3. Check the chezmoi ignore/template behavior for Athena and Apollo and confirm the unit
   remains excluded from the Mac.
4. Review the linked repository diff to ensure it contains only the unit and the
   host-scope rule.

Phase completion means the validated unit source is landed in the managed dotfiles
repository and is available for a normal `chezmoi update` on each target. Do not mutate
either live gateway service during this phase.

## Phase 2: Deploy and verify both gateways

Apply the landed chezmoi state and activate the service on one host at a time. Begin
with read-only preflight checks on each host: confirm the uv-tool gateway executable,
user linger, current port-7629 owner, Tailscale Serve rule, and local health state.

Roll out Athena first because it has no existing gateway process:

1. Run the repository-required `chezmoi update -a --force`, then confirm the target unit
   matches the managed source.
2. Run `systemctl --user daemon-reload` and
   `systemctl --user enable --now sase-gateway.service`.
3. Require the unit to be both `enabled` and `active`, non-transient, and loaded from
   `~/.config/systemd/user/sase-gateway.service`. Confirm its effective `ExecStart`,
   restart policy, and loopback-only port-7629 listener.
4. Require `http://127.0.0.1:7629/api/v1/health` to report the SASE gateway as healthy.
   Verify Athena's tailnet HTTPS health endpoint from Apollo; a self-request from Athena
   currently encounters a Tailscale TLS error even though remote access reaches the
   Serve endpoint, so it is not the cross-machine acceptance probe.

After Athena passes, roll out Apollo with an explicit handover:

1. Apply the landed state with `/home/bryan/bin/chezmoi update -a --force`, reload the
   user manager, and verify the permanent unit before disturbing the healthy transient
   process.
2. Stop `sase-gateway-proof.service` immediately before enabling and starting
   `sase-gateway.service`, preventing two processes from competing for port 7629.
3. If the permanent unit cannot become healthy, capture its user-journal diagnostics and
   restore the known-good transient command with `systemd-run --user`, preserving Apollo
   availability while correcting the managed unit.
4. Once the permanent service passes, confirm the proof unit is inactive/transient-only
   and that the sole port-7629 listener belongs to `sase-gateway.service`.

For final acceptance, restart each permanent service with `systemctl --user restart` and
repeat all local and cross-tailnet health checks. From Athena, require
`sase machine status apollo -j` to remain `ok` with `hello ok`, proving the existing
credential and installation identity survived the supervisor replacement. Recheck
`systemctl --user is-enabled`, `is-active`, the non-transient fragment path, and
`loginctl ... Linger=yes` on both hosts. A reboot is unnecessary: enabled unit state,
linger, and successful explicit restarts provide the required persistence evidence
without disrupting unrelated user services.
