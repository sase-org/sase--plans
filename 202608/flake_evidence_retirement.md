---
tier: tale
size: medium
title: Retire a fixed node's historical flake evidence per node
goal: "`just selection-health --fail-on-new-flake` can be told that one specific node
  has been fixed, so that node's pre-fix failure records stop counting as flake evidence
  while every other node's evidence and the gate's bar for genuinely new flakes stay
  exactly as they are: the gate exits 0 on master with the nine config nodes commit
  3a22ff04f fixed retired, and the four node IDs owned by live beads still reported.

  "
proposed_by: bbugyi200.athena.sase-ns.6.1
bead: sase-ns.6.1
create_time: 2026-08-16 21:11:34
status: wip
---

- **PARENT:** [202608/task_backlog_top5.md](task_backlog_top5.md)
- **BEAD:**
  [sase-ns.6.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ns/sase-ns.6.1.md)

# Retire A Fixed Node's Historical Flake Evidence

## Goal

`just check-full`'s last gate — `just selection-health --fail-on-new-flake` — can be
told that one specific node has been fixed, so that node's pre-fix failure records stop
being permanent evidence, while every other node's evidence and the gate's bar for
genuinely new flakes are untouched.

Concretely: after this work, the gate exits 0 on master, the nine config / config-cache
nodes that commit `3a22ff04f` fixed are no longer reported, and the four node IDs that
belong to live beads are still reported.

## Background (verified live, this workspace, after `just install`)

`python3 tools/selection_health --fail-on-new-flake` on master `4819a0314` reports
exactly the 13 nodes the bead describes:

```
flake baseline gate: 13 reproducible flake(s) exceed tests/reproducible_flake_baseline.txt
  (records after 2026-08-15T17:22:27Z, at most 5 failures per run):
  tests/ace/tui/test_top_bar_order.py::test_override_pills_keep_narrow_top_bar_in_bounds
  tests/main/test_var_integration.py::test_var_cli_end_to_end_refreshes_index_and_round_trips_machine_outputs
  tests/test_bead/test_cli_golden.py::test_bead_cli_golden_contract[stats]
  tests/test_config.py::test_legacy_overlay_is_discovered_but_not_a_complete_owner
  tests/test_config.py::test_machine_overlays_require_matching_selector_and_keep_ordinary_overlays
  tests/test_config.py::test_selected_overlay_identity_cannot_be_overridden_by_other_sources
  tests/test_config_cache.py::test_clear_config_cache_forces_reload
  tests/test_config_cache.py::test_clear_config_cache_resets_config_token_time_gate
  tests/test_config_cache.py::test_current_config_token_refresh_is_single_flight
  tests/test_config_cache.py::test_explicit_invalidation_wins_race_with_background_refresh
  tests/test_config_cache.py::test_first_config_token_read_does_not_start_worker
  tests/test_config_cache.py::test_yaml_content_cache_survives_config_cache_clear
  tests/test_query_profile.py::test_provider_query_schema_derives_fields_from_the_notes_fixture
```

A direct walk of the durable store (`~/.sase/test-selection/gh_sase-org__sase/`),
restricted to the same window the gate uses (`recorded_at` after `2026-08-15T17:22:27Z`,
a recorded change set, at most 5 failures per run — 109 eligible records), gives the
last recorded failure per node:

