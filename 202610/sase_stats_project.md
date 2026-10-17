---
tier: epic
title: "sase stats project: one shared Projects report with honest model attribution"
goal: "`sase stats project` and the Admin Center's Statistics → Projects tab render one
  Rust-backed report through the same loader, view model, legends, and section
  renderers. The report shows landed commits and committed plans attributed to a
  provider/model with visible coverage, a lifetime code snapshot, and this machine's
  agent runs. Every new agent commit and proposed plan records its model at write time.

  "
decisions:
  stats_decision_record:
    ask:
      "Add a decisions-web record: stats count each fact at its durable source, and
      footers stamp only underivable facts?"
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: provenance
    title: Write-time model provenance and the commit-record projection fix
    depends_on: []
    size: medium
    description:
      "provenance: stamp `SASE_MODEL=<provider>/<model>` on agent commits and
      `proposed_with` on proposed plans (including the sase-core plan-schema change),
      and stop projecting PR URLs into commit records' `sha`."
  - id: inventory
    title: Per-project linked-repo resolution and declared-by provenance
    depends_on: []
    size: medium
    description:
      "inventory: fix sase-1lk so another project's linked repos never absorb the cwd
      project's entries, and record whether each linked repo was declared by the project
      or by user-wide config."
  - id: foundation
    title: Statistics registry, neutral render primitives, palette, and range grammar
    depends_on: []
    size: medium
    description:
      "foundation: a zero-visual-diff refactor that promotes the view specs and legends
      into `sase.stats.registry`, moves the loader out of the TUI package, extracts
      Textual-free render primitives and the provider palette, and adds one shared range
      parser."
  - id: core-engine
    title: sase-core project_stats wire, git history walk, cache, and footer attribution
    depends_on: []
    size: large
    description:
      "core-engine: add `sase_core::project_stats` with the full request/response wire,
      a hardened git runner, the cached incremental history walk, the Output and
      lifetime aggregates with footer attribution, per-facet seams, and the
      `project_stats_query` binding."
  - id: core-runs
    title: Runs facet over the existing agent_stats fold
    depends_on:
      - core-engine
    size: small
    description:
      "core-runs: fill the Runs facet and Patch rows by calling
      `agent_stats::query_run_stats` once, adding a code-repos-only run-commit count,
      and prove the numbers match today's Projects tab."
  - id: core-attribution
    title: Agent directory join and the committed-plans facet
    depends_on:
      - core-engine
    size: medium
    description:
      "core-attribution: build the read-only agent directory from agents-sidecar pages
      and the local run index for attribution steps 2-4, and count committed plans
      (rename-aware first adds) credited to the proposing model."
  - id: core-code
    title: Code snapshot, role classifier, tokei gate, and records tier
    depends_on:
      - core-engine
    size: large
    description:
      "core-code: measure role-classified lines over each code repo's committed tree
      with a per-blob incremental cache, run the tokei dependency gate (falling back to
      non-blank lines), and summarize the records tier."
  - id: loader
    title: Python project-stats loader, facade, and view model
    depends_on:
      - foundation
      - inventory
      - core-engine
    size: medium
    description:
      "loader: add `sase.stats.project.load_project_stats`, which resolves projects,
      tiers their repos, builds the request, calls the binding, and returns a frozen
      `ProjectStatsView`, plus the facade, pin bump, and the `sase disk` cache row."
  - id: render
    title: Shared Rich and Markdown section renderers
    depends_on:
      - loader
    size: medium
    description:
      "render: add the shared report groupings and legends to the registry, and build
      pure section renderers (card, project, repo, model, and patch tables, legend,
      coverage) and the Markdown renderer over `ProjectStatsView` and a `RenderContext`,
      with width-pinned goldens."
  - id: cli
    title: The sase stats group and the project command
    depends_on:
      - render
      - core-runs
      - core-attribution
      - core-code
    size: medium
    description:
      "cli: register `sase stats` and `sase stats project` with rich/markdown/json
      output, the versioned JSON envelope, fetch, exit codes, root help, completion
      snapshot, and the generated CLI reference page."
  - id: tui
    title: Statistics → Projects tab migration
    depends_on:
      - render
      - core-runs
      - core-attribution
      - core-code
    size: large
    description:
      "tui: route the Projects tab through `load_project_stats` with progressive paint
      and revalidating ticks, render it with the shared renderers and the new
      project/repo/model/patch groupings, relabel Overview's tile, and update help,
      legends, goldens, and docs."
  - id: acceptance
    title: Cross-surface parity, budgets, and real-data verification
    depends_on:
      - provenance
      - cli
      - tui
    size: small
    description:
      "acceptance: add the CLI-to-TUI parity and registry option-map tests, verify
      invariants and coverage on this machine's real data, measure the performance
      budgets, run the full visual suite, and record follow-ups."
proposed_by: bbugyi200.apollo.research.1s.linker.w0
decided_by: auto
create_time: 2026-10-16 22:05:14
status: wip
---

- **PROMPT:**
  [prompts/202610/sase_stats_project.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/sase_stats_project.md)

# Plan: `sase stats project`, one shared Projects report with honest model attribution

## 1. Summary

This epic adds a new top-level `sase stats` command group. Its first and only view in
this epic is `sase stats project`. It also migrates the Admin Center's **Statistics →
Projects** sub-tab onto the same data source and code paths.

The design follows the consolidated research report
`research:202610/sase_stats_project_with_model_attribution/sase_stats_project_with_model_attribution.md`.
The user agreed with every requirement it recommends (R1–R13). It builds on the earlier
repo-truth design
`research:202610/project_stats_cli_design/project_stats_cli_design.md`, whose
Code-section contract it keeps. Section 3 records the few places where this plan refines
the research.

The core ideas:

- **Count each fact at its durable source.**
  - Commits come from git history: landed commits on each code repo's default branch.
  - Committed plans come from the project's plan store: first adds of plan documents.
  - Runs, success, agent time, and Patches come from this machine's run index, reusing
    today's fold.
  - Today's Projects "Commits" column counts commits recorded by runs on this machine,
    which are pre-land workspace SHAs and mostly sidecar bookkeeping. That column leaves
    the Projects table.
- **Attribute models honestly.**
  - Precedence: a `SASE_MODEL` footer, then an exact SHA in the agents sidecar, then the
    footer's agent name in a read-only agent directory, then `?`. Commits with no agent
    footer go to `∅`.
  - Model rows always include `?` and `∅` and always sum to the total. A coverage line
    names the cause of any gap.
- **Write provenance at the source first.** The `provenance` phase stamps
  `SASE_MODEL=<provider>/<model>` on agent commits and
  `proposed_with: <provider>/<model>` on plans, so new history needs no join.
- **One report with three labeled scopes.**
  - **Output:** commits, committed plans, and who made them. It is global and
    range-scoped.
  - **Code:** a lifetime snapshot of size, tests, age, and activity.
  - **Runs:** this machine's agent runs, range-scoped.
  - Every header names its clock and coverage, and names the machine whenever a section
    reads local stores.
- **"Same code path" means:** one Rust query (`sase_core::project_stats`), one Python
  loader (`sase.stats.project.load_project_stats`), one view model, one legend registry,
  and shared Rich section renderers. The TUI never shells out to the CLI, and the CLI
  never imports Textual.

## 2. Context (verified on sase master `ea84bd8786` and sase-core `181072b5`, the current pin)

### 2.1 The Statistics pane today

- **Specs and loader.** `src/sase/ace/tui/modals/statistics_pane_data.py` holds the
  `StatisticsView` literal (8 views),
  `ProjectsGroupBy = Literal["project","patch","drilldown"]`, `StatisticsViewSpec` /
  `STATISTICS_VIEW_SPECS`, `VIEW_ORDER` and the other lookup tables,
  `StatisticsViewData`, and `load_statistics_view(...)`.
  - The loader always runs `query_run_stats` (tribe grouping), `query_activity_stats`, a
    previous-window query, and `build_statistics_views`, whichever view is shown.
- **Legends.** `VIEW_LEGENDS` lives in `statistics_pane_legends.py`. The help modal
  (`statistics_help_modal.py`) builds its glossary from it and duplicates the Projects
  group labels in `_projects_group_label`.
- **Projects renderers.** `statistics_pane_projects.py`
  (`StatisticsProjectsRenderingMixin`).
  - Its `_percent` / `_share_bar` are `NotImplementedError` stubs. So are those in
    `statistics_pane_macros.py` and `statistics_pane_perf.py`. The real versions in
    `statistics_pane_views.py` win through the MRO.
  - The mixin also owns `_legend_note`, `_project_cell`, and `_format_timestamp`, which
    other views use.
  - Project cells and the scope chip are colored with `categorical_color(project_key)`
    (`src/sase/telemetry/render/palette.py`). Every other project-tag surface uses
    `project_accent` (`src/sase/project_accents.py`).
- **Pane behavior.** `statistics_pane.py` sets `_REFRESH_INTERVAL_SECONDS = 30.0` and
  loads through `run_worker(..., thread=True, exclusive=True)`.
  - Stale results are rejected by view, range, project filter, and macro focus.
  - `g` cycles the Projects grouping without a reload.
  - The Overview "Commits" tile (`statistics_pane_rendering.py`, about lines 170-177)
    shows run-recorded commits, and `OVERVIEW_TILE_TARGETS` sends it to Projects.
- **Numbers.**
  - `sase.stats.query.query_run_stats` calls the Rust binding `agent_stats_query_runs`
    (`sase_core::agent_stats::run::query::query_run_stats`) under
    `agent_artifact_index_operation_lock()`.
  - Commits are `len(done.step_output.meta_commits)` per run.
    `success_rate = completed / runs`. Agent time is the sum of run durations.
- **Ranges.** `src/sase/stats/ranges.py` defines `DEFAULT_PRESET = "7d"`, the presets
  (`today 24h 7d 30d 90d all`), and `parse_custom_range`.
  - `parse_custom_range` accepts `Nh/Nd/Nw`, `YYYY-MM`, `YYYY-MM-DD..YYYY-MM-DD`, and
    `YYYY-MM-DD..`.
  - It rejects a bare `YYYY-MM-DD` and preset words.
  - Ranges are `[start, end)` and are capped at now. The timezone comes from
    `sase.core.time.get_timezone()`.
