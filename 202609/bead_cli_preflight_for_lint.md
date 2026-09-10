---
tier: tale
title: Stop a broken venv CLI from masquerading as a missing epic bead
goal:
  Bead-aware lint stages fail with an accurate, actionable environment error when the
  venv's `sase` CLI cannot run, instead of reporting a valid bead as missing and telling
  the agent to delete a load-bearing whitelist entry.
size: medium
proposed_by: bbugyi200.athena.06i
create_time: 2026-09-09 20:00:36
status: wip
---

# Plan: Stop a broken venv CLI from masquerading as a missing epic bead

## Background

`just symvision` failed with:

```
Error: --epic-symbol 'sase-n4(get_usage_limit_config)': bead 'sase-n4' not found. Remove this --epic-symbol entry.
```

That message is a **false diagnosis**, and following its instruction makes things worse.
The investigation established the following, each item verified by running the command
shown:

1. **The whitelist entry is legitimate and load-bearing.** It was added by commit
   `3201e7fdb` ("feat(llm-provider): add usage-limit detection core", epic phase bead
   `sase-n4.1`, 2026-08-16), which added nine `--epic-symbol "sase-n4(...)"` entries to
   the `_lint-symvision` recipe. Later phases consumed eight of those symbols and
   retired their entries; `get_usage_limit_config` is the last one standing.

2. **Bead `sase-n4` exists and is open.** `sase bead show sase-n4` reports
   `IN_PROGRESS`, with phases `sase-n4.1`–`sase-n4.4` closed and an active child epic
   `sase-n4.5` ("Finish usage-limit auto-disable correctness and surfaces"). Its landing
   audit note explicitly says not to close `sase-n4` until the child epic lands.

3. **Removing the entry does not fix the build, it just relocates the failure.**
   Verified by running the recipe's exact Symvision invocation with that one
   `--epic-symbol` argument dropped:

   ```
   Unused public functions/classes. Make these private if they are used only within the file they are defined. ...
     get_usage_limit_config in src/sase/llm_provider/usage_limit_config.py
   ```

   `get_usage_limit_config` has no cross-file non-test consumer in `src/`: inside `src/`
   it is called only on line 214 of its own defining module
   (`src/sase/llm_provider/usage_limit_config.py`, from `detect_usage_limit`); every
   other reference is a test. Per `sase/memory/symvision.md`, test references never keep
   a public symbol alive.

4. **Symvision maps _any_ non-zero `BD_COMMAND` exit to "bead not found", and discards
   the command's output.** Verified with two stub commands:

   ```bash
   printf '#!/bin/sh\nexit 1\n' > /tmp/fake_bd && chmod +x /tmp/fake_bd
   SASE_SYMVISION_BEAD_STATUS_ONLY=1 BD_COMMAND=/tmp/fake_bd .venv/bin/symvision src/sase \
       --epic-symbol "sase-n4(get_usage_limit_config)"
   # -> Error: --epic-symbol 'sase-n4(get_usage_limit_config)': bead 'sase-n4' not found. ...
   ```

   A second stub that also writes a loud diagnostic to both stdout and stderr before
   exiting 1 produces the _byte-identical_ message: Symvision swallows both streams. So
   `tools/sase_bead` has no channel through which it can report "the CLI is broken
   rather than the bead is missing" — improving the shim's own error text cannot fix
   this.

5. **The actual trigger was a broken venv, not a stale whitelist.** `tools/sase_bead`
   runs `.venv/bin/sase bead ...`, and that binary was aborting during startup:

   ```
   ImportError: cannot import name 'sase_core_rs' from partially initialized module 'sase_core_rs'
     (.../sase-core/crates/sase_core_py/python/sase_core_rs/__init__.py)
   ```

   The `sase_core_rs` extension resolved to the linked `sase-core` checkout's Python
   source directory, where the built native module was absent. `just install` rebuilt
   and reinstalled the extension, after which `just _lint-symvision` passes **with the
   `Justfile` unchanged**.

   That resolution path is not an anomaly, it is one of the two shapes `rust-install`
   normally produces. Its `maturin develop --release` branch installs `sase_core_rs` as
   a bare `site-packages/sase_core_rs.pth` holding a single path line pointing at
   `<sase-core>/crates/sase_core_py/python`, and drops the built `sase_core_rs.abi3.so`
   into that checkout directory; its cached-wheel branch instead installs a
   self-contained `site-packages/sase_core_rs/` package. Both shapes were observed in
   this workspace during the investigation. In the `.pth` shape the native module lives
   outside the venv, so `import sase_core_rs` finds the checkout's `__init__.py` and
   fails exactly as above the moment that one file is missing — which is what the
   original traceback shows.