| node                                                                                         | last failure (UTC)   |
| -------------------------------------------------------------------------------------------- | -------------------- |
| `test_config_cache.py::test_clear_config_cache_resets_config_token_time_gate`                | 2026-08-16T19:55:21Z |
| `test_config_cache.py::test_explicit_invalidation_wins_race_with_background_refresh`         | 2026-08-16T18:35:19Z |
| `test_config.py::test_selected_overlay_identity_cannot_be_overridden_by_other_sources`       | 2026-08-16T18:34:51Z |
| `test_config_cache.py::test_first_config_token_read_does_not_start_worker`                   | 2026-08-16T17:22:33Z |
| `test_config.py::test_machine_overlays_require_matching_selector_and_keep_ordinary_overlays` | 2026-08-16T16:34:50Z |
| `test_config_cache.py::test_yaml_content_cache_survives_config_cache_clear`                  | 2026-08-16T16:21:24Z |
| `test_config_cache.py::test_current_config_token_refresh_is_single_flight`                   | 2026-08-16T16:11:42Z |
| `test_config.py::test_legacy_overlay_is_discovered_but_not_a_complete_owner`                 | 2026-08-16T14:33:16Z |
| `test_config_cache.py::test_clear_config_cache_forces_reload`                                | 2026-08-16T02:47:23Z |
| **live** `test_top_bar_order.py::test_override_pills_keep_narrow_top_bar_in_bounds`          | 2026-08-16T04:29:25Z |
| **live** `test_var_integration.py::test_var_cli_end_to_end_...`                              | 2026-08-16T19:55:21Z |
| **live** `test_cli_golden.py::test_bead_cli_golden_contract[stats]`                          | 2026-08-16T19:55:21Z |
| **live** `test_query_profile.py::test_provider_query_schema_...`                             | 2026-08-16T15:12:02Z |

`git log -1 --format=%cI 3a22ff04f` is `2026-08-16T19:02:36-04:00` =
**2026-08-16T23:02:36Z**, which is after every one of the nine config nodes' last
recorded failure.

That table is also what rules out one of the bead's three directions — see below.

## Direction Chosen, And Why

The bead offers three directions. This plan takes **direction 3's mechanism (a per-node
timestamp window) carrying direction 1's framing (a human explicitly declares a node
fixed, and records what fixed it)**.

The new syntax is a comment directive in `tests/reproducible_flake_baseline.txt`:

```
# fixed-at: <UTC timestamp> <node id>
```

It retires that one node's failure evidence recorded at or before that instant — the
moment its fix landed. Nothing else changes: other nodes' evidence is untouched, the
record still counts as an independent pass for every other node, and any failure of that
node recorded **after** the timestamp is ordinary live evidence.

**Why not direction 2 (N consecutive eligible passes retires the evidence).** It is the
only maintenance-free option, and `_flake_evidence_nodeids`' own docstring argues for
maintenance-free rules — but the table above disproves it here. The still-live
`test_override_pills_keep_narrow_top_bar_in_bounds` (bead `sase-mp`) last failed at
2026-08-16T04:29:25Z and has passed in roughly 90 eligible full runs since; it is
dormant, not fixed. Any N that retires the nine config nodes retires `sase-mp` too,
which the bead's exit criteria call a bug in the mechanism, not a win. Direction 2 is
therefore rejected on evidence rather than on taste, and — per the phase's escalation
rule — this plan does **not** need a `TASK NEEDS APPROVAL` note, because the direction
it takes cannot retire a merely-dormant node at all.

**Why not direction 1 literally (`fixed at <commit>`, retiring evidence from runs whose
head does not contain that commit).** It is more precise about which tree was tested,
but it decides retirement from git reachability, and `git_ancestor_oracle` cannot tell
"this commit is not an ancestor" apart from "this workspace never fetched that commit" —
both answer `False`. Records routinely name heads from sibling workspaces. Erring
towards "not an ancestor" silently discards live evidence (weakens the gate); erring the
other way makes retirement depend on each workspace's fetch state, so the gate's verdict
would differ between workspaces on the same tree. A timestamp needs no git resolution,
is already what the gate's existing file-wide `# effective-after:` lever uses, and reads
deterministically from `recorded_at`, which every schema-2 record carries.

**Known limitation, deliberately accepted.** A workspace running a stale tree can write
a record _after_ the declared fix instant. Such a failure is not retired and the gate
goes red. That is the safe direction (the gate errs red, never silently green), it
self-heals as workspaces rebase and as records age out at `RETENTION_DAYS = 30`, and it
is a transient red rather than the permanent one this bead exists to remove. The
implementer records this limitation in the header comment of the baseline file.

