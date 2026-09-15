---
tier: tale
title: Repair chezmoi LuaRocks installation and failure reporting
size: medium
goal:
  Repair the chezmoi LuaRocks installer so all required rocks install successfully,
  failed attempts preserve existing tools, and the final output identifies every failed
  rock.
proposed_by: bbugyi200.athena.0lb
create_time: 2026-09-15 12:20:32
status: wip
---

# Repair the chezmoi LuaRocks installation failure

## Scope and sizing

Implement this in the `chezmoi` repository. Open it with
`sase repo open chezmoi -r "Implement the approved LuaRocks installer repair"` and use
the returned path. Read its `AGENTS.md`. Paths below are relative to that repository.
The SASE application and upstream dependency repositories need no planned changes.

This is a `tale` with implementation size `medium`: one agent can complete the bounded
dependency diagnosis, installer repair, and shell regression coverage. An epic would add
unnecessary phase handoffs. Authoring this tale is `large` work under the SASE size
guidance; the implementation size accounts for the remaining dependency diagnosis.

## Problem and evidence

The reported `chezmoi update -v -a --force` output ends with a successful installation
of `luacov 0.17.0-1`, followed by `chezmoi: exit status 1`.

The relevant source is `home/.chezmoiscripts/run_onchange_install_luarocks.tmpl`:

- `install_rocks()` deletes the entire user rock tree, then installs `busted`, `nlua`,
  `llscheck`, `luacheck`, and `luacov`, in that order.
- Each unsuccessful install sets `rc=1`, and the loop continues. The function returns
  that accumulated status after the final rock. There is no final list of failures.
- Commit `5fb37dd6` added this error propagation and the Lua 5.1 bootstrap guard.
  Retaining a failure status is correct: a later success cannot repair an earlier
  missing tool.
- A planning-time execution of the actual `install_rocks()` function with mocked
  `luarocks`, cleanup, and logging reproduced the reported pattern. All-success returns
  0; failure of `busted` or `nlua` followed by successful `luacov` returns 1.
- Read-only inspection of this host found LuaRocks 3.11.1 targeting Lua 5.1,
  `nlua 0.3.2-1`, `llscheck 0.8.0-1`, `luacheck 1.2.0-1`, and `luacov 0.17.0-1` in the
  local tree. `busted` is absent, including its executable. This makes `busted` the
  leading suspect, but does not establish the original dependency error.
- Lua 5.1.5, its development headers, and GCC 14.2 are installed. `luasystem` is the
  first missing dependency in the published Busted 2.3.0 dependency order. Read-only GCC
  syntax checks of all eight C sources at `luasystem` tag `v0.7.1` passed with the
  configured Lua include path and compilation flags. These checks do not exercise
  downloading, linking, or installation, and do not establish that dependency as the
  cause.

