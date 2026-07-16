---
tier: tale
goal: 'Epic sase-69 (Artifacts tab) is fully landed: the Commits pane''s keys are
  user-configurable through the keymap registry like the Bugs and Plans panes, stale
  "PRs tab" phrasing in living docs is updated to Artifacts-era terminology, and the
  epic is closed with symvision cleanup and its plan file marked done.

  '
create_time: 2026-07-15 23:14:25
status: wip
prompt: 202607/prompts/close_out_sase_69_epic.md
---

# Plan: Close out epic sase-69 (Artifacts tab)

## Context

Epic sase-69 renamed the ACE TUI's PRs tab to **Artifacts** with four sub-tabs (PRs · Commits · Bugs · Plans). All seven
phase beads are closed and a landing verification pass confirmed every deliverable exists in source, with two gaps that
must be finished before the epic bead is closed:

1. **Commits pane keys are not configurable.** The epic plan (plans sidecar `202607/artifacts_tab.md`) required "All new
   keys are declared in `src/sase/default_config.yml` and the keymap registry" with "default*config.yml as the single
   source". The Bugs and Plans panes honor this (see the
   `plans*\*`and bug-action keys in`src/sase/default_config.yml`, registry fields in `src/sase/ace/tui/keymaps/types.py`, fallback bindings in `src/sase/ace/tui/bindings.py`, and app-level actions in `src/sase/ace/tui/actions/artifacts_plans.py`/`src/sase/ace/tui/actions/artifact_bugs.py`). The Commits pane instead hardcodes all of its action keys (`j/k/y/f/d/a/F/R`) as widget-local `BINDINGS`on`CommitsTimeline`in`src/sase/ace/tui/widgets/artifacts/commits.py`, and both its footer hint bar and the "Commits Pane" section of `src/sase/ace/tui/modals/help_modal/changespecs_bindings.py`print hardcoded key strings. A user who rebinds`refresh_bugs`or`plans_refresh`cannot rebind the Commits pane's`R`.

2. **Stale user-facing copy and one accent literal.** Living docs still say "PRs tab" where the surface is now the PRs
   sub-tab of the Artifacts tab. Separately, `src/sase/ace/tui/widgets/artifacts/plans_pane.py` hardcodes the plans
   accent literal `#AF87FF` in several places instead of using `ARTIFACTS_ACCENTS["plans"]` from
   `src/sase/ace/tui/widgets/artifacts/types.py` (the constant every other pane uses).

Integration review found no conflicts with commits that landed mid-epic (visual-lane hardening, report split, epic
created-status fix, etc.); no other remediation is needed.

## Work item 1: Commits pane keymap configurability

Bring the Commits pane in line with the Bugs/Plans pattern:

- **Config**: add a `# Commits sub-tab` block to `src/sase/default_config.yml` beside the existing Plans/Bugs blocks
  with keys whose defaults match today's hardcoded behavior exactly: `commits_next: "j"`, `commits_prev: "k"`,
  `commits_view_selected: "enter"`, `commits_copy_sha: "y"`, `commits_filters: "f"`, `commits_toggle_sdd: "d"`,
  `commits_toggle_all_projects: "a"`, `commits_fetch: "F"`, `commits_refresh: "R"`. Do not reuse the pre-existing
  unrelated registry names `cycle_commits` / `toggle_commits` (agents-tab actions already in
  `src/sase/ace/tui/keymaps/types.py`).
- **Registry**: add matching fields and display metadata in `src/sase/ace/tui/keymaps/types.py`, mirroring how the
  `plans_*` fields are declared (both the metadata tuple list and the dataclass fields).
- **Fallback bindings + actions**: declare app-level fallback `Binding` entries in `src/sase/ace/tui/bindings.py` and
  move the Commits actions from widget-local `BINDINGS` to app-level actions following the
  `src/sase/ace/tui/actions/artifact_bugs.py` / `artifacts_plans.py` pattern, gated so they are active only on the
  Artifacts tab with the Commits sub-tab active (same `check_action` / command-availability gating used by
  `_BUG_COMMANDS` in `src/sase/ace/tui/commands/availability.py`). Plain list-cursor movement may remain widget-local
  only if it cannot be expressed as a gated app action, but the goal is registry-driven keys for every Commits action.
