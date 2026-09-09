---
tier: tale
title: Rename the task status command to task-status-hooks
goal: "Make `bob task-status-hooks` the canonical public command while preserving hidden
  compatibility dispatch for both older spellings and keeping runtime behavior, output
  schemas, documentation, and release checks coherent.

  "
create_time: 2026-09-09 19:53:25
status: wip
---

# Plan: Rename the Task Status Command to `task-status-hooks`

## Context

The command is registered publicly as `task-status-setter`, dispatched to the native
`TaskStatusSetter` implementation, and hard-codes that spelling in its Clap identity,
examples, error prefix, and human-readable result heading. The same name appears
throughout the CLI integration suite, fixture paths, README, dedicated command
documentation, and install smoke recipe. The older `mark-next-tasks` spelling already
remains available as a hidden compatibility alias whose help and JSON behavior resolve
through the canonical native command.

This repository's most recent rename of this exact command established the compatibility
policy to follow: make the new name the only public spelling while keeping existing
spellings callable but absent from top-level help. A read-only search of the configured
`bob-plugins` and `chezmoi` integration repositories found no additional callers, so
this change does not require a coordinated cross-repository rollout.

## Design and Compatibility Decisions

- Register `task-status-hooks` as the sole public command name, in alphabetical order,
  and use it in the top-level example. Keep `task-status-setter` and `mark-next-tasks`
  as hidden aliases that dispatch to the same native command. Alias invocations should
  stay quiet, return the same status and data as the canonical invocation, and render
  canonical `bob task-status-hooks` usage when help or argument errors are requested.
- Rename the native command/module identity and command-focused internal paths to
  `TaskStatusHooks`/`task_status_hooks` so future work does not perpetuate a stale
  canonical name. Do not change the synchronization algorithm, options, environment
  variables, vault mutations, exit-code contract, or JSON field schema.
- Make every public-facing command reference canonical, including the native help text,
  examples, human result heading, errors, README, command guide, and release smoke
  commands. Mention the two old names only where compatibility is intentionally
  documented or asserted.
- Preserve the project's CLI rules: excellent `-h|--help`, alphabetical public
  subcommands and options, existing short aliases for every public long option, and
  current color/plain-output behavior.

## Implementation

1. Update command registration and native dispatch in `src/runner.rs` and
   `src/native.rs`. Replace the public table entry with `task-status-hooks`, update the
   top-level example, add `task-status-setter` beside `mark-next-tasks` in the hidden
   alias table with descriptions naming the new canonical command, and rename the native
   enum variant/module wiring without introducing a second implementation path.

2. Rename `src/native/task_status_setter.rs` to `src/native/task_status_hooks.rs` and
   make its canonical command constant the single source for native Clap usage,
   examples, diagnostics, serialization context, and human-readable headings. Audit the
   implementation for embedded old spellings, changing user-visible or command-identity
   text while leaving task-status domain terminology and behavior intact.

3. Rename the dedicated guide to `docs/task-status-hooks.md`, update its title, usage,
   canonical-name and compatibility notes, and authoritative-pass references, then
   update all README links and examples, environment-variable descriptions,
   local-install/release smoke instructions, and migration text. Update the `justfile`
   install smoke recipe so a packaged install exercises the new public command. Ensure
   the renamed guide remains included by the existing package manifest.

4. Move command-specific fixtures from `tests/fixtures/task_status_setter` to a matching
   `task_status_hooks` path and update the CLI integration suite's test names, temporary
   labels, fixture includes, invocations, expectations, and failure messages. Strengthen
   the compatibility coverage so both hidden aliases:
   - execute through the native-only path even when script fallback is enabled;
   - show `bob task-status-hooks` in help and argument diagnostics;
   - produce the same successful JSON result as the canonical spelling; and
   - remain absent from top-level help while `task-status-hooks` appears in the correct
     alphabetical position and examples.

5. Audit all tracked references to `task-status-setter`, `task_status_setter`, and
   `TaskStatusSetter`. Retain only explicit legacy alias/migration assertions; remove
   stale public, internal-identity, path, and test references. Also verify
   `mark-next-tasks` occurs only in the intended compatibility documentation and
   coverage.

## Verification

- Exercise focused CLI tests for canonical help, native-only routing, option order,
  top-level help ordering/visibility, canonical-vs-alias JSON parity, and one
  representative idempotent fixture sync.
- Run `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and the full
  `cargo test` suite (or the equivalent `just all`) to catch stale module, fixture, and
  command references across all targets.
- Run `just install-smoke` to build and install the package in isolation and confirm the
  new public command's help works from the installed binary.
- Run `cargo package --list` and confirm `docs/task-status-hooks.md` is packaged and the
  removed guide/path is not.
- Re-run the tracked-reference audit after formatting and tests so remaining old
  spellings are demonstrably limited to the two intentional hidden aliases and their
  migration/compatibility coverage.

## Acceptance Criteria

- `bob task-status-hooks` is the only task-status command displayed by top-level help,
  its placement and options are alphabetical, and all of its help, examples,
  diagnostics, and human result labels use the new spelling.
- `bob task-status-setter` and `bob mark-next-tasks` remain callable hidden aliases and
  expose canonical help/output without appearing in the public command list.
- Canonical and alias invocations preserve existing exit codes, JSON contracts, vault
  edits, dry-run behavior, and idempotence.
- Source identities, tests, fixtures, documentation paths, README references, install
  smoke coverage, and packaged files consistently use `task-status-hooks`, with old
  spellings retained only for deliberate compatibility.

## Out of Scope

Do not alter task ranking, dependency traversal, Blocked-state reconciliation,
Pomodoro-link cleanup, option names, environment variables, output schema, or external
plugin behavior. No linked-repository changes are needed for this rename.