6. **`_setup`'s existing validation does detect this, when it gets the chance.**
   Deleting the installed extension and running `just _lint-symvision` was verified end
   to end: `tools/validate_test_environment` returned its `CORE_BINDINGS_ERROR` bit,
   `_setup` printed "Rebuilding stale or missing sase_core_rs", rebuilt from source, and
   the lint stage then passed with all five `--epic-symbol` entries intact. So the gap
   is not that `_setup` cannot see a broken extension; it is that nothing re-checks the
   one capability these two lint stages depend on at the moment they depend on it, and
   any drift that opens up after validation — concurrent `just` runs in the same
   workspace are a plausible source, several were observed running against shared build
   state — lands as a confidently-worded lie about a bead instead of an environment
   error.

### Root cause

`_lint-symvision` and `_lint-flags` hand `BD_COMMAND=tools/sase_bead` to linters that
cannot tell "the bead CLI failed to start" apart from "the bead does not exist". When
the venv's `sase` CLI is broken for any reason, every bead lookup silently degrades to
"missing", and the linter emits confident, actively harmful remediation advice. `_setup`
is supposed to guarantee a working environment before a lint stage runs, but nothing
verifies the one capability these two stages actually depend on: that `tools/sase_bead`
can execute a bead query at all.

This is a correctness trap for agents, not just a papercut. The error names a specific
file, a specific line to delete, and sounds authoritative; an agent that complies
deletes an in-flight epic's whitelist entry and then, chasing the follow-on "unused
public symbol" error, deletes or privatizes a symbol that epic `sase-n4.5` still needs.

### Non-goals

- **Do not remove or edit the `--epic-symbol "sase-n4(get_usage_limit_config)"` entry**,
  and do not delete, privatize, or add a pragma to `get_usage_limit_config`. Item 3
  above shows the entry is correct as written. It becomes removable only when `sase-n4`
  closes or a real non-test consumer appears — Symvision will say so on its own then.
- Do not vendor, patch, or work around the installed `symvision` package
  (`sase/memory/symvision.md` forbids it). The upstream defect is captured as a task
  bead in step 5 instead.
- Do not change `sase_core_rs` build/install plumbing, and do not try to eliminate the
  `.pth` install shape described in Background item 5. Repairing a drifted workspace is
  already `just install`'s job, and it worked. This plan makes the _failure mode_ honest
  rather than trying to make drift impossible — the preflight is deliberately agnostic
  to why the CLI is broken, so it holds for causes not yet diagnosed.

## Implementation

### Step 1 — Add a bead-CLI preflight tool

Add `tools/check_bead_command` (extensionless Python tool, matching the conventions of
its siblings such as `tools/check_test_wait_helpers` and
`tools/validate_dependency_group`).

