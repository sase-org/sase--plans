---
tier: epic
title:
  "sase tool in the TUI: live ⚒ chips, the ⚒ Runs card, and the Admin Center Tools pane"
goal: "A live ToolRun shows on the row that owns it with its stage progress, and turns
  red when it goes silent. The selected node's header says whether its latest check
  added NEW failures. The Tools deck gains a ⚒ Runs card with a stage waterfall, triage
  items, and a log tail. LLM Calls, the slow-tool list, and monitor/proc Context cards
  link to the run instead of copying it. An Admin Center Tools pane covers project-wide
  runs, failure groups, the catalog, stopping a run, starting a named tool, and the -H
  settlement notification. Everything reads a slim sase-core projection and never
  reconciles or shells out on a UI path.

  "
phases:
  - id: core-glance
    title: Live glance, node summaries, brief lists, and the verdict bucket in sase-core
    depends_on: []
    size: medium
    description: "core-glance: add the fingerprint-free tool_run_live_glance,
      tool_run_briefs and tool_run_node_summaries projections, one shared verdict-bucket
      helper, the silent threshold constant, and agent/owner indexes, with PyO3 bindings
      and round-trip tests.

      "
  - id: core-run-detail
    title: Per-run detail projection with stage timeline and witness counts in sase-core
    depends_on:
      - core-glance
    size: medium
    description: "core-run-detail: add tool_run_detail, which returns one run's brief,
      safe argv, millisecond-normalized stages, the reference run's expected stages,
      triage items with cross-run and cross-agent witness counts, child runs, log
      metadata, and pruning facts, with its binding.

      "
  - id: tool-run-adapter
    title:
      Python adapter, state vocabulary, beta flag, shared log tail, and the chop glyph
      move
    depends_on:
      - core-glance
    size: medium
    description: "tool-run-adapter: move the core pin, add typed Python adapters and
      binding registrations, the ToolRun view vocabulary, the ace_tool_runs beta flag, a
      shared pure log-tail helper used by the CLI, and move the chop link-trail icon
      from ⚒ to ⏲.

      "
  - id: glance-row-chips
    title: ToolRun glance snapshot service and live-only ⚒ row chips
    depends_on:
      - tool-run-adapter
    size: medium
    description: "glance-row-chips: build the TUI ToolRun snapshot service (surface
      token, live drift probe, coalesced worker load, attribution), then render the live
      and silent ⚒ row chips with session inheritance, minute ticking, render-cache
      keys, and row patching behind ace_tool_runs.

      "
  - id: header-chip
    title: Selection-scoped ⚒ header chip, Tool runs field, and copyable run ids
    depends_on:
      - glance-row-chips
    size: medium
    description: "header-chip: add the node-summary loader and LRU, a tool-runs
      detail-header lane, the compact header chip for live and settled verdicts, the
      expanded Tool runs field, and a copy-mode target for the run id.

      "
  - id: tools-deck-cards
    title: Tools becomes a two-card deck with ⚒ Runs first
    depends_on:
      - header-chip
    size: medium
    description: "tools-deck-cards: give the Tools deck two card hosts (a
      ToolRunsDeckView card document and the unchanged LLM Calls panel), with per-card
      availability, the sticky default-card rule, a switcher status segment, active-card
      detail levels, availability for monitors and procs, and outcome-line run blocks.

      "
  - id: runs-card-anatomy
    title: Full ⚒ Runs block anatomy - waterfall, triage, log tail, and honest absence
    depends_on:
      - tools-deck-cards
      - core-run-detail
    size: medium
    description: "runs-card-anatomy: move the core pin past the detail projection and
      render each run block's outcome and context lines, stage waterfall, triage items
      with witness counts, child runs, bounded log tail, three detail levels,
      retention-honest absence, and v/% integration.

      "
  - id: runs-card-live
    title: Live run blocks - in-flight stages, pending stages, follow and hold
    depends_on:
      - runs-card-anatomy
    size: medium
    description: "runs-card-live: make a live run's block progress in place with a pure
      1 Hz elapsed repaint, a detail refetch only on glance drift, pending stages from
      the reference run, silent state, follow-versus-hold on new runs, and an in-place
      settle transition.

      "
  - id: run-links
    title: Link LLM Calls, the slow-tool list, and Context cards to the run
    depends_on:
      - runs-card-anatomy
    size: medium
    description: "run-links: add a verdict suffix and a jump to the run's block on LLM
      Calls rows that ran sase tool run, a live-stage or verdict suffix on the Main deck
      slow-tool list, and a Tool run row on monitor and named-proc Context cards.

      "
  - id: admin-tools-pane
    title: Admin Center Tools pane with Runs, Failures, and Catalog views
    depends_on:
      - runs-card-anatomy
    size: medium
    description: "admin-tools-pane: add the Tools tab to the Admin Center, with a Runs
      list and detail that reuse the Runs block renderer, a Failures signature view, a
      read-only Catalog view, current-project filtering, jump-to-agent, and a generic
      deep-link focus target.

      "
  - id: tool-run-actions
    title:
      Stop, run from the catalog, OpenToolRun notifications, Procs decode, and palette
    depends_on:
      - admin-tools-pane
    size: medium
    description: "tool-run-actions: add a confirmed stop as a durable proc with a typed
      result, a catalog -H launch through a session worker, the OpenToolRun notification
      action (sase-189), the ⚒ decode for tool-run procs, and context-aware palette
      commands.

      "
  - id: cutover
    title: Remove ace_tool_runs, add goldens, inspect live, and bench
    depends_on:
      - runs-card-live
      - run-links
      - tool-run-actions
    size: medium
    description: "cutover: bench j/k and idle ticks with the flag on and off, delete the
      flag's Off branches and close its flag bead, generate and inspect deterministic
      goldens for every surface, and inspect live captures of real runs.

      "
  - id: docs
    title: User docs for ToolRuns in the TUI
    depends_on:
      - cutover
    size: small
    description:
      "docs: document the shipped surfaces, vocabulary, keys, and Admin Center tab
      renumbering in docs/ace.md, docs/tool.md, and docs/configuration.md, and record
      ready-to-apply glossary text as a PROPOSED FOLLOW-UP note."
proposed_by: bbugyi200.athena.0tc
create_time: 2026-09-27 18:32:30
status: wip
---

- **PROMPT:**
  [prompts/202609/tool_runs_tui_surfaces.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/tool_runs_tui_surfaces.md)

# Plan: `sase tool` in the TUI (roadmap E5, "Surfaces")

## 1. Context

`sase tool run` writes a durable, machine-local ToolRun ledger
(`~/.sase/tools/runs.sqlite`): catalog identity, lifecycle, `agent`,
`owner_kind`/`owner_id`, `parent_run_id`, stages, load samples, fingerprints, triage
items, and receipts. Agents run `check` through it constantly, but **the TUI shows none
of it**. An agent sits in `(RUNNING)` for minutes (inline `check` p50 3m55s, p90 15m39s)
with nothing saying it is on stage 7 of 11, and its verdict (NEW vs KNOWN failures) is
buried in reply prose.

**Research.** Read the consolidated report with
`sase artifact read research:202609/sase_tool_tui_integration/sase_tool_tui_integration.md "<why>"`
and the E5 section of
`sase artifact read research:202609/sase_tool_epic_roadmap/sase_tool_epic_roadmap.md "<why>"`.
The report holds the ledger evidence (§2), the ranked user jobs (§3), the
five-researcher disagreements and their resolutions (§4), and the reopen table (§8).
This plan adopts its §10 recommendation. Where they differ, this plan wins; §2 marks
each refinement.

Key evidence that shapes the design:

| Measure                     | Value                                                            | Consequence                                           |
| --------------------------- | ---------------------------------------------------------------- | ----------------------------------------------------- |
| Runs per agent (7 days)     | p50 **1**, p90 3                                                 | A count on rows is noise; never show one              |
| Live runs at once           | **1.38** average, peak 7                                         | A live-only chip is rare enough to mean something     |
| Each agent's latest `check` | NEW 41%, signaled 25%, UNKNOWN 17%, KNOWN-only 8%, **pass 7%**   | A persistent verdict mark would light up ~93% of rows |
| Signaled runs               | p50 **539 s** (a caller-side ~9 min kill)                        | Show "killed at 9m", which is not "failed"            |
| Unsettled zombie            | one monitor-owned `check` "running" 52 h with no sample for 52 h | Heartbeat age exposes zombies without the TUI writing |
| `tool_run_list` payload     | 50 rows ≈ 532 KB (fingerprints)                                  | The TUI needs a slim projection                       |

**Verified on master `24e80d42ef` while planning** (beyond the report):

- **FINAL is always on.** `2d8f2f0566` removed `ace_final_deck`. FINAL is still the
  precedent to copy: a `CardDocumentView` subclass with an off-thread loader and LRU,
  stale-generation rejection, a stat-only signature, a sticky preferred card, a live
  ticker, and run blocks (`src/sase/ace/tui/widgets/decks/final/`).
- **Tools is not a card-document deck.** `CARD_DOCUMENT_DECKS = (MAIN, FINAL)`
  (`widgets/decks/card_documents.py`). The Tools card tab is hard-coded
  (`panel_chrome.py` `CardTab("llm-calls", "LLM Calls")`). `panel_blocks.py` wrappers
  are written "FINAL, else MAIN", so a new document deck would silently route to Main.
  The footer card count is hard-coded to 1 (`actions/agents/_display_detail_footer.py`).
  `LLMCallsVisibilityChanged` sets availability for the **whole** Tools deck from LLM
  calls alone (`panel_files.py`). h/l/H/L route to LLM Calls whenever the focused deck
  is Tools (`actions/agents/_folding.py`,
  `_agent_detail_deck_targets.focused_tools_view`).
- **Attribution values.** `runs.agent` is the concrete turn name (`0t9--code`, `0tb--1`,
  `sase-1b2.20`). A monitor-owned run also carries the _starter's_ `agent`, plus
  `owner_kind=monitor`, `owner_id=<monitor_id>`. `SASE_AGENT_NAME` can name a session
  container when members replace each other in one process
  (`src/sase/agent/identity.py`), so an agent value may name a container.
