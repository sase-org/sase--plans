---
tier: epic
title: Repair the red master CI lanes
goal: "Every GitHub Actions job on the sase default branch — lint, test
  (3.12/3.13/3.14), coverage-contexts, and visual-test — passes on a clean master tree,
  with each failure fixed at its own root cause rather than muted.

  "
phases:
  - id: symvision
    title: Delete the dead glossary and memory-web symbols
    depends_on: []
    size: small
    description: "symvision: retire the ten unused public symbols the config-glossary
      retirement stranded, so the lint job's symvision stage passes again.

      "
  - id: bead-notes
    title: Refresh the bead CLI note fixtures and assertions
    depends_on: []
    size: small
    description: "bead-notes: regenerate the six bead CLI goldens for the
      structured-note wire, retarget the compact-search assertion, and replace the
      hardcoded history date literal with a computed one.

      "
  - id: marker-audit
    title: Re-review the split agent-chat marker-path sites
    depends_on: []
    size: small
    description: "marker-audit: add the six agent-chat resolver call sites that the
      module split moved out of the reviewed marker-path allowlist, each with a real
      lifecycle or exemption rationale.

      "
  - id: visual
    title: Rebaseline the stale ACE PNG goldens
    depends_on: []
    size: medium
    description: "visual: attribute every PNG mismatch to the landed UI change that
      caused it, then regenerate the stale goldens and settle the two marginal nodes
      that flicker between runs.

      "
  - id: pool-cursor
    title: Isolate the pooled-alias round-robin cursor from tests
    depends_on: []
    size: medium
    description: "pool-cursor: stop the machine-global llm_lb.json cursor from leaking
      between concurrently running test files, which is why the pooled-alias consumption
      tests fail only in CI's parallel lane.

      "
  - id: ci-races
    title: Fix the two remaining CI-only test races
    depends_on: []
    size: medium
    description: "ci-races: diagnose and fix the provider-drain relaunch e2e and the
      plugins-pane lazy-fetch node, both of which pass serially and fail only under the
      CI lane.

      "
  - id: land
    title: Integrate, verify, and observe a green master run
    depends_on:
      - symvision
      - bead-notes
      - marker-audit
      - visual
      - pool-cursor
      - ci-races
    size: medium
    description:
      "land: combine every phase tree, run the exhaustive verification lane, reconcile
      the task beads this epic resolves, and confirm a green master CI run."
proposed_by: bbugyi200.athena.0d8
bead_id: sase-th
create_time: 2026-09-09 19:51:23
status: wip
---

- **PROMPT:**
  [prompts/202608/repair_red_master_ci.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/repair_red_master_ci.md)
