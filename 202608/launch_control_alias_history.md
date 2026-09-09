---
status: done
tier: epic
title: Agent history for a model alias in Launch Control
goal:
  Pressing `H` on any alias-bearing Launch Control row opens a pop-up panel that answers
  "which agents actually ran on this alias, and how did they get here?" — a bounded,
  newest-first list of prior runs with the concrete model that answered, a readable
  prompt snippet, and an honest provenance chip that distinguishes a direct
  `%model:@alias` request from an alias reached through another alias and from the
  no-directive default, backed by a new per-alias retention limit config field.
phases:
  - id: provenance
    title: Record the alias resolution trail and its origin at launch
    depends_on: []
    size: large
    description:
      "provenance: make alias resolution return the ordered trail of alias hops it
      traversed, give LaunchSelection that trail plus the origin of the request
      (directive vs no-directive default vs no alias), and persist `model_alias_trail`
      and `model_alias_origin` into agent_meta.json and prompt-step markers at every
      launch, reconcile, re-exec, and follow-up write site."
  - id: core
    title: Rust core — alias projection, schema 22, and the alias-history query
    depends_on: []
    size: large
    description:
      "core: in the sase-core repo, add the alias trail/origin to AgentMetaWire and
      PromptStepMarkerWire, project each run's trail into a new normalized
      agent_artifact_model_aliases table under artifact-index schema 22 with a
      record_json re-projection migration that backfills legacy rows, and expose a
      bounded `query_agent_alias_history` PyO3 binding that returns per-alias groups
      with truncation counts and a directive-stripped prompt snippet."
  - id: wire
    title: Python wire mirror, facade call, and skew probes
    depends_on:
      - core
    size: medium
    description:
      "wire: mirror the new core contract on the Python side — alias trail/origin on the
      marker wires, a new agent_alias_history_wire module with its to_dict/from_dict
      helpers, a `query_agent_alias_history` facade function under the artifact-index
      operation lock — and extend the sase_core_rs validator so a stale wheel fails
      loudly instead of returning empty history."
  - id: config
    title: The per-alias history limit config field
    depends_on: []
    size: small
    description:
      "config: add `llm_provider.model_alias_history_limit` (default 10, minimum 1) to
      the JSON schema, the shipped default config commentary, and a validated accessor,
      and document it in the configuration and LLM references."
  - id: adapter
    title: Frontend-neutral alias-history adapter
    depends_on:
      - wire
      - config
    size: medium
    description:
      "adapter: add the presentation-neutral adapter that composes the core query with
      the configured limit, resolves ProjectSpec keys to configured project names,
      classifies each run's provenance into direct / via-another-alias / default /
      unrecorded, and returns typed view models with the group truncation state."
  - id: panel
    title: The Launch Control agent-history panel and its `H` keymap
    depends_on:
      - adapter
    size: large
    description:
      "panel: add the `H` binding and its context-aware footer entries to Launch
      Control, build the pop-up panel (title summary, grouped rows, two-line detail
      strip, footer) on the panel's existing worker/navigation/jump machinery, add
      full-prompt view, copy, load-more, refresh, and hidden-toggle actions, style it
      alongside the other Launch Control panels, and document it in the ACE reference."
  - id: visual
    title: PNG goldens for the history panel
    depends_on:
      - panel
    size: medium
    description:
      "visual: add deterministic fixtures and PNG snapshot coverage for the history
      panel's populated, grouped, truncated, legacy-provenance, and empty states, and
      confirm no existing Launch Control golden moves."
  - id: floor
    title: Raise the sase-core-rs dependency window
    depends_on:
      - panel
      - visual
    size: small
    description:
      "floor: after the sase-core release publishes, move both bounds of the
      sase-core-rs version window in pyproject.toml to the release carrying artifact
      index schema 22, and confirm the exhaustive gate passes against the published
      wheel."
  - id: verify
    title: Acceptance against real agent history
    depends_on:
      - provenance
      - floor
    size: small
    description:
      "verify: exercise the panel against the real machine-local artifact index — a
      legacy-only alias, a freshly launched alias, a bucket, a truncated alias, and an
      alias with no runs — and confirm the migration backfilled without a full rebuild."
