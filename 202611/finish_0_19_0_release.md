---
tier: tale
size: medium
title: Finish the approved 0.19.0 release and close sase-1s7.7 on evidence
goal:
  sase 0.19.0 is published through the existing steward, independently verified against
  its sealed manifest, and recorded as artifacts; sase-1s7.7 closes normally and parent
  sase-1s7 is re-audited.
proposed_by: bbugyi200.athena.sase-1s7.7.land
proposed_with: codex/gpt-6-astra
bead: sase-1s7.7
create_time: 2026-11-11 15:53:30
status: wip
---

- **PARENT:**
  [202611/release_0_19_0_unblock.md](https://github.com/sase-org/sase--plans/blob/main/202611/release_0_19_0_unblock.md)
- **BEAD:**
  [sase-1s7.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1s7/sase-1s7.7.md)

# Finish the approved 0.19.0 release

This is the remaining work from the landing of **sase-1s7.7**. One agent can finish the
bounded operational recovery, verification, artifact publication, and landing. No
additional implementation phase or land agent is needed. This tale's executor must
perform the closeout below; nobody resumes the interrupted lander afterward.

## Evidence and completed work

Read `sase bead read sase-1s7.7 -r "Need landing findings and follow-up outcomes"` and
every child and note. Read these plans through `sase artifact read`:

- `plan:202611/release_0_19_0_unblock.md` (the epic being finished).
- `plan:202611/ci_green_v0_19_0_pypi_release.md` (parent **sase-1s7**).

Read `file:explicit:33034c169e9a1736ac9a6385` through `sase artifact read` for the
landing audit attached to sase-1s7.7, including the full evidence and API probes. At
audit time on 2026-11-11:

- Train-fixes commit `474b11f90d` is intact on current master `671ff5bde0`:
  `pin_changed` compares source pins only; exact core pin remains policy input;
  `publish.yml` regenerates and commits the release announcement; `release.yml` publish
  and confirm install uv first. Targeted verification passed **102 tests**, including
  the transport retry and candidate evidence contracts.
- Non-epic changes since that commit are `7cc41d89ab` (test split), `0eb9a0f06b`
  (publication caches), release/metadata commits `babc209f4e` and `c366d4524c`, and
  merge `671ff5bde0`. No integration edit was needed. Version fields and generated
  announcement agree on 0.19.0; the exact core dependency is 0.44.2.
- Release-branch rehearsal **42992733019**, head
  `1c87598fc2bf760486dfb77e115e3a4bb8327af1`, attempt 1, was accepted by phase
  sase-1s7.7.2. Required qualification, four exercise legs, and verify-only publish
  passed; the narrow gate-turn visual failure was advisory. Aggregate run status is
  `failure`, so judge individual required jobs. The accepted rehearsal predates the tag
  and merge. Do not dispatch a replacement rehearsal just for an advisory red.
- PR **#322** merged at **671ff5bde04fec5322ba6951adf0640d0dc61b02**, from head
  `c366d4524c3765cfd94ad004f6b562f8317049f4`, at 17:30:58Z. Retained chat
  `sase-1s7.7.3--3` contains the answered gate
  `custom-38f7a4ae-dca2-4350-9dca-0aeda5a43a45`: `approve_merge` selected,
  `merge_command_completed`. **merge_prompt=gate is satisfied; do not re-ask it.**
- Phase **sase-1s7.7.3 is closed canceled**, not done. PyPI sase/0.19.0 and GitHub
  release v0.19.0 both returned 404. No release proof artifacts exist yet.
- Latest listed Publish run **43066640321** (17:17:04Z) and steward run **43065694172**
  (17:11:21Z) both precede the merge. A completed merge does not prove the tag or train
  started.
- Required GitHub job reads fail with HTTP 502: core run **42673707335**, attempt 2,
  `/jobs` and `/attempts/2/jobs`, and the sase rehearsal job list. Connector reads time
  out too. `per_page=1` core pages 1 and 2 succeed, but pages 3 and 4 fail; the API
  advertises four jobs. Complete pagination fails, so smaller pages are **not** a
  demonstrated fix. Transport already retries three times with bounded backoff. Do not
  turn a partial job list into success.

## Constraints and ownership

Use `/sase_memory_read` for `sase_beads.md`, `lint_and_test.md`, and
`sase_artifacts.md`; read `docs/contribute/release-train.md`. Use `/sase_monitor` for
all long commands and waits. Use `/sase_repo` before opening any other repo.

This tale finishes an already implemented and approved release. No production source
change is currently justified by the HTTP errors. Do not increase timeouts, retry
budgets, or acceptance tolerances; do not substitute cached successes, change Rust
policy, bypass qualification, or manually upload distributions. If diagnosis reveals a
real code defect, preserve evidence, keep the epic open, and plan that specific defect
rather than pretending an API outage is fixed by a speculative edit.

Allowed release actions are the original plan's **bare**
`gh workflow run publish.yml -R sase-org/sase` and observe-only
`gh workflow run release-steward.yml -R sase-org/sase` (no `live` input). The existing
scheduled steward alone dispatches `release.yml` release/resume. Never manually dispatch
those modes, change repository variables/environments or branch protection, run
sase-core release-plz, merge another PR, or recut/move a tag.

Do not change lifecycle, agents, monitors, or crontabs for sase-1q2.9.x, sase-1r2.6,
sase-1rq.\*, sase-1n6.11.4, or sase-1oe.4. Check for a live owner before an outward
action and avoid duplicating it. Evidence notes and artifact links are allowed. Do not
take on Telemetry contention/budget repairs or advisory visual flakes.

The lander completed follow-up triage before proposing this tale:

- sase-1s7.7.2 note 3 corroborates existing **sase-1s1** (now +2); no new task.
- sase-1s7.7.3 note 1 is routed as a **DISCOVERED ISSUE to sase-1n6**, the still-open
  original epic whose phase sase-1n6.7.1.2 introduced the affected reader in
  `23cccbc969`. No duplicate task was found. Persistent GitHub failure does not
  demonstrate a regression in the correctly bounded parent retry implementation. No
  separate task is created; publication/proof remain this epic's obligation.

Carry these outcomes into the final close note without re-filing them.

## 1. Recover complete observations and advance the existing release

Fetch origin, inspect changes since `671ff5bde0`, and re-inventory PR #322, tag, GitHub
release, PyPI, and scheduled Publish/steward/release.yml runs. The target stays
**0.19.0**; do not silently switch to another version or start a second cycle. If
already published, proceed directly to verification.

Reopen canceled **sase-1s7.7.3** normally when taking over its unfinished work:
`sase bead open sase-1s7.7.3`. Do not reopen phases .1 or .2.

When no post-merge Publish run or matching train is active and no v0.19.0 release
exists, dispatch **one bare publish.yml** run, as already authorized by the epic. Record
its ID and wait for the entire run, not merely a branch movement. This allows
release-please to create the tag and draft after the approved merge. Do not duplicate an
in-flight run. Read its actual failure if it fails.

Recheck the specific failed job endpoints and the complete read-only steward report.
Wait for full evidence, not just an HTTP success on one page. Cross-check returned job
IDs, attempt numbers, names, and conclusions; preserve error diagnostics. Recover the
accepted rehearsal's full job evidence when accessible and confirm its per-criterion
acceptance against phase .2's note 4. Its visual-test failure remains advisory. Notes
about the accepted rehearsal already went to sase-1q2.9.3/sase-1oe.4.

For a service outage, use a **zero-weight monitor**, polling no faster than five minutes
within one absolute **24-hour recovery deadline**, carried unchanged across
continuations. Monitor completion must return here even on a nonzero steward exit; the
successor reads the evidence and chooses the next bounded action. No endless deadline
resets. Use the existing read-only command:

```text
.venv/bin/python tools/release_watch --until-released 0.19.0 --poll 5m --deadline 24h
```

Reduce its deadline to the remaining recovery window on later invocations. A monitor
follow-up must retain this plan, the epic and phase IDs, all run IDs, and the remaining
verification and closeout steps. Read `/sase_monitor` for staging; wait until the start
command exits with Staged, then finish through `/sase_final` with beads kept open.

On `watcher_broken`, first inspect the endpoint error; on `burned`, `abandoned`,
`publish_conflict`, `vetoed`, or exhausted recovery deadline, record the exact blocker
on sase-1s7.7 and sase-1s7. Keep their plans unfinished and report the blocker. Do not
close successful work as done without proof or cancel the epic to clear the queue.
Obtain any newly required authority through the appropriate SASE gate; this plan does
not grant broader release actions.

## 2. Verify the actual release and publish evidence

Require all of these, independently of the old canceled phase status:

1. The actual v0.19.0 release.yml train succeeds, with `confirm` green, and a fresh
   `tools/release_watch --once --json` verdict is `released`. Inspect per-job outcomes,
   not only the aggregate run conclusion.
2. PyPI JSON lists exactly the sealed wheel and sdist. Compare each filename, size, and
   SHA-256 to the retained candidate manifest on the GitHub release using
   `tools/release_candidate show`/`download`. Require exact file-set equality.
3. Both PyPI provenance documents name `sase-org/sase` and `release.yml`.
4. GitHub release v0.19.0 is published, not draft, and marked Latest. Record tag, peeled
   SHA, merge SHA, candidate generation, and release run URL.
5. A clean Python 3.12 installation from public PyPI of `sase==0.19.0` passes
   `sase version -j` and `sase core health --json`, and reports the manifest's exact
   core pin. Use a throwaway isolated subprocess home and SASE/XDG/uv tool directories,
   scrub inherited SASE session variables, and set UV_NO_CACHE=1. Do not install
   globally or repurpose this agent's HOME.

Write and register three explicit artifacts using
`sase artifact create --bead sase-1s7.7.3`: `sase-0.19.0-pypi-release-evidence.md`, raw
`pypi-sase-0.19.0.json`, and `install-check-sase-0.19.0.txt`. The report must include
the identity and job evidence, manifest/PyPI digest table, provenance, clean install,
released verdict, rehearsal URL, and **sase-1s7.5's workflow table with its actual
remaining reds identified**. Read that bead and its notes; do not claim every CI
workflow is green because release qualification passed.

Verify each artifact with `sase artifact show`. Link each as `related` to
bead:sase-1n6.11.4, bead:sase-1r2, and bead:sase-1s7 with the reason from the original
release plan. Note the evidence refs on sase-1n6.11.4, sase-1q2.9.4, and sase-1r2.6;
change none of their statuses. Read-only inspect bot PRs #300/#319/#320/#323/#326 and
external PR #301 so the final release summary can name their current CI state; do not
merge or close them.

## 3. Finish this epic and re-audit its concrete parent

This is the tale's final step. It must not depend on this tale's own future commit SHA,
push, or CI result. Closeout in the same turn as final edits is normal. No land agent
will resume this work for you.

Re-read all descendants and notes, original plan, decisions, and post-audit drift.
Finish any new epic-caused work; triage genuinely distinct unrelated discoveries through
`/sase_new_task`. If any tracked source files changed, run `just fix` then
`sase tool run check` (the recorded `just check`), never `just check-full`. Plan-only
changes use plan validation. Do not expand this tale to repair check-full-only
infrastructure failures.

After actual release proof exists, resolve any symbols keyed to sase-1s7.7.3 and close
the reopened phase normally with the evidence refs. If the assigned tale has its own
open descendant bead, finish and close that normally before attempting its ancestor
close; an unfinished descendant is never a reason for force.

Run **`sase bead epic-symbols sase-1s7.7`**. For every entry, wire it up, privatize it,
delete dead code/tests, or add a valid real non-test-consumer pragma according to
`symvision.md`. Re-key only when a specifically identified still-open later bead still
needs the exemption. Then run:

```text
sase bead close sase-1s7.7 --note "<source/integration verification, completed release and artifact refs, decisions, tests, and both follow-up outcomes>"
just symvision
```

Open the plans repository with `/sase_repo` and set **`status: done`** in
**`202611/release_0_19_0_unblock.md`** (the PLAN path resolved by the bead). Include
this repository in finalization if modified. If close refuses, finish its actual symbol
or descendant blockers, then retry normally. Never use `--force` merely to make close
succeed or to advance a successful nested landing. A deliberate canceled or superseded
resolution is only appropriate for genuinely abandoned work, with an explicit reason; it
is not the success path here.

Next run `sase bead read sase-1s7.7 -r "Need the parent link"`. Its parent is the **plan
bead sase-1s7**, not a phase. Read that parent's previous landing note, every descendant
and note, and `plan:202611/ci_green_v0_19_0_pypi_release.md`; inspect all post-child
commits and rerun descendant/linked-plan readiness checks. Its broad
every-workflow-green requirement was previously incomplete: .4/.5/.6 canceled, with
Telemetry contention and test-cost issues owned elsewhere. This release alone does not
prove that requirement.

If sase-1s7 is fully complete on fresh evidence, resolve
`sase bead epic-symbols sase-1s7`, close with
`sase bead close sase-1s7 --note "<fresh parent readiness and integration proof>"`, run
`just symvision`, and mark **`202611/ci_green_v0_19_0_pypi_release.md`** done. Repeat
through any directly parented plan ancestors found at that time, only while each is
fully complete. Stop at the first incomplete or ambiguous parent, append a specific
blocker note there, and report it. Never force any ancestor closed.

Finish through `/sase_final`, declaring every modified repository. The final answer must
distinguish the closed release epic from any still-open parent, and link PR, tag/train,
PyPI, evidence artifacts, and follow-up outcomes.
