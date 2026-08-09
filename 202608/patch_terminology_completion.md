---
tier: epic
title: Finish the Patch/stitch terminology migration and land epic sase-hn
goal: 'User-facing output and canonical-module prose everywhere say Patch and stitch,
  the terminology audit can actually detect a defect instead of rubber-stamping the
  source tree, and epic sase-hn is closed with its plan marked done.

  '
phases:
- id: audit-classifier
  title: Make the terminology audit content-aware
  depends_on: []
  size: medium
  description: 'audit-classifier: replace path-prefix-only classification with content-aware
    rules, make skipped linked repos a hard error, and produce the authoritative defect
    list the sweep phases work from.

    '
- id: ace-surface
  title: Sweep the ACE surface
  depends_on:
  - audit-classifier
  size: large
  description: 'ace-surface: retire ChangeSpec vocabulary from ACE console output,
    toasts, TUI labels, docstrings, and canonical locals, including the glossary PNG
    snapshot fixture, while retained aliases and saved state keep working.

    '
- id: workflows-cli-surface
  title: Sweep workflows, CLI, and the remaining source tree
  depends_on:
  - audit-classifier
  size: large
  description: 'workflows-cli-surface: fix CLI help text, workflow status messages,
    error strings, and docstrings across every non-ACE source path, plus the garbled
    ChangeSpecI strings, without changing any legacy command contract.

    '
- id: core-and-linked
  title: Sweep the Rust core and linked integrations
  depends_on:
  - audit-classifier
  size: medium
  description: 'core-and-linked: fix sase-core Rust doc comments and audit the four
    linked repositories the phase-7 run never scanned, keeping wire and completion
    compatibility intact in both directions.

    '
- id: land-epic
  title: Verify and land epic sase-hn
  depends_on:
  - ace-surface
  - workflows-cli-surface
  - core-and-linked
  size: medium
  description: 'land-epic: run the full cross-repository verification set, enforce
    the audit as a lint gate, close bead sase-hn with the verification note, run symvision,
    and mark both plan files done.'
proposed_by: bbugyi200.athena.sase-hn.land
parent_bead: sase-hn
create_time: 2026-08-09 00:10:47
status: done
bead_id: sase-hn.8
---

