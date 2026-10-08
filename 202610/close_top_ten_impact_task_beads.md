---
tier: epic
title: Implement and close the ten highest-impact recent task beads
goal: "All ten beads ranked in the 48-hour task-bead impact report are fixed on master
  and closed with resolution done and recorded evidence: sase-1h6, sase-1h2, sase-10d,
  sase-13p, sase-1gx, sase-14o, sase-1h1, sase-1f0, sase-1br and sase-18v. The epic does
  not land while any of them is open or unverified.

  "
decisions:
  release_red_master:
    ask:
      If master CI is still red for unrelated reasons at release time, how should the
      release phase proceed?
    default: fix_master
    choices:
      fix_master:
        The release phase makes Master Gate and Full CI green itself, then ci_watch
        merges.
      ask_bypass:
        After release-specific checks pass, a gate asks you to approve hand-merging the
        release PR.
    why:
      ci_watch's green-master release gate is a deliberate safety check; keep it unless
      you waive it.
    answer: fix_master
  plugin_floors:
    ask:
      Also raise and republish the sase floors of sase-telegram and sase-github if they
      don't resolve?
    default: true
    why: Today their published versions silently fall back to sase 0.1.0.
    answer: true
phases:
  - id: runs-public
    title: Make the instructions run-index module public (sase-1h6)
    depends_on: []
    size: small
    description:
      "runs-public: rename src/sase/instructions/_runs.py to a public module, update
      every importer, verify symvision has no _runs private-import findings, and close
      sase-1h6."
  - id: prompt-store-cycle
    title: Break the prompt_store_mutations import cycle (sase-1h2)
    depends_on: []
    size: small
    description:
      "prompt-store-cycle: make importing sase.history.prompt_store_mutations
      order-independent, add fresh-interpreter regression tests for it and its lazy
      launch callers, and close sase-1h2."
  - id: readonly-bead-store
    title: Read-only bead resolution never initializes or commits (sase-1gx)
    depends_on: []
    size: medium
    description:
      "readonly-bead-store: make get_read_view open an existing store or raise a typed
      unavailable error, move genuine writers to get_project, add no-write regression
      tests, and close sase-1gx."
  - id: test-host-leaks
    title:
      Isolate tests from the live bead store and long basetemps (sase-14o, sase-18v)
    depends_on: []
    size: small
    description:
      "test-host-leaks: isolate the bead and plan resolvers in the absent-store and
      plan-candidate tests, make the three Rich path assertions basetemp-independent,
      and close sase-14o and sase-18v."
  - id: macro-arg-spans
    title: TUI macro-arg detection uses sase-core structural spans (sase-1h1)
    depends_on: []
    size: medium
    description:
      "macro-arg-spans: replace raw comma and paren splitting in the TUI macro-arg
      detector with sase-core argument spans, add quoted-comma and quoted-paren golden
      fixtures in both repos, and close sase-1h1."
  - id: demand-rss-flake
    title: Guarantee a nonzero peak RSS for every recorded run (sase-1f0)
    depends_on: []
    size: medium
    description:
      "demand-rss-flake: make the tree-RSS sampler record a real peak even for children
      that exit before the first sample, prove the node is stable under repetition and
      load, and close sase-1f0."
  - id: deck-scroll-settle
    title: Deterministic deck anchor-scroll settling (sase-1br)
    depends_on: []
    size: medium
    description:
      "deck-scroll-settle: root-cause and fix the deck anchor-scroll settle race behind
      the block-spread pilot timeout, share one settle wait across deck pilots, prove
      stability by repetition, and close sase-1br."
  - id: import-budget
    title: Get under the TUI import budget and make it a ratchet (sase-13p)
    depends_on:
      - macro-arg-spans
      - deck-scroll-settle
    size: medium
    description:
      "import-budget: defer eager TUI startup imports to at least 30 modules under the
      cap, lower the cap to measured plus 20, add an attribution tool, document the
      ratchet policy, and close sase-13p."
  - id: release
    title: Release sase-core and sase, then move plugin floors (sase-10d)
    depends_on:
      - runs-public
      - prompt-store-cycle
      - readonly-bead-store
      - test-host-leaks
      - macro-arg-spans
      - demand-rss-flake
      - deck-scroll-settle
      - import-budget
    size: large
    description:
      "release: publish a sase-core release containing sase's pin, ratchet the release
      branch, get the sase release gates green and publish sase, raise plugin floors,
      verify fresh installs, and close sase-10d."
