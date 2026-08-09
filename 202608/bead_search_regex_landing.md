---
tier: epic
title: Finish and land opt-in regex bead search
goal: "Epic sase-i1 is complete in normal published installs and local development:
  regex matching works consistently in both lanes, literal search keeps its cheap
  substring path, the released core floor is correct, all follow-ups are routed, and the
  epic is closed with its linked plan marked done.

  "
phases:
  - id: published-floor
    title: Restore the published binding floor
    depends_on: []
    size: small
    description: "published-floor: require the already-published regex-capable core,
      refresh the lock, smoke the minimum wheel, and restore normal-install
      compatibility.

      "
  - id: core-correctness
    title: Correct core match semantics and the literal fast path
    depends_on: []
    size: medium
    description: "core-correctness: separate match truth from highlight ranges, restore
      the cheap literal path, support zero-width regex matches, unify errors, and add
      Rust tests.

      "
  - id: adopt-corrected-release
    title: Adopt the corrected core release and verify both lanes
    depends_on:
      - published-floor
      - core-correctness
    size: medium
    description: "adopt-corrected-release: wait for the corrective core release, raise
      the floor, add cross-format integration coverage, and run exhaustive verification.

      "
  - id: land-epic
    title: Verify and close epic sase-i1
    depends_on:
      - adopt-corrected-release
    size: small
    description:
      "land-epic: re-audit the integrated tree, record every follow-up outcome, close
      sase-i1 without force, run post-close Symvision, and mark the plans done."
proposed_by: bbugyi200.athena.sase-i1.land
parent_bead: sase-i1
create_time: 2026-08-09 09:05:10
status: wip
---

- **PROMPT:**
  [prompts/202608/bead_search_regex_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/bead_search_regex_landing.md)