proposed_by: bbugyi200.athena.03t
bead_id: sase-n8
create_time: 2026-09-09 19:50:42
---

- **PROMPT:**
  [prompts/202608/launch_control_alias_history.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/launch_control_alias_history.md)
- **BEAD:**
  [sase-n8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-n8/README.md)

# Plan: Agent history for a model alias in Launch Control

## Goal

Launch Control tells you what an alias points at _right now_. It cannot tell you what
the alias has actually been doing. This epic adds that: press `H` on an alias row and a
pop-up panel lists the previous SASE agents that ran on that alias, newest first,
bounded by a new configurable per-alias limit that defaults to 10.

Each row must answer three questions at a glance:

1. **What actually ran?** The concrete provider/model and reasoning effort that answered
   — not a re-resolution of today's config, which may have changed since.
2. **What was it asked to do?** A readable snippet of the launch prompt.
3. **How did this alias get chosen?** Directly via `%model:@<alias>`, indirectly through
   another alias that resolved to it, or by default because the prompt carried no
   `%model` directive at all.

Question 3 is the hard one and drives most of this plan: SASE does not record it today.

## Why this needs new launch-time data

`agent_meta.json` already carries `model`, `llm_provider`, `reasoning_effort`, and
`model_alias`. `model_alias` is only the **first hop** — the bare alias named by the
`%model` directive, or the alias named by `llm_provider.default_model` when there is no
directive. Two facts are therefore unrecoverable after the fact:

- **Indirection is lost.** A launch with `%model:@coder`, where `coder` is configured as
  `@large`, records `model_alias: "coder"`. Nothing ties that run to `@large`, so a
  `@large` history panel would silently omit it. Re-resolving `@coder` at display time
  is not a fix: config, temporary overrides, and provider disables all move, so
  re-resolution answers "where would this go today", not "where did it go".
- **Origin is lost.** A directive-driven `@large` and a no-directive default that landed
  on `@large` both write the same `model_alias: "large"`. The user's central ask — "was
  this by default because `%model` wasn't used?" — cannot be answered from stored data.

So the launch path must record two additive fields, and everything downstream reads
them. Runs that predate this epic keep only their first hop; the panel must present
those honestly as **unrecorded provenance** rather than guessing.

## Data contract (pinned; phases build against this)

These names are fixed here so `provenance` and `core` can proceed in parallel.

### New `agent_meta.json` / prompt-step marker fields

Both are additive and optional. Absence is meaningful and must never be normalized away.

| Field                | Type        | Meaning                                                                                                                                                                                     |
| -------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `model_alias_trail`  | `list[str]` | Ordered bare alias names traversed to reach the concrete target. Index 0 is the entry alias; the last entry is the final alias before a concrete model. Omitted when no alias was involved. |
| `model_alias_origin` | `str`       | How the entry alias entered the launch: `directive`, `default_model`, or `none`.                                                                                                            |

Invariants:

- When `model_alias_trail` is present and non-empty, `model_alias_trail[0]` equals the
  existing `model_alias`. `model_alias` keeps its current meaning and its current
  writers; nothing that reads it today changes behavior.
- `model_alias_origin` is `directive` when the prompt carried an explicit `%model`
  (including the epic-lander expressions, which reach the runner as a `%model`
  directive), `default_model` when the alias came from the `llm_provider.default_model`
  launch setting because the prompt had no `%model`, and `none` when the launch resolved
  a concrete model with no alias anywhere in the chain.
- A trail records **alias hops only**. Concrete targets, effort suffixes, and provider
  names never appear in it. A selector (`A | B` / `A || B`) contributes the alias that
  owns the selector plus any alias-valued member that was actually selected — never the
  members that were not chosen.