proposed_by: bbugyi200.athena.0y8
decided_by: auto
create_time: 2026-10-08 09:47:14
status: wip
---

- **PROMPT:**
  [prompts/202610/close_top_ten_impact_task_beads.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/close_top_ten_impact_task_beads.md)

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                          | Why                                                                   |
| ------------ | --------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| derives-from | [research:202610/task_bead_48h_impact_ranking/task_bead_48h_impact_ranking.md][1] | Ranks the ten beads this epic closes and supplies each fix direction. |
| implements   | [bead:sase-10d][2]                                                                | Rank 3. Cut a sase-core release, ratchet the floor, publish sase.     |
| implements   | [bead:sase-13p][3]                                                                | Rank 4. Get under the TUI import budget and adopt a ratchet policy.   |
| implements   | [bead:sase-14o][4]                                                                | Rank 6. Isolate absent-store tests from the host's live bead store.   |
| implements   | [bead:sase-18v][5]                                                                | Rank 10. Make snippet and restart CLI tests basetemp-independent.     |
| implements   | [bead:sase-1br][6]                                                                | Rank 9. Fix the deck scroll-settle flake.                             |
| implements   | [bead:sase-1f0][7]                                                                | Rank 8. Fix the zero peak-RSS flake in the demand-run test.           |
| implements   | [bead:sase-1gx][8]                                                                | Rank 5. Read-only bead resolution never initializes or commits.       |
| implements   | [bead:sase-1h1][9]                                                                | Rank 7. TUI macro-arg detection uses sase-core structural spans.      |
| implements   | [bead:sase-1h2][10]                                                               | Rank 2. Break the prompt_store_mutations import cycle.                |
| implements   | [bead:sase-1h6][11]                                                               | Rank 1. Rename the private sase.instructions._runs module.           |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202610/task_bead_48h_impact_ranking/task_bead_48h_impact_ranking.md
[2]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-10d/README.md
[3]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-13p/README.md
[4]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-14o/README.md
[5]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-18v/README.md
[6]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-1br/README.md
[7]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-1f0/README.md
[8]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-1gx/README.md
[9]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-1h1/README.md
[10]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-1h2/README.md
[11]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-1h6/README.md

<!-- sase:links:end -->

# Plan: Implement and close the ten highest-impact recent task beads

Source research. Read it with `sase artifact read` before starting any phase:
`research:202610/task_bead_48h_impact_ranking/task_bead_48h_impact_ranking.md`.

The report ranks the ten task beads created or +1ed between 2026-10-05 and 2026-10-07
whose remaining work matters most. This epic implements every one of them and closes
each with resolution `done`. The four already-delivered beads the report lists
(`sase-1h5`, `sase-17r`, `sase-1fy`, `sase-1gs`) and the triage-owner bug it filed
(`sase-1ha`) are **out of scope**.

## State re-verified while planning (2026-10-08, sase master `56fcf92447`)

The report was written a day earlier. Every phase must re-verify against its own base
tree, but these are the facts at planning time:

- **sase-1h6.** `src/sase/instructions/_runs.py` is still a private module, imported
  across files by `verify.py`, `coverage_diff.py`, `coverage_sessions.py`,
  `coverage_summary.py`, `doctor/checks_instructions.py` (3 sites) and tests. A
  2026-10-08 note on the bead says symvision's private rule no longer reports `_runs`.
  The module's root helpers now show as unused public symbols, which `sase-1hp` tracks.
  The rename is still owed: a private module must not be imported across files.
- **sase-1h2.** `.venv/bin/python -c "import sase.history.prompt_store_mutations"` still
  raises the circular `ImportError`.
- **sase-13p.** A fresh-interpreter `import sase.ace.tui.app` loads **3582** modules
  against `_MAX_MODULE_COUNT = 3570` (strict `<`).