## Design

### 1. `tests/_test_selection_health_correlation.py`

Add an oracle type alias next to the existing three:

```python
RetiredEvidenceOracle = Callable[[str, FullRunRecord], bool]
```

Thread `retired_evidence: RetiredEvidenceOracle | None = None` (keyword-only) through
`_flake_evidence_nodeids`, `reproducible_flake_nodeids`, and `stale_flake_nodeids`. In
`_flake_evidence_nodeids`, a retired `(nodeid, run)` failure is skipped exactly where an
attributable dirty-tree failure already is:

```python
for nodeid in full_run.failures:
    if _is_attributable_dirty_failure(nodeid, full_run):
        continue
    if retired_evidence is not None and retired_evidence(nodeid, full_run):
        continue
    failures_by_node.setdefault(nodeid, []).append((index, changed_files))
```

Keep the skip at exactly that site and do **not** also filter inside
`_has_interleaved_independent_pass`. That leaves the retired run still counting as "the
node failed here" when searching for an interleaved pass, which can only ever _withhold_
evidence, never manufacture it — the same conservative asymmetry the dirty-tree rule
already has. Say so in a comment so the next reader does not "fix" it.

Add the matching accounting helper, mirroring `attributable_dirty_failures` so the gate
can name what it discounted instead of quietly reporting a smaller number:

```python
def retired_flake_evidence(
    full_runs: Sequence[FullRunRecord],
    *,
    retired_evidence: RetiredEvidenceOracle,
    max_failures_per_run: int | None = None,
) -> tuple[tuple[str, str], ...]:
    """``(nodeid, record name)`` pairs a declared fix retired."""
```

Update the `_flake_evidence_nodeids` docstring: it currently states the evidence shape
is permanent once recorded, which is precisely what this change ends.

### 2. `tests/_test_selection_health.py`

Import and re-export `RetiredEvidenceOracle` and `retired_flake_evidence`, keeping
`__all__` sorted as it is today. This module is the stable import surface the tool uses.

### 3. `tools/selection_health`

- `BASELINE_FIXED_AT_PREFIX = "fixed-at:"` next to `BASELINE_EFFECTIVE_AFTER_PREFIX`.
- `FlakeBaseline` gains `retirements: Mapping[str, datetime]` (default empty).
- `load_flake_baseline` parses the directive inside the existing comment branch, after
  the `effective-after` check. Split the value with `split(None, 1)` so the node ID —
  which may legitimately contain spaces, see the
  `test_classify_origin_matches_python_golden[...]` entries already in the file —
  survives intact as the remainder. Raise `ValueError` with the
  `f"{path}:{line_number}: ..."` shape the existing timestamp error uses for: a
  malformed or missing timestamp, a missing node ID, and a duplicate `fixed-at` entry
  for the same node ID. `main` already turns those into exit status 2.
- A module-level builder returns the oracle:

  ```python
  def _retired_evidence_oracle(baseline: FlakeBaseline) -> RetiredEvidenceOracle:
      def _is_retired(nodeid: str, full_run: FullRunRecord) -> bool:
          fixed_at = baseline.retirements.get(nodeid)
          if fixed_at is None:
              return False
          recorded_at = _recorded_at_utc(full_run)
          if recorded_at is None:
              return False  # an unreadable timestamp keeps its evidence
          return recorded_at <= fixed_at
      return _is_retired
  ```

  The `recorded_at is None` branch must stay fail-closed and must carry that comment: it
  is the one place this feature could silently erase evidence.

- `_flake_gate_result` builds the oracle, passes it to both `reproducible_flake_nodeids`
  and `stale_flake_nodeids`, and appends new diagnostic lines (alongside the existing
  stale / unresolved / dirty ones, and like them not affecting exit status) via a new
  `_retirement_lines(retired, baseline)`:
  - when anything was retired: a count plus one `  {nodeid} ({record})` line per pair,
    naming `# fixed-at:` and the baseline file so a reader can find the declaration;
  - for every `fixed-at` entry that retired nothing in the current window: a line saying
    those entries can be removed. That file's header calls its contents "debt to remove,
    not suppressions to grow"; without this line, retirements would accumulate forever
    exactly like the suppressions it warns against.

