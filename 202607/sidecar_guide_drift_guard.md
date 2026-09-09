---
tier: tale
title: Version-guard sidecar guide regeneration
goal: "Sidecar guide regeneration becomes convergent across sase code vintages and stops
  hard-failing workspace validation, so `just check` no longer blocks agents on shared
  plans/research README drift they did not cause and cannot fix.

  "
create_time: 2026-09-09 19:53:23
status: wip
---

# Plan: Version-guard sidecar guide regeneration

## Context

An implementation agent's required `just check` gate failed at its "SASE validation"
stage on generated drift in the plans sidecar guide (`sase/repos/plans/README.md`, a
+2/−1 diff) that was unrelated to the agent's change. The chain is `just check` →
`sase validate` → `sase init --check`, where `plan_repo_init` → `_plan_sidecar_actions`
→ `plan_sdd_sidecar_init_actions` (`src/sase/sdd/_init_files.py`) compares each sidecar
guide file byte-for-byte against the templates bundled with whichever sase code is
running. Any mismatch is a planned action, and
`run_init_check`/`_run_init_onboarding_result` (`src/sase/main/init_onboarding.py`) turn
any planned action into a non-zero exit that fails `sase validate`.

The drift itself was produced by version skew between sase environments that all
regenerate the same shared sidecar repositories. The project config sets
`commit_hooks.after: "sase init -y"`, which runs after every sase commit using the
`sase` on PATH — a uv-tool editable install of the primary checkout, whose vintage
routinely differs from both `origin/master` and the committing workspace. On 2026-07-16
the plans sidecar README ping-ponged three times: a new-template environment upgraded it
(07:55:39), then 19 seconds after the template-changing commit `f8b44c49f` landed
(08:02:27) the after-commit hook ran _older_ primary-checkout code and downgraded the
README back (08:02:46, the +2/−1 revert), and a newer-vintage hook re-upgraded it at
08:19:24. Every workspace whose code disagreed with the currently-pushed README failed
validation during those windows.

Two defects compose the failure:

1. **Last-writer-wins regeneration.** `_seed_sidecars` (`src/sase/sdd/_sidecar_init.py`)
   rewrites, auto-commits ("Initialize SASE {role} sidecar"), and pushes sidecar guide
   files to exactly match the running code's templates, with no notion of which content
   is newer. Older binaries destructively revert newer content in a shared repo.
2. **Gate coupling to shared state.** The workspace validation gate hard-fails on
   sidecar-clone drift that the workspace neither caused nor may fix inside an unrelated
   change, even though the same drift is auto-healed by the next `sase init -y` run.

## Implementation

### Monotonic generation revision for sidecar guides

- Introduce an integer sidecar-guide generation revision constant in the SDD init-files
  module, and stamp every generated sidecar README with an invisible HTML-comment marker
  carrying that revision. Cover both the templated plans/research READMEs (via
  placeholder substitution or deterministic append at render time) and the generic
  custom-role README built by `_generic_sidecar_readme`.
- Teach `plan_sdd_sidecar_init_actions` to parse the marker from the on-disk README
  first (missing or unparseable markers mean revision 0). When the on-disk revision is
  newer than the running code's revision, plan no actions for that sidecar at all —
  neither the README nor its directory-map asset — so stale code neither rewrites newer
  content nor reports it as drift. When the on-disk revision is equal or older, keep
  today's exact-content comparison so regeneration still enforces canonical content,
  upgrades old content, and writes the new stamp.
- Because check mode (`sase init --check`, `sase validate`) and apply mode
  (`sase init -y`, `sase repo init`, `_seed_sidecars`,
  `initialize_materialized_sidecars`) all flow through this planner, every reader and
  writer becomes downgrade-safe regardless of which installed binary or checkout vintage
  runs it, and regeneration converges to the highest revision instead of the last
  writer.
- Enforce revision hygiene with a unit test that hashes the packaged sidecar guide
  inputs (sidecar README templates, directory-map assets, and a fixed-input rendering of
  the generic README) against a committed manifest. If generated sidecar content changes
  without bumping the revision, the test fails with instructions to bump the constant
  and refresh the manifest.

### Advisory sidecar actions in the init gate

- Add an advisory flag to `InitAction` (`src/sase/main/init_plan.py`), defaulting to
  gating. Mark the sidecar guide-file regeneration actions produced in
  `_plan_sidecar_actions` (`src/sase/main/repo_init_handler.py`) as advisory; the
  create-or-connect-sidecar-repository action remains gating because it needs
  interactive confirmation.
- Compute gate status from gating actions only: `run_init_check`, the `--check` and
  non-interactive paths of `_run_init_onboarding_result`, and the
  `run_init_onboarding_all` aggregation treat a plan whose only pending actions are
  advisory as current for exit-code purposes. Apply flows are unchanged: interactive
  `sase init` still lists and offers the work, and `sase init -y` still applies,
  commits, and pushes it.
- Adjust check rendering so advisory-only work is presented as auto-managed maintenance
  rather than "Needs attention", keeping the inventory and diffstat output available for
  humans while making the exit-code semantics honest.
- Update any command help or docs that describe
  `sase init --check`/`sase init repo --check` exit behavior to note that self-healing
  sidecar guide refreshes no longer fail the check.

## Validation

- Unit tests for the planner guard: newer on-disk stamp plans nothing (README and asset
  both skipped); equal stamp with drifted content plans an update; older or missing
  stamp regenerates and writes the new stamp; marker parsing tolerates malformed input.
- The manifest test proves template or generic-README changes without a revision bump
  fail with actionable output.
- Gate tests: `run_init_check` exits 0 when only advisory sidecar actions exist and 1
  when project-repo drift or blockers exist; onboarding check/non-TTY paths and `--all`
  aggregation follow the same rule; apply mode still regenerates advisory actions and
  produces the sidecar init commit.
- An end-to-end style test seeding a temporary sidecar clone with newer-stamped content
  and running the planner with a monkeypatched lower code revision to prove stale
  environments take no action.
- Run `just install` followed by the repository-required `just check` gate (format,
  lint, full tests, PNG snapshots).

## Non-goals and risks

- Do not change `commit_hooks.after`, the uv-tool install layout, or how environments
  choose a sase binary; version skew between environments remains expected, and this
  plan makes shared regeneration convergent under it.
- Do not touch the legacy in-tree SDD store planning (`plan_sdd_init_actions`) or
  bead-store seeding; only the split plans/research/custom sidecar guide path changes.
- No new CLI subcommands or options, and no Rust core changes: sidecar seeding and init
  gating are CLI-side orchestration in this repository, not domain behavior shared with
  other frontends.
- Existing sidecars are unstamped, so the first guarded apply rewrites each sidecar
  README once to add the marker (one "Initialize SASE {role} sidecar" commit per
  sidecar). This is benign and establishes the ordering baseline.
- Pre-guard binaries (for example, a primary checkout that has not pulled this change)
  can still overwrite stamped content because their comparison logic predates the
  marker. The advisory gate keeps `just check` green during that transition window, and
  the ping-pong ends once writer environments update past this change.
- Changing `--check` exit semantics is intentional but observable; any external
  scripting that relied on sidecar guide drift failing the check will see success
  instead, which the docs update calls out.