- **`sase.stats` already exists** as the Textual-free statistics facade (`query.py`,
  `ranges.py`, `views.py`, `_view_builders.py`, and so on). This epic grows it; it does
  not shadow it.
- **Tests.**
  - Unit tests: `tests/stats/`, `tests/ace/tui/_statistics_pane_helpers.py`,
    `test_statistics_pane_*.py`, `test_statistics_legends_states.py`, and
    `test_statistics_help_modal.py`. Some monkeypatch module paths such as
    `sase.ace.tui.modals.statistics_pane_projects.categorical_color` and
    `sase.ace.tui.modals.statistics_pane_data.query_run_stats`.
  - Visual tests:
    `tests/ace/tui/visual/test_ace_png_snapshots_config_center_statistics.py` produces
    `config_center_statistics_projects_120x40.png` and
    `..._projects_drilldown_120x40.png`. Its helpers monkeypatch
    `statistics_pane.load_statistics_view`.
- **Palette modules.**
  - `src/sase/ace/tui/provider_styles.py` imports `textual.color` for `_derived_style`
    blending.
  - `src/sase/integrations/provider_badges.py` (emoji badges) is already
    presentation-neutral.

### 2.2 Provenance today

- **Commit tags.** `src/sase/workflows/commit/runtime_tags.py` defines three key sets:
  - `PRODUCED_RUNTIME_COMMIT_TAG_KEYS = {"AGENT"}`
  - `STALE_RUNTIME_COMMIT_TAG_KEYS = {"AGENT","MACHINE"}`
  - `RUN_OWNED_COMMIT_TAG_KEYS = {"AGENT","TYPE","BEAD"}`

  `_resolve_runtime_commit_tags()` resolves `AGENT` through `resolve_local_agent_name()`
  (`src/sase/agent/identity.py`), which reads `$SASE_ARTIFACTS_DIR/agent_meta.json`.
  Footer rendering goes through Rust (`update_commit_footer`) and needs no key
  whitelist. `SASE_MODEL=` round-trips today; `parse_commit_footer` canonicalizes both
  `SASE_MODEL=` and `MODEL=` to key `MODEL`.

- **Host-owned commits.** Finalizers run `sase stitch create` with `SASE_ARTIFACTS_DIR`
  set to the producing turn's artifacts dir (`finalizers/commit_repair_stitch.py`). The
  child therefore reads that turn's `agent_meta.json`.
- **Model fields.** `agent_meta.json` carries `llm_provider` and `model`, where `model`
  is the provider's model setting (for example `opus`, not a concrete id). It also
  carries `exec_llm_provider` (the execution harness) and `model_alias` (the SASE size
  alias).
  - During a fallback retry, `SASE_MODEL_OVERRIDE` is set in the environment, but the
    meta is not rewritten.
  - `final_transcript_model_provider` (`src/sase/axe/run_agent_exec_finalize_chat.py`)
    is the existing precedence precedent.
  - `format_metadata_model(provider, model)` (`src/sase/history/chat_storage.py`)
    formats `provider/model`.
- **Tag chips.** `src/sase/vcs_log/_tag_style.py` defines `_KEY_PRIORITY`
  (`TYPE, AGENT, MACHINE, PLAN, BUG`).
- **Plan stamps.** `src/sase/main/plan_propose_handler.py` stamps `proposed_by`
  (`acting_agent_name()`, a global name such as `bbugyi200.athena.q8--plan`) and
  `project`, then re-validates.
  - **The sase-core plan validator rejects unknown keys.** `proposed_with` fails with
    `unknown-key` until it joins `SYSTEM_FIELDS` in
    `crates/sase_core/src/plan/validate.rs`.
  - `tests/test_plan_validate.py` asserts the exact system-field list.
- **Commit-record projection bug.** `_commit_result_list_record`
  (`src/sase/axe/run_agent_helpers_state.py`) derives `sha` from `result`, which is a PR
  URL for `create_pull_request`, and it drops `commit_sha`, `method`, and `run_id`.
  `read_commit_result_metadata` in the same file sets `meta_new_commit = result`, which
  has the same flaw.

### 2.3 Data sources

- **Footers.** Agent commits carry a terminal footer such as
  `SASE_AGENT=[bbugyi200.athena.sase-1lj.land][3]`, a session or family global name in
  link form. No commit carries a model.
- **Agents sidecar pages.** Each project's agents sidecar (for sase,
  `~/.sase/projects/gh_sase-org__sase/repos/agents`) holds about 17,000
  `agents/<global_name>/` pages.
  - `meta.json` (schema 2) has `global_name`, `local_name`,
    `owner{username,machine_name}`, `project{key,name}`, and
    `metadata{llm_provider, model, agent_family, agent_family_role, ...}`.
  - `commits.json` has
    `{schema_version, source_run_id, commits:[{sha, subject, committed_at}]}`.
  - Page names end in a role suffix such as `--plan` or `--code`.
  - The Python schemas live in `src/sase/agents_sync/v2_run_io.py`. Nothing in core
    parses them yet.
  - Publication from athena has been degraded since September (`sase-10x`, the
    in-progress `sase-11o.2` backfill). The directory join therefore resolves far fewer
    recent commits. This epic does not own that recovery; coverage improves on its own
    once the backfill lands.
- **Plan store.** `src/sase/sdd/_store_types.py` defines the storage kinds
  `in_tree | local | separate_repo | sidecar_repos`. `resolve_sdd_store(...)` and
  `canonical_sidecar_dir_for_project(project, "plans")`
  (`src/sase/bead/store_locator.py`) find a project's plans. `local` plans are never
  committed.
- **Repo inventory.**
  - `repo_inventory_session()` / `collect_repo_inventory(...)`
    (`src/sase/repo_inventory.py`) produce `RepoRecord`s with
    `kind ∈ {primary, sidecar, linked, external}`.
  - They cannot tell a project-declared linked repo from a user-wide one; both report
    `source: "repos.linked config"`.
  - `sase-1lk` is open: resolving another project's linked repos from inside a checkout
    merges in the cwd project's `repos.linked`.
- **Project resolution.** `load_project_ref_display_snapshot().project_key_for_ref(ref)`
  (`src/sase/project_display_names.py`) resolves a key, alias, or label
  case-insensitively. It is already used by `sase artifact stats`.

### 2.4 sase-core building blocks

- **Cache precedent.** `memory_history::cache` stores
  `{cache_dir}/v{schema}/{sha256}.json`.
  - Equal tips are fresh; an ancestor tip folds; anything else rebuilds.
  - A corrupt file rebuilds. Writes use an `fs2` writer-only `.lock`, an atomic
    tempfile, and a directory fsync.
  - A process memo is keyed by `(path, mtime, len)`.
- **Git runner.** `file_history::runner::run_git_unchecked` / `run_git_checked` is
  bounded (timeout, `max_bytes`, `truncated`) and sets `--no-optional-locks`,
  `GIT_OPTIONAL_LOCKS=0`, and `GIT_TERMINAL_PROMPT=0`.
  - It does **not** scrub `GIT_DIR`, `GIT_WORK_TREE`, or `GIT_INDEX_FILE`.
  - `file_history/blobs.rs::read_blobs` and
    `memory_history/causes.rs::batched_diff_tree` write all of stdin before reading
    stdout, which can deadlock on large batches.
- **Reusable pieces.**
  - `commit_footer::parse_commit_footer(message) -> CommitFooterWire` (terminal block
    only, `SASE_` prefix stripped, `[label][id]` resolved).
  - `plan::read` (`split_frontmatter`, tier classification, flat, sharded, and bundle
    layouts).
  - `file_history`'s rename-aware lineage index.
  - `agent_stats::run::query::query_run_stats(index_path, AgentRunStatsRequestWire)`,
    which returns per-project (`AgentProjectWorkStatsWire`) and per-Patch
    (`AgentPatchWorkStatsWire`) rows.
- **Bindings.** `crates/sase_core_py/src/memory_history/mod.rs` shows the JSON-wire
  binding pattern (`parse_request`, `py.allow_threads`, `serialize_to_py`), with
  round-trip tests in `src/<domain>/tests.rs`.
- **Dependencies.** `tokei` and `rayon` are not in `Cargo.lock`; parallel work uses
  `std::thread::scope`. The workspace hack is managed by cargo-hakari.
