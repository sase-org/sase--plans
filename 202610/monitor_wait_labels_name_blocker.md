---
tier: tale
title: Monitor wait labels name what they wait on
goal:
  A monitor that waits on a bead, agent, or external system says so in its row label,
  and an agent that discovers a prerequisite bead records it as a dependency edge.
size: small
decisions:
  dep_edge:
    ask:
      Should the monitor skill tell an agent to record its prerequisite on its own bead
      before it polls another bead?
    choices:
      record:
        Agents add the dep edge first, so bead views and any relaunch see the
        prerequisite
      label_only:
        Only the monitor label names the blocker; the bead graph stays as authored
    default: record
    why:
      The root gap was a missing edge; a relaunched phase then waits natively on the
      bead.
    answer: record
proposed_by: bbugyi200.athena.14r
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-27 10:39:19
status: wip
---

# Monitor Wait Labels Name What They Wait On

## Why

The Agents tab showed two clans that were both blocked on one chain, but the rows read
as unrelated waits:

- `sase (WAITING CORE)` is `sase-1oe.4`. Its monitor polls
  `tools/ratchet_core_pin --check` until a published sase-core release contains master's
  core pin.
- `sase (WAITING CUTOVER)` is `sase-1n6.10`. Its monitor runs a hand-written Python loop
  that polls `sase bead read sase-1oe.4` every 5 minutes for up to 7 days. "CUTOVER" is
  the `sase-1oe` epic's internal codename, so the row hides the fact that it waits on
  `sase-1oe.4`, the other row on screen.

So it is one chain: `sase-1n6.10` → `sase-1oe.4` → sase-core publication. Nothing is
stuck. sase-core `v0.43.0` was tagged at 13:47 UTC on 2026-10-27 and contains the pinned
commit `800caef4`, and its Release-plz publish job was still running when this plan was
written. Once the wheels reach PyPI, both monitors unwind in order without any action.

Both labels are free text that agents passed to `sase monitor start -s/-S`. SASE does
not know what a monitor waits on, so a label is the only explanation a reader gets.
`/sase_monitor` fixes only the `TESTING`/`TESTED` pair. For other waits it says "pick
your own pair" and names "another bead" as a valid polling target. It gives no guidance
on naming the target.

Monitor history (3,035 monitors) shows the resulting drift. One sase-core publication
wait appears as `WAITING CORE`, `WAITING CORE CUT`, `WAITING FOR CORE`, and
`WAITING ON PYPI`. CI waits appear as `WAITING ON CI`, `WAITING FOR CI`, `WATCHING CI`,
`WAITING FOR PR CI`, and a bare `WAITING`.

`sase-1n6.10` was running at all because `sase-1oe` ("Unblock E2 (sase-1n6)") was filed
with no dependency edge into `sase-1n6`. `sase-1n6.10` depends only on `sase-1n6.9`,
which is closed. The agent found the real prerequisite at run time and improvised the
poller. Only one monitor in history has polled a bead. That corpus is too thin to
justify a native "wait on bead" monitor mode, so this plan fixes the guidance instead.

## Changes

### 1. `src/sase/macros/skills/sase_monitor.md` — `## Status Labels`

Add a short "name what you wait on" rule after the present/past-tense paragraph:

- A wait's label names what it is waiting on, in terms a reader outside your epic would
  recognize. Never use your plan's internal codename, such as `CUTOVER`.
- When the blocker is a SASE bead or agent, put its ID in the label: `WAITING <id>` /
  `WAITED <id>` (for example `WAITING sase-1oe.4` → `WAITED sase-1oe.4`). Use the full
  ID when the label fits within 20 characters; otherwise use the shorthand after the
  final dash. Always put the full ID in `--label`.
- Add a compact table of shared pairs for common external waits, so related monitors
  read as one lane. A GitHub Actions run or PR checks use `WATCHING CI` → `WATCHED CI`.
  A release use `WATCHING RELEASE` → `WATCHED RELEASE`. A timed sleep uses
  `SLEEPING FOR <n>s` → `SLEPT FOR <n>s`. Any other external system is named directly,
  for example `WAITING PYPI`.

Keep the existing sentences that `tests/main/test_init_skills_sources.py` pins: "present
tense", "past tense", "20 characters", and "When no profile supplies labels".

### 2. Same file — `## Sleep Or Wait`

The paragraph that lists "a release tag, PyPI, another bead" gains a short "Waiting on
another bead" subsection. It covers the following points.

> [!decision] dep_edge = record First record the prerequisite on your own bead with
> `sase bead dep add <your-bead> <blocker>`. Skip this when your bead already depends on
> the blocker. Epic phase workers may add this edge to their own phase bead; it is not a
> new bead. Then start the monitor.

> [!decision] dep_edge = label_only Omit the `sase bead dep add` step and keep only the
> labeling and example below.

- Add one canonical example that waits on a bead with `-w 0`, a multi-day `--timeout`,
  `--start-status 'WAITING <id>'`, `--stop-status 'WAITED <id>'`, and
  `--label '<id> prerequisite'`. The example ends with a `--next` that tells the
  follow-up to read the blocker's resolution, because a canceled blocker is not success.
  Its poll command uses `sase bead read <id> -N -f json -r '<why>'`. Do not use
  `sase bead show`, which refuses inside an agent run. The loop exits once the bead's
  own `status` is `closed`, sleeping a few minutes between polls. Keep it to a short
  shell `until` loop, so agents stop writing bespoke Python pollers.
- Verify the exact poll command live before committing. Run its condition once against a
  closed bead and once against an open or in-progress bead, and confirm it ignores
  nested statuses such as children or dependencies in the JSON envelope. Adjust the
  command until both cases behave.

### 3. `docs/reference/automation/monitors.md`

The `-s/--start-status` bullet currently says "a different kind of wait picks its own
pair". Extend it with one sentence: a wait names what it waits on, using a bead or agent
ID or the shared pairs in `/sase_monitor`. Read the `docs_writing` reference memory
before editing.

### 4. `tests/main/test_init_skills_sources.py`

Extend the `sase_monitor` required-substring tuple with the new contract:
`WAITING <id>`, `WATCHING CI`, `--label '<id> prerequisite'`, and, if `dep_edge` =
`record`, `sase bead dep add`.

## Out Of Scope

- A native wait-on-bead monitor mode (`sase monitor start --bead`), or TUI rendering of
  a monitor's blocker as a live bead token. Both wait until bead-polling monitors are
  common.
- Re-bucketing zero-weight monitors out of `Running`. Both rows in the screenshot sit in
  the Running group and count toward "7 running" even though `-w 0` holds no capacity.
  That change touches shared status-bucket semantics and clan label mirroring, so it is
  tracked separately as task `sase-1ow`.
- Do not deploy the regenerated skills. Per the `generated_skills` reference memory,
  `sase skill init --force` runs only from a landed source revision. Use
  `sase skill init --diff` read-only to preview the render.

## Verification

- `sase skill init --diff` shows the expected `sase_monitor` changes for each provider.
- The live poll-command check from change 2 passes for both closed and open beads.
- Run `just check` through `sase tool run check`, following the `lint_and_test`
  reference memory.