- **sase-1h1.** `sase_core_rs.macro_argument_spans` already exists and parses both repro
  inputs correctly. `#m:"a,b",` yields one `arg_value_string` span plus delimiters, and
  `#m("a)b",s` yields the quoted value, the `,` and an `arg_value` for `s`.
  `src/sase/macro/highlight.py` already loads that binding lazily.
- **sase-10d.**
  - sase declares `sase-core-rs>=0.35.0,<0.36.0`; its pin `sase-core-revision.txt` is
    `ec92ecce`, which no release tag contains. The latest core tag is `v0.37.0`.
  - The release-plz PR `sase-org/sase-core#323` (v0.37.1) is open but red, because
    sase-core master CI is red. The failing test is
    `crates/sase_core/tests/bead_read_parity.rs::event_store_supports_read_queries_without_legacy_projection`,
    a stale expectation recorded as a DISCOVERED ISSUE on `sase-1h8` by `sase-1h7.land`.
  - The sase release-please PR `sase-org/sase#299` (0.18.0) is open, and its
    `release-core-floor-smoke` check fails.
  - On athena, ci_watch auto-merges release PRs for sase, sase-github and sase-telegram
    (merge order: sase-github, sase-telegram, sase). It merges only when the commit's
    `Master Gate` is green and a `Full CI` run is green within 6 h. Both are red today.
    The red set includes symvision, the import budget and several unrelated test shards.

## Global constraints for every phase

- **Own your beads.** Each phase names the task bead(s) it owns. Read them first with
  `sase bead read <id> -r "<why>"`; their descriptions, `task_type_fields` and +1
  evidence hold the repro commands. Re-verify that the defect still reproduces at your
  base before changing anything.
- **Close on evidence.** When your fix is complete and verified, close each owned bead
  with `sase bead close <id> --note "<evidence>"` and the default `done` resolution. The
  note names this phase bead, the verification commands, and their results. Never close
  one of the ten as `canceled` or `superseded`, and never close it before the fix is
  verified. If a bead turns out to be already fixed by other work, close it `done` with
  the fixing commit and your reproduction as evidence.
- **No shortcuts.** Never weaken an assertion, raise a timeout or cap just to get green,
  add a symvision pragma or suppression, or `xfail`/skip a node.
- **Rust-core boundary.** Shared backend behavior belongs in sase-core's `sase_core`
  crate. Open it with `sase repo open sase-core` and read its `AGENTS.md`. Any sase code
  that needs a core change also needs `sase-core-revision.txt` moved past that commit
  (`just ratchet-core-revision`; see `docs/rust_backend.md`). Never edit sase-core
  versions or changelogs.
- **Verification.** Run `sase tool run check` in every repo you change. Run targeted or
  repeated pytest nodes directly with `.venv/bin/python -m pytest`. Use `/sase_monitor`
  for anything long-running; never end a turn to wait.
- **Memory.** Read the sase memory notes your work touches: `tui` before TUI changes,
  `symvision` before symvision work, `lint_and_test` before finishing, and `sase_beads`
  before closing beads. Do not edit `sase/memory/**`. If a note becomes inaccurate,
  append a `PROPOSED FOLLOW-UP:` note to your phase bead.
- **Beads.** Phase workers never create beads. Record discovered work as
  `PROPOSED FOLLOW-UP: <summary — detail>` notes on your own phase bead. Notes on other
  existing beads (for example `sase-1hp`, `sase-1h8` or `sase-1a7`) are fine where this
  plan asks for them.
- **TUI import budget.** Phases that touch TUI code must not add eager imports to the
  `sase.ace.tui.app` startup closure. Keep new imports at use sites.

---

## Phase `runs-public`: make the instructions run-index module public (sase-1h6)

1. Rename `src/sase/instructions/_runs.py` with `git mv` to a public module name that
   collides with no existing module or common local name. Good candidates are
   `run_index.py` or `run_records.py`; avoid bare `runs.py`, because importers alias the
   module `as runs`.
2. Update every importer. Use
   `grep -rn "instructions import _runs\|instructions\._runs" src tests tools` to find
   them, not the list above, because it may have grown. Keep the importers' local
   aliases (`runs`, `run_mod`) or rename them consistently. Check
   `src/sase/_symvision_static_refs.py` and any
   `mock.patch("sase.instructions._runs...")` strings in tests.
