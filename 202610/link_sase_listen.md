---
tier: tale
title: Link sase-listen to the SASE project
goal:
  Make sase-listen discoverable as an on-demand linked repository with an accurate
  description of its audio and podcast responsibilities.
size: small
proposed_by: bbugyi200.apollo.49
create_time: 2026-10-02 16:49:23
status: wip
---

# Link sase-listen to the SASE project

## Outcome

Add `sase-listen` to `repos.linked` in `sase/sase.yml` so agents can discover its
purpose and open it with `sase repo open sase-listen`. Publish the same description into
the generated repository inventory in agent instructions.

This is a `small` tale: one agent can make the focused configuration change, refresh its
generated instructions, and verify the result. No phases are needed.

## Description and evidence

Use this exact entry, immediately after `sase-research-artifacts` and before the
`sidecar` mapping, keeping it inside the existing `linked` list:

```yaml
- name: sase-listen
  path: ../sase-listen
  description: >-
    Standalone text-to-speech CLI for turning Markdown into chaptered,
    loudness-normalized MP3 audio editions and publishing private podcast feeds.
```

The description identifies the repo's interface, input, output, and distinctive
responsibilities. It gives agents useful reasons to open the repo without tying the
description to a particular speech provider, playback app, or deployment.

Research used `sase repo open gh:sase-org/sase-listen`. At inspected commit `d74633a`,
its `README.md`, `AGENTS.md`, `pyproject.toml`, `docs/architecture.md`,
`docs/podcast-feed.md`, and `docs/sase-integration.md` establish that:

- It is a standalone CLI with a console entry point and no SASE plugin entry points or
  imports of `sase` / `sase_core_rs`.
- It synthesizes narration from Markdown, packages chaptered MP3s with loudness
  normalization, and publishes private RSS podcast feeds.
- SASE research narration orchestration belongs to `sase-research-artifacts`; Telegram
  delivery belongs to `sase-telegram`. Calling sase-listen an integration plugin would
  misdirect agents working on those features.

The relative path follows the existing sibling-repository convention.
`docs/configuration.md` specifies that linked paths resolve from the project's primary
checkout, and that `auto_clone` defaults to false. Keep this repository available on
demand: omit `auto_clone`, `revision_pin`, and any addition to `plugins.required`. This
task adds source discoverability, not an installation or build dependency.

## Implementation

1. Inspect the current `repos.linked` list and add the entry above exactly once,
   preserving existing entries and the explanatory comment attached to
   `sase-research-artifacts`. Change no application code or dependency files.
2. Use `/sase_memory_write` before refreshing generated instructions. The approved scope
   includes reflecting this new configured repository in `sase/memory/sase.md`, root
   `AGENTS.md`, and the provider instruction shims generated from it. Run
   `sase memory init --check --diff` to inspect the proposed refresh, then
   `sase memory init --no-commit` from the project checkout. Use the generator; do not
   hand-edit its output or add a new memory note. Review generated changes and keep this
   task scoped to the repository inventory refresh. The no-commit option leaves
   completion to the host.
3. Verify the configuration and generated inventory using the checks below. No new
   regression test is needed for an additional declarative repo entry; existing
   configuration and generation checks cover the mechanism.

## Verification and acceptance

- Parse `sase/sase.yml` and confirm one `repos.linked` entry named `sase-listen`, the
  exact sibling path, and the folded description above. Confirm the entry is absent from
  `repos.sidecar` and `plugins.required`.
- Run `sase doctor -C config.repos` and inspect the sase-listen row from
  `sase repo list --json`. Confirm it resolves as a linked repository with the expected
  description and automatic cloning disabled. If the sibling checkout is unavailable on
  the implementation host, report the path issue explicitly; do not substitute an
  ephemeral external checkout path into the config.
- Re-run `sase memory init --check --diff` and confirm the repository inventory is
  current and the generated instructions contain the chosen description.
- Read `lint_and_test.md` through `/sase_memory_read`, run `just fix`, then run the
  required `just check` through `sase tool run check`. Use `/sase_monitor` if
  verification needs a handoff. Do not run `check-full`.
- Inspect the final diff for only the intended configuration and generated instruction
  changes, and run `git diff --check`.

Completion means sase-listen is configured as an on-demand linked source repo, its
description accurately communicates its Markdown narration and podcast responsibilities,
and the required validation passes.