- **Pin.** A declaration that commits both repos gets the pin written automatically. A
  sase-only change that needs an already-landed core commit runs
  `just ratchet-core-revision` (`docs/contribute/rust-backend.md`, "The CI source
  revision pin").

## 3. How this plan refines the research

1. **No feature flag.**
   - Every user-visible surface lands complete in one phase: `cli` lands after all three
     core facets exist, and `tui` migrates the tab in one phase.
   - The core phases and the `loader` and `render` phases have no user-visible callers.
   - The research's `beta` flag was conditional on the tab switch landing partially,
     which this phase graph avoids.
   - Public Python symbols that land before their consumer use Symvision
     `--epic-symbol <epic-bead>(<symbol>)` rows in the `Justfile`. The consuming phase
     deletes them.
2. **The old Projects tables are replaced, not extracted.**
   - The research's zero-diff extraction of the Projects renderers would move code that
     the `tui` phase deletes.
   - `foundation` extracts only what survives: the shared primitives, the palette, the
     registry, and the loader.
   - The new renderers are written once, against the new view model, in `render`.
3. **The full wire lands first.** `core-engine` defines every request and response
   struct for all three facets, with seams that return `null` for facets not yet
   implemented. The three later core phases fill their own files in parallel and change
   little shared code.
4. **Clocks.** Range filtering uses **committer time**, so anyone can check a count with
   plain `git log --since`. Lifetime facts (first and last commit, age, active days) use
   **author time**, matching the prior design and `vcs_log`, and are bucketed into days
   in the SASE timezone. The legends say so.
5. **Lifetime history ships with Output.** Lifetime facts come from the history rows
   Output already walks, so the card's "born · N commits all time" line paints in the
   first worker. Only the LoC snapshot and the records tier wait for the `code` facet.
6. **Config role globs are deferred.**
   - v1 classifies roles from `.gitattributes` (`linguist-generated`, `-vendored`,
     `-documentation`) and built-in heuristics.
   - The prior design's `project_stats.roles` config globs, the latest-release tag, and
     the monthly sparklines are follow-ups (§22). The research's metric contract does
     not list them.
7. **The tokei gate has concrete thresholds** (`core-code`): accept tokei if it adds at
   most 2 MiB to the release `sase_core_rs` wheel and at most 20% to a clean release
   build of `sase_core_py`. Otherwise ship a "non-blank lines" counter with the same
   wire shape.
8. **"Run commits" is code-only.** The `patch` grouping's run-commit column excludes
   commits whose `repo_name` is a record-tier repo (beads, plans, research, agents,
   attachments). This needs one additive request field on `agent_stats` (`core-runs`).

## 4. Metric contract

Every metric has one definition, one clock, and one scope. The legend text lives once in
`sase.stats.registry` and feeds the TUI legend, the `?` help glossary, the CLI `-h`
definitions epilog, and the Markdown footer.

- **Output** (range-scoped, global: identical on every machine with the same refs)
  - **Commits:** commits reachable from each code repo's default-branch ref, merges
    included, deduplicated by SHA across repos. Clock: committer time in `[start, end)`.
  - **Agent share:** commits whose terminal footer has `SASE_AGENT=` / `AGENT=`, divided
    by commits.
  - **By model:** see §5.1. Rows (including `?` and `∅`) sum to commits.
  - **Plans committed:** plan documents (`tier: tale|epic`) first added to the project's
    plan store, rename-aware, by tier. Clock: first-add committer time.
    - `*.plan-decisions.json`, non-plan Markdown, and later edits are excluded.
    - Projects with `local` storage show `—` (not committed), never `0`.
  - **Plans by model:** see §5.2. Rows sum to plans committed.
  - **Model coverage:** resolved ÷ agent-tagged, with unknown counts by reason.
- **Code** (lifetime, global)
  - **First commit, last commit, age, active days, lifetime commits, lifetime agent
    share:** over the union of the project's code repos. Clock: author time; active days
    are distinct author dates in the SASE timezone.
  - **Code lines, test share, doc lines, agent-context files, languages:**
    role-classified lines over the committed tree at the recorded commit
    (`<ref>@<oid>`).
    - Code = source + tests in SLOC (tokei "code" lines), or non-blank lines if the
      tokei gate fails.
    - Doc lines are non-blank.
- **Runs** (range-scoped, **this machine**)
  - **Runs ✓/×, success, agent time, Patches, last run:** today's exact definitions
    (success = completed ÷ all runs; time = summed run durations). Clock: launch time.
  - **Run commits** (patch grouping only): commits recorded by runs, to code repos only.

**Zero versus unknown.** A measured zero prints `0`. An unavailable or inapplicable
value prints `—` in human output and `null` in JSON. A missing repo never becomes 0
lines; it reduces coverage ("6 of 7 code repos").

**Totals.**

- Project totals deduplicate shared repos by identity.
- Portfolio totals deduplicate commits by SHA.
- Model rows across projects aggregate by `(provider, model)` over the deduplicated
  commit set.
- A display row cap (for example, 50 Patch rows) never truncates totals, and the number
  of omitted rows is printed.

**Never in this command:** quality rankings of models, cost per commit, productivity
scores, tables of individual agents or people, or ordinals. Rows sort by count with
stable tiebreaks.

## 5. Attribution

### 5.1 Commit → provider/model precedence

1. A `SASE_MODEL` footer value (`MODEL` key, either spelling), split on the first `/`
   into provider and model.
   - This is authoritative.
   - A value without `/` goes to `?` with reason `malformed_model_tag`.
2. An exact SHA in an agents-sidecar member's `commits.json`, using that page's
   `metadata.llm_provider` / `metadata.model`.
3. The footer's `AGENT` name (the link label) in the agent directory, tried in order:
   1. the `<name>--code` page;
   2. the exact page `<name>`;
   3. a model shared by every member of that family (pages whose `metadata.agent_family`
      or name prefix matches). Differing members mean `session_ambiguous`.
4. Otherwise `?` (model not recorded), with a reason:
   - `page_unpublished`: no page or index row;
   - `page_metadata_empty`: the page exists but has no model;
   - `session_ambiguous`;
   - `malformed_model_tag`.

A commit with no `AGENT` footer goes to `∅` (no agent footer). That means human work, or
agent work from before signing began on 2026-05-24. When a footer repeats a key, the
first `AGENT` and the first `MODEL` win, matching
`memory_history::classify::fill_provenance`.

**The agent directory.** It is a read-only map built at query time from:

- every registered project's agents-sidecar pages at the sidecar's `HEAD`;
- this machine's run index (`agent_artifacts` rows: name, `llm_provider`, `model`),
  globalized with the request's `local_agent_name_prefix` (for example
  `bbugyi200.apollo.`).

A published page wins over an index row for the same name. It is not an import into
local agent state, so decision `agents-sync-publish-only` stays intact. Because
attribution resolves at query time over cached commit rows, a directory change
re-attributes old commits without re-walking history.

### 5.2 Plan → provider/model precedence

1. `proposed_with` frontmatter.
2. `proposed_by` in the directory, trying `<name>--plan` first and then the exact name.
3. Otherwise `?` (a proposer is named but unresolved) or `∅` (no proposer recorded,
   which covers plans from before July 2026).

Frontmatter (`tier`, `proposed_with`, `proposed_by`, `project`) is read from the plan's
blob at the store tip, following renames. A deleted plan uses its last version. That way
a re-proposal, which re-stamps `proposed_with`, credits whoever proposed the version
that stands. The first-add date still decides which range a plan counts in.

The plan store resolves through the project's SDD storage:

- **`sidecar_repos`:** the plans sidecar.
- **`separate_repo`:** that repo, filtered by the plan's `project:` stamp when the repo
  is shared.
- **`in_tree`:** the primary repo's plans path; those commits stay in code totals.
- **`local`:** `plans_committed: null`, shown as `—`.

### 5.3 Model identity

- Group by `(provider, model)` as recorded, for example `claude · opus`.
- Size aliases and configured model aliases are not models. A leaked alias is kept,
  flagged `model_is_alias` in JSON, and dimmed in tables. The request carries
  `known_model_aliases` from config.
- Stamp what the adapter reports, and never invent a concrete model id.

### 5.4 Invariants

- `Σ model rows (including ? and ∅) = commits`.
- `Σ provider subtotals = commits`.
- The same two identities hold for plans.
- Numerators never exceed denominators.
- An unavailable _source_ (a repo not cloned, the index missing) is reported as an issue
  and in coverage, separately from an available commit whose model is unknown.

## 6. Architecture

### 6.1 Boundary

```text
                 sase-core (Rust)                                         sase (Python)
 ┌──────────────────────────────────────────────────┐  ┌───────────────────────────────────────────────┐
 │ project_stats::query(ProjectStatsRequestWire)    │  │ sase.stats.project.load_project_stats(query)  │
 │  ├ git        hardened runner, ref + identity    │◀─│  resolve projects, tier repos (inventory),    │
 │  ├ history    cached incremental log + footers   │  │  plan store, agents sidecars, run index       │
 │  ├ attribution footer → SHA → directory → ?/∅    │  │  → request wire → binding → ProjectStatsView  │
 │  ├ directory  agents-sidecar pages + run index   │  └───────┬────────────────────────────┬──────────┘
 │  ├ plans      plan-store first adds + frontmatter│          │                            │
 │  ├ snapshot   role-classified LoC (tokei gate)   │   sase stats project        Statistics → Projects
 │  ├ runs       agent_stats::query_run_stats       │   (rich/markdown/json)      (worker → same loader)
 │  └ aggregate  range, dedupe, coverage, fingerprint│          └──── sase.stats.render ────┘
 └──────────────────────────────────────────────────┘
          binding: project_stats_query (py.allow_threads)
```

### 6.2 Core wire (`crates/sase_core/src/project_stats/wire.rs`)

The names below are normative; field details may grow additively.

**Constants:**

- `PROJECT_STATS_WIRE_SCHEMA_VERSION = 1`
- `PROJECT_STATS_METRIC_POLICY_VERSION = 1`
- `PROJECT_STATS_CLASSIFIER_VERSION = 1`

**`ProjectStatsRequestWire`:**

- **Versioning and range.**
  - `schema_version`
  - `start_ts`, `end_ts` (exclusive)
  - `now_ts`, so tests are deterministic
  - `timezone` (IANA)
- **Facets.** `facets: [output | code | runs]`. Output always carries lifetime history.
- **Paths.**
  - `cache_dir` (empty means no cache)
  - `run_index_path` (optional)
  - `agents_sidecars: [{project_key, path}]`
- **Attribution inputs.** `local_agent_name_prefix`, `known_model_aliases`.
- **Record repos.** `record_repo_names`, used to make run commits code-only.
- **Run-only projects.** `include_run_only_projects: bool`.
- **Budget.** `budget: {timeout_ms, max_output_bytes}`.
- **`projects[]` (`ProjectStatsProjectSpecWire`).**
  - Identity: `key`, `label`, `state` (`enabled | disabled`).
  - Selection: `selection` (`always | if_runs`).
  - `index_project_names`: the names `agent_stats` uses for this project.
  - `code_repos[]` and `record_repos[]` (`ProjectStatsRepoSpecWire`):
    - `id` and `name`;
    - `kind` (`primary | linked | sidecar`);
    - `path` and `identity` (canonical remote, else empty);
    - `ref_candidates`, for example
      `["refs/remotes/origin/HEAD", "refs/remotes/origin/master", "HEAD"]`.
  - `not_counted[]: {name, reason: user_linked | external}`.
  - `plan_store: {storage, repo_path?, plans_dir, project_filter?}`.

**`ProjectStatsResponseWire`:**

- **Versions and identity.**
  - `schema_version`, `metric_policy_version`, `classifier_version`, `engine_version`
  - `range{start_ts, end_ts}`
  - `fingerprint`: a hash of the query identity, the policy versions, every repo tip,
    the plan-store tips, the agents sidecar HEADs, and the run-index token (path, mtime,
    length)