3. Do not rename the unrelated file-local `_runs()` helpers in
   `src/sase/agents_sync/v2_snapshot_io.py` and
   `src/sase/ace/tui/widgets/decks/final/overview_card.py`.
4. The module's unused public root helpers (`claude_projects_root`,
   `codex_sessions_root`, `grok_cwd_dir`, `grok_sessions_root`) belong to `sase-1hp`. Do
   not fix them here. After the rename, append a note on `sase-1hp` giving their new
   module path.
5. **Verify.**
   - The grep above returns nothing.
   - `sase tool run check` shows no symvision private-import finding mentioning `_runs`
     or the new module.
   - `tests/instructions/` and `tests/doctor/` pass.
6. Close `sase-1h6`.

## Phase `prompt-store-cycle`: break the prompt_store_mutations import cycle (sase-1h2)

**The cycle.** `src/sase/history/prompt_store_mutations.py:9` runs
`from sase.history import prompt_store as store`. The bottom of
`src/sase/history/prompt_store.py` (about line 477) re-imports `add_or_update_prompt`,
`record_failed_launch_prompt` and `rewrite_prompt_text_exact` from the mutations module.
The lazy callers in `src/sase/agent/launch_cwd_agents.py` and
`src/sase/agent/launch_cwd_bead_work.py` import `effective_prompt_origin` from the
mutations module first, so a cold process fails and approved LaunchApprovals launch
nothing.

1. Make the import order-independent at the module level. For example, move the helpers
   the mutations module needs from the facade into a lower-level module that both
   import. Another option is to make the mutations module resolve the facade at call
   time.
   - Keep every public name `prompt_store` re-exports working.
   - Do not "fix" it by only reordering imports in the lazy callers; that hides the
     cycle.
2. **Regression tests.** Each runs a **fresh interpreter** (`subprocess`), because warm
   test processes hide this bug:
   - `import sase.history.prompt_store_mutations` as the first import succeeds and
     exposes `effective_prompt_origin`;
   - importing `sase.agent.launch_cwd_agents` and `sase.agent.launch_cwd_bead_work`
     first, then importing their lazy dependency, succeeds.
3. **Verify.**
   - The bead's original repro command succeeds.
   - Grep for other lazy `prompt_store_mutations` importers and confirm they work cold.
   - `sase tool run check` passes.
4. Close `sase-1h2`.

## Phase `readonly-bead-store`: read-only bead resolution never initializes or commits (sase-1gx)

**The chain.** `get_read_view()` in `src/sase/bead/cli_common.py` falls through to
`get_project()`, which calls `init_beads()`, which calls
`ensure_bare_git_sdd_initialized(root, commit=True)`. A read in a checkout with no store
can therefore create and commit SDD scaffolding.

1. Make `get_read_view()` open an existing usable store, or raise a typed "bead store
   unavailable" error.
   - Reuse an existing error type if one fits.
   - The message must name the cwd and the locations it searched.
   - It must never call `init_beads()`, `ensure_bare_git_sdd_initialized()`,
     `commit_sdd_files()`, or create a new store. It may still use whatever read-safe
     refresh of an **existing** configured store reads rely on today, as long as nothing
     is written into the cwd's checkout.
2. Audit every `get_read_view` caller. Today they are:
   - `bead/cli.py`, `cli_query.py`, `cli_dep.py`, `cli_refs.py`, `cli_history.py`;
   - the attachment commands and `attachment_doctor.py` / `attachment_resolve.py`;
   - `bead/attachments/upload/*`, `cli_attachment_publish.py`;
   - `cli_work_entry.py`, `artifact_cli/create.py`, `artifact_cli/link_migrate.py`;
   - `agent/bead_display.py`, `plan_show/resolve.py`,
     `scripts/sase_clan_summary_epic.py`.

   Any caller that actually mutates must call `get_project()` explicitly. Read-only
   display callers must handle the unavailable error gracefully (a clear message or an
   empty result, as fits each surface) instead of a traceback.

3. **Coordinate with the active epic `sase-1h8`**, which is moving bead reads onto a
   Rust read model.
   - Read its plan (`plan:202610/bead_store_history_independent_performance.md`) and
     rebase onto its latest landed work.
   - Keep this change in the Python resolution layer.
   - If an open `sase-1h8` phase is editing the same functions, note the overlap on your
     phase bead.
