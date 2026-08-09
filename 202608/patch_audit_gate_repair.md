---
tier: epic
title: Repair the Patch/stitch terminology gate and finish the test-tree sweep
goal: 'The Patch/stitch terminology gate passes on an ordinary checkout and in CI,
  the audit can report a defect anywhere under tests/ and smoke/ instead of rubber-stamping
  them, the test tree says Patch and stitch, and epic sase-hn.8 is closed with its
  plan marked done.

  '
phases:
- id: gate-repair
  title: Unbreak the gate and reopen the test-tree work list
  depends_on: []
  size: medium
  description: 'gate-repair: stop the lint gate from hard-failing on unmaterialized
    linked repos, restore the strict all-repos invocation for the explicit audit command,
    and add a temporary opt-in flag that re-enables content-aware tests/ and smoke/
    classification so the sweep phases have an enforceable work list.

    '
- id: test-tui-sweep
  title: Clear the ACE TUI test surface
  depends_on:
  - gate-repair
  size: large
  description: 'test-tui-sweep: retire ChangeSpec vocabulary from the 2709 defects
    under tests/ace/tui by switching canonical call sites to the existing Patch helpers,
    fixing prose, and annotating genuine retained contracts, keeping the PNG goldens
    pixel-inert.

    '
- id: test-rest-sweep
  title: Clear the remaining test surface
  depends_on:
  - gate-repair
  size: medium
  description: 'test-rest-sweep: clear the 244 defects under tests/ outside tests/ace/tui,
    make the sase.ace.changespec compatibility tests self-declaring rather than path-exempt,
    and confirm smoke/ stays clean.

    '
- id: land-epic
  title: Make strict classification the default and land epic sase-hn.8
  depends_on:
  - test-tui-sweep
  - test-rest-sweep
  size: medium
  description: 'land-epic: make content-aware tests/ and smoke/ classification unconditional,
    replace the test that pinned the blanket rule, run the full cross-repository verification
    set, close bead sase-hn.8, run symvision, and mark this plan done.'
proposed_by: bbugyi200.athena.sase-hn.8.land
parent_bead: sase-hn.8
create_time: 2026-08-09 04:14:48
status: done
bead_id: sase-hn.8.6
---