- **`projects[]` (`ProjectStatsProjectWire`).**
  - `key`, `label`, `state` (`enabled | disabled | unregistered`)
  - `output?`
    - `commits`, `agent_commits`, `agent_share`
    - `plans_committed?`, `plans_epic?`, `plans_tale?`
    - `models[]`
    - `commit_coverage`, `plan_coverage`
  - `lifetime?`
    - `first_commit_ts`, `last_commit_ts`, `age_days`, `active_days`
    - `commits`, `agent_commits`
  - `code?`
    - `counter: sloc | nonblank`
    - `code_files`, `code_lines`, `test_lines`, `test_share?`
    - `doc_files`, `doc_lines`
    - `agent_context_files`, `agent_context_lines`
    - `languages[]`
    - `repos_configured`, `repos_measured`
  - `runs?`
    - `runs`, `completed`, `failed`, `other_terminal`, `in_progress`, `waiting`,
      `success_rate?`
    - `total_runtime_seconds`, `distinct_patches`, `unattributed_runs`, `last_run_ts?`
  - `repos[]` (`ProjectStatsRepoWire`)
    - `id`, `name`, `kind`, `tier` (`code | records`), `status`
      (`ok | unavailable | partial`), `issue?`
    - `ref?{name, commit, shallow, fetched_at?}`
    - `output?` (per-repo commits, agent commits, models)
    - `lifetime?`, `snapshot?`
  - `not_counted[]`
  - `models[]` (`ProjectStatsModelRowWire`)
    - `kind` (`resolved | unknown | no_agent`), `provider`, `model`, `model_is_alias`
    - `commits`, `plans`, `plans_epic`, `plans_tale`
    - `unknown_reasons{...}`
  - `patches[]` (`ProjectStatsPatchRowWire`)
    - `name`, `status`, `has_pr`
    - `runs`, `completed`, `failed`
    - `run_commits`, `total_runtime_seconds`, `first_run_ts`, `last_run_ts`
  - `truncated_patch_rows`, `coverage`
- **`portfolio`:** deduplicated totals and portfolio-level `models[]`.
- **`issues[]`:** `{project?, repo?, code, message, hint?}`.

A facet that was not requested is `null`. A requested facet whose source is unavailable
is present, with `null` values and an issue. The binding is
`project_stats_query(request: dict) -> dict`.

### 6.3 Cache (`crates/sase_core/src/project_stats/cache.rs`)

- **Layout.** `{cache_dir}/v{schema}/{sha256(key)}.json`, following
  `memory_history::cache` exactly:
  - equal tips are fresh, tip ancestry folds, and anything else rebuilds;
  - a corrupt or wrong-schema file rebuilds silently;
  - writes take a writer-only lock and are atomic;
  - a process memo is keyed by `(path, mtime, len)`.
- **Code.** Generalize `memory_history::cache`'s load, persist, and memo helpers into a
  crate-internal module if that is mechanical; otherwise mirror them.
- **Kinds of entry:**
  - history rows per repo, keyed by repo identity and git common dir, with the tip
    stored inside;
  - the snapshot per repo, keyed by repo identity and classifier version, holding the
    tree OID and per-path rows keyed by blob OID, so a new tree re-reads only changed
    blobs;
  - the directory per agents sidecar, with `HEAD` inside;
  - plans per plan store, with the tip inside;
  - the records tier per repo, with the tip inside.
- **Location.** Python passes `~/.sase/cache/project_stats`. `sase disk` lists it as a
  rebuildable row. Losing it costs speed, never correctness.
- **Budgets** to validate:
  - warm, one project: under 150 ms;
  - cold history, directory, and plans for sase: about 1–1.5 s;
  - cold LoC: 2–5 s;
  - warm LoC after one new commit: under 0.5 s.

### 6.4 Python packages

- **Facade.** `src/sase/core/project_stats_facade.py`:
  `require_rust_binding("project_stats_query")`, using a literal name (for
  `tools/check_sase_core_rs_bindings`).
- **`src/sase/stats/registry.py`** (`foundation`): view specs, CLI names, groupings, and
  legends.
- **`src/sase/stats/loader.py`** (`foundation`): `StatisticsViewData` and
  `load_statistics_view`, which `tui` makes view-aware.
- **`src/sase/stats/project/`** (`loader`): `ProjectStatsQuery`, project resolution, the
  request builder, `ProjectStatsView`, and
  `load_project_stats(query, *, facets=ALL_FACETS)`. This is the only function both
  surfaces call.
- **`src/sase/stats/render/`** (`foundation` adds the primitives; `render` adds the
  rest): `RenderContext`, the section renderers, composition, and Markdown.
- **`src/sase/stats/envelope.py`** (`cli`): the versioned JSON envelope that every
  future `sase stats` view reuses.
- **`src/sase/integrations/provider_palette.py`** (`foundation`): the Textual-free
  provider palette.
- **CLI modules** (`cli`): `src/sase/main/parser_stats.py`,
  `src/sase/main/stats_handler.py`, and `stats_handler_project.py`.

### 6.5 TUI integration rules (`tui_perf.md`)

- **Ticks revalidate.**
  - Each 30-second tick reruns the query off-thread; the warm cache makes it cheap.
  - When the response `fingerprint` equals the last painted one, the pane neither
    rebuilds the view nor repaints.
- **Progressive paint.**
  - Worker 1 requests `{output, runs}` and paints.
  - Worker 2 requests `{code}`; the card shows `measuring code…` until it lands.
  - First paint never waits on LoC.
- **Stale handling.**
  - A result is applied only if its query identity (view, range, project filter, facets)
    still matches.
  - A failed refresh keeps the last good report with a visible `stale · <time>` marker.
  - A first load shows "loading", never placeholder zeros.
- **Prompt-free.** The TUI never fetches. Every git call is prompt-free with a scrubbed
  environment, and a missing nested clone must never let git walk up into a parent repo.

## 7. CLI contract

```text
usage: sase stats project [-h] [-D] [-F] [-f {json,markdown,rich}] [-g {model,patch,project,repo}]
                          [-j] [-p PROJECT] [-r RANGE]

Show the Admin Center Statistics → Projects view: what each project produced (commits and committed
plans, and which provider/model produced them), how big and how old its code is, and this machine's
agent runs. Output and code facts come from each code repo's default branch and match on every machine;
run facts come from this machine's run log and say so. Same loader and definitions as the TUI tab.

options:
  -D, --include-disabled  Also include disabled projects when no -p is given
  -F, --fetch             Fetch each code repo's default branch before measuring (default: local refs)
  -f, --format {json,markdown,rich}
                          Output format (default: rich)
  -g, --group-by {model,patch,project,repo}
                          Breakdown table, like the TUI's g key (default: project for several projects,
                          model for one)
  -h, --help              show this help message and exit
  -j, --json              Shorthand for --format json
  -p, --project PROJECT   Project name, alias, key, or +tag; repeatable (default: every enabled project)
  -r, --range RANGE       today, 24h, 7d, 30d, 90d, all, Nh/Nd/Nw, YYYY-MM, YYYY-MM-DD, or START..END
                          (default: 7d, the TUI default)

definitions:
  (one line per metric, rendered from sase.stats.registry legends)

examples:
  sase stats project
  sase stats project -p sase -r all
  sase stats project -p sase -g model -r 30d
  sase stats project -p +sase -F -f markdown > sase-stats.md
  sase stats project -p sase -j | jq '.data.projects[0].models[] | {provider, model, commits, plans}'
```

**Behavior:**

- **Group help.** Bare `sase stats` prints the group help, exits 0, and lists only
  implemented views. There are no stubs.
- **Root help.** `stats` joins the curated compact root help with the summary "Show
  Admin Center Statistics views from the command line."
- **Project selection.**
  - With no `-p`: enabled projects (`always`), disabled projects that have runs in range
    (`if_runs`, or `always` with `-D`), and unregistered index projects that have runs
    in range.
  - `-p` resolves through `project_key_for_ref` after stripping one leading `+`.
  - An unknown project exits 2 with "did you mean" suggestions; an ambiguous one exits 2
    listing the candidates.
  - There is no inference from the current directory.
- **Format conflicts.** `-j` combined with a non-JSON `-f` is a usage error (exit 2).
- **Range.** `-r` uses `sase.stats.ranges.parse_stats_range`; bounds are `[start, end)`
  in the SASE timezone.
- **Fetch.** `-F` fetches each code repo's default branch with a prompt-free
  environment. A fetch failure falls back to local refs and adds an issue.
- **Streams.** stdout carries the report. A progress spinner (on an interactive
  terminal, after about 1.5 s) and diagnostics go to stderr.
- **Exit codes.**
  - **0:** success, including empty ranges and partial coverage.
  - **1:** nothing could be measured (binding missing, or every source failed).
  - **2:** usage error.
- **JSON envelope.** One envelope, with additive-only changes:

  ```json
  {
    "schema": "sase.stats.project",
    "schema_version": 1,
    "generated_at": "…",
    "machine": "apollo",
    "query": {
      "projects": ["gh_sase-org__sase"],
      "range": { "preset": "7d", "start_ts": 0, "end_ts": 0, "label": "Last 7 days" },
      "timezone": "America/New_York",
      "group_by": "model"
    },
    "versions": { "sase": "…", "sase_core": "…", "metric_policy": 1 },
    "data": { "…": "the core response wire, verbatim" }
  }
  ```

## 8. Visual design

Mockups show layout only; all numbers are illustrative.

- **Colors.**
  - Borders use the project accent (`project_accent`). The Statistics accent `#FF87D7`
    titles sections.
  - Bars use the provider palette with `provider_badges` emoji (🤖 codex, 🎭 claude, 🚀
    grok, 🦋 muse, 🪐 agy).
  - `?` is dim amber and `∅` is dim gray.
- **Rows.**
  - Provider subtotal rows sit above indented model rows. A single-model provider
    collapses into one row.
  - Each provider shows its top 3 models, then "+k more"; JSON lists every model.
  - `?` and `∅` rows always print, even at 0.
- **Numbers.**
  - Table cells hold exact integers; prose uses friendly numbers (3.1M, 8 months).
  - Scope (range or lifetime, global or "this machine", repo coverage) sits next to the
    numbers it qualifies.
- **Responsive.**
  - 120 or more columns gets the full layout.
  - At 80 columns, drop FIRST/LAST COMMIT, DOC LINES, and EPIC/TALE.
  - Below 60 columns, switch to stacked per-project cards.
  - Use ASCII bars when the terminal lacks Unicode. Honor `NO_COLOR` and `FORCE_COLOR`.
    Never convey meaning by color alone.