- **Store facts.** WAL journal mode (stat both `runs.sqlite` and `runs.sqlite-wal`).
  There is no heartbeat column: the newest sample's `observed_ts` (every 10 s,
  `src/sase/tool/sample.py`) is the only freshness signal. **Stage stamps are epoch
  milliseconds; run, event, and sample stamps are seconds.** The retention bug this
  causes is filed separately as `sase-1bo` and is not part of this epic. Request wires
  use `deny_unknown_fields`, so new filters need new bindings, not new fields on
  `ToolRunListRequestWire`. `tool_run_summary`'s `last` ignores `definition_digest`. The
  verdict request is built in three places today (`triage.rs` twice, `receipt.rs`).
- **Every CLI read reconciles.** `reconcile_unsettled_tool_runs()`
  (`src/sase/tool/liveness.py`) writes to the store and can publish notifications.
  `query.py`, `control.py`, `failures.py`, `receipt_*.py` and `sase tool list` all call
  it first.
- **`-H` refuses inside ACE procs.** `handoff_launch.py` exits 2 when `SASE_PROC_ID`
  names an `origin == "ace"` proc, which covers `_submit_durable_proc` and `:` Command
  Line procs.
- **`sase tool stop` writes no typed operation result**, so a durable-proc submission
  would complete as "did not write a typed result".
- **`src/sase/ace/tui/tools/`** is a stale directory holding only `__pycache__`, left
  over from the LLM Calls rename. Do not name a new package `tools`.
- **Admin Center tabs** are tested to be alphabetical and numbered
  (`tests/ace/tui/test_config_center_tabs.py`). Tools becomes 7 and Updates 8.
- **`⚒` is used twice**, both for the chop/Services link-trail icon
  (`actions/link_trail.py` `_AXE_ICON`, `relations/link_subject.py` `_CHOP_ICON`). `⏲`
  and `⛏` are unused. All are single-cell and covered by bundled fonts.
- **Related beads.** `sase-189` (the notification action) is implemented by
  `tool-run-actions`. `sase-1bi` (monitor scan wire drops `tool_run_id`) is independent:
  this epic joins monitors from the ledger side (`owner_id` = `monitor_id`), so it is
  not a prerequisite. `sase-1b1` (deck views) and `sase-124` (Agents freshness) are
  still landing child epics that touch deck chrome and refresh paths (§3.14).

**Rust boundary** (core memory `rust_core_backend_boundary`):

| Concern                                                                                                                                                                                                  | Owner                                                                                                                                                 |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| What a run _is_ right now: live stage, progress against a reference run, typical duration, last activity, silent threshold, verdict bucket and class counts, witness counts, node scoping by agent/owner | `sase-core` (`tool_run` projections). A web, Telegram, or editor frontend must agree with the TUI                                                     |
| Log-tail reading and retention-honest availability                                                                                                                                                       | Python `src/sase/tool/logs.py`, one pure helper shared by `sase tool show` and the TUI (the logic already lives in Python; moving it is out of scope) |
| Glyphs, colors, words, chips, cards, panes, keys, loaders, caches                                                                                                                                        | Python TUI (presentation only)                                                                                                                        |

## 2. Binding decisions

The report's §9 questions are settled here. **(refined)** marks where this plan departs
from the report, with the reason. The user can override any of them during plan review.