- **BEAD:**
  [sase-th](https://github.com/sase-org/sase--beads/blob/main/pages/sase-th/README.md)

# Plan: Repair the red master CI lanes

## Context

`actstat` reports `sase-org/sase` as the only red repository among the configured set.
`sase-org/sase-core` is green on its last three master commits (`c0958b0`, `151a37d`,
`e02f0cc`), and the failures attributed to it are actually downstream fallout in this
repo from sase-core's released structured-notes wire.

Every CI run on master has been red or cancelled for many consecutive commits. Run
`32826389979` (master `6271aa52d`) and run `32836871279` (master `2d908ca11`) both fail
`lint`, `visual-test`, `coverage-contexts`, and all three `test` legs.

All findings below were reproduced locally on master `770777110`
(`fix(sdd): harden remote sidecar cloning`) after `just install`. Two of the failure
classes visible in the older CI runs are already fixed on that HEAD and are explicitly
**not** in scope:

- the eighteen `src/sase/history/chat_fork/` private-import symvision errors (task
  `sase-tb`), fixed by `7be1396ab`;
- the `_combine_mutation_outcomes` symvision error (task `sase-ta`).

Master moves fast. Every phase must re-measure its own failing set against the current
default branch before editing, and report any node that has since been fixed or newly
broken instead of assuming this plan's inventory is still exact.

## Root causes

Six independent causes produce the red lanes. None of them share a file.

1. **`lint` — symvision, ten unused public symbols.** Retiring the config glossary
   (`cebab38a1`, `b592cfa57`, epic `sase-sq.8.1`) removed the
   `sase glossary read|show|all|list` CLI but left its renderers and audit-log
   summarizers behind with no non-test consumer. This error was masked until `7be1396ab`
   cleared the `chat_fork` private-import errors ahead of it in symvision's output.
   Tracked as task `sase-tg`.

2. **`test` / `coverage-contexts` — bead CLI note fixtures.** Epic `sase-t2`
   ("Timestamped bead notes") changed `notes` from a scalar string to a list of
   structured records plus a flattened `notes_text` projection. Seven test nodes still
   assert the pre-migration shape. The epic's own notes record this drift with six
   corroborations but no phase has refreshed the fixtures.

3. **`test` / `coverage-contexts` — hardcoded date literal.** A bead history test
   asserts a literal `2026-08-24` that was the wall-clock date when the test was
   written. Tracked as task `sase-t9` with three corroborations.

4. **`test` / `coverage-contexts` — marker-path audit allowlist.** Splitting the
   agent-chat CLI into resolver modules (`7318c52b7`, `39bc4bc70`) moved six marker-path
   call sites to new module paths that the reviewed allowlist does not name. Tracked as
   task `sase-tc`.

5. **`visual-test` — stale PNG goldens.** Three landed UI features changed rendering
   without regenerating goldens: the `F fork` footer binding (`69dc50a31`), clan-wide
   minimum remaining runtime on collapsed clan lanes (`86bbbb532`), and the
   glossary-to-memory panel keymap migration (`93d379e0a`). Feeds the standing
   rebaseline backlog task `sase-r5`.

6. **`test` — machine-global pooled-alias cursor.** The `@large` pool's round-robin
   cursor lives at `~/.sase/llm_lb.json`, a real path shared by at least six test files.
   Under CI's parallel lane a concurrent file advances the cursor, so the consumption
   tests observe the wrong pool member. This is an instance of the process-global-state
   flake class owned by epic `sase-j7`.

Two further CI-only nodes have no confirmed cause yet and are handled in their own
phase: a provider-drain relaunch e2e assertion and a plugins-pane lazy-fetch race (task
`sase-td`).

## Delete the dead glossary and memory-web symbols

Reproduce with the lint job's exact invocation:

```bash
SASE_SYMVISION_BEAD_STATUS_ONLY=1 BD_COMMAND=tools/sase_bead .venv/bin/symvision \
  src/sase --exclude-decorator gate_command_entrypoint --exclude-decorator builtin_chop \
  --epic-symbol "sase-n4(get_usage_limit_config)"
```

It reports ten unused public symbols:

| Symbol                              | File                                                 |
| ----------------------------------- | ---------------------------------------------------- |
| `GlossaryReadAgentSummary`          | `src/sase/memory/legacy_glossary_read_log.py`        |
| `GlossaryReadError`                 | `src/sase/memory/legacy_glossary_read_log.py`        |
| `GlossaryReadTermSummary`           | `src/sase/memory/legacy_glossary_read_log.py`        |
| `summarize_glossary_reads_by_agent` | `src/sase/memory/legacy_glossary_read_log.py`        |
| `summarize_glossary_reads_by_term`  | `src/sase/memory/legacy_glossary_read_log.py`        |
| `build_relation_chip_rows`          | `src/sase/ace/tui/modals/glossary_preview_render.py` |
| `filter_glossary_entries`           | `src/sase/memory/web/text_filter.py`                 |
| `fsync_directory`                   | `src/sase/memory/atomic_write.py`                    |
| `render_glossary_catalog`           | `src/sase/memory/web/render.py`                      |
| `render_glossary_closure`           | `src/sase/memory/web/render.py`                      |

Read `sase/memory/symvision.md` with the `/sase_memory_read` skill first and apply its
decision hierarchy per symbol. Do not reach for a pragma or an `--epic-symbol` entry:
none of these has a real consumer, and whitelisting is the last resort. Preliminary
reading of the tree, which the phase agent must confirm independently:

- `fsync_directory` is called at `src/sase/memory/atomic_write.py:97`, inside its own
  file, so it should become private and its in-file caller updated.
- The other nine have no non-test consumer. Tests do not keep a public symbol alive, so
  they are deletion candidates along with their now-dead private helpers and their
  tests.

Take care to remove only what actually died. `src/sase/memory/web/render.py` must keep
`glossary_closure_markdown`, which `legacy_glossary_read_report.py` still imports, and
`glossary_preview_render.py` must keep the chip and property-grid builders that
`snippets_panel_rendering.py`, `memory_panel_rendering.py`,
`memory_panel_web_rendering.py`, `_prompt_glossary.py`, and `glossary_preview_modal.py`
still import. Re-check each surviving sibling independently after each removal, because
deleting one symbol can strand another.

`src/sase/memory/legacy_glossary_read_log.py` stays: `cli_log.py`,
`legacy_glossary_read_report.py`, and `src/sase/ace/tui/glossary_reads.py` still import
`GlossaryReadEvent`, `glossary_read_log_path`, `filter_glossary_read_events`, and
`read_glossary_read_events` from it. Only the error class, the two summary dataclasses,
and the two summarize functions are dead. Prune `__all__` to match whatever survives.

Close task `sase-tg` from this phase.

## Refresh the bead CLI note fixtures and assertions

Two distinct root causes land in the same test package; keep them separate in the commit
message and in the bead records.

**Structured-note fixture drift (epic `sase-t2`).** Six golden contract cases and one
search assertion still encode the scalar `notes` field:

- `tests/test_bead/golden/cli/list_full.stdout`
- `tests/test_bead/golden/cli/list_json.stdout`
- `tests/test_bead/golden/cli/list_json_limit.stdout`
- `tests/test_bead/golden/cli/list_implicit_closed_json.stdout`
- `tests/test_bead/golden/cli/show_json.stdout`
- `tests/test_bead/golden/cli/show_phase_json.stdout`
- `tests/test_bead/test_cli_search.py::test_handle_bead_search_compact_includes_closed_and_match_reason`

The live output is the intended post-migration contract, so the fixtures are what must
move. The observed deltas are exactly:

- JSON: `"notes": ""` becomes `"notes": []` plus a sibling `"notes_text": ""`; a
  populated note becomes a one-element array of `{id, timestamp, author, text}` with the
  flattened string preserved in `notes_text`.
- Full text: the `NOTES` header gains a count suffix (`NOTES (1)`) and each note renders
  as `#<n> · <local timestamp> · <relative age> · <author>` above an indented body,
  replacing the single `[<timestamp> · <author>] <text>` line.
- Compact search: the note preview renders `notes: "private needle note"` instead of
  embedding the `[<timestamp> · unknown] ` attribution prefix in the snippet.

There is no golden-regeneration tool in this repo; the goldens are read from disk by
`_read_expected` in `tests/test_bead/test_cli_golden.py`. Regenerate by capturing the
real CLI output through that module's own `_setup_case` and `_run_cli` helpers so the
`<ROOT>` path substitution and the trailing-newline normalization stay identical, then
diff every regenerated file against its committed version and confirm each hunk is a
note-shape change and nothing else. Do not hand-edit the JSON by eye —
`list_json_limit.stdout` alone carries more than twenty affected records.

Record the fix on epic `sase-t2` rather than filing a new task; that epic owns the
schema change and already carries six corroborations of this drift.

**Hardcoded date literal (task `sase-t9`).** In `tests/test_bead/test_cli_history.py`,
the assertion at line 226 of
`test_history_full_makes_overwritten_note_revisions_readable` reads
`assert "from: [2026-08-24" in output`. `_revision_chain` creates the bead through
`BeadProject.create` with no injected clock, so the first note is stamped with the real
current UTC date and the assertion holds only on the day it was written. The sibling
assertions in that test check the two legacy `2026-07-27` revisions, which are injected
by `_append_legacy_notes_update` and are genuinely fixed — leave them alone. Fix only
the "today" assertion, either by computing the expected date or by asserting on the
stable `unknown] first note` portion of the line. Close `sase-t9`.

## Re-review the split agent-chat marker-path sites

`tests/test_agent_artifact_marker_path_passing_audit.py::test_tracked_marker_path_passing_sites_are_reviewed`
compares discovered marker-path-passing call sites against
`_REVIEWED_PATH_PASSING_CONTEXTS`. Six sites are discovered but unreviewed:

- `src/sase/scripts/_agent_chat_from_name_common.py:resolve_meta_chat_path`
- `src/sase/scripts/_agent_chat_from_name_common.py:completed_response_path`
- `src/sase/scripts/_agent_chat_from_name_common.py:resolve_done_response_path`
- `src/sase/scripts/_agent_chat_from_name_family.py:_resolve_agent_family_member_shell`
- `src/sase/scripts/_agent_chat_from_name_failure.py:_resolve_failed_agent_transcript`
- `src/sase/scripts/_agent_chat_from_name_resume.py:_resolve_family_member_resume_transcript`

These are the same call sites the audit already covered before `7318c52b7` and
`39bc4bc70` split `agent_chat_from_name` into resolver modules; only their module paths
changed. This is an audit, not a rename exercise: read each function and write a
`PathPassingReview` that states what it actually does with the marker path. Give it
`lifecycle_coverage` when it writes a marker and refreshes the Tier 1 artifact index for
the same directory, and `exemption` when it only reads. If any of the six turns out to
mutate a marker without refreshing the index, that is a real defect — fix the code
rather than writing an exemption for it, and say so on the phase bead.

Confirm the reviewed set matches exactly; the assertion is set equality, so a stale
entry left behind by the split fails just as loudly as a missing one. Close task
`sase-tc`.

## Rebaseline the stale ACE PNG goldens

`just test-visual` on master `770777110` fails 15 of 787 nodes. A prior run on
`2d908ca11` failed 16. The stable set, by snapshot name:

```
agents_clan_panel_epic_120x40                    agents_panel_fold_sweep_armed_120x40
agents_clan_panel_epic_hints_120x40              agents_proc_shell_detail_120x40
agents_clan_panel_epic_logical_prompt_hints_120x40  agents_retry_exhausted_120x40
agents_clan_panel_swarm_120x40                   agents_selected_clan_collapse_precedence_120x40
agents_clan_tree_collapsed_120x40                agents_selected_panel_clan_collapse_120x40
agents_clan_unread_collapsed_120x40              agents_selected_row_120x40
agents_group_clan_collapse_precedence_120x40     artifacts_files_idle_placeholder_120x40
help_keymaps_changespecs_120x40
```

Two further nodes flicker between runs and must be settled, not silently accepted:
`artifacts_plans_empty_120x40` (109 changed pixels, 0.007%) failed in CI run
`32826389979` and in a local run at `2d908ca11` but passed at `770777110`;
`config_center_procs_tab_filtered_120x40` failed once locally at `2d908ca11` and passed
at `770777110`. Decide whether each is genuinely stale, genuinely nondeterministic, or
sitting on the tolerance boundary, and fix the cause of any nondeterminism instead of
regenerating around it.

Before regenerating anything, attribute each mismatch to a landed change. Three distinct
UI deltas are already identified from the diff artifacts:

- **Footer binding.** `agents_selected_row_120x40` gains an `F fork` entry that reflows
  the footer hint columns. Introduced by `69dc50a31`
  (`feat(ace): expose shell forks throughout ACE`).
- **Clan runtime.** `agents_clan_panel_swarm_120x40` renders the collapsed clan lane as
  `11m / 14m` instead of `14m`; the longer row widens the tree pane and narrows the
  detail pane, which rewraps the clan member rows. Introduced by `86bbbb532`
  (`feat(ace): show clan-wide minimum remaining runtime on collapsed clan lanes`).
- **Keymap help.** `help_keymaps_changespecs_120x40` follows the glossary panel's
  migration to the memory panel and the extracted keymaps registry in `93d379e0a`.

Every remaining mismatch must be traced to a specific landed commit the same way. A diff
no landed change explains is a regression: stop, do not regenerate it, and report it on
the phase bead.

Use `.pytest_cache/sase-visual/` for the per-node `actual.png`, `expected.png`,
`diff.png`, and `summary.txt` artifacts, and accept intentional changes with
`--sase-update-visual-snapshots`. Goldens live in `tests/ace/tui/visual/snapshots/png/`.
Note that local runs use exact pixel equality while CI allows a small ratio-only
renderer tolerance, so a locally green rebaseline is the stricter check. The suite takes
roughly three minutes, which is inline-able, but hand it to `/sase_monitor` if it starts
stretching the turn.

Report the result on task `sase-r5`, the standing rebaseline backlog bead. That bead's
inventory is five days stale and lists a mostly different set of 36 nodes, so re-measure
rather than trusting it, and note which of its listed nodes are now green.

## Isolate the pooled-alias round-robin cursor from tests

`tests/test_pooled_alias_single_consumption.py::test_two_consecutive_default_launches_alternate_pool_members`
and `::test_explicit_large_directive_and_default_alias_share_pool_cursor` fail in CI
with `('claude', 'opus') == ('codex', 'gpt-5.6-sol')` and pass locally in isolation.

The cause is shared mutable state on a real path, not a product defect. The tests' own
`_pool_cursor` helper reads `Path.home() / ".sase" / "llm_lb.json"`, and
`src/sase/llm_provider/load_balancing.py` writes that same file through
`_STATE_FILENAME`. There is no `HOME` isolation in `tests/conftest.py`, and at least six
test files touch the path:

- `tests/test_pooled_alias_single_consumption.py`
- `tests/test_reasoning_effort_metadata_persistence.py`
- `tests/agent/test_launch_guard.py`
- `tests/llm_provider/test_pool_last_resort_aliases.py`
- `tests/llm_provider/test_load_balanced_aliases.py`
- `tests/llm_provider/test_load_balanced_alias_state.py`

The two failing tests assert absolute cursor positions (`_pool_cursor() == 1`, then
`== 0`) and absolute member identities, so any concurrent advance from another worker
shifts them by one and flips the expected member. Two of the files above additionally
assert the state file does _not_ exist, which a concurrent writer breaks outright.

Fix the leak, not the assertion. Redirect the load-balancer state root to a per-test
temporary directory for every test that touches it — a shared autouse fixture is the
natural shape — so each test starts from a known cursor. Reproduce first: run the six
files together under the parallel lane and confirm the failure, since a serial run will
not show it. Verify the fix by re-running that same combination repeatedly, not by
running the failing file alone.

Record the fix as corroboration on epic `sase-j7`, which owns the process-global-state
flake class, and check whether the same root cause explains any node in the reproducible
flake baseline before closing anything out.

## Fix the two remaining CI-only test races

Both nodes pass serially on master `770777110` and fail in CI, so neither is
reproducible without recreating the lane.

**`tests/fakey/test_provider_drain_e2e.py::test_provider_drain_e2e_flag_on_relaunches_stranded_agent`**
fails with `assert None is not None` in CI runs `32816209010` and `32826389979`, and
passes locally in 3.3 seconds. The assertion is looking for a relaunched agent that has
not appeared yet, which points at a wait that is too short or is watching the wrong
marker under load rather than at broken drain behavior. Note that `provider_drain` is a
live feature flag with an open retirement task (`sase-sx`); confirm the flag's CI
resolution matches the local one before assuming a race.

**`tests/ace/tui/test_plugins_browser_pane_detail.py::test_plugins_pane_lazy_fetches_highlighted_latest`**
fails with `assert None == '2.0.0'` on the 3.12 coverage leg. Task `sase-td` already
diagnoses it: the test waits on the enrich call marker instead of on the applied entry,
so under the parallel lane it observes the pane before the fetched version is applied.
Fix it to await the applied state and close `sase-td`.

For each node, fix the synchronization in the test rather than adding a retry or
extending a timeout. If either turns out to be a product defect, fix the product and say
so on the phase bead. If a node cannot be reproduced even under a recreated lane, do not
guess a fix: record what was tried and what was ruled out so the next agent does not
repeat it.

## Integrate, verify, and observe a green master run

Combine every phase tree and verify it exhaustively. The phases touch disjoint files by
construction, so conflicts should be rare; resolve any that appear in favor of the phase
that owns the file.

Run the full verification lane through a monitor, never inline:

```bash
sase monitor start --command 'just check-full' \
  --start-status TESTING --stop-status TESTED --next '<follow-up action>'
```

Then run `just test-visual` and confirm it is green, including the two marginal nodes
the visual phase settled.

Expect the combined tree to surface failures this epic does not own — master has been
red long enough that other drift is likely hiding behind these six causes, and the
flake-baseline gate is independently blocked (task `sase-j0`). Triage anything new
through `/sase_new_task`; do not absorb unrelated work into this epic and do not let it
block the epic's own acceptance.

Reconcile the beads this epic resolves — `sase-tg`, `sase-t9`, `sase-tc`, `sase-td`, and
corroborations on `sase-t2`, `sase-j7`, and `sase-r5` — and check whether `sase-tb` and
`sase-ta` can now be closed as already fixed by `7be1396ab` and its predecessor.
Consider whether the in-progress epic `sase-m4` ("Stabilize GitHub Actions") should be
closed or explicitly superseded by this one, since it was reopened for exactly this
outcome and its remaining open items (`sase-m4.3` item 1, `sase-m4.5`, `sase-m4.6`)
overlap this epic's acceptance.

Acceptance is a green master CI run observed after the epic lands — not a green local
tree. Confirm it with `actstat --repo sase-org/sase` and record the run URL.