**One project (`sase stats project -p sase`, at least 100 columns):**

```text
╭─ ■ sase  +sase ──────────────────────────────────────── Last 7 days · Oct 09 – Oct 16, 2026 ─╮
│   OUTPUT    412 commits · 99% by agents · 61 plans committed (14 epic · 47 tale)              │
│   CODE      3.1M lines · 46% tests · 82K doc lines · 6 of 7 code repos     lifetime ▸ born   │
│             Feb 14, 2026 · 8 months · commits on 239 of 245 days · 17,755 commits all time    │
│   RUNS      233 runs on apollo · ✓ 189 × 31 · 81% success · 41 h agent time · 18 Patches      │
│   Commits by provider  ██████████████▓▓▓▓▓▓▓▓▓▒▒▒▒▒▒▒░░░░░·   🦋 160  🤖 98  🎭 71  🚀 52  ? 27  ∅ 4 │
│   Plans by provider    ████████████████▓▓▓▓▓▓▒▒▒░·           🎭 31  🤖 18  🚀 6  🦋 4  ? 2        │
╰───────────────────────────────────────────────────────────────────────────────────────────────╯
  BY MODEL · last 7 days                        COMMITS   SHARE    PLANS   EPIC   TALE
  🤖 codex                                            98     24%       18      5     13
                gpt-6-luna                            61     15%        9      2      7
                gpt-6.1-sol                           37      9%        9      3      6
  🎭 claude                                           71     17%       31      8     23
                opus                                  40     10%       30      8     22
                sonnet                                31      8%        1      0      1
  ?  model not recorded                               27      7%        2      0      2
  ∅  no agent footer                                   4      1%        0      0      0
  ──────────────────────────────────────────────────────────────────────────────────────
     total                                           412    100%       61     14     47
  Model recorded for 381 of 408 agent commits (93%) · 27 unresolved: agent page not published
  Commits: code repos' default branches, by commit time · refs fetched 2 h ago (-F to refresh)
  Plans: first committed to the plan store, credited to the proposing model · Runs: this machine only
  More: -g repo · -g patch · -r all · -j        Definitions: sase stats project -h
```

**All projects (`sase stats project`):**

```text
╭─ SASE projects · 3 enabled ─────────────────────────────── Last 7 days · Oct 09 – Oct 16, 2026 ─╮
│   OUTPUT   503 commits · 98% by agents · 77 plans committed                                       │
│   CODE     3.5M lines · 45% tests · 10 of 11 code repos measured                                  │
│   RUNS     412 runs on apollo · 84% success · 63 h agent time                                     │
│   Commits by provider  ███████████████▓▓▓▓▓▓▓▓▒▒▒▒▒░░░░·   🦋 201  🤖 140  🎭 92  🚀 41  ? 25  ∅ 4 │
╰─────────────────────────────────────────────────────────────────────────────────────────────────╯
  PROJECT     COMMITS  AGENT  PLANS   CODE LINES  TESTS     AGE   RUNS ᵃ  SUCCESS  LAST COMMIT
  ■ sase          412    99%     61    3,126,927    46%    8 mo      233      81%  4 min ago
  ■ bob-cli        89    97%     16      369,231    36%    5 mo      170      88%  1 h ago
  ────────────────────────────────────────────────────────────────────────────────────────────
    total         503    98%     77    3,496,158    45%              412      84%
  ᵃ runs launched on this machine (apollo).   -g model: providers and models across all projects
```

**The other tables:**

- **`-g repo`:**
  - Code repos: REPO, KIND, COMMITS, AGENT, CODE LINES, TESTS, DOC LINES, FIRST COMMIT,
    LAST COMMIT, with an unavailable repo dimmed and showing its reason.
  - A "RECORDS · sidecars, never totaled" section: COMMITS, FILES, FIRST COMMIT, LAST
    COMMIT.
  - A "Not counted:" line.
- **`-g patch`:** a Project → Patch tree with RUNS, ✓/×, RUN COMMITS ᵃ, AGENT TIME, LAST
  RUN, and a dim `(no Patch)` row. It absorbs today's `drilldown`.

**The TUI tab after migration:** the scope strip (Range `t/T/c` · Project `p/P` · Group
`g`), then the same card fitted to the pane width, then the selected group table, then
the one-line legend:
`Commits = default-branch commits in code repos · Plans = first committed to the plan store · Runs = this machine`.

## 9. Phase: Write-time model provenance and the commit-record projection fix (`provenance`)

This phase touches sase-core and sase. Commit both repos in one turn so the host writes
the pin.

- **sase-core plan schema.**
  - Add `proposed_with` to `SYSTEM_FIELDS` in `crates/sase_core/src/plan/validate.rs` as
    an optional non-empty string.
  - Surface it on the normalized plan wire beside `proposed_by`, with tests mirroring
    the `proposed_by` ones.
  - Run `sase tool run check` in sase-core.
- **One model-label resolver.**
  - Add `resolve_turn_model_label(artifacts_dir: str | None = None) -> str | None` in
    `src/sase/agent/identity.py`, beside `discover_agent_runtime`.
  - Precedence:
    1. If `SASE_MODEL_OVERRIDE` is set (a fallback retry is active), use
       `sase.llm_provider.registry.resolve_model_provider(value)` without consuming.
    2. Otherwise use `llm_provider` + `model` from
       `<artifacts_dir or $SASE_ARTIFACTS_DIR>/agent_meta.json`.
  - Format with `format_metadata_model`. Return `None` when either half is missing;
    never invent an id.
  - `provider` is the model's provider (`llm_provider`), not the execution harness
    (`exec_llm_provider`), matching `final_transcript_model_provider`.
  - Verify where commit finalizers run relative to the deletion of `SASE_MODEL_OVERRIDE`
    in `run_agent_exec_finalize.py`. Pin the fallback case with a test.
- **Commit footer.**
  - `_resolve_runtime_commit_tags()` adds `MODEL` whenever it adds `AGENT` and the label
    resolves.
  - Add `MODEL` to `PRODUCED_RUNTIME_COMMIT_TAG_KEYS`, to
    `STALE_RUNTIME_COMMIT_TAG_KEYS` (so amend, cherry-pick, and child PRs scrub an
    inherited value through `filter_runtime_owned_tags`), and to
    `RUN_OWNED_COMMIT_TAG_KEYS` (so `sase stitch create --resume` re-stamps it).
  - Update `test_produced_and_stale_runtime_tag_keys_are_deliberately_separate` and add
    tests:
    - a host finalizer stamps the producing turn's model, not the runner process's
      default;
    - the fallback-retry model wins;
    - an inherited `SASE_MODEL` is scrubbed;
    - resume re-stamps it;
    - a non-agent commit gets no `MODEL`;
    - `create_proposal` still adds no runtime tags.
- **Tag chips.** In `src/sase/vcs_log/_tag_style.py`, order `MODEL` directly after
  `AGENT`. Extend `tests/test_vcs_log_tags.py`.
- **Plan stamp.**
  - In `plan_propose_handler.py`, set `stamps["proposed_with"]` from
    `resolve_turn_model_label(artifacts_dir)` next to `proposed_by`. A re-proposal
    overwrites it.
  - Expose `proposed_with` on `_ValidatedPlan` (`src/sase/sdd/plan_validate.py`) and its
    facade, and update the exact field list in `tests/test_plan_validate.py`.
  - Add stamping tests beside `test_plan_command_stamps_proposer_before_formatting`.
- **Projection fix.**
  - `_commit_result_list_record` sets `sha` from `commit_sha` first. It falls back to
    `result` / `commit_result` only when the value is a 7–40 character hex SHA, and it
    never stores a URL or a path in `sha`.
  - It preserves `commit_sha`, `method`, and `run_id`.
  - Apply the same rule to `read_commit_result_metadata`'s `meta_new_commit`.
  - Check consumers (`ace/tui/models/_loaders/_done_common.py` `commit_record_key`,
    `ace/tui/widgets/prompt_panel/_agent_commits.py`) and update
    `tests/test_axe_run_agent_helpers_commit_output.py` and related tests. Add a
    `create_pull_request` record test.
- **Docs.**
  - The footer tag list in `docs/reference/vcs/commit-workflows.md`.
  - The tag sets in `docs/contribute/commit-workflow-orchestrator.md`.
  - System fields in `docs/reference/work-units/plan-format.md`.
- Run `sase tool run check` in sase.

## 10. Phase: Per-project linked-repo resolution and declared-by provenance (`inventory`)

- **Fix `sase-1lk`.**
  - `_resolution_config_cached()` (`src/sase/_linked_repo_config_state.py`) currently
    starts from `load_merged_config()`, whose last layer is the cwd project's local
    `sase/sase.yml`.
  - When the target project is not the cwd project, build the resolution config from
    layers that exclude the cwd project-local layer: defaults, plugins, user, overlays,
    and the target's local config.
  - Add the regression test the bead describes: cwd inside project A, which declares
    `../x`; a sibling `x` exists next to project B; resolving B must not list `x`.
- **Declared-by provenance.**
  - Carry each `repos.linked` entry's origin layer through the merge.
  - Add `RepoRecord.declared_by: Literal["project", "user"] | None` (`None` for
    non-linked kinds).
    - `project` means the project's own `sase/sase.yml` or ProjectSpec.
    - `user` means user-wide config or overlays.
  - Emit it additively in `sase repo list -j`. Rich output may show a dim `(user)` tag;
    it is optional.
  - This is what lets the stats loader put `chezmoi` under "not counted" without a name
    heuristic.
- **Tests.** Inventory and repo-list tests for both values and for the cross-project
  case. Note on this phase's bead whether `sase-1l2` can now reuse `declared_by`; that
  is a `PROPOSED FOLLOW-UP`, not part of this phase.
- **Close the bug.** After verification, close `sase-1lk` with
  `sase bead close sase-1lk --note "<fix and regression test>"`.
- Run `sase tool run check`.

## 11. Phase: Statistics registry, neutral render primitives, palette, and range grammar (`foundation`)

This is a refactor, proven by unchanged Statistics output. The only behavior change is
the additive range grammar.