- A trail never repeats a name; alias cycles already fail closed during resolution.
- Reading code treats a missing `model_alias_origin` as **unrecorded**, not as any of
  the three known values.

### New artifact-index contract (sase-core, schema 21 → 22)

- `AgentMetaWire` and `PromptStepMarkerWire` gain `model_alias_trail: Vec<String>`
  (defaulting to empty) and `model_alias_origin: Option<String>`.
- `agent_artifacts` gains a `model_alias_origin TEXT` column.
- New normalized projection table, mirroring the existing `agent_artifact_aliases`
  lifecycle exactly (written on upsert, deleted on row delete, counted in status):

  ```sql
  CREATE TABLE IF NOT EXISTS agent_artifact_model_aliases (
      artifact_dir TEXT NOT NULL,
      alias        TEXT NOT NULL,
      position     INTEGER NOT NULL,
      PRIMARY KEY (artifact_dir, alias)
  );
  CREATE INDEX IF NOT EXISTS idx_agent_artifact_model_aliases_alias
      ON agent_artifact_model_aliases(alias, artifact_dir);
  ```

- A schema-22 migration re-projects the table from each row's existing `record_json`,
  following the `migrate_output_variable_projection_v21` pattern: use
  `agent_meta.model_alias_trail` when present, else fall back to the single-element
  trail `[agent_meta.model_alias]` when that is non-null, else project nothing. This
  backfills every historical run's first hop without a full filesystem rebuild.

### New binding: `query_agent_alias_history`

Query wire `AgentAliasHistoryQueryWire`:

| Field                  | Default  | Meaning                                                                         |
| ---------------------- | -------- | ------------------------------------------------------------------------------- |
| `aliases`              | required | Bare alias names to report on. Empty is an error, not an implicit "everything". |
| `limit_per_alias`      | `10`     | Maximum runs returned per alias. `0` means unlimited.                           |
| `include_hidden`       | `false`  | Include hidden runs.                                                            |
| `projects`             | `[]`     | Exact ProjectSpec keys; empty means every project.                              |
| `prompt_snippet_bytes` | `240`    | Bounded read budget per returned run; `0` skips prompt reads entirely.          |
| `freshness`            | `cached` | Reuses the existing index freshness enum.                                       |

Result wire `AgentAliasHistoryWire { schema_version, index_path, query, groups }` with
one `AgentAliasHistoryGroupWire` per requested alias — preserving request order, and
present even when it has no runs — carrying `alias`, a
`{limit, total_count, returned_count, truncated}` limit block, and `runs`.

Each `AgentAliasRunWire` carries: `artifact_dir`, `project_name`, `workflow_dir_name`,
`timestamp`, `agent_name`, `workflow_name`, `model`, `llm_provider`, `reasoning_effort`,
`model_alias`, `model_alias_origin`, `model_alias_trail`, `alias_position`, `status`,
`workflow_status`, `has_done_marker`, `hidden`, `started_at`, `finished_at`,
`retry_attempt`, `bead_id`, `cl_name`, `workspace_num`, `prompt_snippet`, and
`used_xprompts`.

`alias_position` is the queried alias's index in that run's trail and is the single
field the provenance chip is derived from — `0` means the alias was the entry point, and
any higher value means it was reached from `model_alias_trail[alias_position - 1]`.

## Panel design

The panel is a `ModalScreen` pushed from Launch Control and it deliberately reuses
Launch Control's own visual grammar — title / option list / two-line description strip /
context footer — so it reads as the same surface rather than a bolted-on window.

**Title** (two lines, matching `_title_text()`'s existing pattern, including the tan
ownership gutter for user-owned aliases):

```
Launch Control › ▌@large · Agent History
claude/opus @xhigh · 43 runs recorded · showing 10 · 31 done · 7 failed · 5 running
```

