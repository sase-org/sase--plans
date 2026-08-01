---
tier: tale
title: Fix pyscripts closer-directory package false positives
goal: Package directories no longer trigger Rule 2 while genuine closer standalone-script directories remain invalid.
proposed_by: bbugyi200.athena.sase-de
create_time: 2026-08-01 10:25:02
status: done
---

- **PROMPT:** [202608/prompts/fix_pyscripts_closer_package.md](prompts/fix_pyscripts_closer_package.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-de](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-de.md)
- **COMMITS:**
  - [93dfb63](https://github.com/sase-org/sase/commit/93dfb63587ece702ffde02b752cfd293da7eb2d8) — fix: update vendored
    pyscripts validator

# Fix pyscripts closer-directory package false positives

## Context

`just _lint-pyscripts` currently reports that the root `tools/sase_bead` script should move into `tests/ace/tui/tools/`
because a clan-context test contains the longer identifier `sase_beads`. The basename search is intentionally
substring-based, but the purported closer directory contains an `__init__.py` and is therefore a Python package. The
validator already skips package `scripts/` and `tools/` directories as sources of standalone scripts, so they must not
be candidates for Rule 2 placement either.

The canonical `pyscripts` source lives in the linked chezmoi repository. The dated copy in this repository must be
refreshed from that source with `pyvendor`, not edited directly.

## Implementation

1. Update the canonical `pyscripts` implementation so Rule 2 considers only non-package `scripts/` and `tools/`
   directories when looking for a closer standalone-script directory. Keep package-directory discovery available for
   verbose reporting and retain the existing skip behavior during validation.
2. Exercise the canonical script against a focused temporary repository shape that contains a root script reference
   beneath a package `tools/` directory, confirming that this valid layout passes. Also exercise a corresponding
   non-package closer directory, confirming that the real Rule 2 violation is still reported.
3. Re-vendor the corrected canonical script into this repository with `pyvendor`, allowing it to replace the prior dated
   vendored copy according to the tool's normal workflow.

## Verification

1. Run `just _lint-pyscripts` and confirm the original clan-context false positive is gone.
2. Run `just check` in the SASE repository, as required for repository file changes.
3. Review both repositories' diffs to confirm that only the canonical script and its generated vendored copy changed,
   with no commit, branch, or PR created.