- **Registry: `src/sase/stats/registry.py`.**
  - Move the view literal, `StatisticsViewSpec`, `STATISTICS_VIEW_SPECS`, and the order,
    label, and description lookups out of `statistics_pane_data.py`. Move `VIEW_LEGENDS`
    out of `statistics_pane_legends.py`.
  - Add these spec fields:
    - `cli_name`, the singular view noun: `overview`, `runner`, `project`, `provider`,
      `activity`, `macro`, `plan`, `perf`; map each from the existing view ids;
    - `groupings` (the Projects groupings stay `project, patch, drilldown` until `tui`);
    - `legends` (term, definition).
  - TUI modules and the help modal read the registry. Remove the duplicated group-label
    table in `statistics_help_modal.py`.
- **Loader move: `src/sase/stats/loader.py`.**
  - Move `StatisticsViewData` and `load_statistics_view` out of
    `statistics_pane_data.py`, unchanged and Textual-free.
  - `statistics_pane.py` keeps importing the name `load_statistics_view`, so the visual
    fixtures' monkeypatch target still works.
  - Update the unit tests' monkeypatch paths (`query_run_stats`, `query_activity_stats`,
    `load_project_display_snapshot`) to the new module.
- **Primitives: `src/sase/stats/render/primitives.py`.**
  - Add `percent`, `share_bar`, `stacked_bar` (with an ASCII fallback), `format_count`,
    `format_compact` (3.1M), `format_duration`, and `format_relative_age`.
  - Delete the `NotImplementedError` stubs in the Projects, Macros, and Perf mixins and
    the real versions in `statistics_pane_views.py`, routing every caller to the
    primitives. Output must stay byte-identical.
- **Palette: `src/sase/integrations/provider_palette.py`.**
  - Move the provider palette data and the derived-style blending out of
    `ace/tui/provider_styles.py`. Use pure RGB math that is numerically identical to the
    current `textual.color` blending.
  - `provider_styles.py` keeps its public API by delegating.
  - Before moving, capture a test table of every provider × style output, and assert
    equality after.
- **Range grammar:
  `sase.stats.ranges.parse_stats_range(text, *, now=None) -> StatsRange`.**
  - Accept the presets, `Nh/Nd/Nw`, `YYYY-MM`, a bare `YYYY-MM-DD` (new: that whole
    local day), `START..END`, and `START..`. Errors say what is accepted.
  - Point the TUI custom-range submit (`statistics_pane_actions.py`) at it.
  - Extend `tests/stats/test_ranges.py`, and update the range syntax in
    `docs/contribute/telemetry.md`.
- **Proof.**
  - Every Statistics unit test passes.
  - A targeted
    `just fix-tui-screenshots --check -- tests/ace/tui/visual/test_ace_png_snapshots_config_center_statistics.py`
    passes with no golden changes. Run it through `/sase_monitor` if it outlasts the
    turn.
- Run `sase tool run check`.

## 12. Phase: sase-core project_stats wire, git history walk, cache, and footer attribution (`core-engine`)

This phase works in sase-core only. Read its `AGENTS.md` first. `mod.rs` files hold only
`mod` and `pub use` lines, files stay at or under 1,500 lines, and `rustfmt` max width
is 80.

- **Module.**
  - Add `crates/sase_core/src/project_stats/` (`pub mod project_stats;` in `lib.rs`,
    alphabetical) with these files: `wire.rs` (split request and response files if
    large), `git.rs`, `history.rs`, `cache.rs`, `attribution.rs`, `aggregate.rs`, and
    `query.rs`.
  - Add facet seam files `runs.rs`, `directory.rs`, `plans.rs`, and `snapshot.rs`. Their
    functions return `None` / an empty directory until their phases land; `query.rs`
    dispatches to them.
  - Define the complete §6.2 wire now.
- **Runner hardening** (in the shared `file_history::runner`, so every caller benefits):
  - Remove `GIT_DIR`, `GIT_WORK_TREE`, `GIT_INDEX_FILE`, `GIT_OBJECT_DIRECTORY`,
    `GIT_ALTERNATE_OBJECT_DIRECTORIES`, `GIT_COMMON_DIR`, and `GIT_NAMESPACE` from the
    child environment.
  - Set `GIT_NO_LAZY_FETCH=1`.
  - Add a non-deadlocking batch helper that feeds stdin from its own thread while stdout
    drains, under the same budgets. Later phases use it for `cat-file --batch` and
    `check-attr --stdin`.
  - Migrating `read_blobs` and `batched_diff_tree` to the helper is a
    `PROPOSED FOLLOW-UP`, unless it is trivial.
- **Repos.**
  - Validate that `rev-parse --show-toplevel` equals the canonicalized spec path.
    Otherwise the repo is `unavailable: identity_mismatch`; this is how a missing nested
    clone is refused instead of walking up.
  - Resolve the first `ref_candidates` entry that names a commit.
  - Record the ref name, the commit, `--is-shallow-repository`, and the mtime of
    `FETCH_HEAD` as `fetched_at`.
  - A repo identity falls back from the spec `identity`, to the root commit, to the
    realpath.
- **History.**
  - Stream `git log -z --format=%H%x1f%P%x1f%at%x1f%ct%x1f%B <tip>` over the full DAG.
  - Parse each message once with `parse_commit_footer`, and keep rows of
    `(sha, committer_ts, author_ts, is_merge, agent?, model_tag?)`.
  - Walk `cached_tip..tip` incrementally when the cached tip is an ancestor
    (`merge-base --is-ancestor`); otherwise rebuild.
  - Budgets are generous and configurable from the request. Truncation marks the repo
    `partial` with a `history_truncated` issue, never silently.
- **Cache.** Implement §6.3 for history rows.
- **Aggregation.**
  - Output per project is the union of code-repo rows, deduplicated by SHA, filtered by
    committer time in `[start, end)`.
  - Lifetime facts follow §4 (author time; active days in the request timezone via
    `chrono-tz`).
  - Per-repo output and lifetime.
  - Portfolio totals.
  - Attribution runs through `attribution.rs` with the §5.1 precedence against a
    `Directory` value that is empty in this phase, so non-stamped agent commits become
    `? page_unpublished`.
  - `model_is_alias` comes from `known_model_aliases`.
  - Commit coverage, issues, and the fingerprint.
  - Enforce the §5.4 invariants with `debug_assert!` plus tests.
- **Binding.**
  - `crates/sase_core_py/src/project_stats/mod.rs` exposes `project_stats_query`
    (`parse_request`, `py.allow_threads`, `serialize_to_py`). Register it in `lib.rs`.
  - Add a registration test and a round-trip test in `tests.rs`.
- **Tests** (hermetic temporary repos; set `user.name` / `user.email` per repo;
  deterministic `GIT_AUTHOR_DATE` / `GIT_COMMITTER_DATE`):
  - **Footers:** both footer spellings; `SASE_MODEL` attribution; a malformed model tag;
    model-looking prose in a body that must not count; repeated tags.
  - **History shape:** merges counted; SHA dedupe across two repos that share history.
  - **Ranges:** `[start, end)` edges; a DST transition in `America/New_York`;
    non-monotonic dates.
  - **Walks:** an incremental fold equals a cold rebuild; a force push rebuilds.
  - **Cache:** a cached query equals an uncached one; a corrupt cache file rebuilds.
  - **Repo safety:** an inherited `GIT_DIR` is ignored; a missing nested clone does not
    walk up; a shallow clone is flagged.
  - **Invariants.**
- **Measure** the cold and warm history walk on the sase repo, and record the numbers in
  this phase's bead notes.
- Run `sase tool run check` in sase-core. This phase changes no sase files.

## 13. Phase: Runs facet over the existing agent_stats fold (`core-runs`)

- **Additive `agent_stats` field.**
  - Add `#[serde(default)] excluded_commit_repo_names: Vec<String>` to
    `AgentRunStatsRequestWire`.
  - The project and Patch commit folds skip `meta_commits` whose `repo_name` is in the
    set. An empty set keeps today's numbers exactly; test that.
- **`runs.rs`.**
  - Call `agent_stats::run::query::query_run_stats(index_path, ...)` once per query with
    `project: None`, the request range, a Patch row cap (`work_top_n`) large enough for
    the displayed rows (truncation still reported through `truncated_patch_rows`), and
    `excluded_commit_repo_names = record_repo_names`.
  - Map project rows through `index_project_names`.
  - Apply `selection`: an `if_runs` project with no runs in range is dropped from the
    response.
  - Append unregistered index projects with runs as `state: unregistered` rows (Output
    and Code `null`) when `include_run_only_projects` is set.
  - Fill `patches[]` with `run_commits` and the portfolio run totals.
  - A missing index is an `index_missing` issue with `runs: null`.
- **Migration oracle.** Over a fixture index, the facet's runs, success, time, and
  Patches equal today's `query_run_stats` per-project and per-Patch rows (commit counts
  aside).
- Run `sase tool run check` in sase-core.

## 14. Phase: Agent directory join and the committed-plans facet (`core-attribution`)

- **`directory.rs`.**
  - For each agents sidecar: resolve `HEAD`, then `ls-tree -r -z HEAD -- agents/` for
    `*/meta.json` and `*/commits.json`.
  - Read the blobs with the batch helper, and parse the §2.3 schemas tolerantly (skip
    and count malformed pages).
  - Build:
    - name → `{provider, model, family, role}`;
    - family → members;
    - SHA → page name.
  - Add the run index's agent rows under `local_agent_name_prefix`; a published page
    wins on conflict.
  - Cache per sidecar by `HEAD`. An incremental `diff-tree` fold over changed pages is
    welcome if simple.
  - Plug the directory into `attribution.rs` for §5.1 steps 2–4 with the exact unknown
    reasons.
- **`plans.rs`.**
  - Resolve the plan store from the request descriptor.
  - Find rename-aware first adds of plan documents under `plans_dir`. Reuse
    `file_history`'s lineage index with that pathspec if it fits; otherwise walk
    `git log --raw -M --diff-filter=AR`.
  - Apply the §4 exclusions. Recognize layouts with `plan::read`'s helpers
    (`split_frontmatter`, tier classification).
  - Read frontmatter at the tip per §5.2, apply the `project_filter` for shared
    `separate_repo` stores, and credit by the §5.2 precedence.
  - Fill `plans_committed`, `plans_epic`, `plans_tale`, `models[].plans*`, and
    `plan_coverage`. `local` storage gives `null`.
  - Cache by store tip, folding on ancestry.