**Rows** — one selectable line per run, plus a non-selectable header row per alias when
more than one alias is being reported (bucket case), using Launch Control's existing
single-blank-row spacer convention:

```
 ✓  2h ago   sase-n7.land       sase   claude/opus @xhigh    direct
 ●  5h ago   03q--mon           sase   claude/opus @xhigh    via @coder
 ✗  1d ago   bobo.w3            bob    codex/gpt-5.6 @max    default
 ✓  3d ago   tmp_260813         sase   claude/sonnet @high   unrecorded
```

The provider/model badge reuses `provider_styles.py`, so provider theming is identical
to the alias rows the user just came from. Project cells render the configured
`PROJECT_NAME`, never the ProjectSpec key.

**Detail strip** for the highlighted run — the resolution trail is the centerpiece:

```
@coder → @large → claude/opus @xhigh · override active at launch
"Refactor the workspace-provider ownership module into focused siblings…"
sase · workspace #15 · bead sase-n7.3 · started 14:22 · ran 38m · #bd/work_phase_bead
```

For an unrecorded-provenance run the first line states plainly that the launch predates
provenance recording and shows only the known first hop — it never renders a speculative
arrow.

**Actions**: `j`/`k`/arrows/`Ctrl+N`/`Ctrl+P` navigate, `'` jumps (the existing adaptive
entry-jump mixin), `Enter` opens the full launch prompt, `y` copies the run's artifact
reference, `Ctrl+K` loads another page beyond the configured limit, `r` refetches with
`revalidate` freshness, `.` toggles hidden runs, `Esc`/`q` returns to Launch Control.

**Empty state**: a non-selectable row naming the alias. When the alias has no runs _and_
no run in the index carries a trail at all, the strip adds the one-time explanation that
provenance recording began with this feature, so a new user does not read an empty panel
as a bug.

## Scope boundaries

- **The limit bounds what is listed, not what is retained.** The user's phrasing was
  "remembering the last 10 agents"; implementing that as deletion would destroy agent
  artifacts that many other SASE surfaces depend on. The config field is a query bound,
  `Ctrl+K` pages past it, and the title always shows `N runs recorded` next to
  `showing M` so truncation is visible rather than silent. Artifact retention is
  untouched.
- **No new CLI subcommand.** The core query is frontend-neutral and reachable by any
  future frontend, but this epic ships only the TUI surface.
- **Alias resolution policy stays in Python.** Rust records and filters the trail Python
  computed; it never resolves an alias. This keeps the documented boundary intact —
  selector policy and override precedence remain Python-owned.
- **Existing `model_alias` semantics are frozen.** Every current reader keeps working
  untouched; the trail is strictly additive.

---

## Phase `provenance` — Record the alias resolution trail and its origin at launch

Give resolution a trail, then persist it everywhere the launch selection is recorded.

### Resolution

