---
tier: tale
title: Observe sase 0.18.1 and finish the release landing
goal:
  Confirm sase 0.18.1 on PyPI with a clean install, then close sase-1nn.8.4 and every
  fully complete direct plan ancestor with retained evidence.
size: medium
proposed_by: bbugyi200.athena.sase-1nn.8.4.land
proposed_with: codex/gpt-6-astra
bead: sase-1nn.8.4
create_time: 2026-10-24 09:01:12
status: wip
---

- **PARENT:**
  [202610/v0_18_1_final_goldens.md](https://github.com/sase-org/sase--plans/blob/main/202610/v0_18_1_final_goldens.md)
- **BEAD:**
  [sase-1nn.8.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nn/sase-1nn.8.4.md)

# Observe sase 0.18.1 and finish the release landing

Finish the interrupted landing of **sase-1nn.8.4**. Its direct parent is plan bead
**sase-1nn.8**, whose direct parent is plan bead **sase-1nn**. This tale's coding agent
owns the entire remaining observation, install proof, and closeout; there is no tale
land agent. Do not stop after starting a fire-and-forget watch.

This is a medium tale because the fixes are already committed. No planned step needs
this tale's future commit, push, or CI result. Long external waits use `/sase_monitor`
with an explicit `--next` that resumes this plan, including on failure. Do not create
another repair/watch epic solely to wait for already-landed work.

## Required context and verified baseline

Read these through audited commands:

- `sase bead read sase-1nn.8.4 -r "Need the landing audit and complete follow-up triage"`.
- Both children, `sase-1nn.8.4.1` and `sase-1nn.8.4.2`, including every note.
- `sase artifact read file:explicit:c12008cebe29a875c6afb8fe "Need independent source and CI evidence"`.
- `sase artifact read plan:202610/v0_18_1_final_goldens.md "Need the epic acceptance and ancestor closeout"`.

At the 2026-10-24 landing audit:

- Golden repair **dc2cc11324f6cd3696603ec56e2458f113936ab1** is on master. It removes
  the two duplicate Python 3.12 help-golden lines and regenerates Archive help for the
  three stranded-work legend rows. Source, image, and retained semantic assertions were
  independently reviewed. Full CI 3.12 and 3.13 pass in both runs below.
- The only later master commit is **c793bff54b0382194527c3c422b0416b11d87a1f**,
  Beads-pane stranded presentation with its own visual updates. It does not change the
  repaired stats or Agents Help paths. Exact-master Master Gate **40087702875** is
  green.
- PR **321**, `chore(master): release 0.18.1`, is OPEN at
  **83e33050eca8a5fbdcfc7e4bd8a61b9e4d7f320a**, contains the repair and current master,
  and PR CI **40088307135** is green. PyPI sase/0.18.1 returns 404.
- Full CI **40088671003** at c793bff54b is in progress. Re-inventory before acting.
- The two previous Full CI runs share dc2cc11324. **40077730487** fails the known TODO
  cursor-highlight flake and two final-row PNGs. **40085700689** passes those cases and
  the entire visual lane (1123 unchanged), but fails the known archive pulse Enter
  flake. Each Python 3.14 leg has one failure / 57117 passed / 138 skipped.
- Watch phase **sase-1nn.8.4.2** closed **canceled**, not done, after those two reds.
  Its bounded watcher worked; this is not a reason to close a release epic done.
- Both Python dependency files pin **sase-core-rs==0.42.3**, containing source pin
  **58aca5fe69158286092e3a33685687a52f8d2b8f**. Prior isolated wheel checks found all
  876 required bindings. Recheck the actual pin if master has moved.
- No epic-symbol entries exist for sase-1nn.8.4. No tracked source edits were made
  during the landing audit and no `just check-full` was run.

## Constraints and completed triage

The root's accepted decisions remain **publish_dispatch=yes**, **gate_rerun=yes**. Merge
authority stays with athena's **ci_watch**. It requires Master Gate green at exact
master HEAD, the newest completed Full CI green and no older than six hours, and all
release PR checks green (skipped is acceptable). It dispatches Full CI when the latest
is over two hours old and none is in flight. Never hand-merge, create a release PR,
dispatch release.yml, run the steward with `--live`, or change its merge-authority flag.

Read **sase-1n6.9** before continuing; it was in progress at audit time. If cutover has
completed or the release target changed, preserve evidence, leave the release epics
open, and plan the actual remaining acceptance under the new contract. Do not continue
the old mutating watch path blindly.

The lander already completed `/sase_new_task` searches, recent-task sweep, active-epic
review, and every proposal disposition, recorded on sase-1nn.8.4:

- `.1` note 1 import budget: supplementary note on **sase-1ic**, no duplicate or extra
  +1; a local environment count, not reproduced by release CI.
- `.1` note 3 narrow gate and renamed-session recaptures: DISCOVERED ISSUE on active
  **sase-1jz**, whose `.3` and notes 1–3 own those exact readiness cases; supplementary
  evidence on related **sase-1ln** without claiming its precise mechanism was proven. No
  new task and no blind golden regeneration.
- TODO cursor overlay: **sase-xh** received +1 with same-SHA fail/pass evidence.
- Archive pulse Enter: **sase-1mj** received post-close +1 and reopened ready.
- Final-row PNG widths 60/80: new **sase-1nx**, large flake task, ready, with retained
  evidence and a related link to sase-1jz. Its cause is unproven; do not absorb a broad
  test investigation merely to perform this operational closeout.
- Resolved CLI-golden task **sase-1nt** is closed done with independent CI proof.
  **sase-1no** has a note resolving its Archive-help DISCOVERED ISSUE; leave that
  unrelated epic open.

Preserve exact assertions, pixels, and timeouts. Do not use skips, xfails, tolerance,
assertion retries, or sleeps as layout waits. File-change verification is `just fix`
then `sase tool run check` (the wrapped `just check`). **Never run just check-full.** An
unrelated check-full-only issue is test infrastructure, not release-epic work. Read the
applicable memories before changes; this plan includes no memory changes. Use
`/sase_repo` for any other repo and audited `sase artifact read` for sidecar context.
Host finalization owns commits. Never end a provider turn with a yielded command
running.

## 1. Re-inventory the release and observe the existing gates

Fetch origin. Re-read the current epic and watch resolution, PR 321 metadata/checks,
current master SHA, Master Gate on that SHA, Full CI history, Publish history, and the
PyPI JSON endpoint. Review every newer non-epic master commit for conflicts with the
repaired golden/source/pin/workflow paths. If PyPI already serves 0.18.1, proceed
directly to the install proof and closeout after confirming the publication chain.

If Full CI 40088671003 (or a newer relevant run) is still active, hand its wait to
`/sase_monitor`, for example:

```sh
sase monitor start -r 'Observe the already-landed sase 0.18.1 release gates' \
  -t 3h -s 'WATCHING CI' -S 'CI WATCHED' \
  --next 'Resume this approved tale. Inspect every failed job in the observed run, refresh master/PR/Publish/PyPI, then complete steps 2-4 including the three release-epic closeouts. A timeout or red is not release completion.' \
  -- gh run watch 40088671003 -R sase-org/sase --interval 120 --exit-status
```

Use the current relevant run ID, not an already-obsolete literal. Wait for monitor start
itself to exit. Do not wait in the provider after the handoff.

For a new red, read its actual logs. Same-node same-SHA passing evidence establishes a
flake; different tests failing in consecutive runs do not establish a deterministic
regression. Reuse the owners above and add only new evidence. If a concrete new
implementation defect is found, plan only that work rather than weakening the gates or
closing this tale successfully. Any code requiring its own landed SHA for release
qualification needs a genuinely separate committed-code phase; do not wait for this
turn's future commit.

Do not dispatch duplicate Full CI runs while one is active. Let ci_watch/the schedule
refresh the heavy lane. If no suitable run exists and none is in flight, the root plan
permits one `gh workflow run full.yml -R sase-org/sase --ref master`. The accepted
gate_rerun decision permits one failed-job rerun per failed **Master Gate** run at
current master, not unbounded Full CI reruns.

## 2. Confirm ci_watch merge, Publish, and PyPI

Once gates are green, let ci_watch merge PR 321. For long waits, use `/sase_monitor`
with an explicit continuation that carries steps 3-4. Poll only read-only release facts,
at five-minute intervals and with a bounded deadline (at most ten hours per watch). A
small temporary watcher may be adapted from the prior epic's Phase watch, but remove its
bead-closing logic and always request a follow-up. This tale has no separate lander to
awaken. Stop the watcher on a terminal Publish failure, two newly completed Full CI
failures, target/cutover change, or deadline; the follow-up diagnoses the result and
leaves incomplete epics open. Do not mutate repo files for the watcher.

The allowed mutations are exactly the accepted existing release operations:

- If PR 321 loses a required landed repair or its core-floor smoke is stale/red, and no
  Publish refresh is active, dispatch once:
  `gh workflow run publish.yml -R sase-org/sase --ref master -f publish_existing=false`.
- After ci_watch merges, dispatch the same command once if no post-merge Publish run
  exists. A scheduled run already covering that merge needs no duplicate dispatch.
- Rerun a failed Master Gate at exact master HEAD at most once per run, as above.

Record exact SHAs, run IDs, job conclusions, and times. Require Publish's `build`,
`install-smoke`, `install-smoke-core-floor`, and `publish` jobs to succeed and require
`https://pypi.org/pypi/sase/0.18.1/json` to list the release artifacts. Do not treat a
green watcher, successful metadata-only Publish, merged PR, or PyPI HTTP 200 alone as
the complete evidence chain.

## 3. Prove the published installation and retain evidence

Create a fresh Python 3.12 venv outside the repo with an isolated child-process home.
Use `UV_NO_CACHE=1` and install **sase==0.18.1 from PyPI**, without an editable checkout
or a source-core override. Never repurpose the shell's HOME and never replace a global
install. Run outside the source checkout so local imports cannot mask the installed
wheel. Use `/sase_monitor --next` if installation may exceed the synchronous ceiling.

Verify installed sase is 0.18.1, its resolved core version equals the exact dependency
declared by the wheel, and `sase core health --json` succeeds in that isolated context.
Record Python version, commands, outputs, PyPI artifact names/digests, release tag/SHA,
master/PR gate evidence, Publish URL and jobs, and any drift reviewed after the audit.

Write a temporary `sase_v0_18_1_release_evidence.md` and register it:

```sh
sase artifact create -p <evidence-file> -l 'sase v0.18.1 PyPI release evidence' --bead sase-1nn.8.4
```

Wait for the registration command to exit. Add its canonical ref in notes on
**sase-1nn.8** and **sase-1nn**. The earlier landing audit is not a substitute for this
actual release evidence. Check that sase-1nt is still correctly resolved and preserve
all follow-up dispositions above in the final close note.

## 4. Finish this epic and each complete direct ancestor

This is the tale's final required step, in the same turn as any last local edits; it
must not depend on those edits' future commit SHA, push, or CI result.

1. Re-read `sase-1nn.8.4`, every child and note, and
   `plan:202610/v0_18_1_final_goldens.md`. Recheck current descendant and linked-plan
   readiness against actual scope, not only CLOSED labels. Once publication/install
   acceptance is met, reopen canceled **sase-1nn.8.4.2** and close it normally with
   actual release evidence, preserving its historical failure notes.
2. Run `sase bead epic-symbols sase-1nn.8.4`. Resolve every entry by wiring it up,
   privatizing, deleting, or a justified non-test consumer pragma under Symvision
   policy. Re-key only when a still-open later bead truly needs the exemption. Then:
   `sase bead close sase-1nn.8.4 --note "<source/commit verification, integration, all follow-up outcomes, release and install evidence ref>"`.
   Run **`just symvision`** after close. Open the plans repo using `/sase_repo` and set
   **`status: done`** in **202610/v0_18_1_final_goldens.md**, using only its printed
   checkout path. Do not merely edit an untracked copy of the plan.
3. Inspect the actual parent link with
   `sase bead read sase-1nn.8.4 -r "Need the parent link"`. The expected parent is plan
   **sase-1nn.8**. Read its previous landing note, all descendants and every note,
   `plan:202610/v0_18_1_release_landing.md`, and post-child drift. Re-run descendant and
   linked-plan readiness checks. If its named canceled watch **sase-1nn.8.3** still
   needs completion, reopen and close it normally with verified release evidence.
   Resolve every `sase bead epic-symbols sase-1nn.8` entry, then
   `sase bead close sase-1nn.8 --note "<full recheck and release artifact>"`. Run
   **`just symvision`** and mark **202610/v0_18_1_release_landing.md** done.
4. Repeat that full recheck for direct plan ancestor **sase-1nn**: prior landing notes,
   all descendants/notes, `plan:202610/v0_18_1_release.md`, post-child drift, descendant
   and linked-plan readiness. Reopen and normally complete its canceled watch
   **sase-1nn.7** if required, using actual acceptance evidence. Resolve all
   `sase bead epic-symbols sase-1nn` entries, then
   `sase bead close sase-1nn --note "<full root verification and release artifact>"`.
   Run **`just symvision`** and mark **202610/v0_18_1_release.md** done. Recheck its
   actual parent link (currently none) and continue only through fully complete direct
   plan ancestors.

If any close refuses leftover symbols, finish that cleanup and retry. If a named phase
was never completed, finish or reopen it; never force a successful nested landing and
never use force merely to make the command succeed. Do not change historical canceled
resolutions without reopening and genuinely completing the work. Stop at the first
incomplete or ambiguous parent, record its blocker in a bead note, and report it. If an
actual parent is a phase, verify its child-plan scope and close only that phase
normally, leaving its containing epic to the already-waiting land agent.

Any tracked sase source changes require `just fix` and `sase tool run check`. For
sidecar-only plan status edits, follow that repo's instructions. Read `/sase_final` and
submit the final declaration as the last action of a normal ending; include the plans
repo obligation. Report the release evidence ref and exactly which epics closed.