- **PROMPT:** [prompts/202608/patch_terminology_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/patch_terminology_completion.md)
- **PARENT:** [202608/patch_and_stitch_terminology.md](https://github.com/sase-org/sase--plans/blob/main/202608/patch_and_stitch_terminology.md)
- **BEAD:** [sase-hn.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hn/sase-hn.8.md)

# Finish the Patch/stitch terminology migration and land epic sase-hn

## Why this plan exists

Epic `sase-hn` ("Rename ChangeSpec to Patch and introduce stitches") closed all seven of
its phases, and its landing verification found the tree green (`just check` passes,
`just symvision` passes, both discovered issues recorded on the epic bead are genuinely
fixed). But the epic's own acceptance contract is not met yet, so it cannot be closed as
done.

The approved epic plan (`plans:202608/patch_and_stitch_terminology.md`) states:

> This is an additive compatibility migration, not a blind search-and-replace. New code
> and **user-facing output** use Patch/stitch. Old spellings may remain **only** in
> explicit, documented compatibility adapters, migration fixtures, immutable historical
> records, or stable public paths retained to prevent broken links. Each such occurrence
> must be covered by the final allowlist audit.

and Phase 2's acceptance criterion:

> canonical callers contain **no ChangeSpec/CommitEntry vocabulary outside the
> compatibility boundary**

Neither holds today. 1171 `ChangeSpec`/`CHANGESPEC` occurrences remain across 224
canonical (non-shim, non-compat-annotated) Python files in `src/sase/`, including live
user-facing output.

### Root cause: the phase-7 audit cannot report a defect in the source tree

`tools/audit_patch_stitch_terminology` reports `0` defects, which is why phase
`sase-hn.7` believed the rename was complete. That result is an artifact of the
classifier, not of the code. In `src/sase/patch_stitch_audit.py`:

- `_is_source_legacy_api_boundary` classifies by **path prefix alone** — it deletes its
  `line` and `match` arguments and returns `True` for every occurrence under
  `src/sase/ace/`, `src/sase/main/`, `src/sase/workflows/`, `src/sase/core/`,
  `src/sase/axe/`, `src/sase/bead/`, `src/sase/xprompt/`, `tools/`, and most other
  source subtrees. It absorbed 3943 occurrences.
- `_is_compatibility_test_or_fixture` returns `True` for all of `tests/` and `smoke/`
  (4059 occurrences), also without reading the line.
- `_is_stable_public_path` returns `True` for any path merely _containing_ `changespec`
  (1316 occurrences).
- `_default_repo_specs` **silently skips** linked repos that are not materialized in the
  workspace. In this workspace only `sase-core` exists, so the "cross-repository" audit
  covered `main` + `sase-core` only; `sase-github`, `sase-telegram`, `sase-nvim`, and
  `chezmoi` were never scanned.

Because a defect is only what falls through _every_ rule, and three rules match on path
alone, the audit is structurally incapable of reporting a source-tree defect. Tightening
it is the first phase here, so later phases have an authoritative work list and CI keeps
the invariant afterwards.

### Confirmed user-facing defects (sample)

These are live strings printed to users today, in canonical (non-shim) modules:

- `src/sase/main/parser_commit.py:34,52,59,85,90,97,105,109` — `sase commit --help`
  text: `"Branch/ChangeSpec name (required for create_pull_request)"`,
  `"Parent ChangeSpec name (overrides auto-detection from current branch)"`,
  `"ChangeSpec status (overrides $SASE_PR_STATUS; default: draft)"`,
  `"List all reverted ChangeSpecs"`, `"NAME of the ChangeSpec to revert"`.
- `src/sase/main/commit_handler.py:150,152,173,187` —
  `"No reverted ChangeSpecs found."`, `"Reverted ChangeSpecs:"`,
  `"Error: ChangeSpec '<name>' not found"`, `"ChangeSpec restored successfully"`.
- `src/sase/main/commit_handler.py:80` —
  `"Set SASE_COMMIT_METHOD_ALLOW_OVERRIDE=1 to force the ChangeSpecI method."` This is a
  **garbled string**: line 78 of the same message says `"CLI commit method"`, so
  `ChangeSpecI` is a mangled `CLI` produced by an over-broad historical rename in
  `ba6d2a1e5`. It predates `sase-hn` but is the same edit surface and is fixed here
  rather than filed separately. The same garble appears in
  `src/sase/xprompt/workflow_hitl.py:98` and `src/sase/xprompt/_directive_alt.py:86`
  docstrings ("the ChangeSpecI HITL handler", "ChangeSpecI auto-daemon routing"), and in
  `src/sase/axe/summarize_hook_runner.py:280` (`action = "JumpToChangeSpec"`) — audit
  each occurrence in context before assuming which word was mangled.
- `src/sase/ace/archive.py:61,73,86,134,146,195,199` —
  `"ChangeSpec does not have a valid PR set"`,
  `"Cannot archive: other ChangeSpecs have this one as their parent"`,
  `"Renaming ChangeSpec to: ..."`, `"Renamed ChangeSpec: ..."`.
- `src/sase/ace/revert.py:77,111,122,130` and `src/sase/ace/restore.py:135,160,169` —
  the same rename/revert/restore prose.
- `src/sase/workflows/commit/commit_tracking.py:600,635,637` —
  `"Created ChangeSpec: <name>"`,
  `"Skipping ChangeSpec: could not detect project name."`.
- `src/sase/workflows/rewind/workflow.py:62,93,95,97` — `"ChangeSpec not found: ..."`,
  `"Updating ChangeSpec entries..."`.
- `src/sase/core/status_wire_conversion.py:252` —
  `"Children of WIP/Draft ChangeSpecs must be WIP, Draft, or Reverted."`
- `src/sase/bead/work.py:573` —
  `"ChangeSpec launch context is missing required field(s): "`.

### Measured inventory

Counted over tracked files, excluding compatibility-shim paths (any path containing
`changespec`/`change_spec`), lines whose text already declares itself legacy/alias/
compat/deprecated, and `src/sase/patch_stitch_audit.py` (whose token list is its
contract):

| Surface                                                 | Occurrences | Files |
| ------------------------------------------------------- | ----------: | ----: |
| `src/sase/ace/**`                                       |         658 |    98 |
| all other `src/sase/**`                                 |         513 |   126 |
| — of which user-facing string literals (whole src tree) |         107 |    61 |
| — of which comments/docstrings (whole src tree)         |         680 |   173 |
| `sase-core` Rust doc comments (`///`, `//!`)            |          68 |    22 |
| `docs/**`                                               |           0 |     0 |

`docs/**` is already clean — phase `sase-hn.6` finished that surface. The 2065 lowercase
`changespec_*` identifier occurrences are mostly **legitimate retained compatibility
names** (wire keys, saved-state keys, metadata fields, tab IDs); they are in scope only
where they are a canonical-side local variable or private helper that no external
contract pins.

### Integration review (already done, no work remaining)

Commits that landed between the epic's first commit (`3e6da8d5f`) and HEAD but are not
the epic's own (`3e6da8d5f`, `6367ef347`, `d9e11c786`, `c7026e50e`, `2634fb475`,
`db632d7fd`) were reviewed. Five added legacy tokens; four were already absorbed by
later epic phases:

- `7b473c789` (glossary → project config) added a `ChangeSpec` glossary term; the
  config-backed glossary now defines `Patch` and `Stitch` and no `ChangeSpec` term.
- `a06f12df8` (bead multi-target dispatch) added `--cl-name | ChangeSpec name` doc rows;
  `docs/beads.md:1549` and `docs/configuration.md:3742` now say `Patch name`.
- `2cf198af6` / `e213d03f9` (module and TUI splits) carried legacy identifiers that the
  later phases renamed.
- `bb07bd865` (prompt glossary interactions) left `term="ChangeSpec"` and the demo
  prompt `"Ask the Agent Clan to review the ChangeSpec glossary wiring"` in
  `tests/ace/tui/visual/_ace_prompt_png_snapshot_helpers.py`. This is the one live
  residue; it is fixed in the `ace-surface` phase below because it drives PNG goldens.

`Justfile:935` also still carries a stale `transition_changespec_status` comment.

## Non-goals

- No behavior, lifecycle, wire-format, or CLI-contract changes. This is terminology and
  audit-fidelity work only.
- Do not rename retained compatibility identifiers: `sase.ace.changespec`,
  `sase changespec`, `--changespec`, `changespec_name`, `changespec_bug_id`,
  `commit_changespec_name`, `meta_changespec`, the `changespecs` tab ID, `COMMITS:`
  section reading, `all_changespecs.json`, `filtered_changespecs.json`,
  `/sase_changespecs`, `docs/change_spec.md`, or the `changespecs-in-practice` blog
  slug. Prose that _names_ one of these retained identifiers is correct as written.
- Do not touch `CHANGELOG.md`, `.beads/`, `sdd/tales/`, `docs/blog/posts/`, or archived
  plans — immutable history.
- Do not hand-edit `AGENTS.md` / `CLAUDE.md` / `GEMINI.md` / `OPENCODE.md` / `QWEN.md`;
  regenerate them with `sase memory init` if a canonical `sase/memory/*.md` note
  changes, and only with explicit user permission for the note itself.

## Shared conventions for every phase

- Read `sase/memory/symvision.md` via `/sase_memory_read` before touching public symbol
  names.
- `just install` first — this workspace may have drifted dependencies.
- The audit is the arbiter: `./tools/audit_patch_stitch_terminology --repo-root .` must
  exit 0 at the end of every phase. Use `--all --json` to inspect classifications and
  `--json` to list defects.
- When an occurrence is genuinely a retained boundary, do not silence it by loosening a
  path rule. Either annotate the line so a content-aware rule matches it (e.g. the line
  already says "legacy"/"compatibility"/"alias", or add a brief comment that says so),
  or add a narrow, justified rule with a test.
- Run `just check` after changes; run `just check-full` in the final phase.

---

## Phase 1: Make the terminology audit content-aware (`audit-classifier`)