### 4. `tests/reproducible_flake_baseline.txt`

Extend the header comment to document the directive, its per-node scope, the convention
that a preceding comment names the bead and the fixing commit, and the stale-tree
limitation above. Then add, in a labelled block after the file-wide `# effective-after:`
line and before the baseline entries, nine retirements at `2026-08-16T23:02:36Z` (the
committer date of `3a22ff04f`), introduced by a comment naming `sase-nv` and that
commit:

```
# Fixed nodes: `# fixed-at: <UTC timestamp> <node id>` retires only that node's
# failure evidence recorded at or before the instant its fix landed. ...
#
# sase-nv: fixed by 3a22ff04f "fix(config): isolate config cache from test-owned
# CONFIG_DIR" (committed 2026-08-16T23:02:36Z).
# fixed-at: 2026-08-16T23:02:36Z tests/test_config.py::test_legacy_overlay_is_discovered_but_not_a_complete_owner
# fixed-at: 2026-08-16T23:02:36Z tests/test_config.py::test_machine_overlays_require_matching_selector_and_keep_ordinary_overlays
# fixed-at: 2026-08-16T23:02:36Z tests/test_config.py::test_selected_overlay_identity_cannot_be_overridden_by_other_sources
# fixed-at: 2026-08-16T23:02:36Z tests/test_config_cache.py::test_clear_config_cache_forces_reload
# fixed-at: 2026-08-16T23:02:36Z tests/test_config_cache.py::test_clear_config_cache_resets_config_token_time_gate
# fixed-at: 2026-08-16T23:02:36Z tests/test_config_cache.py::test_current_config_token_refresh_is_single_flight
# fixed-at: 2026-08-16T23:02:36Z tests/test_config_cache.py::test_explicit_invalidation_wins_race_with_background_refresh
# fixed-at: 2026-08-16T23:02:36Z tests/test_config_cache.py::test_first_config_token_read_does_not_start_worker
# fixed-at: 2026-08-16T23:02:36Z tests/test_config_cache.py::test_yaml_content_cache_survives_config_cache_clear
```

Do **not** touch the `# effective-after:` line and do **not** add or remove any baseline
node ID entry.

## Tests

Both exit criteria (a)/(b)/(c) are covered twice: once at the evidence-bar level and
once end-to-end through the CLI and the real file syntax.

**New file `tests/test_test_selection_health_flake_retirement.py`** (module docstring in
the style of its siblings, explaining that pre-fix evidence for a declared-fixed node is
retired and why the gate's bar is otherwise unchanged). Build records with
`FullRunRecord` directly, as `test_test_selection_health_flake_gate.py` does, and pass a
plain `retired_evidence` lambda keyed on record name or `recorded_at`:

1. a node meeting the evidence bar is dropped when its evidence is retired
   (`reproducible_flake_nodeids(runs, retired_evidence=...) == frozenset()`);
2. a node whose failures continue past the fix point is **still** flagged — retire only
   the earliest failure, leave two later disjoint failures with an interleaved pass;
3. retiring one node does not retire another — two independently flaky nodes in the same
   record sequence, one retired, the other still reported;
4. `retired_evidence=None` reproduces today's result exactly (no behaviour change for
   callers that do not opt in);
5. `stale_flake_nodeids` honours retirement too;
6. `retired_flake_evidence` returns exactly the `(nodeid, record)` pairs that were
   discounted, and `()` when nothing was.

**Appended to `tests/test_selection_health_tool.py`** (keeping the file under the
700-line `toobig` info threshold — it is 500 lines today). Give `_baseline()` a
`retirements: tuple[tuple[str, str], ...] = ()` parameter that writes real
`# fixed-at: <ts> <nodeid>` lines, then add:

7. a `fixed-at` entry after the node's failures makes the gate exit 0, print "no new
   reproducible flakes", and print the retirement diagnostic naming the node;
8. a node that also fails **after** its `fixed-at` timestamp still exits 1 and is still
   named — the guardrail the bead demands be proven by test, not inspection;
9. two flaky nodes, one retired: exit 1, the retired node absent from the reported list,
   the other present;
10. a `fixed-at` entry that retires nothing prints the removable-debt line and does not
    change the exit status;
11. malformed directives exit 2: bad timestamp, and a timestamp with no node ID;
12. a duplicate `fixed-at` for one node exits 2;
13. a record with an unparseable/absent `recorded_at` is **not** retired (fail-closed).

## Verification

Run from this repo's workspace root, after `just install` (these workspaces are
ephemeral):

1. `just check` — inline is fine; hand it to `/sase_monitor` with a `--next` action if
   it runs long. Must be green.
2. `just selection-health --fail-on-new-flake` — must **exit 0**. Confirm in its output
   that all four still-live node IDs are still reported as current flakes covered by the
   baseline (the `no new reproducible flakes (N current, ...)` line plus the retirement
   diagnostic listing only config nodes), and that none of the nine config nodes appears
   as a flake.

   Verify the four explicitly rather than trusting exit 0: re-run with a temporary copy
   of the baseline file that has the nine `# fixed-at:` lines but an empty node-ID list,
   and confirm the gate then reports exactly those four and no config node.

   If a stale-tree workspace has recorded a _new_ post-fix config failure while this
   work was in flight, do not bump the timestamp to paper over it — that would be the
   all-or-nothing move this bead exists to replace. Record the finding in the bead note
   instead.

3. `just check-full` — **only** through `/sase_monitor`
   (`sase monitor start --command 'just check-full' …` with a `--next` action), never
   inline. The gate this phase changes is that recipe's last step, so this is the
   end-to-end proof.

## Bead Bookkeeping

`sase-nv` is this phase's task bead; the epic plan makes task-bead handling the phase
worker's job.

1. `sase bead update sase-nv --status in_progress` before starting the work.
2. `sase bead note sase-nv "<what changed and why this direction>"` when done — the note
   must name the chosen direction and why directions 1 and 2 were rejected, since the
   epic plan asks for that justification explicitly.
3. `sase bead close sase-nv --note "<what was verified>"` if it is finished; leave it
   open with the step-2 note as handoff if not.
4. No `TASK NEEDS APPROVAL` note is expected: the chosen direction cannot retire a
   merely-dormant node, so it does not reduce the gate's ability to catch new flakes. If
   the implementer's work forces a different direction that _would_ weaken the gate,
   leave that note on `sase-nv` instead of asking the user, and stop that part of the
   work.

Then close **only** the phase bead: `sase bead close sase-ns.6.1 --note "<verified>"`.
Do not close `sase-ns.6` or `sase-ns`. Record any unrelated discovered work as
`sase bead note sase-ns.6.1 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`; do not
create beads.

## Out Of Scope

- Bumping the file-wide `# effective-after:` marker. That is the all-or-nothing lever
  this bead exists to replace.
- The four still-live node IDs and their owning beads (`sase-mp` and the var /
  bead-stats / query-profile nodes). They must keep being reported.
- Removing any existing baseline node-ID entry, including
  `test_config_center_state.py::test_save_atomically_replaces_existing_state` — that
  entry belongs to sibling phase `sase-ns.6.2`, which is the phase that may remove it.
  Both phases touch `tests/reproducible_flake_baseline.txt`; neither blocks the other,
  and whichever lands second rebases. This phase only _adds_ a header block and
  `# fixed-at:` lines, so the collision is textual, not semantic.
- Applying retirement to the `summarize` / health-report path. Retirement, like
  `effective-after`, is a property of the gate's committed baseline file, which the
  report path does not read.
