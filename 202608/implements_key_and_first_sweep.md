---
tier: tale
size: medium
title: Derive `implements` from the plan's own bead, then run the first sweep
goal:
  "`sase artifact link` derives `plan implements bead` from a plan's `bead_id:`
  frontmatter — the plan's own bead — instead of `bead:`, which names the agent that
  proposed the plan; and the retroactive derivation sweep epic sase-tw built has
  actually run once on this machine, so the derived graph exists in the store instead of
  only in the code."
proposed_by: bbugyi200.athena.sase-tw.land
bead: sase-tw
---

- **PARENT:**
  [202608/artifact_link_durability_and_derivation.md](artifact_link_durability_and_derivation.md)
- **BEAD:**
  [sase-tw](https://github.com/sase-org/sase--beads/blob/main/pages/sase-tw/README.md)

# Plan: Derive `implements` from the plan's own bead, then run the first sweep

## Problem

Epic sase-tw ("Artifact links that survive, derive themselves, and pay for the turn")
landed all fourteen phases. Its landing verification found two of its own acceptance
conditions unmet. Both are recorded in full on bead `sase-tw`; this plan is only the
remainder.

Epic sase-tw is still open and its landing is waiting on this work. Everything else that
landing needs is already done — the fourteen phases are verified, the epic-symbol
whitelist is clean, flag bead sase-tx is closed, and every phase follow-up is
dispositioned — so the epic's own close-out is not part of this plan and is not work for
its worker.

### The `implements` rule reads the wrong frontmatter key

`src/sase/artifact_links/derive/_plan_implements.py` reads `frontmatter.get("bead")`.
sase-core's plan frontmatter schema (`crates/sase_core/src/plan/validate.rs`) defines
the two keys differently:

- `bead` — "bead id of the agent that proposed this plan, written by SASE"
- `bead_id` — "Epic bead id written by SASE after creation"

`bead_id` is the plan's own bead. `bead` is the proposing agent's bead, which is why
`plan:202608/monitor_land_fixes.md` carries `bead: sase-kp` — the epic its land agent
was landing, not a bead that plan implements.

The epic plan named `bead_id` twice: owner decision 1
("`plan:<relpath> implements bead:<id>` derived from `bead_id:` frontmatter is correct
and directed that way") and its derivation-core section ("Plan `bead_id:` frontmatter →
`implements` (602 edges)"). The coverage report in
`src/sase/artifact_cli/link_health.py` already labels the population
`plan bead_id implements`. Only the rule itself disagrees.

Measured over the 4,003 plans the sweep walks:

| Frontmatter key | Plans | Resolve to a live bead |
| --------------- | ----: | ---------------------: |
| `bead_id`       |   602 |                    541 |
| `bead`          |   207 |                    207 |

The two populations are disjoint (602 + 207 + 3,194 with neither = 4,003). So the rule
as written would assert 207 rows that say a plan implements its own proposing agent's
bead, and would never write the 541 correct ones.

### The retroactive sweep has never run

The epic's derivation-hooks section required running the sweep once and recording the
resulting row counts on the phase bead. That phase auto-closed through
`sase stitch create` with no verification note, and the sweep never ran. Live state on
master 7f6f936de:

- `sase artifact link list --origin derived --limit 0` returns 0 rows.
- `sase artifact doctor` reports
  `plan bead_id implements 0 linked / 207 candidates (0%)` and
  `prompt header cites 0 linked / 141 candidates (0%)`.
- The aggregate holds 465 rows against the epic's ~1,600 projection.
- No `artifact_link_backfill.json` checkpoint exists under `~/.sase/projects/`, and the
  running housekeeping lumberjack — started before the chop landed — does not list
  `artifact_link_backfill` among its chops.

Order matters here. `sase_chop_artifact_link_backfill` is registered in the
`housekeeping` bucket in `src/sase/default_config.yml`, and a workspace `sase` already
lists it (`.venv/bin/sase axe chop list`). The axe running on this machine does not yet
— its orchestrator predates the chop, and the `sase` on `PATH` is a separate install
that has not picked up this version. So nothing is writing derived rows this hour, but
the first thing that does will write whatever the rule says. Land the key fix before
running any sweep, so the wrong 207 rows never enter the durable graph.

## The fix

**Files.** `src/sase/artifact_links/derive/_plan_implements.py`,
`tests/artifact_links/test_plan_implements.py`.

Read `bead_id` instead of `bead`. Update the module docstring, the function docstring,
and the candidate description text (currently "derived from the plan's `bead:`
frontmatter field") so all three name the key the rule actually reads. Every existing
skip case stays: a non-plan ref, unreadable or invalid frontmatter, a missing or blank
field, and a bead id absent from `known_bead_ids` all still yield no candidate.

Do **not** add `bead` as a fallback. A plan does not implement the bead of the agent
that proposed it, and a fallback would put exactly the 207 wrong rows back into the
graph through a second door. If a later consumer wants the proposing-agent association,
that is a different relation and a different decision.

The six tests in `tests/artifact_links/test_plan_implements.py` all use `bead:` in their
fixtures. Convert them to `bead_id:`, and add one test asserting the negative that
motivated this plan: a plan whose frontmatter carries `bead: <live-bead-id>` and no
`bead_id` yields no candidate.

`src/sase/artifact_cli/link_health.py` needs no change — its coverage report calls
`derive_candidate_links`, so its `plan bead_id implements` population follows the rule.

## The sweep run

Run the retroactive sweep the epic's derivation-hooks phase specified, from a workspace
where the `research` sidecar is cloned. That matters:
`sweepable_artifact_link_documents` walks `store.sidecar_roots` per kind and skips a
root that is not a directory, so a workspace without the research clone derives zero
research-swarm `derives-from` rows and silently under-reports the sweep. Check
`sase repo list` before starting, and say in the bead note which workspace was used and
whether research was cloned there.

Drive it through the chop rather than by hand, so the run exercises the code path that
will run hourly:

```bash
.venv/bin/sase axe chop run artifact_link_backfill -V
```

Use the workspace `.venv/bin/sase`, not the `sase` on `PATH`: only the workspace install
carries this chop today. The chop bounds job 1 to a 45-second work budget per tick and
checkpoints swept refs under `~/.sase/projects/`, so it takes several runs to converge
over ~4,000 documents. Keep running it until the summary's `sweep_remaining` reaches 0;
that is the point at which the "a second sweep is a no-op" acceptance means anything.

Persistence commits the touched `links/` companions through the normal store path, so
expect commits in the plans and research sidecars and bead events for bead-endpoint
rows. That is the designed behavior, not a surprise: verify afterwards that no sidecar
is left dirty.

Record on the bead, as numbers rather than adjectives: rows by origin and by relation
before and after, the sweep's scanned/persisted/remaining totals, both doctor coverage
percentages after, and the aggregate row count after.

## Acceptance

- `sase artifact link list --origin derived --limit 0` returns roughly 800 rows: ~541
  `implements`, ~141 `cites`, and ~116 research-swarm `derives-from`. The last term is
  the one that depends on running from a workspace with the research clone; if it comes
  back near zero, the sweep ran from the wrong place. Every `implements` row's target
  must be the plan's own bead — spot-check three against each plan's `bead_id:`
  frontmatter.
- `sase artifact doctor` reports `plan bead_id implements` at 541 of 541 candidates, or
  explains any shortfall on the bead rather than moving the number.
- A second full sweep persists 0 new rows.
- No row in the store asserts `plan:<x> implements bead:<y>` where `<y>` is the plan's
  `bead:` value and the plan carries a different `bead_id:`. If the chop already ran
  before this plan landed, find those rows with
  `sase artifact link list --relation implements` and remove each one with
  `sase artifact link rm`, then say how many were removed.
- `just check-full` passes.