- **PARENT:**
  [202608/bead_search_regex.md](https://github.com/sase-org/sase--plans/blob/main/202608/bead_search_regex.md)

# Finish and land opt-in regex bead search

## Goal

Finish epic bead `sase-i1` so `sase bead search -e/--regex` works through both the Rust
fast path and Python fallback in normal published-package installs, while literal search
preserves its cheap substring-matching path. Correct the defects found by the land
audit, integrate the now-published core release, verify the complete tree, route every
follow-up, close `sase-i1`, run post-close Symvision, and mark its linked plan done.

## Landing audit evidence

- `sase-i1.1` landed core commit `721f20d7710db7a53d622d1527d5be5d255c68b7` and closed
  done. The implementation added the flag, matcher, rendering, and binding, but
  `SearchMatcher::is_match` calls `byte_ranges` rather than a match-only path.
  Consequently literal matching allocates the folded offset table for every searched
  field, contrary to the epic's fast-path invariant, and a valid zero-width regex such
  as `\\b` does not match anything because zero-length highlight spans are discarded.
  Matching and highlighting must be separate operations.
- Rust reports invalid patterns as `invalid regex: ...`; compact/JSON fast-path output
  therefore differs from the Python/full fallback, which rewrites it to
  `invalid search regex: ...`. All formats must return exit code 2 with the same
  `Error: invalid search regex: ...` prefix.
- `sase-i1.2` closed canceled while release `0.21.2` was not yet published. The release
  commit `c416cd0b7db4fbf61be4523f3c9ecbe037361a9b` and tag `v0.21.2` are now on
  `sase-core` master, and PyPI has five `0.21.2` artifacts. The released binding has
  signature
  `(beads_dir, query, statuses=None, issue_types=None, tiers=None, limit=None, regex=False)`
  and a direct regex-keyword smoke passed.
- `sase-i1.3` landed main-repo commit `a3a536a033daebf647439bde081d7e609a8dc99e`, but
  `pyproject.toml` and `uv.lock` still allow/resolve `sase-core-rs 0.21.1`. After a
  normal `uv run` synchronization, the environment loads the `0.21.1` binding without
  `regex`; 14 focused CLI/facade tests fail with
  `TypeError: bead_search() got an unexpected keyword argument 'regex'`. Raising the
  published floor is urgent compatibility work, not optional cleanup.
- Seven non-epic commits landed in `sase` after `sase-i1` began and before its CLI
  commit (`9591b4e37`, `11cd8634d`, `fcc9be44f`, `a787f36f`, `f35fa9548`, `7c7de9c9`,
  `c2c8e883`). Their source changes do not overlap bead search. The one task-workflow
  change in `fcc9be44f` correctly retains literal distinctive-term bead searches and
  should not be converted to regex. Core commit `5c555dcda` landed after the regex core
  commit and shares release `0.21.2`; it does not change bead search.
- Follow-up routing already completed by the land audit: the `sase-i1.1` commit-resume
  proposal is already attached, with the proposing bead named, to active Patch/stitch
  epic `sase-hn.8`; `sase-i1.3`'s two VCS-tag flake entries are already tracked by
  `sase-hk`/`sase-cw` and active flake epic `sase-h8`, while its plan-approval node is
  already corroborated on umbrella task `sase-ct`. The phase observed the baseline gate,
  not another independent test failure, so no extra +1 is warranted. The original plan's
  explicitly out-of-scope Rust/Python matching-line snippet gap is now ready task
  `sase-i4` (medium). Do not absorb `sase-i4` into this epic.

## Invariants

- Plain search does not compile a regex and its per-field truth check remains one
  lowercase conversion plus substring containment; byte-offset work is rendering-only.
- Regex truth uses the compiled regex's match operation, including valid zero-width
  matches. Highlight range generation may omit empty spans but cannot alter whether a
  bead matches.
- The regex is compiled once per search and reused for matching and rendering. Keep an
  explicit bounded regex compilation size limit and the existing case-insensitive
  default/inline opt-out behavior.
- Compact, JSON, and full formats agree on matching, validation, ordering, limits, and
  invalid-pattern exit status/text. JSON retains the additive `regex` boolean.
- A normal dependency synchronization must install a published core that supports the
  keyword used by Python. Local source overrides are additional development coverage,
  not a substitute for the published-floor smoke.
- Work in the linked core checkout only after opening it with
  `sase repo open sase-core`; work in the plan sidecar only after opening `plans` the
  same way.
- Do not change the pre-existing compact-snippet matching-line behavior tracked by
  `sase-i4`.

## Phase 1: Restore the published binding floor

ID: `published-floor`

Depends on: none

Size: small

In the `sase` repo, rebase on current master and check whether concurrent epic
`sase-i3.3` has already raised the floor. Ensure `pyproject.toml` requires at least
published `sase-core-rs 0.21.2` and refresh `uv.lock`; preserve any higher compatible
floor already landed. Run a published-wheel smoke in a throwaway venv that imports the
selected minimum and calls `bead_search(..., regex=True)`. Confirm
`tools/smoke_sase_core_rs_telemetry --print-minimum` reports the selected minimum and
`tools/validate_sase_core_rs_version --published-minimum` passes. Run `just install`,
then run the focused Python CLI, facade, project-delegation, and fast-path tests through
a command that preserves the local core override (not bare `uv run`, which can resync
the environment). Run `just check` and commit the compatibility-floor repair.

## Phase 2: Correct core match semantics and the literal fast path

ID: `core-correctness`

Depends on: none

Size: medium

Open the linked `sase-core` repo through `/sase_repo` and rebase on current master.
Refactor the shared bead matcher so matching and highlight-range extraction are
separate:

1. A match-only method uses `value.to_lowercase().contains(needle)` in literal mode and
   `regex.is_match(value)` in regex mode.
2. The range method retains the Unicode-aware literal offsets and regex `find_iter`,
   dropping only zero-length spans.
3. Pattern compilation uses an explicit bounded size limit and returns the canonical
   `invalid search regex: ...` validation message.

Add regression tests proving a zero-width pattern matches fields while producing no
empty highlight spans, literal matching does not depend on range generation, literal
metacharacters remain literal, and compact/JSON invalid-pattern errors use the same
prefix and exit code. Preserve one matcher compilation per search and all ordering,
filter, and limit behavior. Run `cargo fmt --all -- --check`,
`cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`.
Commit as a release-eligible fix and ensure it reaches core master so release automation
can publish it; do not hand-edit crate versions.

## Phase 3: Adopt the corrected core release and verify both lanes

ID: `adopt-corrected-release`

Depends on: `published-floor`, `core-correctness`

Size: medium

Wait for the release-plz release containing `core-correctness` to merge, tag, and
publish to PyPI; do not cancel this phase merely because publication is asynchronous.
Verify the exact release commit and published artifacts, then raise the `sase-core-rs`
minimum to that release (preserving the `<0.22.0` ceiling unless current master has
moved it deliberately) and refresh `uv.lock`. Update Python compatibility/error
normalization only if needed after the canonical Rust error changes, and add integration
coverage that exercises compact/JSON fast path and full Python fallback against the same
invalid pattern and zero-width match. The tests must assert identical exit code 2 and
`invalid search regex` text across formats, plus successful zero-width matching without
empty ANSI highlights.

In a throwaway venv, install exactly the new minimum and smoke the binding with
`regex=True`. Run the published-minimum telemetry/version tools, `just install`, all
focused bead-search/fast-path/facade/project tests, and `just check-full` because this
phase changes the dependency floor and integrates two repositories. If the full suite
finds unrelated reproducible flakes, route evidence through `/sase_new_task` rather than
weakening the gate. Commit the adoption/integration change.

## Phase 4: Verify and close epic sase-i1

ID: `land-epic`

Depends on: `adopt-corrected-release`

Size: small

Re-run the complete land audit on the rebased combined tree: show `sase-i1` and every
child, re-read all notes and the linked plan, inspect the core/main epic commits plus
the corrective commits, and review non-epic commits that landed after this plan was
written for new integration points. Confirm normal published installs and local-source
development installs both pass the regex contract. Run the core workspace gates and
`just check-full` one final time.

Close `sase-i1` with a note that records the source/commit verification, the immediate
and corrected floor versions, the cross-format/zero-width/literal-performance fixes, the
integration review, and every follow-up outcome: commit resume attached to `sase-hn.8`;
VCS flakes tracked by `sase-hk`/`sase-cw` and `sase-h8`; plan-approval flake on
`sase-ct`; release proposal resolved as epic work; snippet parity filed as ready task
`sase-i4`; and an explicit reason for any later proposal declined. Do not force the
close. After it succeeds, run `just symvision`, remove only stale `sase-i1` whitelist
entries and genuinely unused code it reports, and rerun the relevant verification.
Finally open the `plans` sidecar through `/sase_repo`, set `status: done` in
`202608/bead_search_regex.md`, and also mark this corrective plan done if its durable
archive is distinct. Finish any required commits/publication so both repositories are
clean and the epic is genuinely landed.