| #   | Decision                                                                                                                                                                                                                                                                                                                                               | Why                                                                                                                                                                                                              |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | **One noun, one glyph.** `⚒` (U+2692) means ToolRun everywhere: row, header, card, pane, notification icon, Procs cell. The chop/Services link-trail icon moves to `⏲` (U+23F2), because chops are scheduled jobs                                                                                                                                      | A single visual identity makes every surface point at the same thing. `🔨` is two cells. The move is unconditional (it is not flag-gated)                                                                        |
| D2  | **Four zoom levels:** glance (live-only row chip), selection (header chip), diagnose (`⚒ Runs` card), project/machine (Admin Center Tools pane)                                                                                                                                                                                                        | Each job in the report's §3 lives on exactly one surface                                                                                                                                                         |
| D3  | **No counts on rows, and no persistent verdict marks on rows**, ever in v1                                                                                                                                                                                                                                                                             | p50 is one run per agent, and 93% of latest checks are not "pass". See §6 for the reopen condition                                                                                                               |
| D4  | **Silent is not lost.** A run is _silent_ when `now − last_activity_ts ≥ silent_after_s` (core constant, 60 s = six missed samples). The TUI says "silent Nm/Nh/Nd" in red and never claims an outcome, never reconciles, and never settles anything                                                                                                   | Keystroke and render paths are read-only. Only the owner or a reconcile may settle a run                                                                                                                         |
| D5  | **The verdict bucket is computed in core** (§3.2): `running`, `pass`, `new_failures`, `known_only`, `undetermined`, `stopped`, `killed`, `lost`                                                                                                                                                                                                        | The CLI, TUI, and future frontends can never disagree                                                                                                                                                            |
| D6  | **(refined)** Every glance time is honest and cheap. Row chips use minute resolution. The header uses seconds only if the header already repaints on the 1 s tick, and minutes otherwise. Settled times are absolute (`16:21`), never "6m ago"                                                                                                         | A relative time that doesn't tick is a lie. No new repaint loops                                                                                                                                                 |
| D7  | **Tools deck identity is unchanged** (`λ TOOLS`, key `t`, accent `#87D7FF`). Cards: `⚒ Runs` first, then `LLM Calls`. The Runs card exists only when the node has at least one run. **Default card:** sticky preference → Runs → LLM Calls. **Tools is paged-only**: it never spreads, `P` stays a no-op, and its view policy stays AUTO, as for FINAL | Runs are the short, high-signal card, and stickiness keeps anyone who prefers LLM Calls there. The two cards have unrelated content and scroll                                                                   |
| D8  | **(refined)** The switcher segment is `tools ⚒N M` (N runs, M calls), or `tools ⚒N` when there are no calls. The `⚒N` part is bold accent while a run is live and red while silent. It never uses `" · "` inside a segment                                                                                                                             | The switcher joins decks with `" · "`, so the report's `tools ⚒ 2 · 57 calls` would read as two decks. A fixed-shape segment never jumps width                                                                   |
| D9  | **One block per run** on the Runs card, landing on the newest. `(`/`)` step. A reader on the newest block follows new arrivals; a reader who moved back holds position and sees an arrival dot. A nested run (`parent_run_id` on the same card) renders as a `↳` line inside its parent block, never as its own block                                  | Reuses deck → card → block. Blocks never nest                                                                                                                                                                    |
| D10 | **Attribution is by node kind, never by name-prefix guessing** (§3.3). The live chip goes to the owner first (monitor or proc row), otherwise the agent turn. History (card, header) shows both relationships and dedupes by `run_id`                                                                                                                  | A monitor-owned run belongs to the monitor row. The starter still did start it                                                                                                                                   |
| D11 | **(refined)** **No new single-letter Agents-tab keys.** Runs uses existing idioms: `p t` / `Ctrl+J/K` to reach it, `(`/`)` for blocks, h/l/H/L for detail levels (routed by active card), **`v` hint mode** gains `⚒ run log` targets that open the log in the pager, **`%` copy mode** gains a "tool run id" target, and stop is a palette command    | The report proposed card-local `v`/`y` keys. `v` is already hint-mode "view in pager" and `%` is already copy mode; extending those idioms instead of adding card-local letters keeps the Agents tab predictable |
| D12 | **Stopping is allowed for every owner kind**, with consequence copy and Cancel focused (§3.11). It runs as a durable proc `sase tool stop RUN -j`, which gains a typed result                                                                                                                                                                          | It is the only fix for a silent run short of killing the agent. Durable ops appear in the Procs tab and survive quit                                                                                             |
| D13 | **Catalog `r`** hands off `sase tool run -H <tool>` at the current project's primary checkout root through a session worker that calls the hand-off launcher in-process, after a confirm that shows argv and root. It never uses `_submit_durable_proc` or the `:` proc path                                                                           | `-H` is the designed human hand-off (fail-closed reservation, one owning proc), and it refuses inside ACE-origin procs by design                                                                                 |
| D14 | **No rerun, and no reconcile button** in v1                                                                                                                                                                                                                                                                                                            | There is no CLI rerun contract, rerunning in an agent's workspace would race the agent, and reconcile belongs to a backend job (§6)                                                                              |
| D15 | **Remote rows** get no chip and no Runs card. The Tools empty state, if shown, says "ToolRun history lives on `<machine>`" and never shows zero                                                                                                                                                                                                        | The ledger is machine-local, so absence must never read as "no runs"                                                                                                                                             |
| D16 | **Privacy:** only `display_argv` reaches the TUI. `private_argv` never leaves core (it already doesn't; tests pin it)                                                                                                                                                                                                                                  | Redaction must not regress through a new surface                                                                                                                                                                 |
| D17 | **One `ace_tool_runs` beta flag** gates every user-visible surface until `cutover` deletes it (§3.13)                                                                                                                                                                                                                                                  | Phases land piecemeal                                                                                                                                                                                            |
| D18 | **Admin Center Tools pane**, filtered to the current project by default (`A` toggles all projects), with Runs / Failures / Catalog views                                                                                                                                                                                                               | The ledger is machine-wide, and only 21 of 1,250 runs have no agent. A 4th top-level tab is not warranted                                                                                                        |

Naming hygiene: say **"tool run"** or **"ToolRun"**. Never say "tool call" for a
ToolRun, and never put a ToolRun count next to an LLM-call count without its `⚒`.
"Stage" means a ToolRun stage, not a workflow step.

## 3. Design specification (all phases implement against this)

### 3.1 Vocabulary

| Term              | Meaning                                                                                                                                                                                                                 |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Live run**      | A run in state `created` or `running`                                                                                                                                                                                   |
| **Silent run**    | A live run with `now − last_activity_ts ≥ silent_after_s` (D4)                                                                                                                                                          |
| **Last activity** | `max(created_ts, running_ts, newest sample observed_ts, newest stage start or finish converted to seconds)`                                                                                                             |
| **Reference run** | The newest run with the same `project`, `tool_name`, and `definition_digest` whose state is `succeeded` or `failed` and whose stages are all complete. It supplies the expected stage count and the expected stage list |
| **Position**      | `stages_done + 1` while a stage is in flight, otherwise `stages_done`. Progress `k/n` shows only when `k ≤ n`; otherwise the chip falls back to elapsed                                                                 |
| **Node**          | One Agents-tab sase node: an agent turn, session container, monitor turn, named proc, or clan                                                                                                                           |
| **Selector**      | The core query key for a node: `{key, agents[], owners[], since_ts}` (§3.3)                                                                                                                                             |
| **Label**         | `tool_name`, or for ad-hoc runs the basename of `display_argv[0]`, capped at 12 characters                                                                                                                              |

### 3.2 State vocabulary

Every state shows glyph + word + color together, so goldens and colorblind users never
depend on color. The mapping lives in one pure module,
`src/sase/tool/view_vocabulary.py` (created by `tool-run-adapter`). Colors reuse the
finalizer palette (`src/sase/finalizers/view_vocabulary.py`) where one exists.

**Bucket precedence (core, first match wins):**

1. state `created` or `running` → `running`
2. `terminal_cause` in `stop_requested`, `interrupt` → `stopped`
3. `terminal_cause` in `signal`, `timeout` → `killed`
4. state `lost`, or `terminal_cause` in `owner_lost`, `wrapper_lost`, `launch_failed` →
   `lost`
5. triage verdict `pass` → `pass`; `new_failures` → `new_failures`; `no_new_failures` →
   `known_only`; `undetermined` → `undetermined`

| Bucket                        | Glyph          | Words (examples)                                                                        | Color                                                           |
| ----------------------------- | -------------- | --------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| running                       | `⚒` + progress | `check 7/11`, `check 2m`, `check starting` (created), `check stopping` (stop requested) | bold `#87D7FF` (Tools accent, so the chip points at its detail) |
| running, silent (TUI-derived) | `⚒⚠`           | `check silent 4m`                                                                       | bold `#FF5F5F`                                                  |
| pass                          | `✓`            | `pass`                                                                                  | `#5FD75F`                                                       |
| new_failures                  | `✗`            | `3 NEW · 1 KNOWN`                                                                       | `#FF5F5F`                                                       |
| known_only                    | `≈`            | `known only · 2 KNOWN`                                                                  | `#87AF87` (calm: the known red, not yours)                      |
| undetermined                  | `?`            | `2 UNKNOWN`, or `untriaged` when the reason is `not_triaged`                            | `#FFAF5F`                                                       |
| killed                        | `⊘`            | `killed at 9m00s · signal`, `timed out at 30m00s`                                       | `#D75FFF` (the finalizer "didn't get to finish" violet)         |
| stopped                       | `⊘`            | `stopped at 1m02s`                                                                      | dim                                                             |
| lost                          | `⊘`            | `lost · wrapper_lost`                                                                   | dim `#FF5F5F`                                                   |

**Severity** (used to pick one chip among several tools, and for session inheritance of
settled verdicts): new_failures > undetermined > killed > lost > stopped > known_only >
pass. Live always outranks settled, and silent outranks live.

Silent and elapsed durations format as `<1m`, `Nm` (under 60 m), `Nh` (under 72 h),
`Nd`. Stage and run durations use the CLI's `format_duration_ms`
(`src/sase/tool/render.py`).

### 3.3 Attribution and scope

The TUI maps each node to a selector; core matches a run to a selector when
`agent ∈ agents` **or** `(owner_kind, owner_id) ∈ owners`, **and**
`created_ts ≥ since_ts`. `since_ts` is the node's start time minus 60 s when the node
has one, which guards against a reused name.

| Node                              | Selector                                                                                  | Live row chip                                                          | Runs card and header                                                                        |
| --------------------------------- | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Agent turn                        | `agents=[agent_name]`                                                                     | Runs with this `agent` **and no owner**                                | All runs with this `agent`. Handed-off ones are labeled `→ monitor <name>` or `→ proc <id>` |
| Session container                 | Union of member selectors, plus its own name (a run's `agent` can name the container, §1) | The most severe live chip of its members (silent > live), then its own | Union, deduped by `run_id`, each block labeled with its turn                                |
| Monitor turn                      | `owners=[(monitor, monitor_id)]`                                                          | Its owned live runs                                                    | Owned runs, annotated with the starter's `agent`                                            |
| Named proc                        | `owners=[(proc, proc_id)]`                                                                | Its owned live runs                                                    | Owned runs                                                                                  |
| Clan container, tribe             | None in v1                                                                                | None (members carry their own)                                         | No Runs card. The Tools empty state points at Admin Center → Tools                          |
| Remote row (`fleet_origin_alias`) | None                                                                                      | None                                                                   | None. The empty state says "ToolRun history lives on `<machine>`" (D15)                     |

Glance attribution for the chip is exclusive: owner first, then an exact match on a
concrete turn's `agent_name`, then an exact match on a session container's name. A live
run whose parent run is also live on the same node folds into its parent (no `+N`). The
pure resolver lives in `src/sase/ace/tui/tool_runs/attribution.py`.

### 3.4 Core projections (contracts C1–C4)

All live in a new `sase_core::tool_run` projection module (for example
`tool_run/projection/`), are read-only (`with_read_store`), and select **lean columns in
one statement per table**. They never select `fingerprint_*_json`, `private_argv_json`,
`launch_envelope_json`, or `launcher_json`, and never call the per-row `load_run`.
Stage, sample, and triage lookups are batched by `run_id IN (…)`. A missing store
returns an empty result with `store_exists: false`. Missing optional columns and triage
tables are tolerated the way `load_run` and `triage_tables_present` already are. Request
wires use `deny_unknown_fields`; result wires are lenient. The wire schema version stays
`TOOL_RUN_WIRE_SCHEMA_VERSION` (1) because these are new wires, not changed ones.

**C0. Shared helpers.**

- `verdict_summary_for_runs(conn, run_ids) -> HashMap<run_id, ToolRunVerdictSummaryWire>`
  builds the verdict through the existing `tool_run_triage_verdict` logic once per run
  and applies §3.2's bucket precedence. `ToolRunVerdictSummaryWire` is
  `{bucket, verdict?, failure_kind?, reasons[], new, known, flaky, unknown, unlabeled}`,
  with `ToolRunVerdictBucketWire` as a snake_case enum. The two store-side copies
  (`triage.rs` show path, `receipt.rs::load_triage_facts`) move onto the same
  request-building helper only where existing tests prove their outputs stay identical.
  Otherwise, leave them and record a PROPOSED FOLLOW-UP.
- `pub const TOOL_RUN_SILENT_AFTER_SECONDS: i64 = 60;`, echoed in every glance result as
  `silent_after_s`.
- Indexes, all `CREATE INDEX IF NOT EXISTS` in `SCHEMA_SQL`:
  `runs(agent, created_ts DESC)`, `runs(owner_kind, owner_id, created_ts DESC)`, and
  `runs(project, tool_name, definition_digest, created_ts DESC)`. They appear after the
  next write open. Queries must stay correct, and only slower, before then.

**C1. `tool_run_live_glance`** — every unsettled run on the machine.

- Request: `{schema_version, now_ts?}`.
- Result:
  `{schema_version, store_exists, last_write_ts?, silent_after_s, truncated, runs: [ToolRunGlanceWire], diagnostics[]}`.
  It is capped at 200 runs, newest first, and sets `truncated` beyond that, so a pile of
  unreconciled zombies stays bounded.
- `ToolRunGlanceWire`:
  - identity: `run_id`, `tool_name?`, `label`, `state`, `launch_mode`
  - attribution: `project?`, `agent?`, `workspace?`, `bead?`, `owner_kind?`,
    `owner_id?`, `parent_run_id?`
  - timing: `created_ts`, `running_ts?`, `last_activity_ts`
  - progress: `current_stage?: {description, started_ms}`, `stages_done`,
    `stages_expected?`, `reference_run_id?`
  - typical: `typical_ms?`, `typical_samples` (the same median rule as `summarize`)
  - `stop_requested`

**C2. `tool_run_briefs`** — the lean filtered list the Admin pane uses.

- Request:
  `{schema_version, project?, tool?, states[], agents[], owners[], since_ts?, limit (1..=500, default 100), cursor?}`.
- Result:
  `{schema_version, store_exists, runs: [ToolRunBriefWire], next_cursor?, diagnostics[]}`.
- `ToolRunBriefWire`:
  - identity: `run_id`, `tool_name?`, `label`, `state`, `launch_mode`
  - outcome: `exit_code?`, `signal?`, `terminal_cause?`
  - attribution: `project?`, `agent?`, `workspace?`, `bead?`, `owner_kind?`,
    `owner_id?`, `parent_run_id?`
  - timing: `created_ts`, `running_ts?`, `settled_ts?`, `duration_ms?`, `typical_ms?`
  - `verdict: ToolRunVerdictSummaryWire`
  - flags: `detail_pruned`, `stop_requested`

**C3. `tool_run_node_summaries`** — per-node history for the selected node.

- Request:
  `{schema_version, nodes: [ToolRunNodeSelectorWire] (1..=64), per_node_limit (1..=100, default 20)}`.
  `ToolRunNodeSelectorWire` is `{key, agents[], owners: [{kind, id}], since_ts?}`.
- Result:
  `{schema_version, store_exists, silent_after_s, nodes: [ToolRunNodeSummaryWire], diagnostics[]}`.
- `ToolRunNodeSummaryWire`:
  `{key, total_runs, truncated, live: [ToolRunGlanceWire], latest_by_tool: [ToolRunBriefWire], runs: [ToolRunBriefWire]}`.
  `runs` is newest first and bounded. `latest_by_tool` has one entry per label.

**C4. `tool_run_detail`** (phase `core-run-detail`) — one run, for one card block.

- Request:
  `{schema_version, run_id, witness_window_days (default 7, max 30), item_limit (default 50, max 200)}`.
- Result:
  `{schema_version, found, brief?, display_argv[], stages[], expected_stages[], triage_items[], items_truncated, child_runs: [ToolRunBriefWire], logs: ToolRunLogMetadataWire, detail_pruned, diagnostics[]}`.
- `stages[]` entries:
  `{description, started_ms, finished_ms?, elapsed_ms?, exit_code?, incomplete, output_bytes?, counts: {new, known, flaky, unknown}}`.
  Every stage stamp is milliseconds, and the field names say so.
- `expected_stages[]` entries: `{description, elapsed_ms}`, taken from the reference
  run. It is filled only when the run is live, or when it settled before reaching every
  reference stage, so the card can show "pending" and "not reached".
- `triage_items[]` entries:
  `{class?, stage_key, display, locator_paths (≤3), occurrences, witness_runs, witness_agents, first_seen_ts, last_seen_ts}`.
  They are ordered NEW, UNKNOWN, unlabeled, KNOWN, FLAKY. Witness counts are distinct
  runs and distinct agents with the same `(signature, extractor_version)` inside the
  window, using the same grouping semantics as `tool_run_failures`.

**Bindings.** `tool_run_live_glance`, `tool_run_briefs`, `tool_run_node_summaries`, and
`tool_run_detail` are registered in `crates/sase_core_py/src/telemetry/mod.rs`
`register_telemetry`. They use the same
`(store_path, request: dict, busy_timeout_ms=250)` shape as `tool_run_list`, and each
gets a round-trip test in `telemetry/tests.rs`. New code calls `sase_core::tool_run::…`
directly and adds no `prelude.rs` aliases.

### 3.5 TUI data flow and performance contract

This follows `tui_perf` rules 1, 2, 4, 5, 6, 7, 8, 11, 13, and 14. All TUI ToolRun code
outside the deck lives in a new package, `src/sase/ace/tui/tool_runs/` (not `tools/`).

1. **Glance snapshot service** (`tool_runs/snapshot.py`). It holds one app-level
   immutable `ToolRunGlanceSnapshot` (runs by attribution key, `silent_after_s`, store
   token, and a load generation).
   - **Idle detection.** A new `ace_refresh_tokens` surface, `tool_runs`, stats
     `runs.sqlite` and `runs.sqlite-wal` in `_surface_tokens.py` (`SurfaceTokenRoots`,
     `SurfaceTokenSnapshot`, a `probe_tool_runs_token` probe). It rides the existing 10
     s auto-refresh. A quiet tick opens no ToolRun file. Update the two test helpers
     that build `SurfaceTokenSnapshot` positionally.
   - **Live cadence.** While the last snapshot holds at least one live run and the
     Agents tab is visible, a stat-only drift probe piggybacks on the existing 1 s
     countdown tick at most every 2 s (the `ProcObserver._store_rows` pattern). It adds
     no new timer or loop.
   - **Load.** On drift, a coalesced (scheduled/running/pending) `spawn_pump_free_task`
     runs `asyncio.to_thread(tool_run_live_glance)`. It defers while `NavigationGate` is
     navigating, re-reads the tab and selection after the await, applies by attribution
     key, and patches only rows whose chip token changed (the bead-warmup pattern,
     `actions/agents/_loading_bead_warmup.py`, including its rebuild escalation on a
     failed patch).
   - **Failure modes.** A busy or locked store, a newer schema, or a thread error keeps
     the last snapshot. A missing binding (stale wheel: `require_rust_binding` raises
     `AttributeError`) disables ToolRun surfaces for the session with one log line, no
     toast storm, and no crash.
2. **Node summaries** (`tool_runs/summaries.py`, built by `header-chip`). A worker
   loader runs behind `DetailPanelDebouncer` for the selected node only. Results go in
   an LRU (64 entries) keyed by `(selector key, store token)`. The header lane, the Runs
   card availability, and the card itself all read the same entry. The loader re-reads
   the selected identity after every await and rejects stale generations (copy
   `FinalDeckView`).
3. **Run detail** (`widgets/decks/tool_runs/loader.py`, built by `runs-card-anatomy`). A
   worker load runs for the visible block only. Results go in an LRU (32) keyed by
   `(run_id, settled_ts)`, or `(run_id, "live", store token)` while live. Settled runs
   are effectively immutable. The log tail is read by the shared Python helper in the
   same worker call.
4. **Render paths are pure.** Row, header, switcher, and block formatters are pure
   functions of in-memory state plus `now`. Nothing stats, opens SQLite, or reads a log
   on a render or keystroke path.
5. **Never on any UI path:** `reconcile_unsettled_tool_runs`, the CLI handlers, a
   `sase tool` subprocess, receipt lookup or report, or `sqlite3` from Python.
6. **Budgets.** j/k p95 stays under 16 ms with the flag on (bench). A glance call takes
   a few ms with ≤10 live runs. A quiet auto-refresh tick reloads no ToolRun surface
   (`refresh.auto_tick` `surfaces_reloaded` excludes `tool_runs`).

### 3.6 Row chip (live only)

Illustrative (the row layout is the existing one):

```
│ 🎭 sase (RUNNING) ⚒ check 7/11 0t9                          🏃 2m13s / 30m36s
│ 🚀 sase (RUNNING) ⚒⚠ check silent 4m sase-1b2.20             🏃 14m02s
│ 🎭 sase (RUNNING) ⚒ check 3/11 0tb
│    └─ ⚙ (CHECKING) ⚒ check 3/11 0tb--mon                     🏃 0m42s
│ 🎭 sase (DONE) 0t8                                           16:24:42 · 52m01s
```

- **When.** Only while the node has a live run (§3.3). A settled run leaves the row on
  the next glance.
- **Text.** `⚒ <label> <k>/<n>`, or `⚒ <label> <elapsed>` when there is no reference run
  or `k > n`. `created` shows `starting`, and a stop request shows `stopping`. Silent
  shows `⚒⚠ <label> silent <age>`. A second live run on the same node appends `+N`.
- **Slot.** Right after the status `)` and after the `⊛` chip when both apply
  (`_agent_list_render_agent_status.py`, after `_append_finalizer_chip`). The chip is at
  most 20 cells; the label truncates with `…` first.
- **Style.** §3.2. The chip is never the only signal: silent carries `⚠` and the word.
- **Ticking.** Rows whose chip text depends on time (elapsed fallback, silent age) join
  the 1 s runtime patch set (`row_runtime_or_wait_ticks` / `_runtime_suffix_ticks`). The
  chip's time is minute-quantized, and the minute bucket is part of the chip token, so a
  tick repaints only when the text changes. This also covers a zombie on a row that does
  not otherwise tick.
- **Cache.** Add a `tool_run_chip_token(agent)` (a pure side-cache lookup) inside
  `_runtime_signature` in `_agent_list_render_cache.py`, so containers invalidate
  through its existing recursion into children. Add the flag state to the render key as
  a deliberate key edit.
- **Width changes.** Progress changes width by at most one cell, which is within
  `patch_row`'s slack. Appearing, disappearing, and turning silent may exceed it; accept
  the existing rebuild escalation for those rare transitions, and never rebuild on a
  progress change.
- **Untouched.** Status buckets, ordering, filters, capacity, `agent_row_is_in_flight`,
  and row actions. Tests assert this.

### 3.7 Header chip and the expanded field

The compact header row 2 (`_identity_header_compact.py`) gets one `⚒` chip after the
activity / `⊛ finalizing` chip, visible whichever deck is open:

| State                   | Chip                                                                                                       |
| ----------------------- | ---------------------------------------------------------------------------------------------------------- |
| Live                    | `⚒ check · lint (mypy) 7/11 · 2m13s / typ 4m13s`. Elapsed turns amber `#FFAF5F` past typical. Never an ETA |
| Silent                  | `⚒⚠ check · test (scoped) · silent 2d` (red)                                                               |
| new_failures            | `⚒ check ✗ 3 NEW · 1 KNOWN · 4m12s · 16:21`                                                                |
| known_only              | `⚒ check ≈ known only · 2 KNOWN · 4m05s · 16:02`                                                           |
| undetermined            | `⚒ check ? 2 UNKNOWN · 4m20s · 15:48`, or `? untriaged`                                                    |
| killed / stopped / lost | `⚒ check ⊘ killed at 9m00s · signal · 15:40`, `⊘ stopped at 1m02s`, `⊘ lost · wrapper_lost`                |
| pass                    | `⚒ check ✓ 4m01s · 15:31`                                                                                  |

- The chip covers the node's latest run per label. A live run wins; otherwise the most
  severe settled one (§3.2), plus `+N` when other labels exist.
- Data comes from a new `tool-runs` lane in `DetailHeaderSummary`
  (`_agent_display_state.py`, modeled on the `slow-tools` lane), fed by §3.5.2. Live
  facts are overlaid from the glance snapshot, so the chip moves without a summary
  reload.
- The expanded header gains a `Tool runs:` field (after BUG in
  `_agent_display_header_metadata.py`), with one entry per label:
  `check ✗ 3 NEW 6c3d5107 · test ✓ 81ef0cb1`.
- `%` copy mode gains a "tool run id" target (`_copy_target_standard.py`,
  `AGENT_COPY_TARGETS`, and a free key in `default_config.yml`'s copy section). It
  copies the full 32-hex id of the header chip's run. This turns today's dead-text run
  ids into one keystroke.

### 3.8 Tools deck and the `⚒ Runs` card

**Structure (refined from the report).** Tools keeps two independent card hosts:

- `ToolRunsDeckView(CardDocumentView)`, under `widgets/decks/tool_runs/view.py`. Its
  document holds exactly one card, `runs` (tab `⚒ Runs`), with one `CardBlock` per run.
- The unchanged `AgentLLMCallsPanel`, for the `llm-calls` card.

`DeckPanel` resolves Tools' active card id (`runs` | `llm-calls`) and shows exactly one
host. `CARD_DOCUMENT_DECKS` gains `TOOLS`. `document_view` / `document_for` /
`panel_blocks.py` / rail sync / search corpus / scroll watchers / `_deck_is_empty` /
`cycle_card` become **exhaustive per-deck dispatch**: a `match` over `DeckId` with an
explicit Tools arm that returns the Runs view only while the Runs card is active and
`None` otherwise. No "else MAIN" fall-through remains, and a test pins that every
`DeckId` has an arm.

**Chrome.**

- Tabs are `⚒ Runs │ LLM Calls`. A node without runs keeps today's single `LLM Calls`
  tab. A monitor or named proc without LLM calls shows only `⚒ Runs`.
- Blurb: "Tool runs and LLM calls". Empty copy: "No tool runs or LLM calls for this
  node" (plus D15's remote line).
- The picker shows `2 runs · 57 calls`.
- The switcher uses D8's `status_segments` entry (`panel_chrome.py`, a `_tools_switcher`
  set from a `ToolRunsDeckLoaded` message).
- The footer card count reflects the tab count, and Ctrl+J/K appear when it is above 1.
- The default card is D7's rule, via a new `tools_default_card(card_ids, preferred)`
  modeled on `final_default_card`. Stickiness is recorded with
  `set_preferred_card_state` (not `set_preferred_card`, which re-shows Main).

**Availability.**

- The Tools probe becomes `has LLM calls OR has runs`. The Runs part reads the
  node-summary LRU plus the glance snapshot, with no I/O. It is `None` while the first
  load is in flight.
- The probe opens for monitor turns and named procs that own runs, which the
  transcript-based gate (`availability.py`, `supports_slow_tool_sources`) excludes
  today. Pinned attempts, clans, and remote rows stay without Runs.
- `LLMCallsVisibilityChanged` updates only the LLM Calls card's availability and is
  merged with Runs, never overwriting the deck.

**Blocks.** One `CardBlock(run_id, …)` per run, with
`BlockMeta(number, label, glyph, accent, status_bucket, kind="tool_run")` from §3.2 and
headers stamped with `DECK_BLOCK_META_KEY`. The rail reads
`1 ⊘ check  2 ✗ check  3 ✗ check`. Exactly one run gives a block-less document with no
rail, as FINAL does. Follow and hold use the existing `reconcile_cursor` / `arrived_ids`
(D9). When the node summary is `truncated`, the card ends with a dim
`+N older runs · Admin Center → Tools` line; it never pretends the list is complete.

**Block anatomy** (illustrative, STANDARD level):

```
⚒ check  ✗ 3 NEW · 1 KNOWN · 0 FLAKY                 4m12s (typ 4m13s) · 16:21
  run 6c3d5107 · inline · turn 0t9--code · bead sase-1b2.20 · ws 13
  $ just check
  ✓ fmt (python)          0.3s ▏
  ✓ lint (ruff)           4.1s ▏
  ✓ lint (mypy)          1m03s ▕██████▌
  ✗ lint (symvision)     1m07s        ▕███████                  2 NEW
  ✓ keep-sorted           0.8s               ▏
  ✗ test (scoped)        1m31s               ▕█████████▍        1 NEW · 1 KNOWN
  NEW    lint (mypy)        _tree.py:622 Name "prefix_key" already defined
                            35 runs · 33 agents · since 09-26
  KNOWN  lint (symvision)   intent_accept in monitor/no_new_receipt.py
                            44 runs · 12 agents · since 09-26
  ↳ child run 81ef0cb1 test ✓ 1m02s
  ── log tail · last 12 of 2,341 lines · 1.2 MB retained ──
  …
```

- **Outcome line.** Glyph, label, bucket words and counts, duration against typical, and
  absolute settle time. A killed run reads `⊘ killed at 9m00s · signal`, followed by a
  dim line: "the caller killed the command; this is not a test failure".
- **Context line** (dim). Run id (8 hex), launch mode (`inline` / `→ monitor <name>` /
  `→ proc <id>`), turn, bead, and workspace. Then `$ <display_argv>` on its own line.
- **Stage waterfall.** One row per stage: glyph (`✓ ✗ ▶ ·`), description, duration, and
  a Gantt bar whose **offset** is proportional to the stage start within the run and
  whose **length** is proportional to elapsed. Bars use eighth-block characters for
  sub-cell precision and are at least one cell. Passed stages are dim green, failed red,
  running gold ending `…`, pending dim `·` with `typ Ns`, and unreached stages dim
  `– not reached`. Per-stage triage counts sit right-aligned. Below a 70-cell panel
  width the bars drop and the table stays.
- **Triage items.** Class tag (colored, with the word), stage, display text truncated
  with `…`, and a second dim line with witness counts. They answer "is this mine?"
  without leaving the card.
- **Child runs.** Nested runs are `↳` lines with their own bucket (D9).
- **Log tail.** Read by the shared helper (§3.12). The header states line count, total
  size, and truncation.
- **Honest absence.** Say "detail pruned · summary retained", "log pruned (retention)",
  "owner log unavailable", or "no log recorded". Never render an empty success.
- **Detail levels** (h/l/H/L, routed by the active card):
  - COMPACT: outcome, context, non-passing stages plus one `✓ N stages passed · 2m58s`
    line, and the top 3 triage items.
  - STANDARD (default): everything above, with at most 20 items (`+N more`) and a
    12-line tail.
  - FULL: adds locator paths, a 60-line tail, and full argv.

**Live blocks** (`runs-card-live`). A live block renders the in-flight stage `▶` with a
growing bar and the pending reference stages. The outcome line reads
`⚒ check  ▶ running · test (scoped) 7/11 · 2m13s / typ 4m13s`, and a silent block reads
`⚒⚠ check  silent 2d · last activity 09-25 16:03 · test (scoped)` plus a dim hint: "The
TUI never settles runs. `sase tool runs` reconciles, or stop it from the palette."

- A pure 1 Hz repaint updates elapsed and bar growth from the cached detail. It runs
  only while the card is visible and the run is live, and it reuses FINAL's live gate
  (`decks/final/live.py`: visible, not navigating, not typing).
- Detail is re-fetched only when the glance token drifts.
- On settle, the same block (same id) re-renders as settled. It never moves the cursor
  or an older selected block.

**`v` and `%`.**

- `v` hint mode (`actions/hints/_files.py`) gains one `⚒ run log <8hex>` target per
  visible run block. It opens the retained log in the pager (`PagerDocument`, text
  section) off-thread.
- `%` copy mode's "tool run id" target (§3.7) copies the **selected block's** run id
  when the Runs card is active.

### 3.9 Links into the run (`run-links`)

- **LLM Calls.** A Bash row whose command runs `sase tool run …` (or a
  `sase monitor start` that wraps one) gains a suffix `→ ⚒ check ✗ 3 NEW` (bucket glyph
  and short words) in both the Text and markdown twins (`_llm_calls_panel_timeline.py`).
  - The join is computed in the LLM Calls worker against the node summary's runs:
    - same node;
    - run `created_ts` within `[call start − 2 s, call end + 2 s]`;
    - `display_argv` tokens match the command's tool argv.
  - The fallback scrapes the 32-hex run id from the call output, which the compact
    footer prints (`src/sase/tool/executor_display.py`).
  - The suffix is a click target that selects that run's block on the Runs card: select
    the block cursor, `show_card`, store the active card, refresh chrome, and sync the
    rail. `v` hint mode lists the same jump as `⚒ run <8hex>`.
- **Main deck slow-tool list** (`widgets/prompt_panel/_agent_slow_tools.py`). A
  _running_ `sase tool run` Bash row gains `· ⚒ lint (mypy) 7/11` from the glance
  snapshot. A settled one gains the bucket suffix.
- **Monitor and named-proc Context cards** gain a `Tool run` row:
  `⚒ check ✗ 3 NEW · 04349ecf`, clickable to the Runs card.

### 3.10 Admin Center → Tools pane (`admin-tools-pane`)

A new `CenterTabSpec("tools", …)` sits between Statistics and Updates, so Tools is 7 and
Updates is 8. Update the numbering test, `help_modal/binding_common.py` ("1-8 jump"),
and the docs rows. The pane uses the Projects-pane sub-view switcher (`PanelTabStrip` +
`ContentSwitcher`, `[`/`]`, click). Each view is a list plus a detail region. It loads
on a thread worker with `_reload_pending` coalescing. While the pane is the active tab,
it stat-probes the same two store files at most every 2 s (the Procs pane's active-only
polling pattern) and reloads only on drift. It pauses when hidden.

```
 ⚒ Tools   Runs │ Failures │ Catalog                           +sase · A all projects
 ⚒⚠ check  silent 2d          sase-1b2.20--mon  test (scoped)        2f5ce886 │ ⟨block⟩
 ⚒  check  7/11 · 2m13s       0t9--code         lint (mypy)          6c3d5107 │
 ✗  check  3 NEW · 1 KNOWN    0t8--code         4m12s · 16:21        81ef0cb1 │
 ≈  check  known only         0t7--code         4m05s · 16:02        1a2b3c4d │
 ✓  test   pass               —                 1m02s · 16:05        04349ecf │
 enter detail · a agent · s stop · v log · y copy id · / filter · [ ] view · A all
```

- **Runs** (default). Silent runs are pinned first in red, then live, then settled
  newest first (C1 + C2). `/` filters with `tool:`, `state:`, `agent:`, and `verdict:`
  tokens. The detail region uses **the same block renderer** as the Runs card. Keys:
  `enter` focuses detail, `a` jumps to the owning Agents node (close the modal, set the
  tab, then `_reveal_agent_row`, and select the run's block), `v` opens the log in the
  pager, `y` copies the id, and `s` stops (added by `tool-run-actions`).
- **Failures.** Rows come from `tool_run_failures` (a pure-read binding called directly,
  never `src/sase/tool/failures.py`, which reconciles): class, tool/stage, signature
  display, runs, agents, first and last seen. `enter` lists the affected runs and agents
  in the detail region, and `a` jumps to one. This answers "is master red, and who is
  blocked by it?"
- **Catalog.** `load_project_tool_catalog_at(<current project's primary root>)` plus one
  `tool_run_summary` per tool (pure read) gives name, stage count from the reference
  run, LAST (bucket glyph, words, time), and TYPICAL (`n=`). `r` (added by
  `tool-run-actions`) runs the tool.
- **Deep links.** `_open_config_center` gains a generic `focus_target` (a
  `ToolRunFocusTarget(run_id)` here) that the pane holds as _pending_ until its first
  load lands, then selects. It is not delivered-once like `proc_focus_target`. Opening
  when a Config Center modal is already showing switches its tab instead of pushing a
  second modal.
- **Keys** live in a keymap dataclass and a `default_config.yml` section with metadata
  and a help section, following the configurable-pane pattern (`keymaps/app_keymaps.py`,
  `keymaps/registry.py`).

### 3.11 Actions (`tool-run-actions`)

**Stop.**

- `sase tool stop RUN -j` gains typed result emission: an `ops/names.py` `TOOL_STOP`
  plus `emit_operation_result`, mirroring `ops/commands/monitor.py`.
- The TUI submits it through `_submit_durable_proc` with `durable_fingerprint`,
  `durable_request_payload`, and a concurrency key per run id. Register the producer
  site in `_proc_producer_sites_actions.py`.
- The confirm is a `ConfirmActionModal` with DANGER and Cancel focused. Its copy depends
  on the owner:
  - **Inline agent run:** "Stop ⚒ check (run 6c3d5107)? The agent's `sase tool run`
    exits 143 and it will see a failed check. The agent keeps running."
  - **Monitor-owned:** "Stop ⚒ check owned by monitor `<name>`? The monitor stops and
    its follow-up agent will not launch."
  - **Proc-owned hand-off:** "Stop ⚒ check? Its hand-off proc is killed, and the run
    settles as stopped."
  - **Nested run:** the TUI resolves the outermost stoppable ancestor and says "This run
    belongs to run `<parent>`; stopping it stops both."
- Entry points: Admin Runs `s`; the palette "Stop live tool run" for the selected node's
  live run (or the selected Runs block).
- The outcome is a toast. The row chip turns `stopping`, then leaves on settle.

**Run from the catalog.**

- `r` shows a confirm with the tool's argv and the root, with Run focused, because this
  is the user's explicit action and the argv is repo-reviewed.
- It then calls the hand-off launcher (`handoff_launch.py`) **in-process** from a
  `_submit_session_worker` body, passing the root explicitly. If the launcher reads the
  process cwd or env, add explicit parameters; never mutate the TUI's cwd or env.
- It never uses the `:` proc path or `_submit_durable_proc` (D13).
- On success, select the reserved run in Runs. On refusal, toast the launcher's typed
  message.

**OpenToolRun notification (`sase-189`).**

- `deliver_handoff_settlement` (`src/sase/tool/notify.py`) publishes
  `action="OpenToolRun"` (PascalCase like the existing actions) with the same
  `action_data` and a deterministic id. With the flag off it keeps `action=None`.
  Monitor-owned runs still never notify.
- `_notification_dispatch.py` routes it:
  - when the run has a visible local node (§3.3 resolver), select that node and its Runs
    block (`p t`);
  - otherwise, open Admin Center → Tools focused on the run.
- Add `⚒` to `ACTION_BADGES`/`ACTION_ICONS`, a toast entry, the docs table, and the
  dispatch test table. Update `tests/tool/test_settlement.py`'s `action is None`
  assertion to cover both flag states.
- In the phase close note, say that `sase-189` is implemented, so the land agent can
  close it.

**Procs rows.** A proc tagged `tool-run:<id>` (from `owner_tags` in `handoff.py`) gains
a `⚒ <label>` marker. The label comes from the proc label `tool:<name>`, in the pure
`procs_pane_render.py`, following `_append_monitor_marker`. Enter (or its hint token)
switches the modal to Tools → Runs focused on that run.

**Palette** (keyless, context-sensitive, with availability rules in
`commands/_availability_agents.py`):

- "Show tool runs": the focused panel shows Tools with the Runs card active. Available
  when the node has runs.
- "Stop live tool run": available when the selected node has a local live run.
- "Open Tools pane".
- "Run project tool…": opens Tools → Catalog.

### 3.12 Module map

| Area                                    | Location                                                                                                                                                 |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Core projections, helpers, indexes      | sase-core `crates/sase_core/src/tool_run/` (new projection module, `store/connection.rs` `SCHEMA_SQL`), bindings in `crates/sase_core_py/src/telemetry/` |
| Python adapters and typed results       | `src/sase/core/tool_run.py` (`ToolRunGlance`, `ToolRunBrief`, `ToolRunNodeSummary`, `ToolRunDetail` dataclasses, `from_wire`)                            |
| State vocabulary                        | `src/sase/tool/view_vocabulary.py`                                                                                                                       |
| Shared log tail                         | `src/sase/tool/logs.py` `tool_run_log_tail(...)`, used by `sase tool show --log` and the TUI                                                             |
| Flag helper                             | `src/sase/ace/tui/tool_runs/flag.py` `tool_runs_enabled()`                                                                                               |
| Snapshot, attribution, summaries, chips | `src/sase/ace/tui/tool_runs/{snapshot,attribution,summaries,chips}.py`                                                                                   |
| Runs card                               | `src/sase/ace/tui/widgets/decks/tool_runs/{view,document,blocks,waterfall,loader,live}.py`                                                               |
| Admin pane                              | `src/sase/ace/tui/modals/tools_pane*.py`                                                                                                                 |

### 3.13 Feature flag scaffolding

`tool-run-adapter` creates the flag with
`sase flag new ace_tool_runs -k beta --when-enabled … --when-disabled … --remove-when …`,
following `sase memory read sase_flags.md`. Do not hand-add a registry entry. Paste the
printed registry and schema entries.

- **On:** row chips, the header chip and field, the copy target, the Runs card and Tools
  chrome changes, the `v` targets, LLM Calls / slow-tool / Context links, the Admin
  Tools tab, the OpenToolRun action, the Procs marker, and the palette commands.
- **Off:** rows, headers, Tools, Admin Center, notifications, and Procs render exactly
  as today. The Admin Center has 7 tabs.
- **Remove when:** `cutover` has verified goldens, live captures, and the j/k bench with
  the flag on.

These are unconditional, with no visible TUI change: core projections, adapters, the
vocabulary module, the log-tail helper (CLI output must stay byte-identical), the
`sase tool stop` typed result, and the `⚒`→`⏲` link-trail move (D1). Every gated
behavior gets on and off tests. `cutover` deletes the Off branches and `flag.py`,
removes the registry entry, and closes the flag bead in the same change.

### 3.14 Cross-repo, lint, and sequencing protocol

- **sase-core phases** open the checkout with `sase repo open sase-core -r "<why>"`,
  read its `AGENTS.md`, follow its "Add a core function and expose it to Python" recipe,
  use `feat:` commits, keep files at or under 1,500 lines with facade-only `mod.rs`, and
  pass `sase tool run check` **inside that checkout**. The host finalizer commits and
  pushes that repo.
- **Pin moves.** `tool-run-adapter` moves `sase-core-revision.txt` past `core-glance`,
  and `runs-card-anatomy` moves it past `core-run-detail` (`just ratchet-core-revision`,
  then `just rust-install`). Register new binding names as string literals in
  `require_rust_binding(...)`, `tools/validate_sase_core_rs` `REQUIRED_BINDINGS`, and
  `src/sase/_symvision_static_refs.py`.
- **Symvision.** A public symbol consumed only by a later phase gets an `--epic-symbol`
  whitelist entry (`sase memory read symvision.md`), which the consuming phase removes.
  Prefer wiring a real consumer in the same phase.
- **Config and keys.** Every new key or config value updates
  `src/sase/default_config.yml`, the schema, the settings parser, and their parity tests
  (core-memory gotcha).
- **In-flight epics.** Before editing deck chrome (`panel_chrome.py`, `panel_blocks.py`,
  `titles.py`, `card_documents.py`, `_agent_detail_decks.py`), `tools-deck-cards` reads
  `sase bead read sase-1b1 -r "<why>"`. If `sase-1b1.8` is still open, rebase onto
  master first and keep edits additive to its files. `glance-row-chips` uses only the
  existing surface-token and patch paths so it cannot conflict with `sase-124.8`'s
  freshness work.
- **Memory.** This plan does not authorize memory edits. **Do not edit `sase/memory/`**;
  `docs` records the proposed text instead.

## 4. Phases

### 4.1 `core-glance`: Live glance, node summaries, brief lists, and the verdict bucket in sase-core

Work in the sase-core checkout (§3.14).

1. Add the projection module with C0–C3 (§3.4): wires, the verdict-summary helper and
   bucket precedence, `TOOL_RUN_SILENT_AFTER_SECONDS`, reference-run and typical lookups
   (the same median rule as `summarize`, sharing its helper rather than copying it), and
   `last_activity_ts` (stage stamps ÷ 1000).
2. Add the three indexes to `SCHEMA_SQL` with `IF NOT EXISTS`.
3. Bindings `tool_run_live_glance`, `tool_run_briefs`, and `tool_run_node_summaries` in
   `telemetry/mod.rs` and `register_telemetry`.

**Tests:**

- The bucket precedence table over every state × terminal cause × verdict.
- Selector matching: agent only, owner only, both, `since_ts`, and dedupe.
- The reference-run choice: same digest only, incomplete stages skipped, and none.
- `stages_done` and the current stage from a real `stage_started`-only row.
- `last_activity_ts` from samples vs stages (the ms/s conversion).
- The 200-run glance cap and `truncated`.
- A missing store, missing triage tables, and a pre-index store.
- A `private_argv` redaction canary: a run with private argv never appears in any
  result's JSON.
- Round trips for each binding.
- A payload-size assertion: a 50-run brief page stays under 64 KB with fingerprints
  present in the store.

### 4.2 `core-run-detail`: Per-run detail projection with stage timeline and witness counts in sase-core

Work in the sase-core checkout.

1. Implement C4. Stages come in `started_ms` order, with per-stage triage counts.
   `expected_stages` comes from the reference run and is filled only when the run is
   live or settled before the reference's last stage.
2. Build triage items in class order with witness counts, using `tool_run_failures`'s
   grouping semantics (a shared helper, not a copy) inside `witness_window_days`. Add
   child runs by `parent_run_id` as briefs, and `detail_pruned` when a settled run has
   no stage rows but its summary says stages ran.
3. Binding `tool_run_detail` with a round-trip test.

**Tests:** found/not found; live with an in-flight stage and pending expected stages;
settled early (not reached); triage ordering and witness counts across two agents;
`item_limit` truncation; children; the pruned-detail fixture; and the redaction canary.

### 4.3 `tool-run-adapter`: Python adapter, state vocabulary, beta flag, shared log tail, and the chop glyph move

1. **Pin.** Move the pin past `core-glance` (§3.14) and register the three binding
   names.
2. **Adapters.** Add `ToolRunGlance`, `ToolRunBrief`, `ToolRunVerdictSummary`,
   `ToolRunNodeSummary`, and the result types as frozen dataclasses with tolerant
   `from_wire` (unknown keys ignored; missing optional keys become `None`), plus
   `tool_run_live_glance()`, `tool_run_briefs()`, and `tool_run_node_summaries()` in
   `src/sase/core/tool_run.py`, defaulting the store path as the neighbors do. Leave the
   `tool_run_detail` adapter for `runs-card-anatomy`.
3. **Vocabulary.** Add `src/sase/tool/view_vocabulary.py` with §3.2: `ToolRunStateStyle`
   (glyph, word, color), a bucket lookup, severity ordering, duration and age
   formatters, and pure chip-text builders (`row_chip_text`, `header_chip_text`,
   `switcher_runs_text`) that take plain values and `now`.
4. **Flag.** `sase flag new ace_tool_runs` (§3.13) and `tool_runs/flag.py`.
5. **Log tail.** Extract a pure
   `tool_run_log_tail(run_id, logs_metadata, owner_kind, owner_id, lines, max_bytes) -> ToolRunLogTail(availability, source, lines, total_bytes, truncated)`
   into `src/sase/tool/logs.py`, from the log-selection logic in `query.py`'s show path
   (run logs vs owner log, retention). `sase tool show`'s output must stay
   byte-identical; add a parity test.
6. **Glyph.** Move the chop icon: `link_trail.py` and `link_subject.py` use `⏲`. Update
   `tests/ace/tui/test_link_rail.py` and `test_link_follow_entry_points.py`. If any
   golden shows the link trail, rebaseline it with targeted
   `just fix-tui-screenshots -- <selectors>` and inspect it.

**Tests:** adapter round trips against the real binding; vocabulary tables for every
bucket, silent ages, `k>n` fallback, and truncation to 20 cells; flag on/off helper;
log-tail availability cases (available, truncated, pruned, owner log missing, not
recorded); and the emoji-font audit for `⏲` and `≈`.

### 4.4 `glance-row-chips`: ToolRun glance snapshot service and live-only ⚒ row chips

1. **Snapshot service** (§3.5.1): the surface token, the ≤2 s live drift probe on the
   countdown tick, the coalesced pump-free load, the `NavigationGate` deferral, apply by
   attribution key, patching, and the failure modes. Add a `tool_runs_loads` counter to
   the `refresh.auto_tick` trace span.
2. **Attribution** (§3.3) as a pure module over `Agent` rows (`is_monitor` +
   `monitor_id`, `is_named_proc` + `proc_id`, `agent_name`, containers,
   `fleet_origin_alias`).
3. **Row chip** (§3.6): rendering, session inheritance, clan and remote exclusion, the
   ticking set, the cache token inside `_runtime_signature`, and the flag in the render
   key.

**Tests:**

- The attribution table, including a monitor-owned run chipped on the monitor row only,
  a container-named agent, and a nested run folded into its parent.
- Chip text per state, `+N`, 20-cell truncation, and `⊛` + `⚒` together.
- A silent transition driven purely by `now`, with no reload.
- A session inherits silent over live.
- A quiet tick opens no ToolRun file (trace counter).
- Drift coalescing: three drifts give one load.
- A busy store keeps the last snapshot.
- A missing binding disables surfaces with one log line.
- A patch, not a rebuild, on a progress change.
- Flag off is byte-identical.
- Record `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` p95 with the flag on.
- Inspect a flag-on `sase screenshot` of a real `sase tool run check` advancing.

### 4.5 `header-chip`: Selection-scoped ⚒ header chip, Tool runs field, and copyable run ids

1. **Node summaries** (§3.5.2): the selector builder from §3.3, the worker loader, the
   LRU, and stale rejection.
2. **Lane and chip** (§3.7): the `tool-runs` `DetailHeaderSummary` lane, the compact
   chip with the live overlay from the snapshot, and the time policy (D6). Check whether
   the header already repaints on the 1 s tick before choosing seconds.
3. **Field and copy.** The expanded `Tool runs:` field and the `%` "tool run id" target
   (a new copy-mode key in `default_config.yml`).

**Tests:** one chip per bucket; live past typical turns amber; multi-label severity and
`+N`; a session union deduped; a monitor annotated with its starter; remote shows no
chip; the copy target copies the full id; debounced loads with a stale generation
rejected on j/k; flag off byte-identical; and a flag-on `sase screenshot` inspection.

### 4.6 `tools-deck-cards`: Tools becomes a two-card deck with ⚒ Runs first

Read §3.14's in-flight rule first.

1. **Host.** Add `ToolRunsDeckView(CardDocumentView)` with a `build_tool_runs_document`
   that makes one `runs` card, whose blocks carry the outcome line only (the full
   anatomy comes next). Add a worker loader over the node-summary LRU, a
   `ToolRunsDeckLoaded` message, and a `get_tool_runs_text` search corpus.
2. **Dispatch** (§3.8): `CARD_DOCUMENT_DECKS` gains TOOLS, and every "FINAL else MAIN"
   chain becomes exhaustive per-deck dispatch, with a test that every `DeckId` has an
   arm. `DeckPanel` shows exactly one Tools host by active card. Tools stays paged-only.
3. **Chrome.** Tabs from availability, the switcher segment (D8), picker counts, blurb
   and empty copy, the footer card count and Ctrl+J/K, the sticky default card
   (`tools_default_card`, `set_preferred_card_state`), and persistence of the `tools`
   preference.
4. **Availability.** The per-card merge, the monitor and named-proc opening, and
   `LLMCallsVisibilityChanged` narrowed to its card.
5. **Detail levels.** h/l/H/L route by active card. The Runs card gets its own level
   state (COMPACT / STANDARD / FULL, default STANDARD). Update the footer labels.

**Tests:**

- The default-card rule: sticky wins, Runs when present, otherwise LLM Calls.
- A no-runs node keeps a single LLM Calls tab.
- A monitor with runs and no calls shows Runs only.
- The switcher segment never contains `" · "`.
- Ctrl+J/K round trip.
- `(`/`)` on two runs, with no rail on one.
- h/l on Runs leaves the LLM Calls level unchanged, and vice versa.
- Search finds block text.
- `P` is a no-op on Tools.
- The FINAL and Main deck unit and golden suites stay pixel-identical.
- Flag off: Tools is exactly today's deck.

### 4.7 `runs-card-anatomy`: Full ⚒ Runs block anatomy - waterfall, triage, log tail, and honest absence

1. **Pin.** Move the pin past `core-run-detail`, and add the `tool_run_detail` adapter
   and `ToolRunDetail` types.
2. **Detail loader** (§3.5.3): the worker load for the visible block, including the log
   tail via the shared helper, the LRU, and stale rejection.
3. **Blocks** (§3.8): outcome and context lines, the waterfall (a pure `waterfall.py`
   that takes stages, run span, and width and returns `Text`, with the narrow fallback),
   triage items with witness counts, child runs, the log tail, honest absence, and three
   detail levels. Put the renderer in `blocks.py` as a pure function the Admin pane can
   import.
4. **`v` and `%`.** Add the `⚒ run log` hint targets that open the pager, and make the
   copy target use the selected block.

**Tests:** renderer tables for new_failures, known_only, undetermined (UNKNOWN and
untriaged), pass, killed (with the explanation line), stopped, lost, pruned detail,
pruned log, owner log unavailable, and truncated tail; waterfall offsets and lengths,
sub-cell rounding, minimum one cell, and the narrow fallback; COMPACT collapsing passed
stages; `display_argv` only; the LRU hit on a settled run; and a flag-on
`sase screenshot` inspection of a failed check block.

### 4.8 `runs-card-live`: Live run blocks - in-flight stages, pending stages, follow and hold

Implement §3.8's live blocks:

- the in-flight `▶` stage and growing bar;
- pending reference stages with `typ`;
- silent block copy;
- the pure 1 Hz repaint under FINAL's live gate;
- a refetch only on glance drift;
- follow vs hold with arrival dots;
- the in-place settle transition.

**Tests:**

- The repaint does no I/O (patch the loader and assert zero calls across ticks).
- It stops when the card is hidden or navigation is active.
- A new run arrives while on the newest block (follows) vs on an older block (holds,
  dot).
- Running → settled keeps the cursor and the block id.
- A silent block renders from `now` alone.

Inspect a real live check in a flag-on `sase screenshot` session with `--keep`,
capturing twice at least 10 s apart.

### 4.9 `run-links`: Link LLM Calls, the slow-tool list, and Context cards to the run

Implement §3.9: the LLM Calls join (primary join and scrape fallback), the suffix in
both twins, the click jump and `v` hint jump to the Runs block, the slow-tool live and
settled suffixes, and the Context card `Tool run` row for monitors and named procs.

**Tests:**

- The join: exact argv, time window edges, two runs in one call window (the nearest
  wins), and scrape-only.
- A non-tool Bash row gets no suffix.
- The jump selects the right block from either card.
- The slow-tool suffix appears while running and after settle.
- A monitor Context row without `sase-1bi` (ledger owner join only).
- Flag off is byte-identical.

### 4.10 `admin-tools-pane`: Admin Center Tools pane with Runs, Failures, and Catalog views

Implement §3.10: the tab spec and renumbering (the test, help copy, and docs rows), the
lazy factory (keep `test_config_center_tabs.py`'s lazy-import guard green), session
state, the three views with list and detail, current-project filtering with `A`, `/`
filter tokens, `enter`/`a`/`v`/`y`, the pane keymap and config section, and the generic
pending `focus_target` deep link. Gate the tab on the flag, so there are 7 tabs when
off.

**Tests:**

- Tab order and numbers in both flag states.
- Silent pinned first.
- Filters.
- A focus target set before the load lands selects after it.
- Jump-to-agent selects the node and its block.
- Failures `enter` lists agents.
- Catalog rows from a fixture catalog.
- No reconcile: patch `reconcile_unsettled_tool_runs` to raise and exercise every view.
- Flag-on `sase screenshot` inspection of each view.

### 4.11 `tool-run-actions`: Stop, run from the catalog, OpenToolRun notifications, Procs decode, and palette

Implement §3.11:

- the `sase tool stop` typed result (unconditional);
- the stop flow with per-owner copy and nested-ancestor resolution;
- the catalog `r` session-worker hand-off with its confirm;
- the OpenToolRun producer and dispatch (`sase-189`);
- the Procs marker and jump;
- the four palette commands with availability rules.

Record `sase-189` as implemented in the phase close note.

**Tests:**

- The stop op writes a typed result and the durable proc completes cleanly.
- The confirm copy per owner kind, with Cancel focused.
- A nested run resolves to its parent.
- `r` refuses cleanly when the launcher refuses, and never goes through
  `_submit_durable_proc` (assert the path).
- OpenToolRun routes to the node when visible, otherwise to the pane.
- Monitor-owned runs still never notify.
- Deterministic id and exactly-once delivery are unchanged.
- The Procs marker is decoded from tags only.
- Palette availability.
- Flag off: `action=None`, no marker, no commands.
- Inspect a real `-H` run started from the Catalog, its settlement notification, and
  Enter opening the run, in a flag-on live session.

### 4.12 `cutover`: Remove ace_tool_runs, add goldens, inspect live, and bench

1. **Bench first**, with the flag still present. Run the j/k bench flag off and on in
   SINGLE, LEFT_RIGHT, and with Tools pinned in the second panel. Also run an idle-tick
   trace (`SASE_TUI_TRACE=1`, the `docs/perf_runbook.md` idle recipe) with no live run
   and with one live run. p95 must be under 16 ms, and a quiet tick must reload no
   `tool_runs` surface. Record the numbers in the close note, and fix any regression.
2. **Remove the flag:**
   - delete every Off branch and `tool_runs/flag.py`;
   - make `OpenToolRun` unconditional;
   - collapse on/off tests to single-state;
   - remove the registry and schema entries;
   - close the flag bead in the same change.

   `sase flag show ace_tool_runs` and `tools/check_feature_flags` must be clean.

3. **Goldens.** Add a new `tests/ace/tui/visual/test_ace_png_snapshots_tool_runs.py`
   with deterministic fixtures: one loader seam each for glance, node summaries, and
   detail, monkeypatched with fixed dicts, pinned clocks, and no SQLite. Cover:
   - rows (live, silent, starting, `+1`, with `⊛`) at 120, 80, and 60 columns;
   - header chips for each bucket;
   - the Runs card: failed check, live, killed, pass, pruned, a monitor node, and a
     session with rail and 3 runs, at 120 and 80;
   - the Tools/Reply split;
   - the switcher and picker;
   - the Admin Runs, Failures, and Catalog views;
   - the notification modal with the ⚒ badge;
   - the Procs row marker;
   - the link trail with `⏲`.

   Generate them with targeted `just fix-tui-screenshots -- <selectors>` (use
   `/sase_monitor` if long), and inspect **every** new PNG. Then rebaseline the existing
   goldens that legitimately change (the switcher's `tools` segment, picker counts,
   Admin tab numbering), and inspect them group by group in the retained visual report.
   Generation is not approval.

4. **Live inspection** with `sase screenshot`:
   - a real agent's `check` advancing and settling (row, header, card);
   - a monitor-owned check;
   - a silent run (the existing zombie, if still present, or a scratch
     `sase tool run -- sleep 600` whose wrapper is SIGSTOPped past 60 s, then resumed
     and stopped);
   - the Admin pane;
   - a `-H` run end to end.

   Fix any defects found.

5. **Check.** `sase tool run check` is green. Do not run `just check-full` unless
   explicitly instructed.

### 4.13 `docs`: User docs for ToolRuns in the TUI

Describe only shipped behavior.

- **`docs/ace.md`:**
  - the `⚒` glyph and §3.2 vocabulary;
  - the row chip (live only, silent);
  - the header chip and `Tool runs` field;
  - the Tools deck's two cards, default and sticky rules, block anatomy, detail levels,
    live behavior, and the `v`/`%` integrations;
  - LLM Calls and slow-tool links;
  - the Admin Center Tools tab (7), Updates now 8, and the global-keys rows;
  - stop and run semantics with their consequences;
  - the OpenToolRun notification row;
  - the palette commands;
  - the `⏲` link-trail icon in the glyph legend.

  Fix the stale deck rows that omit FINAL/`n` while there.

- **`docs/tool.md`:** a short "In the TUI" section pointing at the above, and restating
  that the TUI never reconciles.
- **`docs/configuration.md`:** the Tools pane keymap section, the copy-mode key, and the
  Admin tab numbering.
- **Memory.** Do not edit `sase/memory/`. Record a `PROPOSED FOLLOW-UP:` note on this
  phase's bead with ready-to-apply text for:
  - the `glossary:agent-data-deck` strand (Tools holds `⚒ Runs` and LLM Calls);
  - the `glossary:llm-calls` strand ("future ToolRuns" becomes the sibling Runs card);
  - the `glossary:tool-run` strand (where runs surface in the TUI);
  - an optional new strand, **Verdict Bucket**.

## 5. Verification

**Per phase:**

- targeted pytest (or targeted `just test -p` in sase-core) for the touched areas;
- `just fix`, then `sase tool run check` in every repo the phase changed;
- UI phases also inspect a flag-on `sase screenshot`.

**Goldens** change only in `cutover`, except the D1 link-trail move in
`tool-run-adapter`. Every other phase leaves existing goldens pixel-identical (flag off,
or no visual change).

**The epic is done when:**

- starting `sase tool run check` in an agent shows `⚒ check k/n` advancing on its row
  and leaves on settle;
- a monitor-owned check chips the monitor row, not its starter;
- a silent run turns red with no TUI write;
- selecting any agent answers "did its latest check add NEW failures?" from the header;
- `p t` shows the Runs card with waterfall, triage, and log tail, and `(`/`)` walk runs;
- LLM Calls rows jump to their run;
- Admin Center → Tools lists, filters, and jumps, and Failures reaches affected agents;
- a Catalog `r` hand-off settles into an OpenToolRun notification that opens the run;
- stop works for inline, monitor, and proc owners with accurate consequence copy;
- no UI path reconciles (a patched reconcile raises across the suite);
- the j/k bench p95 is under 16 ms and a quiet tick reloads nothing, with both flag
  states recorded;
- goldens and live captures are inspected;
- the `ace_tool_runs` flag bead is closed;
- `sase tool run check` is green in sase and sase-core.

## 6. Out of scope (candidate follow-ups; record them as PROPOSED FOLLOW-UP notes, do not build)

| Don't build                                                                                       | Reopen when                                                                                   |
| ------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Any run count on rows (D3)                                                                        | Never, as a raw count                                                                         |
| A persistent verdict mark on rows                                                                 | At least 70% of agents' latest checks are pass or KNOWN-only for a week (15% today)           |
| A 5th deck, a 4th top-level tab, or child nodes for inline runs                                   | Runs become many-per-node, or agent-less runs become common                                   |
| A FINAL Verification card, or renaming Tools                                                      | A FINAL _link_ from a receipt's `verdict_provenance` to its run is a fine follow-up           |
| ETAs, or a top-bar `⚒N` live count                                                                | E6 forecasts reach ≥80% interval coverage, or E7 admission gives the count a capacity meaning |
| TUI reconcile, or a "reconcile" button                                                            | A backend job that reconciles silent runs and notifies is the right follow-up                 |
| In-TUI rerun                                                                                      | A CLI rerun contract exists                                                                   |
| Per-turn `⚒` receipts in Reply, and jumpable run ids in bead notes                                | Evidence that users look for runs from Reply or beads                                         |
| Statistics views (adoption, verdict rates, stage hot spots, receipt coverage) in the Tools pane   | After this epic, alongside E6                                                                 |
| Receipt coverage in the Catalog                                                                   | The receipt path gets a cheap read that does not re-fingerprint                               |
| OpenToolRun on mobile and Telegram (they map unknown actions to Unsupported)                      | Mobile or Telegram users hand off tool runs                                                   |
| Fixing the ~9-minute caller kill (`sase-17e`, `sase-17g`) and the retention unit bug (`sase-1bo`) | They have their own beads                                                                     |
| Converging tool procs with `sase-11y` oneshot service procs                                       | Per roadmap §3.2's reopen condition                                                           |
