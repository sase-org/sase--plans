---
tier: tale
title: Raise toobig_split launch capacity to 5
goal: New agents the toobig_split job launches admit at queue capacity 5.
size: small
proposed_by: bbugyi200.athena.0tk
create_time: 2026-09-28 09:22:33
status: wip
---

# Plan: Raise toobig_split launch capacity to 5

## Outcome

Agents proposed by the Athena `toobig_split` job start under queue capacity 5. Today
those launch prompts contain the line `%queue(capacity=3)`. `%q:3` is the alias of that
same capacity, and `%q:5` is the alias of `%queue(capacity=5)`.

## Where the 3 comes from

`toobig_split` is the only job on the Athena `run_every` routine in the chezmoi-managed
file `home/dot_config/sase/sase_athena.yml`. That routine sets `wait_runners: 3`. The
job script `bugyi_chop_toobig_split` does not author a queue directive. Each proposal
ends in a `%if` line-count gate and `%auto #split_file:<path>`.

When SASE plans the proposals, the routine value arrives as `lumberjack_wait_runners`
and `scaffolded_prompt` appends `format_queue_directive(capacity=<wait_runners>)`
because the chop prompt has no capacity directive of its own. The formatter emits
`%queue(capacity=3)`. Live members of clan `toobig-67` already show that line and
`queue_capacity` 3. The `#split_file` xprompt body has no queue directive.

`wait_runners` exists only on the routine, and the routine passes one value to every job
it runs. `run_every` has no other job, so changing its `wait_runners` from 3 to 5
changes exactly the prompts this job launches. `refresh_docs` keeps `wait_runners: 1` on
its own routine.

Capacity 5 means a default-weight split agent starts when occupied weighted load plus 1
fits in 5. The sequential `%wait` chain, the `%if` 700-line recheck, the `toobig-` clan
inhibit, and the global `max_running_agents` budget stay as they are.

## Implementation

1. Open the chezmoi repo with `sase repo open chezmoi` and edit only
   `home/dot_config/sase/sase_athena.yml` in the checkout that command prints. Under
   `axe.routines.run_every`, set `wait_runners: 5`. Leave
   `axe.routines.refresh_docs.wait_runners` at 1, and leave every other field in the
   file unchanged.
2. Land that commit on `master` of `bbugyi200/dotfiles` through the normal SASE commit
   path for the opened repo. The checkout from `sase repo open` and
   `~/.local/share/chezmoi` are two clones of that repo. Chezmoi applies from
   `~/.local/share/chezmoi`. After the commit is on `origin/master`, run
   `chezmoi update -a --force` so the source clone fast-forwards and
   `~/.config/sase/sase_athena.yml` receives `wait_runners: 5` under `run_every`. Leave
   the target file unedited by hand.
3. Restart the scheduler with `sase scheduler restart`. The orchestrator loads axe
   config once and each routine process keeps that snapshot. A new `run_every` process
   is what makes the next scheduled scan inject capacity 5. The restart does not rewrite
   prompts already stored for clan `toobig-67`. Those agents keep `%queue(capacity=3)`.
   The next clan the job proposes after the routine is running with the new config uses
   `%queue(capacity=5)`.

The bugyi-chops tests that call `prepare_chop_proposals` with
`lumberjack_wait_runners=3` supply their own sample threshold. They are not the job
configuration, and the SASE formatter stays as it is. This tale does not edit
`bugyi-chops` or the sase repository.

## Verification

1. The dotfiles diff changes one integer: `run_every.wait_runners` from 3 to 5.
   `refresh_docs.wait_runners` remains 1.
2. After `chezmoi update -a --force`, `~/.config/sase/sase_athena.yml` matches that
   source field, and `sase config show` reports `run_every` wait runners 5 and
   `refresh_docs` wait runners 1.
3. `sase axe job run toobig_split -n -f -L run_every` previews proposals and launches
   nothing. Each preview prompt contains one `%queue(capacity=5)` line. Confirm with
   `sase agent list` that the command added no agents. `-n` is what keeps the run from
   launching; `-f` only lets the preview proceed while clan `toobig-67` still inhibits a
   normal tick.
4. `sase scheduler restart` completes, and a later scheduled `toobig_split` run is
   executing in a routine process that started after the target file contained
   `wait_runners: 5`.
