---
tier: tale
title: Fix Telegram bead project discovery
goal:
  Telegram's /bead command reliably lists active beads from enabled project workspaces
  and never substitutes an unrelated ambient bead store when project discovery fails.
size: medium
proposed_by: bbugyi200.athena.w4
create_time: 2026-08-08 17:15:35
status: wip
---

# Fix Telegram `/bead` project discovery and ambient-store fallback

## Problem

Telegram's `/bead` picker currently reports 15 `bryan-*` epic beads even though the
active SASE project has a different set of open and in-progress beads. The picker is not
reading stale Telegram state; it is rendering the successful output of the wrong bead
store.

The failure is reproducible as follows:

1. `sase project list --state=enabled --json` fails in a fresh process with an
   `ImportError` for `resolve_project_alias_ref` from a partially initialized
   `sase.project_aliases` module.
2. The import chain is
   `project_aliases -> project_alias_prompts -> sase.xprompt -> loader_memory -> sase.memory -> memory.read_log -> project_aliases`.
   The eager `sase.memory.notes` dependency added to the xprompt memory loader closes
   the cycle only when project aliases are imported first, which is the order used by
   the `sase project` command. Existing in-process tests import the modules in a
   favorable order and therefore miss the regression.
3. `sase-telegram`'s `_iter_known_project_workspaces()` logs any failed or malformed
   `sase project list` result and returns the same empty list used for a legitimate
   zero-project result.
4. `_show_bead_selection()` interprets that empty list as permission to run
   `sase bead list --status=open --status=in_progress --format=json` without a project
   `cwd`. The Telegram chop's ambient directory is the user's home directory, whose
   legacy home bead store contains exactly the 15 open `bryan-1` through `bryan-f` epics
   shown in the screenshot.

The fix must restore project enumeration at its source and make the Telegram boundary
fail closed so a future discovery regression cannot silently display a valid-looking but
unrelated bead store.

## Implementation

### Restore fresh-process SASE project imports

- Remove the eager xprompt-to-memory package import edge that creates the cycle while
  preserving the public memory-xprompt loader behavior. Prefer deferring the memory note
  implementation import until memory xprompts are actually loaded (with
  type-checking-only imports where needed) over broad package initialization changes.
- Add a regression test that starts a fresh Python interpreter with the same import
  order as the `sase project` entry path. Assert that importing the project handler (or
  invoking the project-list entry path) succeeds without a partially initialized module
  error. The test must not rely on another test having imported `sase.memory` first.
- Keep the existing memory-xprompt discovery, validation, precedence, and metadata tests
  green so the cycle fix does not disable or reorder the feature that exposed the bug.

### Make Telegram project discovery fail closed

- Change enabled-project discovery to distinguish these states explicitly: successful
  discovery with projects, successful discovery with zero projects, and discovery
  failure (missing command, nonzero exit, invalid JSON, or invalid payload). Do not
  encode failure as an empty project list.
- In `/bead` list handling, retain the deliberate project override and legitimate
  zero-project/context compatibility paths, but never run an unscoped ambient
  `sase bead list` after project enumeration has failed. Send one concise actionable
  chat message and keep the full subprocess diagnostic in logs.
- Apply the same distinction anywhere project enumeration is reused for `/bead <id>`
  lookup so a discovery error cannot trigger an unrelated ambient-store search or
  obscure a successful chat-scoped lookup.
- Add focused Telegram tests for nonzero project-list exit, missing executable,
  malformed JSON, and a successful empty list. Assert that failure cases do not launch
  an ambient bead-list subprocess, while the valid empty/context behavior remains
  compatible. Preserve the existing all-project aggregation, active-status filtering,
  JSON parsing, duplicate-ID labeling, callback routing, and bounded error rendering.
- Update the inbound integration documentation to state that project enumeration
  failures are reported rather than falling back to the chop's working directory.

## Verification

- In the SASE repository, run `just install`, the focused fresh-interpreter import and
  memory-xprompt tests, then `just check` as required for repository changes.
- From a neutral directory, run `sase project list --state=enabled --json` and confirm
  it exits successfully with the enabled-project JSON contract.
- In the linked `sase-telegram` repository opened through `sase repo open`, run
  `just install`, the focused `/bead` and bead-format tests, then `just check`.
- Reproduce a project-list failure in the Telegram test harness and confirm the bot
  emits the bounded discovery error without executing ambient `sase bead list`.
- Re-run the successful discovery path against representative enabled projects and
  confirm the rendered picker entries come only from their resolved workspaces, with no
  `bryan-*` entries sourced from the user's home bead store.

## Scope and safety

- Do not modify, close, or migrate the legacy `bryan-*` beads; they are evidence of the
  wrong-store read, not the cause of this incident.
- Do not change bead status semantics or the open/in-progress filters.
- Keep changes limited to import-boundary repair in SASE plus discovery/error handling,
  tests, and directly affected documentation in `sase-telegram`.