- **Hints + help**: render the Commits pane footer hint bar from the keymap registry (mirror
  `ArtifactsPlansPane._hints_text` in `src/sase/ace/tui/widgets/artifacts/plans_pane.py`), and switch the "Commits Pane"
  section of `src/sase/ace/tui/modals/help_modal/changespecs_bindings.py` to dynamic key display (the `d(...)` helper)
  like the Bugs/Plans sections.
- **Preserve instrumentation and behavior**: the perf samples asserted by `tests/ace/tui/bench_artifacts_jk.py`
  (`commits.next` / `commits.prev` action names, 20 samples per direction, p95 < 16 ms) must keep working after the
  refactor, as must Enter-to-open (`CommitViewModal`), the tracked-task fetch, and the debounced detail flow covered by
  `tests/ace/tui/test_commits_pane.py`.
- **Tests**: keep the existing commits pilot/renderer tests green; extend them (or the keymap tests) with a case proving
  a config override remaps a Commits action; update `tests/ace/tui/modals/test_help_modal.py` if the section content
  changes. Because default keys and hint labels stay identical, PNG goldens should not shift — if `just test-visual`
  reports diffs, inspect `.pytest_cache/sase-visual/` and only accept intentional changes.

## Work item 2: Docs terminology sweep and accent constant

- Update stale "PRs tab" phrasing to Artifacts-era terminology — "the PRs sub-tab" / "the Artifacts tab" as context
  dictates (the Artifacts framing already used in `docs/ace.md`'s tab table is the reference voice):
  - `docs/ace.md` (several spots: onboarding/empty-state note, the two `o`/`O` grouping notes, the always-grouped
    grouping section, query history, the `JumpToChangeSpec` row, mentor-comment stats section)
  - `docs/getting_started.md` (commit-workflow paragraph)
  - `docs/mentors.md` (three references: `,C` modal, inline comment counts, `,M` kill-mentors)
  - `docs/notifications.md` (jump-target description)
  - `docs/query_language.md` (intro sentence)
  - `docs/blog/posts/hello-sase-your-first-15-minutes.md` and `docs/blog/posts/changespecs-in-practice.md` (light touch:
    keep the posts' narrative intact, just correct the UI reference so readers can find the surface)
- Leave internal identifiers and code comments that say "ChangeSpecs tab" untouched — the internal tab id staying
  `changespecs` is an explicit epic non-goal.
- In `src/sase/ace/tui/widgets/artifacts/plans_pane.py`, replace the hardcoded `#AF87FF` plans-accent literals with
  `ARTIFACTS_ACCENTS["plans"]` (module-level constant import). Only the plans-accent usages change; other colors in that
  file (gold/cyan/teal counters) are intentional palette choices and stay as-is.

## Final phase: land the epic

After both work items are complete and `just check` passes:

1. Close the epic bead: `sase bead close sase-69`.
2. Run `just symvision` — closing the epic expires its epic-symbol whitelist entries. Remove the stale sase-69 entries
   and any unused code it reports.
3. Open the plans sidecar repo with the `/sase_repo` skill and set `status: done` in the frontmatter of the epic plan
   file `202607/artifacts_tab.md` (the PLAN path shown by `sase bead show sase-69`).
4. Run `just check` one final time if any repo files changed in steps 2-3.

## Testing

- `just install`, then `just check` after each work item.
- `just test-visual` to confirm the PNG snapshot suite is unaffected (defaults unchanged ⇒ goldens stable).
- Manual sanity: with a keymap override for one commits key (e.g. `commits_refresh`) in a test config, the remapped key
  drives the action and the help modal/footer hints display the override.

## Risks

- **Keymap collisions**: new `commits_*` names must not collide with existing registry fields; the gating must keep
  Commits keys inert on other tabs/sub-tabs (mirror the existing `check_action` tests in
  `tests/ace/tui/test_artifacts_scaffold.py`).
- **Golden churn**: hint-bar rendering changes could shift PNGs; keeping labels and default keys byte-identical avoids
  regeneration.
- **Perf regressions**: moving bindings to app-level actions must not add event-loop work on the j/k path; the
  `bench_artifacts_jk.py` budget (p95 < 16 ms) is the gate.