- **PROMPT:** [prompts/202608/patch_audit_gate_repair.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/patch_audit_gate_repair.md)
- **PARENT:** [202608/patch_terminology_completion.md](https://github.com/sase-org/sase--plans/blob/main/202608/patch_terminology_completion.md)
- **BEAD:** [sase-hn.8.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hn/sase-hn.8.6.md)

# Repair the Patch/stitch terminology gate and finish the test-tree sweep

## Why this plan exists

Epic `sase-hn.8` ("Finish the Patch/stitch terminology migration and land epic
`sase-hn`") closed all five of its phases, but its landing verification found two
defects that the epic itself introduced. Both are in the final phase's commit
`cac21c867` ("fix: enforce Patch terminology audit gate", `sase-hn.8.5`). The epic
cannot be closed until they are resolved, because unresolved issues caused by an epic
remain that epic's work.

### Defect 1: the new lint gate fails on every ordinary checkout, including CI

`cac21c867` wired `_lint-patch-stitch-terminology` into `just lint` (`Justfile:258`),
`just check` (`Justfile:554`), and `just check-full` (`Justfile:573`). The recipe at
`Justfile:285-286` invokes the audit with no flags:

```
_lint-patch-stitch-terminology: _setup
    {{ venv_bin }}/python tools/audit_patch_stitch_terminology --repo-root .
```

`main()` in `src/sase/patch_stitch_audit.py:597-600` exits `1` whenever default
discovery could not find every expected linked repo, unless the caller passes
`--allow-missing-linked-repos`. `_discover_default_repo_specs`
(`src/sase/patch_stitch_audit.py:143-154`) looks for `sase-github`, `sase-telegram`,
`sase-nvim`, and `chezmoi` under `sase/repos/linked/`, which are materialized on demand
and are absent from a normal workspace and from a CI checkout.

Reproduced on current `master` in a clean workspace:

```
$ just _lint-patch-stitch-terminology
- scanned repos: main, sase-core
- missing expected repos: sase-github, sase-telegram, sase-nvim, chezmoi
error: recipe `_lint-patch-stitch-terminology` failed on line 286 with exit code 1
```

`.github/workflows/ci.yml:123` runs `just lint` on a bare `actions/checkout` of this
repo alone, so the gate fails on every pull request and every master push.
`sase repo list` shows the four repos cloned in at most 3 of 24 workspaces, so
`just check` and `just check-full` are broken for nearly every agent as well. Phase
`sase-hn.8.5` reported `just check-full` green because its own workspace still had the
four repos materialized by phase `sase-hn.8.4`.

The audit must keep scanning every repo that _is_ present, and must keep reporting the
ones it skipped — the summary line already does that — but a routine lint gate that
cannot materialize linked repos must not hard-fail. `--allow-missing-linked-repos`
exists for exactly this caller.

### Defect 2: the test-tree classifier was reverted to a path-only rubber stamp

The entire premise of epic `sase-hn.8` was that `tools/audit_patch_stitch_terminology`
could not report a defect because three of its rules matched on path alone. Phase
`sase-hn.8.1` (`a4a340679`) fixed that, including `_is_compatibility_test_or_fixture`:

```python
def _is_compatibility_test_or_fixture(repo, path, line, match, context) -> bool:
    del repo
    if not (path.startswith("tests/") or path.startswith("smoke/")):
        return False
    return (
        _declares_compatibility_boundary(context)
        or _mentions_retained_serialized_marker(line, match)
        or _mentions_retained_public_marker(line)
    )
```

`cac21c867` reverted it to the blanket form it had before the epic began
(`src/sase/patch_stitch_audit.py:333-339`):

```python
def _is_compatibility_test_or_fixture(repo, path, line, match, context) -> bool:
    del repo, line, match, context
    if not (path.startswith("tests/") or path.startswith("smoke/")):
        return False
    return True
```

The same commit added `test_classifier_accepts_test_tree_fixture_tokens`
(`tests/test_patch_stitch_terminology_audit.py:86`), which locks the blanket behavior
in.

This is the one thing the approved epic plan explicitly forbids:

> When an occurrence is genuinely a retained boundary, do not silence it by loosening a
> path rule.

and it re-creates the exact structural blindness the epic existed to remove: no
occurrence anywhere under `tests/` or `smoke/` can be a defect. It also contradicts
phase `sase-hn.8.1`'s acceptance criteria, on which phases `sase-hn.8.2` and
`sase-hn.8.3` (both of which scoped "the corresponding tests") were built.

Measured on current `master` by restoring only that predicate and re-running the audit:

| Classification             | blanket rule | content-aware rule |
| -------------------------- | -----------: | -----------------: |
| `legacy-data-test-fixture` |         4402 |               1449 |
| `defect`                   |            0 |               2953 |

2953 hidden defects across 295 files, by directory:

| Path                                   | Defects |
| -------------------------------------- | ------: |
| `tests/ace/tui/**`                     |    2709 |
| `tests/test_revert.py`                 |      81 |
| `tests/ace/changespec/**`              |      65 |
| `tests/test_archive.py`                |      45 |
| `tests/ace/test_changespec_archive.py` |      25 |
| `tests/ace/deltas/**`                  |      18 |
| everything else under `tests/`         |      10 |
| `smoke/**`                             |       0 |

The most common matched tokens are `changespecs` (1504), `changespec` (425),
`ChangeSpec` (151), and `ChangeSpecGroupingMode` (117). The work list is a genuine mix
of three kinds, and each kind has a different correct resolution:

1. **Canonical call sites that should use the canonical name.** The canonical helpers
   already exist and the legacy spellings are plain aliases: `make_patch`
   (`src/sase/ace/testing/fixtures.py:12`, aliased at `fixtures.py:45`),
   `PatchGroupingMode` (`src/sase/ace/tui/models/patch_groups/_buckets.py:25`, aliased
   at `grouping_strategy.py:20`), `build_patch_tree`
   (`src/sase/ace/tui/models/patch_groups/_tree.py:151`, aliased at
   `changespec_groups/_tree.py:21`), and the `AcePage(patches=...)` keyword
   (`src/sase/ace/testing/ace_page.py:119` retains `changespecs=` as an alias). Example:
   `tests/ace/tui/test_changespec_jk_navigation.py:11` imports `make_changespec` when
   `make_patch` is available.
2. **Prose describing the current concept.** Example:
   `tests/ace/tui/models/test_agent_groups_layout.py:1` —
   `"""Tree layout shape: project / ChangeSpec / name-root banners."""`; and
   `tests/ace/tui/bench_tui_jk.py:276` —
   `"""Measure j/k latency on the ChangeSpecs tab with 50 synthetic ChangeSpecs."""`
3. **Genuine retained contracts the content-aware rules do not yet recognize.** The
   `changespecs` tab ID is listed under the epic plan's Non-goals as retained, but
   `await page.expect_state("tab", "changespecs")` is not matched by
   `_RETAINED_PUBLIC_MARKERS` (`src/sase/patch_stitch_audit.py`, which lists
   `sase.ace.changespec`, `sase changespec`, `--changespec`, `sase_changespecs`,
   `/sase_changespecs`, `docs/change_spec.md`, `changespec-tags`,
   `changespecs-in-practice`). These need either an in-line annotation — the pattern
   already used at `tests/test_agent_group_revival_e2e.py:231`,
   `initial_tab="changespecs",  # legacy tab id` — or a narrow, justified marker with a
   test.

A test that deliberately exercises a retained alias is legitimate and must keep doing
so; it just has to say so, exactly as `src/` compatibility boundaries already do.

### What verification confirmed is genuinely done

Recorded here so the final phase does not redo it:

- CLI help is canonical: `sase commit --help` reads `Branch/Patch name`,
  `Parent Patch name`, `Patch status`; `sase changespec --help` dispatches to
  `usage: sase patch` and reads "Inspect and maintain Patch lifecycle records."
- The user-facing strings named in the epic plan are fixed in `src/sase/ace/archive.py`,
  `revert.py`, `restore.py`, `src/sase/main/commit_handler.py`,
  `src/sase/workflows/commit/commit_tracking.py`,
  `src/sase/core/status_wire_conversion.py`, and `src/sase/bead/work.py`.
- The garbled `ChangeSpecI` strings are gone; the only remaining `ChangeSpecI` matches
  are the `ChangeSpecInfoPanel` legacy alias declarations.
- Both `DISCOVERED ISSUE` notes on bead `sase-hn` are resolved:
  `src/sase/workflows/commit_utils/entries.py:7` imports `PROJECT_SPEC_SECTION_HEADERS`
  from `sase.ace.patch.section_order`, and `parse_timestamp_value` is public at
  `src/sase/ace/tui/models/patch_groups/_buckets.py:64` with the private alias kept
  module-local.
- `just _lint-symvision` passes; the Justfile carries no `--epic-symbol` entries, so
  nothing expires when `sase-hn.8` closes.
- Maintained docs are clean: one `ChangeSpec` occurrence outside `docs/blog/posts/`, at
  `docs/change_spec.md:49`, describing the legacy `## ChangeSpec` parser block.
- No child bead of `sase-hn.8` carries a `PROPOSED FOLLOW-UP:` note. The single such
  note in the wider `sase-hn` tree (`sase-hn.6`, 2026-08-09T02:19:45Z) was resolved
  within that same phase.
- Integration review: the only two non-epic commits since `sase-hn.8` began (2026-08-09
  00:10:48 EDT) are `57e99f9f6` and `64f9383f1`, both documentation-only from the
  `chop.refresh_docs` agent. Both already use Patch/stitch vocabulary and the audit
  reports zero `docs/` defects at HEAD, so no integration work remains.

## Non-goals

- No behavior, lifecycle, wire-format, or CLI-contract changes. This is terminology and
  audit-fidelity work only.
- Do not rename retained compatibility identifiers. The epic plan's Non-goals list still
  governs: `sase.ace.changespec`, `sase changespec`, `--changespec`, `changespec_name`,
  `changespec_bug_id`, `commit_changespec_name`, `meta_changespec`, the `changespecs`
  tab ID, `COMMITS:` section reading, `all_changespecs.json`,
  `filtered_changespecs.json`, `/sase_changespecs`, `docs/change_spec.md`, and the
  `changespecs-in-practice` blog slug.
- Do not rename test files whose subject is a retained alias (`tests/ace/changespec/**`,
  `tests/ace/test_changespec_archive.py`); fix what they say, not what they are named.
- Do not touch `CHANGELOG.md`, `.beads/`, `sdd/tales/`, `docs/blog/posts/`, or archived
  plans.
- Do not re-sweep `src/**`, `docs/**`, `sase-core`, or the linked repositories. Those
  surfaces are verified clean above.
- Do not close bead `sase-hn`. That epic has its own land agent (`sase-hn.land`, the
  assignee that created `sase-hn.8`), and `sase-hn` becomes closable only once
  `sase-hn.8` is closed by the final phase here.

## Shared conventions for every phase

- `just install` first — workspace dependencies may have drifted.
- Read `sase/memory/symvision.md` through `/sase_memory_read` before touching public
  symbol names.
- The audit is the arbiter. During phases 2 and 3 measure with
  `./tools/audit_patch_stitch_terminology --repo-root . --allow-missing-linked-repos --strict-test-fixtures --json`;
  the `--strict-test-fixtures` flag is introduced by phase 1 and removed by phase 4.
- Never silence an occurrence by loosening a path rule. Either annotate the line so a
  content-aware rule matches it, or add a narrow marker with a test that justifies it.
- Prefer switching a canonical call site to the canonical name over annotating it. Only
  annotate when the test's subject genuinely is the retained alias.
- Run `just check` after changes; the final phase runs `just check-full`.
- ACE TUI tests include PNG snapshot goldens. Renaming a fixture keyword or helper must
  not change rendered pixels; if `just test-visual` reports a diff, inspect
  `.pytest_cache/sase-visual/` before concluding anything, and do not pass
  `--sase-update-visual-snapshots` for a change that was supposed to be inert.

---

## Phase 1: Unbreak the gate and reopen the test-tree work list (`gate-repair`)

**Depends on:** nothing. **Size:** medium.

Goal: make `just lint`, `just check`, and `just check-full` pass on an ordinary checkout
again, and give phases 2 and 3 an authoritative, enforceable defect list — without
turning the gate red in the meantime.

1. Change `_lint-patch-stitch-terminology` (`Justfile:285-286`) to pass
   `--allow-missing-linked-repos`. The summary line still names the repos it skipped, so
   the skip stays visible; it just stops being fatal for a gate that cannot materialize
   those repos.
2. Restore the strict, all-repos invocation for the explicit entry point. `cac21c867`
   made `audit-patch-stitch-terminology` (`Justfile:497-499`) delegate to the private
   recipe; give it back its own direct invocation with no opt-out, so the
   cross-repository verification command still fails loudly when a linked repo is
   missing.
3. Add a `--strict-test-fixtures` flag to `main()` in `src/sase/patch_stitch_audit.py`.
   When set, `_is_compatibility_test_or_fixture` uses the content-aware predicate that
   phase `sase-hn.8.1` shipped in `a4a340679`
   (`_declares_compatibility_boundary(context)` or
   `_mentions_retained_serialized_marker(line, match)` or
   `_mentions_retained_public_marker(line)`); when unset, current blanket behavior is
   retained. Default unset so the gate stays green while phases 2 and 3 work. Document
   in the flag's help that it is a temporary sweep instrument removed by phase 4.
4. Tests in `tests/test_patch_stitch_terminology_audit.py`:
   - assert `just`'s gate invocation shape — the audit exits `0` with missing linked
     repos when `--allow-missing-linked-repos` is passed and `1` when it is not (the
     existing `test_cli_fails_for_missing_expected_linked_repos_unless_allowed` may
     already cover this; extend rather than duplicate);
   - assert that under `--strict-test-fixtures` a `tests/` line whose prose describes
     the current concept as a ChangeSpec classifies as `defect`, and that a `tests/`
     line declaring itself a legacy alias does not;
   - leave `test_classifier_accepts_test_tree_fixture_tokens` in place for now — it
     documents current default behavior and phase 4 replaces it.
5. Record the per-directory strict defect counts in the phase's bead note so phases 2
   and 3 can verify they cleared their slice.

**Acceptance:** `just lint` and `just check` pass in a workspace with no linked repos
materialized; `just audit-patch-stitch-terminology` still exits non-zero there;
`./tools/audit_patch_stitch_terminology --repo-root . --allow-missing-linked-repos`
exits `0`; the same command with `--strict-test-fixtures` exits `1` and reports the 2953
defects; `just check` passes.

---

## Phase 2: Clear the ACE TUI test surface (`test-tui-sweep`)

**Depends on:** `gate-repair`. **Size:** large.

Scope: `tests/ace/tui/**` — 2709 defects. Path-disjoint from phase 3 so the two can run
in parallel.

1. Switch canonical call sites to canonical names: `make_changespec` → `make_patch`,
   `ChangeSpecGroupingMode` → `PatchGroupingMode`, `build_changespec_tree` →
   `build_patch_tree`, `AcePage(changespecs=...)` → `AcePage(patches=...)`, and the
   local variables and helper names that follow from them (`changespecs`,
   `changespec_tree`, `changespec_group_key`, `changespec_grouping_mode`,
   `changespec_group_fold_registry`, `changespec_idx`, `changespec_row`). The heaviest
   files are `test_changespec_grouped_navigation.py` (95),
   `models/test_changespec_groups_layout.py` (91), `test_changespec_grouping_cycle.py`
   (87), `test_changespec_grouping_integration.py` (83), `test_jump_to_entry_history.py`
   (82), `test_changespecs_onboarding.py` (70).
2. Fix prose in docstrings, test names, and comments that calls the current concept a
   ChangeSpec — for example `models/test_agent_groups_layout.py:1` and
   `bench_tui_jk.py:276`.
3. Leave retained-contract values alone and annotate them so a content-aware rule
   matches: the `changespecs` tab ID in `expect_state("tab", "changespecs")` and
   `initial_tab="changespecs"`, saved-state keys, and keymap IDs. Follow the existing
   pattern at `tests/test_agent_group_revival_e2e.py:231`. If one annotation would have
   to be repeated more than a handful of times, propose a narrow addition to
   `_RETAINED_PUBLIC_MARKERS` instead and add a classifier test for it.
4. Keep — and clearly label — the tests whose subject is the legacy alias itself, so the
   aliases stay covered after the canonical rename.
5. The visual suite lives in this subtree. Every change here should be pixel-inert: run
   `just test-visual` and confirm it passes without updating goldens. If a golden
   genuinely must change, inspect `.pytest_cache/sase-visual/` diffs first and justify
   it in the bead note.

**Acceptance:**
`./tools/audit_patch_stitch_terminology --repo-root . --allow-missing-linked-repos --strict-test-fixtures --json`
reports zero defects under `tests/ace/tui/`; `just test-visual` passes with no golden
updates; `just check` passes; the legacy aliases named in Non-goals still resolve and
are still exercised by at least one test.

---

## Phase 3: Clear the remaining test surface (`test-rest-sweep`)

**Depends on:** `gate-repair`. **Size:** medium.

Scope: everything under `tests/` outside `tests/ace/tui/`, plus `smoke/` — 244 defects.
Path-disjoint from phase 2.

1. `tests/test_revert.py` (81), `tests/test_archive.py` (45),
   `tests/ace/test_changespec_archive.py` (25), `tests/ace/deltas/**` (18), and the ~10
   remaining scattered occurrences: apply the same three-way resolution as phase 2 —
   canonical call site, prose fix, or annotated retained contract.
2. `tests/ace/changespec/**` (65) is the compatibility package's own test directory. Its
   subject _is_ the retained `sase.ace.changespec` facade, so most occurrences should
   stay; make each one self-declaring so the content-aware rule matches it rather than
   relying on the path. Do not rename the directory.
3. `smoke/**` currently has zero strict defects. Confirm that holds at the end rather
   than assuming it.

**Acceptance:**
`./tools/audit_patch_stitch_terminology --repo-root . --allow-missing-linked-repos --strict-test-fixtures --json`
reports zero defects under `tests/` outside `tests/ace/tui/` and zero under `smoke/`;
`just check` passes; the retained aliases these tests cover still resolve.

---

## Phase 4: Make strict classification the default and land epic sase-hn.8 (`land-epic`)

**Depends on:** `test-tui-sweep`, `test-rest-sweep`. **Size:** medium.

1. Delete the `--strict-test-fixtures` flag and make its behavior the default:
   `_is_compatibility_test_or_fixture` becomes the content-aware predicate
   unconditionally, matching what phase `sase-hn.8.1` shipped. Remove the flag from
   `main()` and its help text.
2. Replace `test_classifier_accepts_test_tree_fixture_tokens`
   (`tests/test_patch_stitch_terminology_audit.py:86`) with a pair that pins the real
   contract: a `tests/` line declaring itself a legacy fixture is
   `legacy-data-test-fixture`, and a `tests/` line whose prose describes the current
   concept as a ChangeSpec is a `defect`. Fold in the equivalent assertions phase 1
   added under the flag so the file has one coherent contract, not two.
3. Re-run the full verification set on the combined tree: `just install`,
   `just check-full`, `just rust-check`, `just test-visual`, `just docs-check`,
   `just docs-pdf-check`, `sase memory init --check`, `sase skill init --diff`,
   `just validate-committed-plans`, and
   `./tools/audit_patch_stitch_terminology --repo-root . --allow-missing-linked-repos`
   (must exit `0`).
4. Confirm defect 1 stays fixed after the classifier change: `just lint` must pass in a
   workspace with no linked repos materialized. This is the regression that broke CI;
   verify it explicitly rather than inferring it from `just check-full`.
5. Materialize the four linked repos through `/sase_repo` and run
   `just audit-patch-stitch-terminology` (the strict, all-repos entry point) to confirm
   it exits `0` across all five repositories. Phase `sase-hn.8.4` verified this;
   re-confirm it once here because the classifier changed underneath it.
6. Close the epic:
   `sase bead close sase-hn.8 --note "<what was verified across steps 1-5 and this plan>"`.
   The note must record: the two epic-caused defects found at landing and how each was
   fixed; the verification summarized under "What verification confirmed is genuinely
   done" above; that no child bead carried a `PROPOSED FOLLOW-UP:` note; and the
   integration review of the two non-epic commits since the epic began.
7. After closing, run `just symvision`. There are no `--epic-symbol` entries for
   `sase-hn.8`, so nothing should expire — but if the close surfaces newly unused public
   symbols, resolve them by the `sase/memory/symvision.md` decision hierarchy: delete,
   privatize, pragma, and only then whitelist.
8. Set `status: done` in the frontmatter of this plan file. The epic's own plan file
   `plans:202608/patch_terminology_completion.md` already reads `status: done`; leave
   it. Do not touch `plans:202608/patch_and_stitch_terminology.md` — that is `sase-hn`'s
   plan and belongs to `sase-hn`'s land agent.

**Acceptance:** `sase bead show sase-hn.8` reports the epic closed with resolution
`done`; `just check-full` passes; `just lint` passes with no linked repos materialized;
`just audit-patch-stitch-terminology` exits `0` across all five repositories;
`just symvision` is clean; this plan file reads `status: done`.
