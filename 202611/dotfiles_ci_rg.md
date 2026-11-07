---
tier: tale
title: Turn dotfiles CI green by dropping the test-only ripgrep dependency
goal: bbugyi200/dotfiles CI (lint and test (ubuntu-latest)) is green at the master
  tip after the test-only rg call in tests/bash/doctl_login_test.sh is replaced with
  portable grep, with ci_evidence attached and phase bead sase-1rq.5 closed.
size: small
proposed_by: bbugyi200.athena.sase-1rq.5
proposed_with: claude/opus
bead: sase-1rq.5
status: done
---

- **PARENT:**
  [202611/red_ci_repair.md](https://github.com/sase-org/sase--plans/blob/main/202611/red_ci_repair.md)
- **BEAD:**
  [sase-1rq.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1rq/sase-1rq.5.md)

# Plan: Turn dotfiles CI green by dropping the test-only ripgrep dependency

This plan carries out phase `sase-1rq.5` ("Turn dotfiles CI green") of epic `sase-1rq`
("Repair red GitHub Actions CI across five projects"). The epic's DECISIONS are final:
`test_quarantine = quarantine_flakes` and `release_merge = gate`. This phase has no
release, so `release_merge` does not come up. No quarantine is expected either, because
the failure is deterministic and has a known root cause.

## Repository

The `chezmoi` linked repo. Open it with `sase repo open chezmoi -r "<reason>"` and work
only in the path it prints. Its `origin` is `git@github.com:bbugyi200/dotfiles.git`,
which is the GitHub repo `bbugyi200/dotfiles` named by the epic. Read that repo's
`AGENTS.md` before editing.

**Do not run `chezmoi apply`, `chezmoi update`, or anything else that touches Bryan's
live home directory.** The repo's own `AGENTS.md` has a gotcha saying to run
`chezmoi update -a --force` after commits. The epic contract overrides it for this
phase. The change only touches `tests/`, which is outside the chezmoi source root
(`.chezmoiroot` = `home`), so applying would change nothing anyway. Say so in the final
response.

## What Was Verified While Planning

- The `CI` workflow (`.github/workflows/ci.yml`) has two jobs, `lint` and
  `test (ubuntu-latest)`. On master, `test` has failed on every push since `9d4011ad`
  ("feat(doctl): add guided Apollo recovery login"). The latest red run is `42368492034`
  at tip `e9316957` ("chore: run sase init"). `lint` passes.
- That run's failed log has exactly one failure:
  `Tests: 292 passed, 9 skipped, 1 failed, 302 total`. The failing test is "Help and
  usage errors" (`test_help_and_usage_errors`), with
  `./tests/bash/doctl_login_test.sh: line 195: rg: command not found`. Because
  `test-bash` fails, `just test` stops there, and `test-python` has not run in CI since
  then. No files under `home/lib/xfile` have changed since the last green push
  (`af323fbe`), so `test-python` is not expected to break.
- Line 195 is the only `rg` call in `tests/`:
  `options="$(rg '^  -[hlry],' "$STDERR_FILE")"`. The script under test,
  `home/bin/executable_doctl_login`, does **not** call `rg`, so this is a test-only
  portability bug, not a missing runtime dependency. Provisioning ripgrep in CI would
  hide that, so the fix is the portable `grep` switch.
- Reproduced locally. Running `bashunit tests/bash/doctl_login_test.sh` with every
  `PATH` entry that holds `rg` removed gives the same failure (16 passed, 1 failed). On
  athena, `rg` lives in `~/.cargo/bin`, the same directory as `just`. So run `bashunit`
  directly for this check, not through `just`.

## Step 1: Fix The Test

In `tests/bash/doctl_login_test.sh`, change line 195 from `rg` to plain `grep`. The
pattern `^  -[hlry],` is a valid basic regex and means the same thing to grep:

```bash
  options="$(grep '^  -[hlry],' "$STDERR_FILE")"
```

Make no other behavior changes. While in the file, check that nothing else in `tests/`
or the `Justfile` test recipes uses a tool the CI `test` job does not install. Fix any
such use the same way, and mention it in the final response. Planning found none.

## Step 2: Verify Locally

From the chezmoi checkout:

1. **Regression proof without ripgrep.** Build a `PATH` with every directory that holds
   `rg` removed. Confirm with `command -v rg` that it is gone, then run
   `bashunit tests/bash/doctl_login_test.sh`. All 17 tests must pass.
2. **Repo recipes.** Run `just test`, which is what CI runs. Also run `just lint` and
   `just fmt-check` if their tools are available locally. These are the repo's own check
   recipes; `just check` = `fmt-check lint test`. The change is a one-line shell edit,
   and no linter covers `tests/bash`. So a lint or format tool that is missing locally
   does not block landing; record which recipes ran. If `just test` fails for a reason
   that also reproduces on the unmodified tree (`git stash`, rerun, `git stash pop`), it
   is not caused by this change. Record it as a `PROPOSED FOLLOW-UP:` (see Bead Rules)
   and continue.

## Step 3: Land And Watch CI (First Turn Ends Here)

Agents never commit or push; host completion lands the change after the turn ends.

1. **Start the CI watch with `/sase_monitor` before declaring.** Use a zero-weight wait
   (`-w 0`) with labels `WATCHING CI` → `WATCHED CI`. The command must:
   - Poll `gh run list -R bbugyi200/dotfiles -b master -w CI -e push --json …` about
     once a minute until a run exists for the landed fix commit. Match on `displayTitle`
     equal to the commit subject declared in item 2. As a fallback, accept the first
     push run whose `headSha` is not `e9316957…` or an earlier SHA.
   - Stop with a clear `TIMEOUT` message and nonzero exit if no such run appears within
     about 90 minutes, so a commit that never landed cannot hang the monitor.
   - Then run `gh run watch <id> -R bbugyi200/dotfiles --exit-status`, and finally print
     `gh run list -R bbugyi200/dotfiles -b master -w CI -L 5 --json databaseId,headSha,displayTitle,event,attempt,status,conclusion,createdAt,updatedAt`
     so the follow-up turn has the data it needs.

   Give it a `--next` prompt telling the follow-up turn to read the monitor output. That
   turn either finishes per Step 4 or iterates per Step 5, under bead `sase-1rq.5` and
   this plan.

2. **Submit the commit decision through `/sase_final` as the last action.** Use an
   explicit `commit` decision for the chezmoi repo (it is a repo obligation, because it
   was opened with `/sase_repo` and modified). Use a conventional subject such as
   `fix(tests): use grep instead of rg in doctl_login_test`. The `sase` primary
   workspace should have no changes. Do not close the bead in this turn: green is not
   proven yet.

## Step 4: Follow-Up Turn: Confirm Green, Attach Evidence, Close The Phase

"Green", per the epic contract: the latest `CI` run on master is `completed` /
`success`, on a head SHA at or after the fix commit, with both `lint` and
`test (ubuntu-latest)` succeeded and none `cancelled` or `skipped`. Re-read
`gh run list` yourself; don't rely only on the monitor output. Note whether the green
came on the first attempt.

When green:

1. Publish a `ci_evidence` JSON variable through `/sase_var`. Its fields: `repo`
   (`bbugyi200/dotfiles`), `default_branch` (`master`), `tip_sha`, and `runs` (one entry
   for workflow `CI`, with `workflow`, `run_id`, `url`, `head_sha`, `event`, `attempt`,
   `conclusion`, `completed_at`). Also `fixes` (the landed commit SHA and a one-line
   summary), `quarantined` (`[]`), and `notes`. The notes cover the test-only nature of
   the bug, that `chezmoi apply` was deliberately skipped, and any first-attempt caveat.
2. Run `sase bead epic-symbols sase-1rq.5`. Resolve or re-key any listed entry; none is
   expected, since this change adds no Python symbols.
3. Close only this phase:
   `sase bead close sase-1rq.5 --note "<what was verified: run id/url, tip sha, local bashunit-without-rg proof>"`.
   **Never close epic `sase-1rq` or any ancestor**, and never hand-set bead status.
4. In the final response, repeat the evidence as a short Markdown table. Include the
   reproducing command `gh run list -R bbugyi200/dotfiles -b master -w CI -L 5`. Then
   end the turn through `/sase_final` (no repo changes in this turn).

## Step 5: If CI Is Still Red

- **Same job, new failure from this change:** fix it, re-verify per Step 2, and repeat
  Step 3 with a new monitor.
- **New failure from someone else's commit:** fix it if the fix is quick, same loop.
  Otherwise, record it in the evidence notes and as a `PROPOSED FOLLOW-UP:`.
- **A test fails and then passes on an unchanged tree (proven flake):** under
  `quarantine_flakes`, it may be skipped. Record a
  `PROPOSED FOLLOW-UP: flake — <test> …` note for each such test, and list them in
  `quarantined`. Never raise timeouts, drop jobs, or soften lint.
- **Monitor `TIMEOUT` (fix never landed):** check `git log origin/master` in the chezmoi
  checkout. If the fix commit is missing, investigate why host completion did not land
  it before redoing anything.
- Never change repository variables, secrets, branch protection, or workflow
  permissions. Never force-push or rewrite history.

## Bead Rules For This Phase

- Do not create beads. Record discovered follow-up work as
  `sase bead note sase-1rq.5 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`. The
  epic's land agent triages these.
- A check failure that reproduces identically on the clean base tree does not keep this
  bead open. Record it as a `PROPOSED FOLLOW-UP:` and close anyway once CI is green.
- Do not run `sase bead open sase-1rq.5`; the bead is already `in_progress`.

## Out Of Scope

- Provisioning ripgrep in CI, or changing `.github/workflows/ci.yml`. It isn't needed,
  because the dependency is test-only.
- Upgrading local `bashunit` (athena has 0.36.0; CI installs the latest, 0.52.0 at
  planning time).
- Any `home/` source change, and any `chezmoi apply` / `update`.
