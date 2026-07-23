---
tier: tale
title: Fix receipt-owned plugin detection
goal: Receipt-injected SASE plugins appear installed consistently in the Admin Center
  and plugin catalog.
create_time: 2026-07-23 11:49:54
status: done
---

- **PROMPT:** [202607/prompts/fix_receipt_owned_plugin_detection.md](prompts/fix_receipt_owned_plugin_detection.md)

# Fix receipt-owned plugin detection in the Admin Center

## Problem

The SASE Admin Center Updates tab and `sase plugin list/show` report the catalogued `bugyi-chops` community plugin as
not installed even though it is installed in SASE's managed uv tool environment.

The observed runtime state is internally contradictory:

- `uv-receipt.toml` contains `bugyi-chops` as a git-injected requirement.
- `importlib.metadata` in the SASE uv tool interpreter finds `bugyi-chops 0.2.0` and its three `bugyi_chop_*` console
  scripts.
- `sase plugin install bugyi-chops` and the `--git` variant both return `already_installed: true`, so reinstalling is an
  idempotent no-op.
- `sase plugin list/show` and the Admin Center report only two installed plugins and render `bugyi-chops` as absent.
- `sase version` omits `bugyi-chops` because generic plugin discovery only recognizes a `sase_*` entry-point group, a
  `sase-*` distribution name, or a console script beginning with `sase_chop_`.

`bugyi-chops` is a valid catalogued chop package with intentionally explicit script names (`bugyi_chop_*`). Axe resolves
configured chop scripts by their exact names and the chop diagnostics pass, so renaming the scripts or forcing a
plugin-specific entry point is not the right fix for installed-state reconciliation.

The root cause is that plugin mutation commands correctly treat the managed uv receipt as the source of truth, while the
catalog merge uses only heuristic runtime discovery. A receipt-owned package without one of those heuristic signals
therefore becomes installable-but-never-installed in the UI.

## Goal

Make every package explicitly injected into SASE's managed uv tool receipt appear installed in the plugin catalog and
Admin Center when its normalized distribution name matches a catalog entry, while preserving entry-point group metadata
and the existing discovery fallback outside managed uv installs.

After the fix, the live `bugyi-chops` row must report version `0.2.0` as a git install, expose installed-only actions
rather than the install action, and no longer suggest `sase plugin install bugyi-chops`.

## Implementation

1. Extend `src/sase/plugins/installed.py` with a best-effort, injectable reader for the current managed uv receipt.
   - Probe the running SASE interpreter with the existing uv-tool detection utility.
   - When it is a managed install, load and deduplicate the receipt's injected requirements and use their normalized
     distribution names as authoritative plugin membership.
   - Resolve each injected requirement against live `importlib.metadata` to obtain its installed version without
     importing plugin code.
   - Merge receipt-owned packages after the existing SASE entry-point inventory and generic plugin candidates,
     preserving the richer contributed entry-point groups when already known.
   - Treat a missing/non-uv install, malformed or temporarily unreadable receipt, missing distribution metadata, and
     other best-effort inspection failures as non-fatal; retain the current discovery behavior rather than breaking
     read-only catalog/UI loading.
   - Reuse the same receipt-aware membership for `any_plugins_installed()` so onboarding does not claim a managed
     receipt-owned plugin is absent.

2. Keep catalog matching conservative.
   - Continue matching by the existing PEP 503-normalized catalog repo/name candidates.
   - Do not classify every transitive environment distribution as a SASE plugin and do not broaden console-script
     matching to arbitrary executable names.
   - Do not change install/update/uninstall semantics: those operations already use the receipt correctly.

3. Add focused regression coverage in `tests/test_plugin_catalog_installed.py`.
   - Model a `bugyi-chops` distribution with only `bugyi_chop_*` console scripts and prove generic candidate discovery
     alone does not recognize it.
   - Model the same distribution as an injected receipt requirement and assert that the installed index contains version
     `0.2.0` with no fabricated SASE entry-point groups.
   - Assert that existing inventory groups win when a receipt-owned plugin is also found through entry points.
   - Cover deduplication, a missing installed distribution, a non-uv runtime, and receipt/probe failure fallback.
   - Assert receipt-only membership is reflected by `any_plugins_installed()`.
   - Add or extend catalog/JSON-level coverage to prove a community entry named `bugyi-chops` is rendered as installed
     rather than merely testing the internal index.

4. Update `docs/plugins.md` so the documented model matches the implementation.
   - State that managed uv receipt injection is authoritative for catalog and Admin Center installed membership.
   - Explain that SASE entry points and recognized console/distribution naming remain the fallback for unmanaged
     environments and the source of contributed-group metadata.
   - Clarify that exact-name chop script packages need not rename their scripts to `sase_chop_*` merely to appear
     installed when SASE installed them.

## Validation

1. Run the focused installed-catalog and catalog/CLI tests, including the new receipt-only community-plugin regression.
2. Run `just install`, as required for an ephemeral SASE workspace, followed by `just check`.
3. Revalidate against the real managed environment without mutating it by invoking the updated source with the existing
   SASE uv-tool interpreter and catalog cache:
   - `sase plugin show bugyi-chops --offline --json` through the updated source reports `installed: true`, version
     `0.2.0`, and git install type.
   - `sase plugin list --offline --json` reports three installed plugins.
   - The install command remains an `already_installed` no-op.
4. Confirm no reinstall was performed and no uv receipt or installed package set changed as part of the fix.

## Non-goals and safety

- Do not uninstall/reinstall `bugyi-chops`; it is already installed, and a reinstall does not address the detection
  split.
- Do not modify `bugyi-chops` packaging or rename its public console scripts.
- Do not repair the separate local development-version incompatibilities reported by `uv pip check`; they are unrelated
  to the Admin Center false negative and would expand this change's scope.
- Do not change plugin catalog discovery, community-plugin trust warnings, latest-version fetching, or update
  eligibility beyond consuming the corrected installed state.
