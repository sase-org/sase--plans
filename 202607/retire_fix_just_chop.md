---
tier: tale
title: Retire the fix_just Axe chop
goal: Remove the fix_just chop from its package, scheduled configuration, installed
  runtime surface, and current documentation without disturbing the remaining bugyi-chops
  jobs.
create_time: 2026-07-20 07:43:40
status: wip
prompt: 202607/prompts/retire_fix_just_chop.md
---

# Plan: Retire the `fix_just` Axe chop

## Outcome and ownership

Completely retire the runnable `fix_just` Axe chop while preserving unrelated automation and historical audit data. The
active feature is split across two source repositories:

- the external `bugyi-chops` repository owns the `bugyi_chop_fix_just` console entry point, its dedicated
  `bugyi_chops.fix_just` implementation, tests, and package documentation; and
- the linked chezmoi repository owns the `axe.lumberjacks.run_every.chops.fix_just` scheduler entry deployed as
  `~/.config/sase/sase_athena.yml`.

The main SASE repository does not own a live definition. Its documentation already identifies the old `fix_just` xprompt
workflow as retired, and its remaining `fix_just` strings are historical or generic fixture values rather than an
executable chop. Leave those records intact: removing audit history, migration backups, old ChangeSpecs, or the
retirement note would erase useful provenance without making the chop less runnable.

This is a tale because one coding agent can make and validate the small, ordered changes across both owning
repositories. The package must remain installed because it also supplies `toobig_split` and the two recent-commit audit
chops.

## Remove the package feature

Open `bbugyi200/bugyi-chops` through the audited SASE repository workflow and retire only the `fix_just` public surface:

- delete the dedicated `src/bugyi_chops/fix_just.py` implementation and `tests/test_fix_just.py` coverage;
- remove `bugyi_chop_fix_just` from `[project.scripts]` in `pyproject.toml`;
- bump the pre-1.0 package version to the next minor version because removing a console command is a breaking public API
  change; and
- update the README's script count/table and remove the `fix_just` behavior/configuration section, while retaining the
  installation, SDK contract, `toobig_split`, and recent-audit documentation.

Update the GitHub repository description/catalog-facing summary so it no longer advertises “just repairs.” Do not
uninstall or archive `bugyi-chops`, and do not alter its other three console scripts.

## Remove the scheduled definition

Open the linked chezmoi repository through `sase repo open` and remove the complete `fix_just` mapping from
`home/dot_config/sase/sase_athena.yml`: script, description, cadence, ChangeSpec inhibition rule, and workspace
variable. Keep the containing `run_every` lumberjack and its `toobig_split` chop unchanged. Do not edit the generated
destination file under `~/.config/sase` directly; deploy the canonical chezmoi change through the repository's normal
apply/update workflow.

## Validate source and artifacts

Run the full `bugyi-chops` `just check` gate. In addition to lint, type checking, tests, coverage, wheel/sdist builds,
and Twine validation, inspect the built distributions or install them into an isolated environment and prove that:

- no `bugyi_chops.fix_just` module is shipped;
- no `bugyi_chop_fix_just` console entry point is registered; and
- the three retained console scripts remain present.

Search the current `bugyi-chops` and chezmoi source trees for `fix_just` and `bugyi_chop_fix_just`; any remaining match
must be removed or explicitly justified as non-current provenance. Validate the chezmoi YAML and render the effective
SASE configuration from the changed source before applying it, confirming that the `run_every` lumberjack still contains
`toobig_split` but no `fix_just` entry.

## Roll out without a stale scheduler window

Land the package and chezmoi changes before refreshing the installed environment. Enter Axe maintenance mode for the
deployment so the already-running daemon cannot launch the old chop between updates. Apply the committed chezmoi state
as required by that repository, then update the installed `bugyi-chops` plugin from its Git source so SASE retains the
other plugin chops but replaces the old distribution and restarts Axe against the new configuration. Exit maintenance
mode only after the runtime checks pass; if rollout validation fails, leave maintenance active and report the exact
recovery state rather than resuming a partially updated scheduler.

Verify the deployed result with the verbose/configured/available chop inventory and chop doctor:

- neither a configured `fix_just` chop nor a discoverable `bugyi_chop_fix_just` executable/entry point exists;
- the installed Python environment cannot resolve `bugyi_chops.fix_just`;
- `toobig_split`, `recent_bug_audit`, and `recent_improvement_audit` still resolve to the updated `bugyi-chops`
  installation;
- the effective SASE config and deployed `sase_athena.yml` contain no `fix_just` definition; and
- Axe starts cleanly with no missing-script diagnostic.

Historical Axe run directories and completed agents may continue to mention `fix_just`; they are records, not live
definitions, and should not be destructively purged.
