---
tier: tale
title: Enable and initialize the sase agents sidecar
goal:
  The SASE project explicitly enables its public agents sidecar, initializes the missing
  remote safely, and verifies availability without publishing historical agent bundles.
create_time: 2026-09-09 19:53:10
status: wip
---

# Enable and initialize the `sase--agents` sidecar

## Goal

Put the SASE repository into its intended steady state: its managed `agents` sidecar is
explicitly enabled, the public `sase-org/sase--agents` repository exists and is
initialized at the stable machine-level hidden clone path, and read-only inventory/sync
checks no longer report the project as disabled.

This tale deliberately stops short of `sase agent sync`: enabling and seeding the
transport repository should not also publish the backlog of commit-associated agent
transcripts.

## Diagnosis

- `sase/sase.yml` currently has an explicit `repos.sidecar` entry named `agents` with
  `disabled: true`.
- That project-local entry is authoritative: sidecar resolution omits it,
  `sase repo list` has no `agents` row, `sase repo path agents` fails, and
  `sase agent sync --check --json -p sase` reports
  `agents sidecar is disabled or unavailable`.
- The remote `sase-org/sase--agents` does not currently exist.
- Commit `44ccbe84c` introduced the opt-out while implementing consent-gated
  agents-sidecar initialization. The completed implementation transcript records the
  reason: `just check` required the repository to declare the new intrinsic sidecar, but
  that agent did not have authority to create a public remote and publish agent data. It
  therefore used `disabled: true` as a conservative bootstrap state.
- This is not a resolver or synchronization bug. Managed projects already inject/resolve
  the hidden sidecar correctly when it is enabled, and the existing test suite covers
  enablement, hidden-path inventory, consent, initialization, and disabled opt-outs.

## Implementation

1. Enable the existing project-local sidecar declaration in `sase/sase.yml`.
   - Replace `disabled: true` with `visibility: public`.
   - Keep the existing `name` and description.
   - Use an explicit visibility value instead of relying on the public default because
     the repository contains full agent prompts/responses, metadata, and commit
     associations, and the prior opt-out was specifically about lacking publication
     consent.
   - Do not change global defaults, resolver code, schemas, docs, tests, memory files,
     or generated instruction shims; the runtime behavior already implements the desired
     contract.

2. Preview the enabled initialization before creating anything.
   - Run `sase repo init --diff --no-commit`.
   - Confirm that the agents action targets `sase-org/sase--agents` with public
     visibility and the stable hidden path
     `~/.sase/projects/<project-key>/repos/agents`, never a numbered workspace's
     `sase/repos/agents`.
   - Confirm that the seed is limited to the privacy README, schema-v1 empty manifest,
     and tracked empty `agents/` directory.

3. Initialize the missing sidecar through the supported consent-gated workflow.
   - Run `sase repo init --no-commit` interactively.
   - Accept only the agents-specific prompt that names the public
     `sase-org/sase--agents` repository and warns that future synchronization can
     publish full chats and associated metadata.
   - Let repo initialization create the remote, clone it at the hidden machine-level
     path, seed it, and push the initialization commit.
   - Do not run mutating `sase agent sync`; historical/local agent bundles remain
     unpublished by this tale.

4. Verify the resulting repository and project state.
   - Run `sase repo init --check` and require an exit-zero, no-work result.
   - Inspect `sase repo list --json` and require one `agents` sidecar row for project
     `sase`, with slug `sase--agents`, `auto_clone: false`, an existing hidden clone,
     and the canonical GitHub remote.
   - Run `sase repo path agents` and require the stable hidden clone path.
   - Verify GitHub metadata for `sase-org/sase--agents`: it exists, is public, and has
     the initialized default branch.
   - Run `sase agent sync --check --refresh --json -p sase` only as a read-only status
     check. It may report unexported local agents, but it must not report `disabled`,
     `not_created`, a missing clone/upstream, or a configuration error.
   - Inspect the hidden clone without editing it and confirm the expected README,
     manifest, and `agents/` placeholder are present.

5. Validate the tracked SASE repository change.
   - Run `just install` first because this is an ephemeral workspace.
   - Run the mandatory `just check`.
   - Run `git diff --check` and review the final diff. The primary repository diff must
     be limited to the intended `sase/sase.yml` enablement; the sidecar's seed commit
     lives in its own repository.

## Failure handling

- If the remote is created but a later clone, seed, or push step fails, preserve the
  partial state and rerun `sase repo init --no-commit`; initialization is designed to
  adopt existing remotes and seed idempotently.
- If preflight resolves a different owner, slug, host, visibility, or clone path, stop
  before confirmation and diagnose the effective provider/config layers rather than
  authorizing the wrong resource.
- Do not delete or recreate a partially initialized remote as automatic cleanup; that
  would be a separate destructive action.

## Acceptance criteria

- `sase/sase.yml` explicitly enables the agents sidecar with public visibility and
  contains no agents-sidecar `disabled: true` opt-out.
- Public repository `sase-org/sase--agents` exists with the deterministic initial
  sidecar contents.
- SASE inventory and path resolution expose the intrinsically hidden, non-auto-cloned
  sidecar at the machine-level project path.
- Read-only sync status recognizes the sidecar as available and configured, without this
  tale publishing any agent bundles.
- `sase repo init --check`, `just check`, and `git diff --check` all pass.
