---
tier: tale
title: Fix Claude Code agent-CLI update failing with exit 217 (orphaned-npm misdetection)
goal: 'SASE detects a native Claude Code install correctly even when an orphaned npm
  global package lingers, so the agent-CLI update runs `claude update` instead of
  a doomed `npm install -g …`, and the comprehensive update stops reporting Claude
  Code as failed.

  '
create_time: 2026-07-22 08:36:45
status: done
---

- **PROMPT:** [202607/prompts/claude_cli_orphan_npm_detection.md](prompts/claude_cli_orphan_npm_detection.md)

# Plan: Fix Claude Code agent-CLI update failing with exit 217

## Symptom

The ACE comprehensive update ("SASE updated") repeatedly finishes with the warning toast **"⚠ SASE updated with Agent
CLI issues"** and the line:

> • Claude Code: failed — command failed with exit 217. See https://code.claude.com/docs/en/installation

SASE itself updates fine; only the Claude Code agent-CLI leg fails, and it fails on every update attempt.

## Root cause (confirmed end-to-end)

The failure is a **install-method misdetection** caused by a leftover ("orphaned") npm global package coexisting with a
native Claude Code install. The chain:

1. **Claude Code is a _native_ install.** `claude` resolves to `~/.local/bin/claude`, a symlink into
   `~/.local/share/claude/versions/<v>`. `~/.claude.json` reports `installMethod: native` and
   `autoUpdatesProtectedForNative: true`.

2. **An orphaned npm global package still exists.** `@anthropic-ai/claude-code` remains under the active Node's global
   `node_modules` (e.g. an nvm `.../lib/node_modules/@anthropic-ai/claude-code`), pinned at an old version, but its
   `bin/claude` symlink is gone — the native installer took over the `claude` command. Crucially, **`npm ls -g --json`
   still lists the package**.

3. **`_detect_install_method` classifies Claude as `InstallMethod.NPM`.** In `src/sase/agent_clis/detect.py` the NPM
   branch (currently ~lines 180–189) fires when **any** of three conditions holds:
   - `package in npm.installed_packages` ← true because of the orphan
   - `_path_under(executable, npm.root)` ← false; the native binary is elsewhere
   - `_path_under(executable, npm.prefix)` ← false; same reason

   The first disjunct wins on nothing more than an `npm ls -g` listing, even though the resolved executable is the
   native binary and does **not** live in the npm tree. `sase agent-cli list` confirms the misdetection: Claude Code is
   shown with `METHOD: npm` while its version/binary come from the native install.

4. **The wrong update command is planned and run.** Because the method is `NPM`, `plan_agent_cli_status`
   (`src/sase/agent_clis/operations.py`, ~lines 284–299) builds
   `("npm", "install", "-g", "@anthropic-ai/claude-code@latest")`. In this dual-install / nvm-managed-global environment
   that npm command fails, exiting **217**. `runner.py` captures the return code, `operations.py` (~lines 191–208) marks
   the result `FAILED` with `command failed with exit 217. See …`, and `post_update_toast.py` renders the warning toast.

5. **It recurs on every update** because the orphan persists. The number `217` is just npm's own failure exit code
   passed through verbatim — it is _not_ a guard in the `@anthropic-ai/claude-code` postinstall (its `install.cjs`,
   verified for both the pinned and latest versions, only ever sets `exitCode = 1`). The exit code is incidental; the
   defect is that SASE should never be running `npm install -g` for a native install in the first place.

**Verification performed during diagnosis:** running `claude update` (the command the correct `SELF_MANAGED` path would
produce) succeeds with **exit 0** and updates the native install; it also prints its own "orphaned npm package — Fix:
`rm -rf …`" warning. This proves the native/self-managed path is the correct one and the npm path is the broken one.

## The fix

Make **npm ownership of the _active_ binary**, not mere package presence in `npm ls -g`, the trigger for
`InstallMethod.NPM`. The resolved executable is the source of truth for "what `claude` actually runs."

In `_detect_install_method` (`src/sase/agent_clis/detect.py`), change the NPM branch so it classifies as `NPM` only when
either:

- the resolved executable actually lives under the npm tree (`_path_under(executable, npm.root)` or
  `_path_under(executable, npm.prefix)`), **or**