Primary package references consulted:
[Busted package metadata](https://luarocks.org/modules/lunarmodules/busted/2.3.0-1),
[Busted rockspec](https://luarocks.org/busted-2.3.0-1.rockspec), and
[LuaSystem package metadata](https://luarocks.org/modules/lunarmodules/luasystem).

No installer, configuration, package installation, or managed home file was changed
during planning. The original full error output is unavailable. Do not present the
missing-dependency notice for `datafile` as the error: the supplied output shows that
dependency resolving successfully.

## Implementation

### 1. Capture the underlying failure before choosing its repair

Recheck the local LuaRocks version, Lua version, configured compiler/include paths, and
installed rocks. Use explicit tree selection for an isolated reproduction of
`luarocks --lua-version=5.1 --tree=<temporary-tree> install --deps-mode=one busted`. Use
temporary build/cache locations where supported, retain complete stdout and stderr, and
capture the real process status if output is piped through `tee`. Keep the current
compiler and Lua configuration so the reproduction is representative. Do not invoke the
existing installer for this step, because it deletes the live rock tree.

If Busted fails, identify the first actual download, build, or installation error and
the failing dependency/version. Apply the smallest durable correction justified by that
evidence in the chezmoi installer or its directly related managed configuration. For
example, a confirmed prerequisite or Lua configuration problem needs that
prerequisite/configuration corrected; an upstream compatibility problem needs a verified
supported release or narrowly scoped workaround with an explanatory comment. Do not
upgrade LuaRocks, pin unrelated packages, disable compiler errors, or replace Lua 5.1
merely because the original tail is ambiguous.

If isolated Busted succeeds, exercise all five required rocks with the same isolated
tree and compare the affected host's actual configuration. Record that the original
dependency failure was not reproduced; do not invent a permanent version pin or claim an
upstream bug. Successful recovery plus the installer corrections below can resolve a
transient failure, but diagnostic changes alone do not establish recovery.

### 2. Make installation recoverable and failures clear

Update `home/.chezmoiscripts/run_onchange_install_luarocks.tmpl`:

- Remove unconditional deletion of `~/.luarocks`. Use LuaRocks' normal installation
  behavior to converge the required packages and let a rerun retry missing packages.
  Preserve unrelated rocks and configuration in the user tree. This removes the
  full-tree deletion; it does not promise transactional rollback of LuaRocks upgrades.
- Keep the five required rocks and their existing order. Continue attempting the
  remaining rocks after a failure, so one failure does not prevent independent tools
  from installing.
- Capture each failed rock's name and original exit code. Preserve live LuaRocks
  stdout/stderr. Print a per-rock failure message and an unmistakable final summary
  naming every failure after the loop, with a retry command consistent with the
  effective Lua version and tree. Send failure diagnostics to stderr.
- Return nonzero whenever any required installation failed and 0 only when all required
  installation commands succeeded. Capture `$?` in a way that does not accidentally
  capture the inverted status of `!` or the status of a logger/`tee`. The last
  successful `luacov` installation must never conceal an earlier failure.
- Preserve the existing behavior that stops before installing rocks when bootstrapping
  LuaRocks fails. Limit bootstrap changes to those justified by the reproduced error.
- Keep Bash compatibility with the repository's Linux/macOS use; ordinary indexed arrays
  are sufficient. A generic logging framework or new installer CLI is not needed.

### 3. Add focused shell regression coverage

Add `tests/bash/install_luarocks_test.sh`, following the existing bashunit conventions
in `tests/bash/`. Prefer testing the actual template script with fake external commands
and isolated paths over copying its functions into the test. The template's only current
substitution is in a comment, so it can be executed as Bash for these tests; also check
rendering separately with chezmoi.

Use a temporary directory per test and stubs for LuaRocks and any bootstrap/package
commands reached by the cases. Adapt the hard-coded utility source path in the harness
or provide a fixture for it. Use task-specific variables such as `test_home`; do not
repurpose the agent's global home or run a real package installation from a test. Tests
must never erase the real rock tree, run sudo, or use the network.

Cover observable behavior:

1. All five installs succeed: all are attempted in order and the script returns 0.
2. Busted fails with a distinctive status, then Luacov succeeds: the script returns
   nonzero and the final stderr summary identifies Busted and its actual status.
3. Multiple failures, including the final rock: every failure is summarized once in the
   final summary, all required attempts occur, and the script returns nonzero.
4. Existing rock/configuration sentinel files survive both successful and failed runs; a
   subsequent successful retry returns 0 without destroying those sentinels.
5. LuaRocks bootstrap fails: the failure propagates, no rock installs occur, and the
   existing tree survives.
6. If step 1 establishes a deterministic prerequisite/configuration defect, add a
   focused regression for that condition and the successful repaired path.

The existing `just test-bash` recipe discovers `tests/bash/*_test.sh`, so no new CI job
or task-runner target should be necessary.

## Verification and completion

From the opened chezmoi repository:

1. Run `bash -n` on the changed template and the new test file. Render the template
   using `chezmoi execute-template` and check the rendered script with `bash -n` too;
   rendering must not execute the installer.
2. Run `bashunit tests/bash/install_luarocks_test.sh`, then `just test-bash`. Check the
   diff for unrelated changes and whitespace errors with `git diff --check`.
3. After the isolated reproduction/repair succeeds, exercise the repaired installer
   against an isolated tree, including a second run. A test-only LuaRocks wrapper can
   translate `--local` to the explicit temporary `--tree` and add `--deps-mode=one`
   after the `install` subcommand while calling the real LuaRocks executable; verify
   that no call targets the live tree. Verify all five required tools are installed and
   Busted starts successfully with that tree's Lua paths. Preserve the reproduction
   error and successful result in the implementation report.
4. Follow the repository's host-owned completion flow. Its `AGENTS.md` requires
   `chezmoi update -a --force` after a commit lands; use the user's verbose form,
   `chezmoi update -v -a --force`, for final live verification after the repaired source
   has landed. Confirm the live required rocks, including `busted`, are present and the
   update exits 0. Coordinate this with the host's landing/apply mechanism rather than
   creating a manual commit or applying an unlanded checkout globally.

Success means all required rocks install and the user's update succeeds, existing
unrelated rocks survive, and future failed installs end with actionable diagnostics and
a nonzero status. Report any unreproduced original error or outstanding live-apply
verification explicitly; passing mock tests alone is not evidence that the original
package failure is repaired. A failure in an unrelated chezmoi script should be reported
separately rather than attributed to this LuaRocks fix.
