---
tier: tale
title: Recover the lost sase-m6.7.1.6 conformance, docs, and perf-gate work
goal:
  Relation and grouping actions are reachable on every pane whose contract declares
  them, the conformance harness enforces that reachability, the synthetic notes fixture
  covers the third-party relation and grouping case, the contract and banner-row
  documentation exist, and the navigation performance gate is re-measured on the landed
  tree.
size: medium
proposed_by: bbugyi200.athena.03c
create_time: 2026-08-16 09:27:29
status: done
---

- **PROMPT:**
  [prompts/202608/recover_artifacts_conformance_phase.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/recover_artifacts_conformance_phase.md)
- **AGENTS:**
  - [bbugyi200.athena.03c](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.03c.md)
- **COMMITS:**
  - [78a9130](https://github.com/sase-org/sase/commit/78a9130f753609fab8a6adb9d3245afb05574d46)
    — fix(tui): honor Artifacts pane contract actions

# Plan: Recover the lost `sase-m6.7.1.6` conformance, docs, and perf-gate work

## Why this plan exists

The `sase-m6.7.1` child epic ("Relations, reveal, and grouping as Artifacts contract
features") has all six phase beads marked `CLOSED` with `resolution: done`, but the
sixth phase never landed. `sase-m6.7.1.6` — "Conformance, docs, and the relation
performance gate" — was closed at `2026-08-16T12:51:16Z` and produced **no commit, no
patch, no branch, and no dirty workspace**. Its work is gone.

Every other phase committed within a minute of its own close:

| Phase | Closed (UTC) | Landed commit                                                           |
| ----- | ------------ | ----------------------------------------------------------------------- |
| `.1`  | 07:14:45     | `2abe188aa` feat(artifacts): declare pane relation and grouping facts   |
| `.2`  | 08:31:07     | `708c25452` feat(artifacts): add host-owned RelationIndex               |
| `.5`  | 09:16:40     | `f5dda81f3` feat(artifacts): put every pane on the shared fold registry |
| `.3`  | 10:55:05     | `a0b6cd16b` feat(tui): generalize artifact relation navigation          |
| `.4`  | 11:56:21     | `30c9ba23b` feat(artifacts): add reversible relation reveal lens        |
| `.6`  | 12:51:16     | **none**                                                                |

`master` and `origin/master` are both at `30c9ba23b` (12:00:26Z), which predates `.6`'s
close by 51 minutes. `sase patch search '"m6.7"'` and `'"conformance"'` return nothing,
no branch matches the phase, and every sibling workspace `sase_10`–`sase_25` is either
clean at or behind `30c9ba23b`.

The phase bead's own closing note claims it "Verified harness relation/grouping checks,
notes fixture family+related+grouping (`hello__a` filename family plus related link and
`by_status` grouping), action reachability, and docs" and that markdown Prettier ran
against `docs/artifacts_pane_contract.md`. None of those artifacts exist on `master`.
The agent did the work and lost it before it landed.

This plan restores exactly that deliverable set. It is scoped to one follow-up coding
agent and is sized `medium` to match the original phase.

## Current state, verified against `30c9ba23b`

Run these to reconfirm before starting; each is the evidence for one work item.

```bash
# 1. Relation/grouping actions are unreachable outside Patches.
.venv/bin/python -c "
from sase.ace.tui._app_action_availability import check_app_action
class App:
    class _S: _blocks_global_config_center_open=False
    screen=_S(); focused=None; _screen_stack=()
    current_tab='artifacts'; current_artifacts_pane_key='stitches'
    current_artifacts_subtab='stitches'; active_artifacts_contract=None
app=App()
acts=['cycle_grouping_mode','cycle_grouping_mode_reverse','expand_or_layout',
      'hooks_or_collapse','hooks_or_collapse_all','expand_all_folds',
      'start_ancestor_mode','start_child_mode','start_sibling_mode']
for pane in ('patches','stitches','beads','plans','files'):
    app.current_artifacts_pane_key=pane; app.current_artifacts_subtab=pane
    print(pane, [a for a in acts if check_app_action(app,a,(),lambda a,p: True) is False])
"
# 2. Harness has zero relation/grouping checks.
grep -c -i "relation\|grouping" tests/ace/tui/artifacts_contract/harness.py   # -> 0
# 3. Fixture was never extended.
ls tests/ace/tui/artifacts_contract/fixtures/notes/   # -> provider.yml, hello.md only
# 4. Contract doc does not exist.
ls docs/artifacts_pane_contract.md                    # -> No such file
```

The base tree is green: `pytest tests/ace/tui/artifacts_contract/ -q` passes 105 tests
**while the reachability bug is live**, which is itself proof the harness has no
reachability check.

### Already landed — do NOT redo

These were delivered by earlier phases. Read them, build on them, do not rewrite them.

- `_rule_relations` / `_rule_grouping` in
  `src/sase/ace/tui/_artifact_tab_contract_rules.py` (phase `.1`). The later-phase
  exemption has **already** shrunk to `STATUS_COUNTERS` and `SHELL` (lines 528–529) —
  that part of `.6`'s description is done.
- The "Relation panel slot" section in `docs/artifacts_pane_visual_grammar.md` (phase
  `.3`).
- `tests/ace/tui/artifacts_contract/test_relation_goldens.py` (phase `.3`).
- `check_declared_actions_are_registered` in `harness.py` already proves every declared
  capability's actions are _registered_. What is missing is that they are _reachable_.
- `ref.relations` and `ref.grouping` provider parsing already exists
  (`extract_provider_relations` / `_RELATION_KEYS` / `_GROUPING_KEYS` in
  `src/sase/ace/tui/_artifact_tab_contract_provider.py`). The fixture only needs the
  declaration; no parser work is required.

## Work items

### 1. Make declared relation and grouping actions reachable

`NON_PRS_ARTIFACT_ACTIONS` (`src/sase/ace/tui/actions/artifacts.py:40`) was never
extended when `f5dda81f3` routed `h`/`l`/`H`/`L`/`o`/`O` onto the shared fold registry
and `a0b6cd16b` mounted relation navigation across panes. `check_app_action`
(`src/sase/ace/tui/_app_action_availability.py:89-94`) therefore returns `False` for all
nine actions on every non-Patches pane, so Textual skips the binding entirely.

All nine bindings and action methods already exist (`bindings.py:203,234-236`,
`default_config.yml:375-378,411-412,435-437`, `actions/agents/_grouping.py:248,267`,
`actions/agents/_folding.py:86`, `actions/navigation/_tree.py:86,102`). Only the
allowlist gate is missing.

Add to the allowlist so they reach Stitches and Plans: `cycle_grouping_mode`,
`cycle_grouping_mode_reverse`, `expand_or_layout`, `hooks_or_collapse`,
`hooks_or_collapse_all`, `expand_all_folds`, `start_ancestor_mode`, `start_child_mode`,
`start_sibling_mode`.

**Preserve the key-collision decision the lost phase already made.** On `files`, `o` is
owned by `files_open_external`; on `beads`, `o` is owned by `beads_open_bug`. Keep
`cycle_grouping_mode` / `cycle_grouping_mode_reverse` gated off on those two panes
rather than stealing `o`. Assigning a non-colliding grouping key there belongs to
`sase-m6.9` — record it as `PROPOSED FOLLOW-UP:` on the bead, do not fix it here.

Gate reachability on the pane's compiled contract (`PaneCapability.RELATIONS` /
`PaneCapability.GROUPING`) rather than a hard-coded pane-id list, so a sidecar that
declares a relation property earns its jumpers without shipping code — that is the
epic's whole premise.

**Performance constraint, learned the expensive way.** The lost phase's bead recorded
that it had to gate contract lookup in `check_app_action` to relation/grouping/
query-history actions only, because `check_app_action` runs on every keypress and an
unconditional contract lookup regressed `j`/`k`. Do the same: look the contract up only
for the actions that need it, never on the navigation path.

### 2. Extend the conformance harness

`tests/ace/tui/artifacts_contract/harness.py` (156 lines, untouched since `a0b6cd16b`)
gets new checks appended to `PANE_CONFORMANCE_CHECKS`, parametrized by the existing
`iter_conformance_cases` over every resolved sub-tab including degraded and synthetic
providers:

- **Relations resolve.** For each `PaneRelationDecl` on `descriptor.resolved_contract`,
  the relation resolves to a reachable `ArtifactEntryTarget` or records a diagnostic. A
  declared-but-silently-empty relation must fail.
- **Grouping banners are navigable and foldable.** For each `PaneGroupingModeDecl`, the
  mode produces banner rows that are reachable as navigation stops and respond to
  fold/unfold — collapsed banners are first-class `j`/`k` targets per `docs/ace.md:679`.
- **Declared actions are reachable, not merely registered.** Assert `check_app_action`
  does not return `False` for every action in
  `CAPABILITY_HOST_ACTIONS[RELATIONS | GROUPING]` on each pane whose contract enables
  that capability. This is the check whose absence let work item 1 ship broken; it must
  fail against unpatched `master`.

Confirm that last point by stashing the item-1 fix and watching the new check go red. A
conformance check that passes both before and after the fix is not testing anything.

### 3. Extend the synthetic `notes` fixture

`tests/ace/tui/artifacts_contract/fixtures/notes/` currently holds only `provider.yml`
and `hello.md`. Add:

- A `ref.relations` block in `provider.yml` declaring a `link` relation and a `family`
  relation, plus a `ref.grouping` block with a `by_status` mode over the
  already-declared `status` property.
- A second, suffixed document `hello__a.md` so the filename-family edge has a real
  counterpart.

Valid `ref.relations` fields are exactly `name`, `kind`, `label`, `source`,
`target_pane`, `inverse`, `directed`, `transitive`; `ref.grouping` takes `default_mode`
and `modes`. Both are already validated by `extract_provider_relations`.

The point is that a **third-party provider, not a built-in**, is the conformance case
for the declared-property edge and the filename family. The fixture must stay data-only
— `test_synthetic_fixture_is_declarative_data_only`
(`tests/ace/tui/artifacts_contract/test_synthetic_provider.py:26`) is the guard, and it
asserts on the exact file list, so update it to expect `hello__a.md`.

### 4. Documentation

- **Create `docs/artifacts_pane_contract.md`.** Write the three relation primitives
  (`hierarchy`, `family`, `link` — `RelationKind` in `_artifact_tab_model.py:126`), the
  `ref.relations` and `ref.grouping` declaration blocks with their validated field sets,
  and the reveal lens and its reversibility guarantee. This document is _completed_ by
  `sase-m6.10`; this phase starts it.
- **Extend `docs/artifacts_pane_visual_grammar.md`** with the banner row's treatment.
  The "Relation panel slot" section (line 31) already landed; the banner row section is
  what is missing.
- Run `just fmt-md` / `just fmt-md-check` — the lost phase hit markdown Prettier
  failures on exactly these files.

### 5. Re-capture the performance gate

The `p95` numbers on the phase bead were measured against a tree that no longer exists,
so they do not certify `master`. Re-measure per `docs/perf_runbook.md`:

```bash
SASE_TUI_PERF=1 .venv/bin/python tests/ace/tui/bench_artifacts_jk.py
```

Gate: navigation `p95` under 16 ms on every converted pane. Record the numbers on the
bead.

Known and **not** caused by this work: `stitches.next` / `stitches.up10` hover at 16–18
ms on unmodified `master` too (recorded baseline: `stitches.next` 16.47, `stitches.up10`
17.95). Do not chase it here; the lost phase already traced it to `CommitsTimeline`
key-to-paint, not relation-index rebuild, and filed it for follow-up. Re-record the
baseline alongside the measurement so the comparison stays honest.

### 6. PNG goldens

The relation panel and banner rows move pixels. Run `just test-visual`, inspect
`.pytest_cache/sase-visual/` actual/expected/diff artifacts, and regenerate **only**
goldens whose sole difference you have actually looked at. Retired task `sase-lo` warns
that a blanket `--sase-update-visual-snapshots` silently absorbs unrelated drift.

## Bead handling

`sase-m6.7.1.6` is closed `done` but is not done, and the parent epic `sase-m6.7.1` and
phase `sase-m6.7` cannot honestly close until this lands. The implementing agent should
**not** hand-edit bead status. Ask the project owner whether to reopen `sase-m6.7.1.6`
or to track this as a new phase under `sase-m6.7.1`; `sase-m6.7.1` is currently assigned
to `sase-m6.7.1.land`, and `sase-m6.land` is WAITING on the whole `sase-m6` clan, so the
choice affects the landing sequence.

Carry forward the lost phase's still-valid `PROPOSED FOLLOW-UP:` notes rather than
rediscovering them:

- `SHELL` still binds to `_rule_later_phase` despite `sase-m6.5` having landed →
  `sase-m6.10`, with `STATUS_COUNTERS`.
- Files `o` / Beads `o` grouping-cycle key collision → `sase-m6.9`.
- `sase monitor show` can abort with `ValueError` on stale non-monitor artifact records.
- `AcePageGroup` notification baseline can leak in large selected lanes (45 teardown
  errors under `just check`, 45/45 passing in isolation).

## Non-goals

- Do not fix the `SHELL` / `STATUS_COUNTERS` later-phase exemption (`sase-m6.10`).
- Do not assign a new grouping key on Files or Beads (`sase-m6.9`).
- Do not chase the pre-existing Stitches `j`/`k` timing.
- Do not re-do phases `.1`–`.5`; they landed and are green.

## Verification

`just install` first — workspaces are ephemeral and dependencies drift.

This phase touches `tests/ace/tui/test_artifacts_*` broadly, so `just check`'s scoped
lane will escalate. Run `just check-full` through `/sase_monitor`
(`sase monitor start --command 'just check-full' …`) with a `--next` action so the
follow-up agent acts on the result. It routinely outruns a single agent turn and must
never be run inline.

Done means:

1. The nine actions are reachable on every pane whose contract declares the capability,
   verified through `check_app_action`, with Files/Beads `o` deliberately excepted.
2. The new conformance checks fail against unpatched `master` and pass after.
3. `pytest tests/ace/tui/artifacts_contract/` passes with the extended fixture.
4. `docs/artifacts_pane_contract.md` exists; the visual grammar doc covers banner rows;
   `just fmt-md-check` passes.
5. `SASE_TUI_PERF=1` `p95` recorded on the bead, under 16 ms except the documented
   pre-existing Stitches baseline.
6. `just check-full` green, PNG goldens individually reviewed.