**Depends on:** nothing. **Size:** medium.

Goal: turn `tools/audit_patch_stitch_terminology` into a tool that can actually fail, so
the later phases have an authoritative work list and CI holds the line afterwards.

1. In `src/sase/patch_stitch_audit.py`, replace `_is_source_legacy_api_boundary` with a
   content-aware rule. A `src/`-tree occurrence may only be classified as a retained
   boundary when the _line or its immediate context_ justifies it — for example it
   declares itself legacy/compat/alias/deprecated, it names one of the retained
   identifiers listed under Non-goals, or it lives under a designated compatibility-shim
   path (`src/sase/ace/changespec/`, `src/sase/core/changespec.py`,
   `src/sase/main/changespec_handler.py`, `src/sase/main/parser_changespec.py`,
   `src/sase/workflows/commit/changespec_operations.py`,
   `src/sase/workflows/commit/changespec_queries.py`,
   `src/sase/workspace_provider/changespec.py`). Everything else in `src/` is a defect.
2. Apply the same treatment to `_is_compatibility_test_or_fixture` (today: all of
   `tests/` and `smoke/`) and `_is_stable_public_path` (today: any path containing
   `changespec`). A test file exercising a retained alias is fine; a test whose prose or
   fixture text describes the _current_ concept as a ChangeSpec is a defect.
3. Tighten `_is_external_legacy_boundary` the same way for `sase-core`, and note that
   its `sase-github`/`sase-telegram` branch is currently unreachable — the first
   `if repo in {...}` block already matches those repos, so the second block is dead
   code. Fix or remove it.
4. Make missing linked repos loud, not silent: `_default_repo_specs` must either
   materialize/require every expected repo or report the ones it skipped, and `main()`
   must exit non-zero when an expected repo was not scanned unless the caller passes an
   explicit opt-out flag. Print the scanned repo list in the summary.
5. Extend `tests/test_patch_stitch_terminology_audit.py`: assert that a synthetic
   canonical-source line with `ChangeSpec` prose is classified `defect`, that a
   synthetic declared-compat line is not, and that a skipped expected repo is reported.
6. Wire the audit into `just check` (a `_lint-*` stage) so the invariant cannot silently
   regress, but only after phases 2-4 have driven the defect count to zero — until then
   run it manually. Coordinate: add the gate in the final phase if adding it here would
   red the tree.

**Deliverable for later phases:** commit the tightened audit, then run
`./tools/audit_patch_stitch_terminology --repo-root . --json > /tmp/patch_audit_defects.json`
and record the resulting per-area defect counts in the phase's bead note so the sweep
phases can verify they cleared their slice.

**Acceptance:** the audit reports a non-zero defect count that matches the inventory
above in shape; every retained-boundary category from `sase-hn.7`'s close note
(generated-provider-copy, immutable-history, legacy-compatibility-boundary,
legacy-data-test-fixture, legacy-serialized-data, stable-public-path) still has real
members; `just check` passes.

---

## Phase 2: Sweep the ACE surface (`ace-surface`)

**Depends on:** `audit-classifier`. **Size:** large.

Scope: `src/sase/ace/**` (658 occurrences across 98 files) plus its tests, excluding the
`src/sase/ace/changespec/` compatibility package.

1. Fix user-facing output first: `archive.py`, `revert.py`, `restore.py`,
   `change_actions.py`, `hooks/processes.py`, `tui/actions/agents/_toasts.py`,
   `tui/widgets/_patch_list_banner.py`, `tui/widgets/__init__.py`, `deltas/refresh.py`,
   `handlers/workflow_handlers.py`. Every console message, toast, banner, error string,
   and log line that describes the concept must say Patch (or stitch, for a `STITCHES:`
   entry).
2. Then comments and docstrings in canonical ACE modules — the heaviest files are
   `query/matchers.py` (23), `operations.py` (22), `comments/operations.py` (14),
   `revert.py` (14), `sync_cache.py` (14), `scheduler/checks_runner.py` (13),
   `scheduler/workflows_runner/starter.py` (13), `hooks/processes.py` (12), `restore.py`
   (12), `scheduler/hooks_runner.py` (11).