- the package is listed by `npm ls -g` **and the npm tree location is unknown** (both `npm.root` and `npm.prefix` are
  `None`) — a conservative fallback so genuine npm installs on minimal environments where `npm root -g`/`npm prefix -g`
  don't resolve are still detected.

When the npm tree _is_ locatable and the executable is not inside it, the `npm ls -g` listing is an **orphan** and must
not override the true install method. Detection then falls through to `SELF_MANAGED` (native, via `self_update_argv`),
so the planned command becomes `claude update`.

This is a targeted change to the disjunction; the surrounding precedence (bundled → not-installed → npm → homebrew →
self-managed → unknown) is unchanged.

### Why this is safe for genuine npm installs

For a real npm-global Claude, the on-`PATH` `claude` is `<npm-prefix>/bin/claude`, which is under `npm.prefix`, so
`_path_under(executable, npm.prefix)` is true and the CLI still classifies as `NPM` and still updates via
`npm install -g`. Only the orphan-with-native-binary case changes behavior — which is exactly the bug.

## Files to change

- `src/sase/agent_clis/detect.py` — adjust the NPM branch of `_detect_install_method` as described. This is the only
  production change required; `operations.py`, `runner.py`, and the provider metadata are correct and stay as-is.

## Tests

Update and extend `tests/agent_clis/test_detect.py`:

- **`test_npm_evidence_beats_self_update_strategy`** currently encodes the buggy behavior — an executable at
  `/other/tool` (outside npm) with the package merely listed is asserted to be `NPM`. Re-target it to assert npm
  ownership when the executable is genuinely under the npm tree (e.g. executable under `npm.prefix` or `npm.root`), so
  it continues to protect real npm-install detection.
- **Add a regression test** for this bug: a native-style executable located _outside_ the npm tree, with
  `installed_packages` containing the package and `npm.root`/`npm.prefix` set → asserts `InstallMethod.SELF_MANAGED`
  (the orphan does not mask the native install).
- **Add a fallback test**: package listed but `npm.root`/`npm.prefix` both `None` → still `InstallMethod.NPM`,
  preserving detection on minimal environments.
- Consider a higher-level `detect_agent_cli_statuses` test that mirrors the real scenario (a resolved native
  executable + a `run_fn` whose `npm ls -g --json` returns the package while `npm root -g`/`npm prefix -g` return a tree
  that does not contain the executable) and asserts the resulting status is `SELF_MANAGED`.

Optionally add an `tests/agent_clis/test_operations.py` assertion that a `SELF_MANAGED` Claude status with
`self_update_argv=["update"]` plans `(executable, "update")` rather than an `npm install -g` argv, to lock in the
end-to-end command choice.

## Validation

Because this repo uses ephemeral workspace checkouts, run `just install` first, then:

- `just check` (ruff + mypy + tests) must pass.
- Targeted: `just test tests/agent_clis` (or the equivalent) for fast iteration.
- Manual sanity in the affected environment: `sase agent-cli list` should show Claude Code with `METHOD: self managed`
  (not `npm`) while an orphan is present, and `sase agent-cli update claude -n` should plan `claude update` rather than
  `npm install -g …`.

## Risks and considerations

- **Rust core boundary.** Per `rust_core_backend_boundary`, confirm there is no equivalent install-method detector in
  `../sase-core`. This logic is host-environment probing (spawning `npm root/prefix/ls -g`, filesystem path checks) that
  lives entirely in Python under `src/sase/agent_clis/`; the fix should stay Python-side. If a mirror exists in
  sase-core, mirror the change there instead and route Python through it.
- **Degraded npm probing.** The conservative fallback (package listed but tree location unknown → still `NPM`) avoids
  regressing genuine npm installs on hosts where `npm root -g`/`npm prefix -g` don't resolve.
- **No behavior change for non-Claude runtimes.** Codex/Qwen (genuine npm) keep their npm classification because their
  executables are under the npm tree; Antigravity (`native`) and bundled runtimes are unaffected.

## Out of scope (possible follow-ups)

- Surfacing the orphaned-npm condition to the user (SASE could detect and warn, echoing `claude update`'s own "Fix:
  `rm -rf …`" hint) is a nice-to-have but not required to stop the failure.
- Cleaning up the user's specific orphaned package is an environment remediation, not a code change: removing the
  leftover `@anthropic-ai/claude-code` from the active Node's global `node_modules` would also unblock updates, but the
  detection fix is what makes SASE robust regardless of whether the orphan is ever cleaned up.