4. **Regression tests.**
   - Set up a temporary git checkout with no store and with `HOME`, `SASE_*` and store
     resolution isolated, so the host store cannot be found. Reuse the isolation pattern
     from `test-host-leaks` if it lands first.
   - Exercise `get_read_view()` and at least one CLI read path from each of these
     groups: `bead history|refs|dep list`, the attachment doctor, the `artifact create`
     bead lookup, and agent bead display.
   - Assert the typed unavailable outcome, no `sdd/` directory or other new files, no
     new commits, and a clean `git status`.
5. **Verify.** Run `sase tool run check`, and `tests/test_bead/` plus the touched
   callers' tests.
6. Close `sase-1gx`.

## Phase `test-host-leaks`: isolate tests from the live bead store and long basetemps (sase-14o, sase-18v)

**sase-14o.** These tests neutralize one input to store resolution and let the resolver
find the host's live store:

- `tests/doctor/test_checks_beads.py::test_project_beads_skips_when_store_is_absent`:
  `_find_existing_beads_dir` in `src/sase/doctor/checks_beads.py` still walks to the
  host store.
- `tests/completion/test_candidates_project_providers.py::test_bead_candidates_without_a_store_returns_empty_list`:
  `_resolve_beads_dir` in `src/sase/completion/candidates/catalog_sdd.py` is not
  cwd-bound.
- Sibling, same change:
  `tests/completion/test_candidates_resource_providers.py::test_plan_candidates_emit_canonical_references`,
  where the live plan archive leaks extra plan refs.

Isolate the resolver itself, not one of its inputs. Patch the resolver, or point every
candidate root at an empty `tmp_path`. If a resolver-isolation fixture already exists
from the `sase-ql` and `sase-ml` precedents, extend it rather than adding a new pattern.

**sase-18v.** These assertions compare a full `tmp_path`-derived path against Rich
output that truncates or wraps it under long agent basetemps:

- `tests/main/test_snippet_cli_add.py::test_add_rich_format_states_created_action`
- `tests/main/test_snippet_cli_delete.py::test_delete_rich_format_prints_restore_and_removed_path`
- `tests/test_agent_restart_cli.py::test_wipe_failed_exits_1_and_prints_recovery_dir`

Make each assertion independent of basetemp length. For example, render with a width
that cannot truncate and normalize wrapping before asserting, or assert on a short
stable suffix. Follow the `sase-13u` precedent, which fixed 11 nodes of this class. Do
not change what real users see.

**Verify.**

- Run all six nodes with raw `.venv/bin/python -m pytest` on this host, which has a live
  store.
- Run them again with a deliberately long `--basetemp`, a path of at least 150
  characters.
- Run the containing test files in full.
- Run `sase tool run check`.

Then close `sase-14o` and `sase-18v`.

## Phase `macro-arg-spans`: TUI macro-arg detection uses sase-core structural spans (sase-1h1)

`src/sase/ace/tui/widgets/_macro_arg_assist_detection.py` uses raw string scanning:

- `_colon_active_input_index` uses `value.count(",")`;
- `_paren_active_input_index`, `_selected_positional_values` and `_used_named_arg_names`
  use `body.split(",")`;
- detection stops at the first `")" in body`.

So `#m:"a,b",` offers `flag` instead of `env`, and `#m("a)b",s` opens no menu. The TUI
disagrees with the binder and the LSP.

1. Read the `tui` memory and core memory `rust_core_backend_boundary`.
2. Derive the active input index, the selected positional values and the used named
   arguments from `sase_core_rs.macro_argument_spans`.
   - Reuse the lazy binding loader in `src/sase/macro/highlight.py`, or factor it into a
     shared helper; do not add an eager import.
   - Alternatively, if the core completion-context builder the LSP uses already answers
     this question, call that instead. Pick whichever keeps the TUI, LSP and binder on
     one parser.
   - Delete the Python splitting logic it replaces.
3. If the spans cannot express a needed case, extend sase-core with tests and bump
   sase's pin. Cases to check include a cursor inside an unterminated quote and nested
   `(`. Do not re-grow a Python parser.