3. Rename canonical-side local variables and private helpers where no contract pins them
   (`changespec` → `patch`, `_find_changespec_end_line` → `_find_patch_end_line`), but
   leave anything reachable from a public import, saved state, wire key, or the
   `sase.ace.changespec` facade alone.
4. Fix `tests/ace/tui/visual/_ace_prompt_png_snapshot_helpers.py:101,341` — the demo
   prompt text and `term="ChangeSpec"` should exercise a real current glossary term.
   This changes PNG goldens: run `just test-visual`, inspect
   `.pytest_cache/sase-visual/` actual/expected/diff artifacts, and only then re-run
   with `--sase-update-visual-snapshots`.
5. Update ACE agent instruction files under `src/sase/ace/` (`AGENTS.md` and its
   provider shims) only if they carry stale concept prose, and regenerate rather than
   hand-editing the shims.

**Acceptance:** the audit reports zero defects under `src/sase/ace/**`; `just check`
passes; `just test-visual` passes; ACE console output, toasts, and TUI labels say Patch
and stitch; old saved state, keymaps, and the `changespecs` tab ID still load.

---

## Phase 3: Sweep workflows, CLI, and the remaining source tree (`workflows-cli-surface`)

**Depends on:** `audit-classifier`. **Size:** large.

Scope: every `src/sase/**` path except `src/sase/ace/**` (513 occurrences across 126
files), plus `Justfile` comments and the corresponding tests. Path-disjoint from phase 2
so the two can run in parallel.

1. User-facing output: `main/parser_commit.py` (8 `--help` strings),
   `main/commit_handler.py` (7, including the garbled `ChangeSpecI`),
   `workflows/rewind/workflow.py` (7), `main/parser_ace.py` (5),
   `workflows/commit/commit_tracking.py` (4), `workflows/commit/patch_operations.py`
   (4), `workflows/mentor.py` (4), `main/search_handler.py` (3),
   `workflows/accept/workflow.py` (3), `workspace_provider/plugins/bare_git_submit.py`
   (3), `workspace_provider/submission_utils.py` (3), `core/status_wire_conversion.py`
   (2), `core/wire_conversion.py` (2), `integrations/_mobile_helper_catalog.py` (2),
   `main/parser_commands.py` (2), `bead/work.py` (1), `axe/*_runner.py`. CLI help text
   is a contract surface for users: canonical help must advertise `sase patch` /
   `--patch` vocabulary while the `sase changespec` / `--changespec` aliases keep
   working and keep their exit codes.
2. Comments and docstrings — heaviest: `workflows/commit/patch_operations.py` (32),
   `status_state_machine/field_updates.py` (24), `workflows/commit_utils/entries.py`
   (17), `axe/check_cycles.py` (11), `vcs_provider/_base.py` (11). Note that
   `workflows/commit/patch_operations.py` and `main/patch_handler.py` were renamed to
   Patch names by this epic but their prose was never updated — those are the clearest
   misses.
3. Fix the garbled `ChangeSpecI` occurrences in `xprompt/workflow_hitl.py:98` and
   `xprompt/_directive_alt.py:86`, and check `axe/summarize_hook_runner.py:280`
   (`action = "JumpToChangeSpec"`) against its consumers before renaming — if a
   notification consumer pins that action string, retain it and annotate it as a wire
   value instead.
4. Remove the stale `Justfile:935` comment referencing the
   `transition_changespec_status` orchestrator (or update it to the current symbol).
5. Leave every retained identifier from Non-goals intact, and leave
   `tools/validate_sase_core_rs`'s
   `REQUIRED_PYTHON_COMPAT_SYMBOLS = ("changespec_to_wire",)` alone — it asserts the
   compatibility binding still exists.

**Acceptance:** the audit reports zero defects outside `src/sase/ace/**`;
`sase commit --help`, `sase patch --help`, and `sase changespec --help` all describe
Patches; `just check` passes; legacy commands keep identical exit codes and machine
output.

---

## Phase 4: Sweep the Rust core and linked integrations (`core-and-linked`)

**Depends on:** `audit-classifier`. **Size:** medium.