- **Tests:**
  - **Commit attribution:** session-container footers with mixed member models give
    `session_ambiguous`; SHA beats name; empty page metadata gives
    `page_metadata_empty`; the `--code` member is preferred; an index-only agent
    resolves; a published page wins over the index.
  - **Plan attribution:** the `--plan` member lookup.
  - **Plan counting:** first add across a cross-directory rename;
    `*.plan-decisions.json` excluded; an edit after the add is not double-counted; a
    re-proposal credits the latest `proposed_with`; `local` storage reports `null`; a
    shared store filters by project.
  - **Invariants.**
- **Measure** the cold and warm directory build over the real sase agents sidecar (about
  17,000 pages; target about 0.5 s cold), and record it on the bead.
- Run `sase tool run check` in sase-core.

## 15. Phase: Code snapshot, role classifier, tokei gate, and records tier (`core-code`)

- **Snapshot (`snapshot.rs`, `classify.rs`).**
  - Per code repo, at the measured commit: `ls-tree -r -z -l <commit>`.
  - Read `.gitattributes` overrides with batched
    `git check-attr --source <commit> -z --stdin linguist-generated linguist-vendored linguist-documentation`.
  - Classify every entry into exactly one role, or into an exclusion with a reason:
    - **Roles:** source, tests, docs, agent_context, config_data, other.
    - **Exclusions:** generated, vendored, binary.
  - Role precedence:
    1. `.gitattributes`.
    2. Built-in heuristics:
       - test directories and filename patterns count as tests;
       - prose languages count as docs;
       - `sase/memory/`, `sase/macros/`, skills, and agent instruction files
         (`AGENTS.md`, `CLAUDE.md`, and similar) count as agent context;
       - data formats and lockfiles count as config/data;
       - golden and snapshot directories and minified files count as generated;
       - NUL-containing blobs and LFS pointers count as binary.
  - Read text candidates through the batch helper, and skip blobs over a size cap as
    `generated`.
  - Gitlinks are skipped and counted; symlinks are not followed.
- **The tokei gate.**
  - Add `tokei` with `default-features = false`. Use only blob APIs: language from the
    extension, plus a SASE-owned shebang check, then `parse_from_slice` with an explicit
    `Config`, never the ambient `tokei.toml`.
  - Regenerate the workspace hack (`cargo hakari generate && cargo hakari manage-deps`).
  - Measure three things: the release `sase_core_rs` wheel size delta, the clean release
    build time delta for `sase_core_py`, and the new crates in `Cargo.lock`.
  - **Accept** if the wheel grows by at most 2 MiB and the clean build time by at most
    20%. Otherwise remove tokei and ship `counter: "nonblank"` (non-blank lines for
    code) with the same wire shape and a policy note.
  - Record the measurements and the outcome on this phase's bead.
- **Incremental cache.** Per §6.3: per-path rows keyed by blob OID, so a new tree
  re-reads only changed blobs. A classifier version change rebuilds.
- **Records tier.** For each record repo at its tip: commits (`rev-list --count`),
  tracked files (`ls-tree` count), and first and last commit. There are no content
  reads, and results are cached by tip.
- **Aggregation.**
  - `code` per project, deduplicated by repo identity, with `repos_configured` /
    `repos_measured`.
  - Per-repo `snapshot`.
  - Portfolio code totals.
  - `test_share = null` when there are no code lines.
- **Tests:**
  - **Line edges:** CRLF; a final unterminated line.
  - **Classification:** binary and LFS blobs; `.gitattributes` overrides; a gitlink and
    a symlink.
  - **Repo edges:** an empty repo; a zero-code repo (`—`, no division by zero).
  - **Cache:** a one-commit incremental snapshot equals a cold one.
  - **Invariants:** role partitions sum to included files; tests ⊆ code.
- **Measure** cold LoC and warm-after-one-commit LoC on sase, and record them.
- Run `sase tool run check` in sase-core.

## 16. Phase: Python project-stats loader, facade, and view model (`loader`)

- **Facade and pin.**
  - Add `src/sase/core/project_stats_facade.py`.
  - Move `sase-core-revision.txt` past the landed core commits with
    `just ratchet-core-revision`. `core-engine` must have landed.
  - Add `project_stats` to the module table in `docs/contribute/rust-backend.md`.
- **Query.** `ProjectStatsQuery` is a frozen dataclass:
  - `project_refs: tuple[str, ...]`
  - `range: StatsRange` and `preset: str | None`
  - `include_disabled: bool`
  - `group_by` (render-only)
  - `fetch: bool`
- **Resolution** (§7 behavior).
  - Strip one `+`, then call `project_key_for_ref`.
  - Raise `ProjectStatsUsageError` for unknown projects (with `difflib` suggestions over
    keys, labels, and aliases) and for ambiguous ones.
  - Apply the enabled, `if_runs`, and `-D` rules.
- **Repo tiering** (from `repo_inventory_session()` /
  `collect_repo_inventory(..., include_disabled=True)`):
  - **Code:** the primary plus linked repos with `declared_by == "project"`.
  - **Records:** sidecars.
  - **Not counted:** `declared_by == "user"` linked repos (`user_linked`) and externals.
  - **Clone choice:** the inventory's canonical clone, else the first existing
    registered clone in deterministic workspace order. With no clone, mark the repo
    unavailable (`not_cloned`).
  - **`ref_candidates`:** `refs/remotes/origin/HEAD`, then
    `refs/remotes/origin/<default branch if known>`, then `HEAD`.
- **Other request inputs.**
  - The plan-store descriptor from `resolve_sdd_store(primary, PRIMARY_WORKSPACE_NUM)`.
  - Agents-sidecar paths for every registered project.
  - The run index path, `default_agent_artifact_index_path()`.
  - `local_agent_name_prefix`, derived from `globalize_owned_agent_name`.
  - `known_model_aliases` from config.
  - `timezone` from `get_timezone()`.
  - `cache_dir` = `~/.sase/cache/project_stats`, through the existing SASE-home path
    helper.
  - `record_repo_names`.
- **Calling the binding.** When the `runs` facet is requested, hold
  `agent_artifact_index_operation_lock()` around the binding call.
- **View model.** `ProjectStatsView` is a set of frozen dataclasses mirroring §6.2:
  - projects, repos, models, patches, portfolio, issues, coverage;
  - `fingerprint`, `machine`, `generated_at`, and the raw wire `data` kept for the JSON
    envelope.
  - Helpers: provider subtotals, the effective `group_by` default, and display labels
    from the `ProjectDisplaySnapshot`.
- **Errors.**
  - `ProjectStatsUnavailableError` covers a missing binding and the case where every
    requested source failed.
  - Partial results are not errors.
- **`sase disk`.** Add a `cache/project_stats` `DiskFootprintRow` next to
  `cache/rust-prebuild` in `src/sase/core/disk_footprint_inventory_rows.py`, marked
  rebuildable.
- **Tests:**
  - **Resolution:** cwd inside another project; display name versus key; alias; `+tag`;
    a disabled project named explicitly; unknown with a suggestion; ambiguous.
  - **Tiering:** a user-linked repo goes to not counted; a sidecar goes to records; an
    external goes to not counted.
  - **Request:** a request-builder golden.
  - **View model:** built from a full fixture wire, including null facets and
    unregistered rows.
  - **Binding smoke:** the binding over temporary repos.
- Add `--epic-symbol` rows for public symbols that `render`, `cli`, or `tui` will
  consume.
- Run `sase tool run check`.

## 17. Phase: Shared Rich and Markdown section renderers (`render`)

- **Registry constants.**
  - Add `PROJECT_REPORT_GROUPINGS = ("project", "repo", "model", "patch")` and
    `PROJECT_REPORT_LEGENDS` (one entry per §4 metric, plus `?` and `∅`) to
    `sase.stats.registry`.
  - The live Projects view spec keeps its old groupings and legends until `tui` adopts
    these constants, so the old tab never shows definitions for numbers it does not
    display.
  - `cli` reads the same constants, so the two parallel phases share one source.
- **`RenderContext`.** `RenderContext(width, color, unicode, now, timezone, machine)`
  with `RenderContext.from_console(console, *, now=None, machine=None)`. Every clock,
  zone, and hostname input is injectable, so goldens are host-independent.
- **Section renderers** (pure functions of `(view, ctx)` returning Rich renderables):
  - `render_project_card` (one project or the portfolio);
  - `render_project_table`, `render_repo_table` (with the records section and the
    not-counted line), `render_model_table`, and `render_patch_tree`;
  - `render_legend` (from `PROJECT_REPORT_LEGENDS`) and `render_coverage_footer`.
- **Composition.**
  - `render_project_stats(view, group_by, ctx)` composes the card, the group table, the
    coverage line, and the legend.
  - `render_project_stats_markdown(view, group_by, *, command_line, versions) -> str`
    produces GitHub-flavored tables with no ANSI or box drawing. It escapes pipes,
    backticks, and newlines in names, and ends with the generating command and versions.
- **Rules.**
  - §8 throughout: project-accent borders; provider palette and badges from
    `provider_palette` and `provider_badges`; `?` and `∅` styling; exact integers;
    neutral sort order.
  - Breakpoints at 120, 100, 80, and 60 columns; an ASCII fallback; `NO_COLOR` and
    `FORCE_COLOR` honored through Rich.
  - The card shows `measuring code…` while `code` is null because the facet is pending
    (a view flag), and `—` when the facet was measured as unavailable.
- **Goldens.**
  - Text exports at 120, 100, 80, and 60 columns with color off, for one project, all
    projects, each `-g`, an empty range, a `local` plan store, an unavailable repo, and
    the code-pending state.
  - Markdown goldens and an ASCII fallback golden.
  - All with a pinned clock, zone, and machine.
- Add `--epic-symbol` rows for symbols that `cli` and `tui` will consume.
- Run `sase tool run check`.

## 18. Phase: The sase stats group and the project command (`cli`)