4. **Golden fixtures.** Add quoted-comma and quoted-paren cases, plus a named-arg case
   after a quoted value, to `macro_arg_choice_completion.json` in **both**
   `crates/sase_core/tests/fixtures/` (sase-core) and `tests/fixtures/` (sase). They are
   mirrored, so keep them byte-identical.
5. **Verify.**
   - Run `tests/macro/test_macro_choice_projection_parity.py`,
     `tests/ace/tui/widgets/test_macro_arg_choice_tui_parity.py` and the detector's unit
     tests.
   - Run the bead's three repro inputs.
   - Run `sase tool run check` in sase, and in sase-core if you changed it.
   - The `sase.ace.tui.app` import count does not grow.
6. Close `sase-1h1`.

## Phase `demand-rss-flake`: guarantee a nonzero peak RSS for every recorded run (sase-1f0)

`tests/tool/test_demand_runs.py::test_foreground_run_records_context_usage_and_grant`
intermittently records `peak_tree_rss_kib == 0`. That is about 1 in 10 serial runs, and
more under load. The likely cause is that the foreground child exits before the tree-RSS
sampler takes a nonzero sample. The sampler lives in `src/sase/tool/sample.py`,
`executor_run.py` and `demand.py`. Production data is clean: one zero-peak record in 632
runs, and it was a `true` no-op. So the fix is about contract clarity and lane noise,
not data repair.

1. Confirm the mechanism. Instrument the sampler or reason from the code; record the
   finding on your phase bead.
2. **Preferred contract:** every recorded run has a real, nonzero peak. Make the
   executor take its first tree sample immediately after spawn. When the process exits
   before any nonzero sample, use the child's own `ru_maxrss` as the peak floor; it is
   available from `os.wait4` or the reaping path's rusage. If the executor's reaping
   makes that infeasible, use the closest equivalent that yields a measured value, not a
   guess.
3. Only if a measured floor is truly unavailable may the contract instead say that `0`
   means "not sampled". In that case, document it where the field is defined and make
   the test assert that contract deterministically. A sleep-based race is not
   deterministic.
4. **Verify.**
   - The node passes at least 50 serial repetitions.
   - It passes at least 20 repetitions under parallel load, for example run alongside
     `pytest -n 8` of `tests/tool/`.
   - The rest of `tests/tool/` passes.
   - `sase tool run check` passes.
5. Close `sase-1f0`.

## Phase `deck-scroll-settle`: deterministic deck anchor-scroll settling (sase-1br)

`tests/ace/tui/widgets/decks/test_deck_block_spread_pilot.py::test_block_spread_bracket_top_aligns`
times out in its 5 s `wait_for(pilot, lambda: int(_scroll(panel).scroll_y) == target)`.
It fails about 1 in 6 isolated serial runs, and more under load. Its sibling node
`test_scroll_derived_cursor_and_streaming_stays…` and `sase-1a7`
(`tests/ace/tui/widgets/decks/test_deck_spread_pilot.py::test_files_ctrl_j_scrolls_page_anchor_to_top`)
time out on the same kind of wait.

1. Read the `tui` memory.
2. Root-cause the race. Candidates:
   - a smooth or animated scroll that is retargeted when layout changes;
   - a landing target computed before layout settles and then stale;
   - a deferred `scroll_to` that is dropped when another refresh lands.

   Record the mechanism on your phase bead.

3. Fix it at the root.
   - If the product scroll is nondeterministic, fix the product: settle the anchor
     scroll deterministically after layout.
   - If only the test waits on the wrong thing, give deck pilots one shared settle
     helper (for example, wait for the view's landing target to be applied and layout to
     be idle) and use it in all affected nodes.
   - Do not just raise the timeout.
4. **Verify.**
   - The node passes at least 30 isolated serial runs (`-p no:randomly`) and a run under
     parallel load.
   - The sibling node and the `sase-1a7` node pass the same loop.
   - `sase tool run check` passes.
   - If the fix also clears `sase-1a7`, you may close it with the same evidence;
     otherwise add a note to it.
5. Close `sase-1br`.

## Phase `import-budget`: get under the TUI import budget and make it a ratchet (sase-13p)