In `src/sase/llm_provider/model_alias_resolution.py`, add `alias_trail: tuple[str, ...]`
to `_ResolvedModelAlias` and accumulate a hop each time the resolver commits to a bare
alias name. Every branch of the inner `resolve()` closure that currently adds to `seen`
is a hop: the launch-override redirect, the temporary-override branch and both of its
provider-disable fallback paths, the configured/implicit target branch, the selector
branch (which contributes the owning alias plus the chosen member's own hops), and the
implicit-fallback-reference branch. Order matters and set membership does not, so
accumulate an ordered tuple rather than deriving one from `seen`.

Two traps to respect:

- Selector member resolution runs for **every** member to compute availability, but only
  the selected member's trail may survive. Build each member's trail independently and
  keep only `member_results[index]`'s.
- `fail()` must return an empty trail. A failed chain resolves to the input string, and
  attributing partial hops to a failed resolution would put a run under an alias it
  never used.

`resolve_model_alias()` keeps returning a bare string;
`resolve_model_alias_with_effort()` carries the trail through unchanged.

### Launch selection

In `src/sase/llm_provider/launch_selection.py`, add `alias_trail: tuple[str, ...]` and
`alias_origin: str` to `LaunchSelection`, and set the origin per branch: `directive`
when `directives.model` supplied the expression, `default_model` when the
`llm_provider.default_model` launch setting did, and `none` when the resolved chain
contained no alias hop at all. Assert-free consistency: an empty trail always pairs with
origin `none`, and a non-empty trail never pairs with `none`.

The default-alias branch resolves through the launch-model-setting helpers rather than
`directives.model`; thread the trail out of that path too so a no-directive launch gets
the full `@large → …` chain and not just the setting's first hop.

### Persistence

Add `model_alias_trail: list[str]` and `model_alias_origin: str | None` to
`AgentMetadataInputs` in `src/sase/axe/run_agent_directive_metadata.py`, write them in
`build_agent_meta()` under the same "omit when empty" convention the neighboring model
fields use, and add both keys to `preserved_agent_metadata()`'s durable set so a runner
re-exec cannot lose them (the list needs its own type guard — the existing loop only
preserves non-empty strings).

Then cover every writer:

- `src/sase/axe/run_agent_directives.py` — the non-consuming preview branch populates
  both fields from the preview selection; the preserved-metadata branch reuses the
  preserved values exactly and must not recompute, because a re-exec must not advance a
  pooled alias cursor or re-resolve against config that moved.
- `src/sase/xprompt/workflow_executor_steps_prompt.py` — the authoritative consuming
  resolution. This is the reconciliation point of record: extend both the prompt-step
  marker write and the anonymous-workflow `update_meta_fields` reconcile so the stored
  trail describes the selection that actually answered.
- `src/sase/xprompt/workflow_executor.py` — `_save_prompt_step_marker` gains the two
  fields and preserves them from an existing marker exactly as it already does for
  `model_alias`.
- `src/sase/axe/run_agent_exec_plan_accept.py` — follow-up launches rewrite
  `model_alias`; rewrite the trail and origin alongside it, and clear both when the
  follow-up has no alias, so a stale trail cannot outlive the alias it described.

### Verification

Unit tests for the resolver covering: a direct alias, a two-hop alias chain, a three-hop
chain, a round-robin selector whose selected member is itself an alias, an
ordered-fallback selector, a temporary override short-circuit, an override paused by a
provider disable that falls back through the underlying target, a launch alias override,
a concrete `%model:opus` (empty trail, origin `none`), a cycle, and a depth-limit
overflow (both empty trails). Launch-selection tests asserting the origin for the
directive and no-directive branches. Metadata tests asserting round-trip through build,
preserve, re-exec, workflow reconcile, and plan-accept follow-up.

---

## Phase `core` — Rust core: alias projection, schema 22, and the alias-history query

All work is in the linked `sase-core` repo (open it with `/sase_repo`), under
`crates/sase_core/src/agent_scan/` plus the `sase_core_py` binding crate.

### Wire and scanner

Add `model_alias_trail` (defaulting to an empty vector) and `model_alias_origin` to
`AgentMetaWire` and `PromptStepMarkerWire` in `agent_scan/wire.rs`. The marker wires are
a deliberately compact projection that drops unknown keys, so without this the new
metadata never reaches `record_json` and every other piece of this epic silently reads
empty history. Confirm the scanner round-trips both fields.

### Index

Bump `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` 21 → 22 and add:

- the `model_alias_origin` column on `agent_artifacts`, via the existing
  `ALTER TABLE … ADD COLUMN` helper;
- the `agent_artifact_model_aliases` table and its alias index from the pinned contract;
- writes in `upsert_record` and deletion in the row-delete path, mirroring
  `agent_artifact_aliases` exactly (delete-then-insert on upsert, so a re-indexed run
  cannot accumulate stale aliases);
- `agent_artifact_model_aliases_rows` on `AgentArtifactIndexStatusWire`;
- `migrate_model_alias_projection_v22`, which re-projects the table from `record_json`
  for every existing row using the pinned trail-then-first-hop fallback.

The migration is the reliability crux: it must be a pure re-projection with no
filesystem access, because it runs on ~7k rows during a normal ACE startup path.

### Query

Add `query_agent_alias_history` in `agent_scan/index.rs`, modeled on
`query_agent_output_variable_history`: one statement per requested alias joining the
projection table to `agent_artifacts`, ordered by `timestamp` descending, applying the
hidden and project filters, and using `limit_per_alias` for the returned window while a
companion `COUNT(*)` supplies `total_count` so truncation is reported rather than
guessed.

Read `raw_xprompt.md` for each **returned** row only (never the counted set), bounded by
`prompt_snippet_bytes`. Produce the snippet by skipping leading blank lines and leading
`%directive` / `#xprompt` lines, collapsing internal whitespace, and truncating on a
character boundary with an ellipsis. A prompt that is nothing but directives yields an
empty snippet — correct and expected, since `used_xprompts` already tells the frontend
what such a launch invoked. A missing or unreadable file yields `None` and never an
error.

Export the binding from the `sase_core_py` crate.

### Verification

Rust tests over a temporary index for: a single alias; multiple aliases in one call
including one with zero runs; the ordering and truncation counts; membership by a
non-entry trail position with the right `alias_position`; hidden and project filters;
legacy rows that carry only `model_alias`; the v21→v22 migration backfilling from
`record_json`; prompt-snippet directive stripping, truncation, and the missing-file
case; and empty `aliases` rejected as an error. Run `just check` from the sase-core repo
root — never `cargo test -p sase_core` alone, which skips the binding tests.

---

## Phase `wire` — Python wire mirror, facade call, and skew probes

Mirror the core contract exactly; a drift here surfaces as an empty panel rather than an
error, which is the worst possible failure mode.

- Add `model_alias_trail` and `model_alias_origin` to `AgentMetaWire` and
  `PromptStepMarkerWire` in `src/sase/core/agent_scan_wire_markers.py`, and reflect the
  new status counter in `src/sase/core/agent_scan_wire_records.py`.
- Add `src/sase/core/agent_alias_history_wire.py` with the query, group, limit, and run
  dataclasses plus `agent_alias_history_query_to_dict` /
  `agent_alias_history_from_dict`, following the tolerant rehydration convention
  (`known_field_kwargs`) so a newer writer never crashes an older reader.
- Add `query_agent_alias_history()` to `src/sase/core/agent_scan_facade.py`, holding the
  artifact-index operation lock exactly as the output-variable history query does.
- Extend `tools/validate_sase_core_rs` with a schema-22 probe so a stale installed wheel
  fails loudly at install/check time instead of quietly returning no history.

Verify with wire round-trip tests including unknown-key tolerance and missing-optional
defaults, and a facade test asserting the lock is taken and the payload is converted.

---

## Phase `config` — The per-alias history limit config field

Add `llm_provider.model_alias_history_limit`:

- `src/sase/config/sase.schema.json` — integer, `minimum: 1`, `default: 10`, in the
  `llm_provider` properties block beside the other launch scalars, described as the
  number of prior agent runs Launch Control lists per model alias before truncating.
- `src/sase/default_config.yml` — a commented entry in the `llm_provider` block, in the
  established comment-the-default style, stating plainly that it bounds what the panel
  lists and does not delete agent history.
- `src/sase/llm_provider/config.py` — `get_model_alias_history_limit()` following
  `get_configured_max_running_agents()`'s shape: read the merged config, accept only a
  true `int` at or above 1, and fall back to a module-level
  `DEFAULT_MODEL_ALIAS_HISTORY_LIMIT` for anything else. A bad config value must never
  break the panel.
- `docs/configuration.md` (the `llm_provider` table) and `docs/llms.md` (near the alias
  documentation) get the new row and a sentence on retention vs. display.

Verify with accessor tests for the default, a valid override, zero, a negative, a
non-integer, a bool (which `type(value) is int` must reject), and a missing section.

---

## Phase `adapter` — Frontend-neutral alias-history adapter

Add `src/sase/llm_provider/alias_history.py`: the seam between the core query and any
frontend, so provenance classification cannot drift between surfaces.

It exposes one load function taking the aliases to report on plus optional
limit/hidden/project/freshness overrides, and returning typed view models:

- It defaults `limit_per_alias` from `get_model_alias_history_limit()`.
- It maps each run's ProjectSpec key to the configured project name through
  `sase.project_display_names`, falling back to the key only when no name is known.
- It classifies provenance into exactly four cases from `alias_position` and
  `model_alias_origin`, and no others:

  | Case       | Condition                                     | Rendered as   |
  | ---------- | --------------------------------------------- | ------------- |
  | Direct     | `alias_position == 0`, origin `directive`     | `direct`      |
  | Default    | `alias_position == 0`, origin `default_model` | `default`     |
  | Indirect   | `alias_position > 0`                          | `via @<prev>` |
  | Unrecorded | origin missing/unknown                        | `unrecorded`  |

  An `alias_position > 0` run is indirect regardless of origin, because the entry alias
  is what the origin describes; the strip still names the origin when it is known.

- It preserves per-group truncation state and the requested/returned counts, and derives
  the status rollup (done / failed / running) the panel's title line shows.
- It computes a display duration from `started_at`/`finished_at` and leaves formatting
  to the frontend.

Unit tests cover each classification case, an unknown future origin string (must degrade
to unrecorded, never raise), display-name mapping including the unknown-project
fallback, truncation state, empty groups, and limit resolution from config.

---

## Phase `panel` — The Launch Control agent-history panel and its `H` keymap

### Entry point

Add `("H", "alias_history", "History")` to `ModelsPanel.BINDINGS`. Uppercase bindings
are established in this codebase and `H` does not collide with the existing lowercase
`h` (leave bucket). Panel-internal keys are intentionally hardcoded rather than
configuration-driven, matching `o`/`x`/`e`/`r`/`p`.

Implement `action_alias_history()` in a new `models_panel_history.py` mixin, resolving
the highlighted row to a set of aliases:

- an `AliasView` row → that one alias;
- a `LaunchModelSettingRow` whose snapshot has a referenced alias → that alias;
- a `BucketView` row → every member alias, so `H` is never dead at the top level, where
  buckets are most of what is selectable;
- the effort / runner-limit / threshold scalar rows, and a launch-model row pointing at
  a concrete model → a clear toast naming why there is no alias history, and no modal.

Add `H` to the context-aware footer strings in `models_panel_display.py` for exactly the
row kinds that support it, following the existing footer convention.

### The panel

New modules alongside the existing panel family: the modal itself, a rendering module
for row and detail-strip construction, and a state module holding the snapshot dataclass
and the thread-worker load function.

The modal composes on `OptionListNavigationMixin` and the adaptive entry-jump mixin, and
mirrors `ProviderRoutingModal`'s worker discipline: never load in a message handler,
always `run_worker(..., thread=True, exclusive=True, group=…)`, cancel outstanding
workers in `on_unmount`, refuse to close mid-write, and surface load failures as a
warning toast with the panel still usable. It opens immediately with a loading row so
the first paint never waits on the query.

Layout, rendering, and actions follow the **Panel design** section above. Reuse
`provider_styles.py` for the model badge and the existing ownership accent so the panel
is visually continuous with the rows it was opened from. Guard programmatic
`OptionList.highlighted` assignments with the synchronous flag pattern the sibling
panels already use.

`Ctrl+K` re-queries with a doubled `limit_per_alias` and keeps the highlighted run
selected; `r` re-queries with `revalidate` freshness; `.` toggles `include_hidden`. Each
re-query goes through the same single worker path — no second load code path.

Add `#alias-history-*` rules to `src/sase/ace/tui/styles.tcss` next to the
`#provider-routing-*` block, sharing its width and chrome budget.

Document the panel in `docs/ace.md`'s Launch Control section: a paragraph on what the
panel answers, the row anatomy, the four provenance labels including what `unrecorded`
means for pre-existing runs, the keymap table addition for `H`, and the panel's own key
table.

### Verification

Textual tests driving the real modal: opening from an alias row, a launch-setting row, a
bucket row (grouped output), and a rejected scalar row; row content for each provenance
case; the detail strip's trail rendering and its unrecorded variant; truncation and
`Ctrl+K` paging; refresh; hidden toggle; the empty state and its first-run explanation;
worker failure surfacing as a toast; and `Esc` returning to a Launch Control whose own
state is unchanged. Add the footer-content assertions to the existing keymaps test.

---

## Phase `visual` — PNG goldens for the history panel

Add fixtures under `tests/ace/tui/visual/` beside
`_ace_models_panel_png_snapshot_fixtures.py`, with a frozen clock so relative times
(`2h ago`) are deterministic, and PNG snapshot coverage for: a populated single-alias
panel showing all four provenance labels; a grouped bucket panel; a truncated panel with
its `showing M of N` title; a legacy-only panel where every run is unrecorded; and the
empty panel.

No existing Launch Control golden may move. If one does, the `H` binding leaked into a
shared footer or the title layout, and that is a regression to fix rather than a golden
to accept. Run `just test-visual` and accept only the new goldens with
`--sase-update-visual-snapshots`.

---

## Phase `floor` — Raise the `sase-core-rs` dependency window

Every preceding phase works in a development install because `just install` builds the
extension from the workspace's sase-core checkout. Released installs resolve the
published wheel, so `pyproject.toml` must move.

Adding public wire fields and a new binding is a minor-version event for sase-core, so
**both** bounds of the constraint currently reading `sase-core-rs>=0.27.11,<0.28.0` need
to move — raising only the floor would leave the new release outside the window. Read
the version release-plz actually published for the `core` phase and set the window to
that release's minor series. Land it as its own
`build(deps): raise sase-core-rs floor to <version>` commit, matching the prior floor
bumps in this repo's history.

If the core release has not published yet, do not guess a version and do not proceed —
report the block. A wrong window either fails to install or admits a wheel whose index
lacks schema 22, which would present as an empty history panel rather than an error.

Then run `just check-full` through `/sase_monitor` and confirm the new schema-22 probe
in `tools/validate_sase_core_rs` passes against the published wheel.

---

## Phase `verify` — Acceptance against real agent history

Exercise the shipped panel against the real machine-local artifact index, which holds
thousands of pre-existing runs and is the only place the migration's backfill can be
judged:

1. Confirm the v22 migration ran on the existing index without a full rebuild, and that
   `agent_artifact_model_aliases` is populated for historical rows that carry
   `model_alias`.
2. Open `H` on a built-in size alias with substantial legacy history: rows appear, every
   one labeled `unrecorded`, and the strip explains why rather than inventing a trail.
3. Launch one agent with `%model:@<alias>` and one with no `%model` directive, then
   reopen the panel: the first shows `direct`, the second shows `default`, and both
   detail strips render the full trail.
4. If a chained custom alias exists (or after adding one), launch through it and confirm
   the run appears under the **target** alias labeled `via @<entry>`.
5. Open `H` on a custom bucket and confirm grouped output with per-alias limits.
6. Confirm a truncated alias reports `showing M of N` and that `Ctrl+K` extends it.
7. Open `H` on an alias with no runs and on a scalar setting row; confirm the empty
   state and the toast respectively.
8. Capture a `SASE_TUI_PERF=1` sample while opening and paging the panel to confirm no
   key-to-paint regression on the Launch Control path.

Report any deviation as a blocking finding rather than adjusting the plan's contract.