- **Parser.**
  - Add `src/sase/main/parser_stats.py` implementing §7 exactly: sorted options, a short
    alias for every long option, the help text, the `definitions:` epilog rendered from
    `PROJECT_REPORT_LEGENDS`, `-g` choices from `PROJECT_REPORT_GROUPINGS`, and
    examples.
  - Register it in `parser_registry.py` and `parser_full_registrars.py`, add the
    `entry.py` dispatch, and add `stats` to `_COMPACT_ROOT_COMMANDS` in
    `parser_root_help.py` with the §7 summary.
  - Bare `sase stats` prints the group help and exits 0.
- **Handler.** `src/sase/main/stats_handler.py` and `stats_handler_project.py`:
  - parse the range (`parse_stats_range`) and check the format conflict;
  - fetch with `-F` (prompt-free; failures become issues);
  - call `load_project_stats(query)`;
  - render rich, markdown, or JSON;
  - show the spinner on stderr only when interactive and slow;
  - return the §7 exit codes.
- **Envelope.**
  - `src/sase/stats/envelope.py` builds the §7 envelope:
    - `versions` holds the sase version, the `sase_core_rs` version, and the metric
      policy;
    - `machine` comes from the identity helpers;
    - `data` is the wire verbatim.
  - It is generic over `schema`, so future views reuse it.
- **Pin.** Ratchet `sase-core-revision.txt` again if `core-runs`, `core-attribution`, or
  `core-code` landed after the `loader` pin.
- **Generated artifacts.**
  - `just fix` regenerates `docs/reference/cli/stats.md` and the CLI index. Add the page
    to the hand-maintained `mkdocs.yml` nav in its alphabetical "Commands" section.
  - `just sync-completion-spec` updates `tests/completion/snapshots/cli_spec.json`.
  - Update `tests/main/test_parser_root_help.py` and confirm
    `tests/main/test_parser_narrowing.py`.
  - Read `docs_writing.md` before writing any prose.
- **Tests:**
  - the exit-code matrix (unknown project, ambiguous project, bad range, `-j` plus
    `-f rich`, binding missing);
  - envelope shape and that `data` is passthrough;
  - cross-format agreement: rich and Markdown numbers equal the JSON;
  - a `-h` golden;
  - bare `sase stats`.
- Remove the `--epic-symbol` rows this phase consumes.
- Run `sase tool run check`.

## 19. Phase: Statistics → Projects tab migration (`tui`)

Read `tui.md`, `tui_perf.md`, `tui_screenshot.md`, and `ace.md` first.

- **Routing.**
  - `sase.stats.loader.load_statistics_view` becomes view-aware. `projects` calls
    `load_project_stats(query, facets={"output", "runs"})` and returns
    `StatisticsViewData(project_stats=..., views=None)`. The other seven views keep
    today's composite path.
  - Overview keeps its run-based `ProjectsView` for Top projects.
- **Pane** (`statistics_pane.py`):
  - **Progressive load.** After the first result paints, start a second thread worker
    (its own group) for `{"code"}`. Merge it only when the query identity still matches.
  - **Fingerprint skip.** A result whose `fingerprint` equals the painted one does not
    repaint.
  - **Failures.** A failed refresh keeps the last good report with `stale · <time>`. A
    first load shows a loading state.
  - **Pump.** Keep all callbacks pump-thin (rules 1, 2, 4, 10).
- **Rendering** (`statistics_pane_projects.py`).
  - Compose `render_project_card`, the selected group table, and `render_legend` with a
    `RenderContext` fitted to the pane width.
  - Delete the old tables and helpers no longer used. Move the still-shared
    `_legend_note`, `_project_cell`, and `_format_timestamp` to
    `statistics_pane_views.py`.
  - The machine name in Runs labels comes from an injectable seam.
- **Groupings.**
  - `ProjectsGroupBy = Literal["project", "repo", "model", "patch"]`. The Projects view
    spec's `groupings` becomes `PROJECT_REPORT_GROUPINGS`.
  - `g` cycles them without a reload. When one project is selected, `project` is skipped
    and the effective grouping is `model`.
  - `patch` renders the Project → Patch tree, replacing `drilldown`.
  - Update `_PROJECTS_GROUP_LABELS`, the group chip, and the help modal's controls text,
    which reads from the registry.
- **Overview.** Relabel the Commits tile "Run commits (this machine)" and update its
  legend entry. The tile still targets Projects.
- **Color.** Use `project_accent` for Statistics project cells and the scope chip,
  replacing `categorical_color(project_key)`.
- **Registry.** Update the Projects description to "Compare what projects produced,
  which models produced it, their code, and this machine's runs." Make
  `PROJECT_REPORT_LEGENDS` the Projects view's `legends`, and delete the old Projects
  legend entries (Share, Wall, run-recorded Commits) that no longer describe a displayed
  number.
- **Tests.**
  - Update the rendering, loading, filters, legends, help-modal, bindings, and
    scope-header tests. Retarget the `categorical_color` monkeypatches.
  - Assert the Projects path makes exactly one `load_project_stats` call per load and no
    activity or previous-window queries.
  - Add a renderer-parity test: `render_project_stats(view, group, ctx)` exported to
    text equals the pane's painted renderable at the same width.
- **Visual.**
  - Regenerate `config_center_statistics_projects_120x40.png`. Replace the drilldown
    snapshot and its golden with a `patch` snapshot, and add `model` and `repo`
    snapshots. Regenerate the Overview golden for the relabel.
  - Pin the epoch, zone, hostname, fixture tree, and loaded-state wait.
  - Run the targeted `just fix-tui-screenshots -- <statistics selectors>` through
    `/sase_monitor`. Inspect every created, updated, and removed golden, then prove host
    independence with two `--check` runs, one under `TZ=UTC`.
- **Docs.**
  - `docs/reference/tui/admin-center.md` (Statistics → Projects).
  - `docs/reference/tui/machines-and-tools.md` (Statistics Tab).
  - `docs/contribute/telemetry.md` (the Projects row, groupings, and `(no Patch)` note).
  - Any keymap doc that names the `drilldown` grouping.
- **Perf.** With `SASE_TUI_PERF=1`, j/k on the Statistics tab stays under 16 ms at p95,
  and `~/.sase/logs/tui_stalls.jsonl` gains no rows across a few refresh ticks. Record
  the numbers on the bead.
- Run `sase tool run check`.

## 20. Phase: Cross-surface parity, budgets, and real-data verification (`acceptance`)

- **Parity tests.**
  - For a frozen fixture query, the CLI's `-j` `data` equals the wire the TUI worker
    received.
  - A registry option-map test asserts that every option the `project` view declares is
    reachable from both surfaces: range (`-r` / `t T c`), project (`-p` / `p P`), and
    grouping (`-g` / `g`). The registry declares the CLI-only options (`-D`, `-F`, `-f`,
    `-j`) explicitly.
- **Real data on this machine.**
  - Run `sase stats project -p sase -r all -j` and check:
    - the §5.4 invariants;
    - that the coverage line is present;
    - that lifetime commits for the primary repo equal
      `git rev-list --count <measured ref>`.
  - Confirm that commits landed after `provenance` resolve at step 1 (`SASE_MODEL`) and
    that plans proposed after it carry `proposed_with`.
  - Run the bare, `-g repo`, `-g patch`, `-f markdown`, and 80-column forms, and inspect
    them.
  - Capture a live `sase screenshot` of Statistics → Projects and inspect the PNG.
- **Budgets.** Measure the §6.3 budgets and TUI first paint, and record them on the
  bead. A budget missed by more than 2× is fixed at its cause or recorded as a
  `PROPOSED FOLLOW-UP` with evidence.
- **Visual suite.** Run the full `just fix-tui-screenshots` through `/sase_monitor`.
  Inspect the report, and confirm that no unrelated goldens changed and that stale
  goldens were pruned.

> [!decision] stats_decision_record

- **Decision record** (only if `stats_decision_record` is yes).
  - Use `/sase_memory_write` to add a `decisions` strand stating:
    - statistics count each fact at its durable source (git history for commits, the
      plan store for plans, the run index only for runs);
    - every `sase stats` header names its machine when a section reads local stores;
    - footers stamp only provenance that cannot be derived elsewhere (`SASE_MODEL`,
      unlike the removed `SASE_MACHINE`).
  - Give the alternatives, the cost, and the reopen condition, then run
    `sase memory init`.
  - If the decision is no, record it as a `PROPOSED FOLLOW-UP` only when no human
    reviewed the plan.
- **Follow-ups.** Record the §22 items as `PROPOSED FOLLOW-UP:` notes on this phase's
  bead.
- Run `sase tool run check`.

## 21. Verification conventions (all phases)

- Run `sase tool run check` in every repo a phase changes, and never raw `just check`.
  Run `just fix` (or at least `just fmt`) first. Run `just install-venv` first if this
  workspace's venv is stale, including after core changes, so `sase_core_rs` rebuilds
  from the linked checkout.
- Do not run `just check-full` unless explicitly instructed.
- Hand long visual runs to `/sase_monitor`.
- Phase workers create no beads. They append `PROPOSED FOLLOW-UP:` notes to their own
  phase bead. The one exception is the `inventory` phase closing `sase-1lk`.
- Follow `cli_rules.md` for every option and help string, and `gotchas` for config
  defaults.

## 22. Non-goals and follow-ups

- **The other seven views.**
  - `sase stats overview|runner|provider|activity|macro|plan|perf`: each is a CLI name
    plus a Markdown renderer over the existing loader.
  - Move the Overview commits tile onto the new source.
  - Add a `list` catalog once three or more views exist.
- **More data.**
  - Fleet-wide Runs through the gateway.
  - Churn and growth series, monthly sparklines, and the latest-release tag.
  - Cost and tokens.
  - `--as-of`.
- **Classification.**
  - Config role globs (`stats.project.roles`) on top of `.gitattributes`.
  - CSV or `auto` formats, if a script or paste workflow misses them.
- **Infrastructure.**
  - Migrate `file_history::blobs::read_blobs` and
    `memory_history::causes::batched_diff_tree` to the non-deadlocking batch helper.
  - `sase-1l2` reusing `declared_by`.
  - A stable plan id, if measured rename churn proves material.
- **Out of scope by design:** model quality rankings, per-agent or per-person tables,
  and rewriting history.