- Accept `--bd-command` (default `$BD_COMMAND` or `tools/sase_bead`, mirroring
  `tools/check_feature_flags`'s `DEFAULT_BD_COMMAND` resolution) and resolve it relative
  to the repo root the same way `resolve_bd_command` in `tools/check_feature_flags`
  does.
- Execute one cheap, read-only bead query through that command and capture stdout,
  stderr, and the exit status. Prefer an invocation that succeeds regardless of which
  beads exist (for example a `list`-shaped query), so the probe tests the CLI rather
  than the contents of the bead store.
- Exit 0 and print nothing on success, so the common case is silent.
- On failure, exit non-zero and print an actionable diagnostic on stderr that includes
  the resolved command, the exit status, the captured stderr (truncated), and the
  remedy: the bead CLI in this venv is broken, so bead-aware lint stages would report
  every bead as missing; run `just install` and retry. Explicitly warn that any "bead
  ... not found" advice from those stages must not be acted on until this passes.
- Give it a module docstring explaining _why_ the probe exists — that Symvision and
  `tools/check_feature_flags` conflate a failed lookup with a missing bead, and that
  Symvision discards `BD_COMMAND` output entirely, so the check has to run as its own
  step before the linter starts.

### Step 2 — Gate the two bead-aware lint stages on the preflight

In the `Justfile`, run the preflight as the first command of both `_lint-symvision` and
`_lint-flags`, before the linter invocation, using the same
`SASE_SYMVISION_BEAD_STATUS_ONLY=1 BD_COMMAND=tools/sase_bead` environment those recipes
already set, so the probe exercises exactly the path the linters use.

Keep both recipes' existing arguments byte-for-byte otherwise — in particular the five
`--epic-symbol` entries in `_lint-symvision`.

### Step 3 — Fix the in-repo twin of the Symvision defect

`show_bead` in `tools/check_feature_flags` has the identical conflation:

```python
if result.returncode != 0:
    return None
```

Its docstring says "or `None` if missing", but it returns `None` for _any_ failure,
including a CLI that cannot start. The correct precedent already exists a few lines
above in the same file: `_list_flag_beads` raises
`RuntimeError(f"cannot list flag beads via {resolved}: {detail}")` on a non-zero exit.

Make `show_bead` distinguish the two: a genuine "no such bead" result keeps returning
`None`, while an infrastructure failure raises with the same shape of message
`_list_flag_beads` uses. Determine the genuine-miss signal from the CLI's actual
behavior (check the exit status and/or stderr that
`sase bead show <unknown-id> --format json` produces) rather than assuming one; record
what you found in the docstring. Update the docstring so it no longer claims `None`
means "missing" in all cases.

### Step 4 — Tests

- `tests/test_justfile_lint.py`: add dry-run assertions (following the existing
  `_dry_run` helper and `test_private_symvision_stage_uses_published_cli` style) that
  both `_lint-symvision` and `_lint-flags` invoke the preflight before their linter.
  Also add a regression assertion that `_lint-symvision` still passes
  `--epic-symbol "sase-n4(get_usage_limit_config)"`, so a future agent acting on the
  false "not found" advice trips a test instead of silently breaking `sase-n4.5`.
- New tests for `tools/check_bead_command` (a new
  `tests/test_check_bead_command_tool.py`, patterned on
  `tests/test_check_feature_flags_tool.py`): a stub `BD_COMMAND` that exits 0 makes the
  tool exit 0 and print nothing; a stub that exits non-zero makes it exit non-zero with
  a message naming the resolved command and `just install`.
- `tests/test_check_feature_flags_tool_beads.py` (or the sibling module that already
  covers `show_bead`): assert `show_bead` still returns `None` for a genuine miss and
  now raises for an infrastructure failure.

### Step 5 — File the upstream Symvision defect

Symvision is a published dependency and must not be patched here, so file a task bead
capturing the upstream fix: `--epic-symbol` resolution should distinguish "bead lookup
failed" from "bead not found", and should surface the `BD_COMMAND` stdout/stderr it
currently discards instead of emitting "Remove this --epic-symbol entry" advice it
cannot justify.

Use the `/sase_new_task` skill before creating it, as required. Include the reproduction
from item 4 of the Background section (the two-line stub `BD_COMMAND` that exits 1
yields the byte-identical false error) so the upstream fix has a concrete test case.

### Step 6 — Record the trap in Symvision memory

`sase/memory/symvision.md`'s "Epic symbols" section currently states the entries are
"self-cleaning" and that Symvision "tells you to drop one when the bead is
missing/closed" — advice that is exactly what mislead here. It needs a caveat: a
`bead '<id>' not found` error can be an environment lie, because Symvision reports any
`BD_COMMAND` failure that way; confirm with `sase bead show <id>` before removing an
entry, and confirm the symbol is genuinely unused by re-running Symvision without that
one entry.

Memory files are not ordinary files. Use the `/sase_memory_write` skill first and follow
whichever route it authorizes (direct edit and republish, ask the user, or file a
`memory` task bead). Do not hand-edit `sase/memory/symvision.md` outside that flow.

## Verification

Read `sase/memory/lint_and_test.md` with the `/sase_memory_read` skill before verifying;
it owns the `just check` vs `just check-full` split and the `just install` prerequisite
for ephemeral workspace clones.

1. `just install` first — an ephemeral workspace clone may have drifted dependencies,
   and this plan's whole subject is what happens when it has.
2. `just _lint-symvision` and `just _lint-flags` both pass, unchanged output apart from
   the new silent preflight.
3. Negative check that the preflight actually fires: point the recipes' bead command at
   a stub that exits non-zero (for example by temporarily invoking the preflight
   directly with `--bd-command /tmp/fake_bd`) and confirm the failure names the broken
   CLI and `just install`, rather than any bead being "not found".
4. `just check` as the repo requires after any code change.