The guard is
`tests/ace/tui/test_app_import_budget.py::test_tui_app_import_stays_under_startup_budget`,
with `_MAX_MODULE_COUNT = 3570` and a strict `<`. The closure measured 3582 at planning
time. The cap has been raised 3290 → 3400 → 3485 → 3530 → 3570, and the bead has been
closed and reopened twice. Per the report: adopt an explicit ratchet **and** actually
defer imports. Do not redefine success as `<=`, and do not raise the cap.

This phase depends on `macro-arg-spans` and `deck-scroll-settle`, so it measures the
tree after this epic's TUI changes.

1. Read the `tui` memory.
2. **Attribution tool.** Add `tools/tui_import_closure`, an extensionless Python tool
   that passes `tools/typecheck_extensionless_tools`.
   - It prints the fresh-interpreter closure count for `import sase.ace.tui.app`.
   - With `-d/--diff <rev>`, it measures the same import at `<rev>` from a temporary
     detached worktree, using the current venv with `PYTHONPATH` pointed at the
     worktree's `src`. It then prints the `sase.*` modules added and removed.
   - It cleans the worktree up.
   - Follow the CLI rules (sase memory `cli_rules.md`) for its options and `-h`.
3. **Defer.** Cut eager startup edges until the measured closure is **at least 30 below
   3570** (≤ 3540).
   - Use use-site and `TYPE_CHECKING` imports. Precedents: `sase-14n.15.1` and
     `5963c52e8`.
   - Start with the growth the bead's latest +1s attribute by commit:
     `core.wait_epic_follow_view`, `artifact_links.projection._agent_created_epic`,
     `69a4431521` (+4), `f10bf6cd24` (+2) and `62604c10b7`. Then go after whole subtrees
     that are not needed for first paint.
   - Add the heavy modules you defer to the test's `deferred_modules` probe.
4. **Ratchet policy.**
   - Lower `_MAX_MODULE_COUNT` to the new measured closure plus 20, and keep the strict
     `<`.
   - Replace the narrative cap-history comment with a short policy, the measured count
     and the date. The policy: the cap only moves down. Raising it requires that a named
     module be genuinely needed on the first-paint path, with deferral impossible. The
     comment must name that module and its commit, and the raise covers only the
     attributed amount.
   - On failure, the assertion message prints the count, the cap, and the exact
     `tools/tui_import_closure --diff <merge-base>` command to run.
   - Document the policy and the tool in `docs/perf_runbook.md`.
5. **Verify.**
   - The test passes in isolation and in `sase tool run check`.
   - Record before and after counts and CPU seconds on the phase bead.
   - Check that the count is not plugin-dependent (the bead's original caveat), so CI's
     Master Gate passes too.
6. Close `sase-13p`.

## Phase `release`: release sase-core and sase, then move plugin floors (sase-10d)

This phase is `large`. Its worker plans first, against the release state at that time.
It runs after every other phase, so the published sase contains this epic's fixes and
the two standing master-red causes (`sase-1h6`, `sase-13p`) are gone.

**Approving this epic authorizes these outward actions:**

- merging sase-core's release-plz PR;
- dispatching `publish.yml`;
- letting ci_watch merge and publish the sase release;
- committing plugin floor changes through the normal host-owned completion path so their
  release processes publish them.

**It does not authorize:**

- deleting or yanking any PyPI release;
- release-plz `manual-version` recovery;
- hand-editing versions or changelogs;
- force pushes;
- bypassing a release gate, except as decision `release_red_master` allows.

### Acceptance

1. **Core release.** A `sase-core-rs` release whose tag contains sase's current
   `sase-core-revision.txt` pin is published **complete** on PyPI: every release-matrix
   wheel plus the sdist, none yanked.
   - First make sase-core master CI green. If
     `bead_read_parity.rs::event_store_supports_read_queries_without_legacy_projection`
     is still red, apply the expectation fix recorded on `sase-1h8` (assert the warning
     is **absent** for event stores) and note it on `sase-1h8`. Fix any other red there
     per sase-core's `AGENTS.md`.
   - Then merge the current release-plz PR once its checks are green. At planning time
     that is `#323`.
2. **Ratchet.** Dispatch `publish.yml` with `publish_existing=false` so
   `sync-release-metadata` ratchets the release-please branch to the new core.
   - The release PR's `release-core-floor-smoke` must pass (at planning time, that PR is
     `#299`).
   - `tools/probe_core_floor` must report no missing capabilities against the new floor.