Repo-disjoint from phases 2 and 3, so it can run in parallel with them. Use `/sase_repo`
to open every repository before reading or writing it — never guess a path.

1. `sase-core`: fix the ~68 Rust doc comments that describe the current concept as a
   ChangeSpec — `crates/sase_core_py/src/lib.rs` (12), `crates/sase_core/src/parser.rs`
   (7), `query/matchers.rs` (5), `status/field_updates.rs` (5), `wire.rs` (5),
   `query/searchable.rs` (4), `tests/python_wire_parity.rs` (4), `agent_stats/wire.rs`
   (3), `status/name.rs` (3), `status/wire.rs` (3). Keep serde aliases, `ChangeSpecWire`
   compatibility names, legacy `## ChangeSpec` parser fixtures, and genuine VCS-commit
   APIs unchanged. `crates/sase_core/src/parser.rs:3` also points at a stale path
   (`sase_100/src/sase/ace/changespec/parser.py`) — retarget it at
   `src/sase/ace/patch/parser.py`. Do not hand-edit release-plz-owned versions. Verify
   with `cargo fmt --all -- --check`,
   `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace`.
2. Materialize `sase-github`, `sase-telegram`, `sase-nvim`, and `chezmoi` and run the
   tightened audit against each. Phase `sase-hn.5` updated them, but the phase-7 audit
   never scanned them from this workspace, so their state is unverified. Fix any
   canonical-side prose or user-facing label the audit flags; retain mixed-version
   compatibility paths.
3. Verify per repo: `just check` in `sase-github` and `sase-telegram` against the local
   SASE/core build; headless `tests/*.lua` in `sase-nvim` against the local
   `sase-xprompt-lsp`; `just check` in `chezmoi`; `git diff --check` in each.

**Acceptance:** `./tools/audit_patch_stitch_terminology --repo-root .` scans all five
repos (no silent skips) and reports zero defects; every linked repo's own checks pass;
wire and completion compatibility in both directions still works.

---

## Phase 5: Verify and land epic sase-hn (`land-epic`)

**Depends on:** `ace-surface`, `workflows-cli-surface`, `core-and-linked`. **Size:**
medium.

1. Re-run the full verification set on the combined tree: `just install`,
   `just check-full`, `just rust-check`, `just test-visual`, `just docs-check`,
   `just docs-pdf-check`, `sase memory init --check`, `sase skill init --diff`, and
   `./tools/audit_patch_stitch_terminology --repo-root .` (must exit 0 across all five
   repos).
2. If phase 1 deferred it, add the audit as a `just check` lint stage now that the
   defect count is zero, so the invariant is enforced going forward.
3. Close the epic:
   `sase bead close sase-hn --note "<what was verified across steps 1-2 and this plan>"`.
   The note must record: all seven phases verified against their commits; both
   discovered issues on the epic bead resolved (the `sase commit` post-rebase import
   hazard — `commit_utils/entries.py` now imports `sase.ace.patch.section_order` and a
   fresh interpreter resolves it; and the Symvision `_parse_timestamp_value` private
   import — `parse_timestamp_value` is public in
   `src/sase/ace/tui/models/patch_groups/_buckets.py` and `just symvision` passes); the
   single `PROPOSED FOLLOW-UP:` note (`sase-hn.6`, 2026-08-09T02:19:45Z) self-resolved
   within that same phase; the integration review of the 40 non-epic commits since
   `3e6da8d5f`; and this plan's completion of the audit-fidelity gap.
4. After closing, run `just symvision`. There are currently **no** `--epic-symbol`
   entries in the Justfile, so nothing should expire — but if closing surfaces newly
   unused public symbols (compatibility aliases that only the epic's in-flight code
   referenced), resolve them by the `sase/memory/symvision.md` decision hierarchy:
   delete, privatize, pragma, and only then whitelist.
5. Set `status: done` in the frontmatter of **both** plan files:
   `plans:202608/patch_and_stitch_terminology.md` (the epic's plan, currently
   `status: wip`) and this plan file.

**Acceptance:** `sase bead show sase-hn` reports the epic closed with resolution `done`;
`just check-full` and the audit pass; `just symvision` is clean; both plan files read
`status: done`.