3. **sase published.** ci_watch merges the release PR, and `publish.yml` publishes sase,
   once Master Gate is green on master's tip and Full CI is fresh-green.

   > [!decision] release_red_master = fix_master

   Inventory every red Master Gate and Full CI failure at that time.
   - Fix deterministic failures that no active epic owns.
   - For failures an active epic owns, check whether that epic has already fixed them.
     If not, apply the minimal fix and note it on the owner.
   - Do not hand-merge the release PR past a red gate.

   > [!decision] release_red_master = ask_bypass

   Once the core release, the ratchet and every release-PR check are green, and the
   remaining red is outside the release path, raise a SASE gate (`/sase_gate`). It lists
   the red failures and asks the user to approve hand-merging the release PR with the
   ci_watch configured merge method. Merge only on approval.

4. **Install proof.**
   - `uv pip compile` of a requirements file containing only `sase` resolves to the new
     version, not `0.1.0`.
   - A fresh `uv tool install sase==<new>` (or a clean venv) runs `sase core health`
     successfully.
5. **Plugin floors.** Open each plugin with `sase repo open` and follow its own release
   process from its `AGENTS.md`.
   - Raise `sase-research-artifacts`' `sase-core-rs` floor and its published smoke pin.
     The minimum is the first release containing `f55c63b` (≥ 0.35.0); move them to the
     new floor where they need it. Publish it.

   > [!decision] plugin_floors
   - Check whether the latest published `sase-telegram` and `sase-github` resolve to the
     new sase with `uv pip compile`. Where they do not, raise their `sase` floor and let
     their release PRs publish; ci_watch merges them.
   - Verify each plugin resolves to the new sase.

6. **Close.** Close `sase-10d` with evidence: versions, the PyPI file counts and the
   `uv pip compile` outputs.

Use `/sase_monitor` for every wait: CI runs, release-plz and publish workflows, and PyPI
propagation.

---

## Landing (epic land agent): all ten beads closed before the epic lands

The land agent must not close the epic bead until every check below passes on the landed
master tip.

1. **Status audit.** Run
   `sase bead read sase-1h6 sase-1h2 sase-10d sase-13p sase-1gx sase-14o sase-1h1 sase-1f0 sase-1br sase-18v -r "<why>"`.
   Every one must be `closed` with resolution `done` and a close note that cites
   verification evidence.
2. **Re-verify each fix on master**, using quick checks rather than the full lanes:
   - `sase-1h6`: no `instructions import _runs` or `instructions._runs` anywhere in
     `src`, `tests` or `tools`, and no symvision private-import finding for that module.
   - `sase-1h2`: a fresh-interpreter
     `.venv/bin/python -c "import sase.history.prompt_store_mutations"` succeeds.
   - `sase-13p`: the import-budget node passes in isolation, and the cap is ≤ 3570.
   - `sase-1gx`: the read-only regression tests pass.
   - `sase-14o` and `sase-18v`: the six nodes pass under raw pytest with a long
     `--basetemp` on this host.
   - `sase-1h1`: the bead's three repro inputs behave correctly.
   - `sase-1f0`: 20 serial repetitions pass.
   - `sase-1br`: 20 serial repetitions pass.
   - `sase-10d`: PyPI serves the new sase and sase-core-rs, and `uv pip compile` of
     `sase` resolves to it.
3. **If any bead is open, or its check fails:** reopen it (and its phase bead) with
   `sase bead open`, then finish the work. Either do it in the landing turn if it is
   small, or rerun `sase bead work <epic-id>` so the owning phase runs again. Then
   repeat steps 1–2.
   - Never close one of the ten as `canceled` or `superseded`.
   - Never close the epic while any of the ten is unresolved, even if every phase bead
     is closed.
4. Run `sase tool run check`. Turn every phase's `PROPOSED FOLLOW-UP:` notes into task
   beads through `/sase_new_task` where warranted.
5. Only then close the epic bead.
