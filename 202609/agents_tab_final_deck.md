---
tier: epic
title: Finalizers on the Agents tab - FINALIZING rows, Reply receipts, and the ⊛ FINAL
  deck
goal: 'Sase finalizers become first-class on the Agents tab at four zoom levels over
  one provider-neutral data layer. At a glance, rows show a FINALIZING phase and a
  ⊛ chip while finalizers run or after a non-success. In context, each shell''s Reply
  phase ends with a short ⊛ FINAL receipt. To diagnose, a new ⊛ FINAL deck has an
  Overview card plus one card per finalizer instance and one card block per run, with
  attempts, operations, steps, typed evidence, diagnostics and gated live tails. For
  authors, the Overview and a read-only `sase final status` run view explain selection,
  declarations and drift. A controller progress journal (which also records handoff
  skips), uniform operation records, a step channel, bounded live logs, an agent_meta
  summary, and one Rust-core projection feed every surface. Nothing in the model is
  commit-specific, so future finalizers render well on day one.

  '
phases:
- id: core-status-wire
  title: finalizer_status summary field on the Rust agent-scan wire
  depends_on: []
  size: small
  description: 'core-status-wire: in sase-core, add the tolerant FinalizerStatusSummaryWire
    and AgentMetaWire.finalizer_status, coerce it leniently in the scanner, bump the
    artifact-index schema with a no-op refresh migration, and cover parity and tolerance
    with tests.'
- id: core-run-view-model
  title: FinalizerNodeView projection - decoders, precedence, and selection
  depends_on: []
  size: medium
  description: 'core-run-view-model: in sase-core, add the finalizer run_view module
    with request/response wires, tolerant capped decoders for every finalizer artifact,
    the ten source-precedence rules, run disposition and instance statuses, DAG order,
    selection explanation, declaration timeline, cycles, drift and the attention hint.
    Also add the project_finalizer_node_view binding.'
- id: core-run-view-detail
  title: FinalizerNodeView detail - attempts, operations, evidence, and runs
  depends_on:
  - core-run-view-model
  size: medium
  description: 'core-run-view-detail: extend the projection with attempts, operations
    (schema-v1 and legacy commit records), steps, carriage-return-collapsed live tails,
    typed evidence and headline selection, attempt-scoped diagnostic dedupe, warning
    counts, failure reason lines, and multi-run node composition. Prove it with commit,
    command and plugin fixtures.'
- id: journal-and-summary
  title: Controller progress journal, handoff skips, and the row summary writer
  depends_on: []
  size: medium
  description: 'journal-and-summary: add the best-effort finalizers/progress.jsonl
    writer and the agent_meta finalizer_status tracker. Wire run, cycle, declaration,
    recovery, instance and attempt events plus phase_skipped (with a new pending_handoff_kind
    helper) into the controller, write the planned summary at plan seal, and touch
    the refresh pulse on transitions.'
- id: operation-records
  title: One uniform operation record across every executor
  depends_on:
  - journal-and-summary
  size: medium
  description: 'operation-records: add an OperationRecorder that writes schema-v1
    attempt-N.<op>.outcome.json records and journal op events for builtin@command,
    plugin execute/verify (plus preflight describe/validate), commit stitches (extending
    today''s outcome.json) and conflict-repair model turns. Fix the conflict-repair
    hard-coded commit instance id.'
- id: step-channel-and-live-sink
  title: Step channel, stitch steps, and bounded live output
  depends_on:
  - operation-records
  size: medium
  description: 'step-channel-and-live-sink: add SASE_FINALIZER_STEPS_FILE, emit_step
    and the SDK step() helper, and structured sase stitch create steps including warn
    steps. Tee subprocess output to bounded rotating .live files from run_bounded_subprocess,
    and refresh the summary''s step and warnings through a throttled wait-loop tick.'
- id: status-summary-adapter
  title: Python mirror and Agent model field for finalizer_status
  depends_on:
  - core-status-wire
  size: small
  description: 'status-summary-adapter: move the sase-core pin past core-status-wire,
    mirror finalizer_status on the Python AgentMetaWire with a tolerant nested converter,
    add the Agent model field through both enrichment paths and dedup, bump the index
    schema constant, and add the validate_sase_core_rs scan probe. There is no visible
    change.'
- id: glance-surfaces
  title: FINALIZING rows, ⊛ chips, header chip, and Reply receipts behind ace_final_deck
  depends_on:
  - status-summary-adapter
  - journal-and-summary
  size: medium
  description: 'glance-surfaces: create the ace_final_deck beta flag and the shared
    finalizer view vocabulary. Add pure row-state helpers, the FINALIZING status word
    in the Running bucket, ⊛ row chips with the session supersede rule, the identity-header
    activity chip, and the ⊛ FINAL Reply receipt at every Reply assembly site including
    hint twins. Test with the flag on and off.'
- id: deck-spec-registry
  title: DeckSpec registry and explicit per-deck dispatch
  depends_on: []
  size: medium
  description: 'deck-spec-registry: replace the parallel per-deck tables with one
    DeckSpec record per deck and an active_deck_cycle() accessor. Turn every else-means-Tools
    fall-through into explicit dispatch, and route cycle, subtitle, picker, catalog,
    layout and persistence through the accessor, with no behavior or pixel change.'
- id: per-deck-preferred-cards
  title: Per-deck sticky preferred cards
  depends_on:
  - deck-spec-registry
  size: small
  description: 'per-deck-preferred-cards: replace DeckPanelState''s single preferred_card
    slot with a per-deck mapping. Scope card cycling, re-show and resolve_active_card
    to the focused deck, and persist a preferred_cards map inside schema v1 while
    still writing and reading the legacy preferred_card key as Main''s preference.'
- id: card-document-view
  title: A generic card-document view and block host beyond Main
  depends_on:
  - deck-spec-registry
  size: medium
  description: 'card-document-view: extract a deck-parameterized CardDocumentView
    from MainDeckView. Generalize the spread/paged decision, measurement cache namespace,
    separators, scroll watching, transitions and the DeckPanelBlocksMixin block host
    so any card-document deck gets cards, spread, anchors and card blocks. Main is
    unchanged.'
- id: run-view-adapter
  title: Python run-view facade, artifact collector, and end-to-end proof
  depends_on:
  - core-run-view-detail
  - step-channel-and-live-sink
  - status-summary-adapter
  size: medium
  description: 'run-view-adapter: move the pin past core-run-view-detail and add the
    typed finalizer_run_view facade and binding-guard registrations. Add a TUI-agnostic
    capped artifact collector with liveness facts and stat signatures, and an end-to-end
    test that drives the real controller and executors into the projection.'
- id: final-cli-status
  title: Read-only sase final status run view
  depends_on:
  - run-view-adapter
  size: small
  description: 'final-cli-status: add `sase final status [<agent>]` with -d/--artifacts-dir
    and -f/--format pretty|json over the shared collector and projection. It prints
    colored output in the shared vocabulary, keeps help alphabetical, and defaults
    to the calling agent inside a SASE turn.'
- id: final-deck-shell
  title: Register the ⊛ FINAL deck with its loader, availability, and chrome
  depends_on:
  - glance-surfaces
  - per-deck-preferred-cards
  - card-document-view
  - run-view-adapter
  size: medium
  description: 'final-deck-shell: add DeckId.FINAL behind ace_final_deck at every
    registration site. Add the off-thread FinalDeckView loader with subject/generation
    stale rejection and a stat-signature cache, the no-I/O availability probe, the
    status-colored subtitle and tab status strip, the default-card rule, sticky FINAL
    cards, pinned-attempt support, and the p n / p N keys.'
- id: final-overview-card
  title: The Overview card
  depends_on:
  - final-deck-shell
  size: small
  description: 'final-overview-card: render the run-level Overview card. It shows
    the plan in DAG order with selection reasons, configured-but-unselected instances
    dimmed, the declaration timeline, controller cycles only when above one, drift,
    run-level diagnostics, the runs ledger, calm skipped/not-reached/unavailable states
    and CLI pointers.'
- id: final-instance-cards
  title: Generic instance cards with commit and command enrichers
  depends_on:
  - final-deck-shell
  size: medium
  description: 'final-instance-cards: render one provider-neutral card per finalizer
    instance with why/trigger/declared lines, attempt sections (latest or failing
    expanded), operations, steps, typed evidence, deduped diagnostics, log and protocol
    hint targets, export and search. Add additive builtin@commit and builtin@command
    enrichers, proven against a plugin fixture.'
- id: final-run-blocks
  title: One card block per run on session containers
  depends_on:
  - final-overview-card
  - final-instance-cards
  size: small
  description: 'final-run-blocks: give every FINAL card on a session container one
    CardBlock per shell that ran finalizers, with roster-matched BlockMeta and block
    ids. Skipped and not-triggered shells appear only in the ledger. The rail, [ /
    ] and newest landing work in FINAL through the generalized block host.'
- id: final-live
  title: Live tails, following, and the 1 Hz tick for the selected agent
  depends_on:
  - final-run-blocks
  size: medium
  description: 'final-live: add the ace.agent_decks.final_tail_delay_seconds gate
    and a sanitized in-card live tail of at most 12 lines. The tail follows the newest
    run and attempt with the arrival marker, pauses on scroll-up, and runs a pump-free
    1 Hz elapsed/tail refresh only while FINAL shows the selected agent''s active
    finalization.'
- id: final-cutover
  title: Remove the flag, add goldens, inspect live, and bench
  depends_on:
  - final-live
  - final-cli-status
  size: medium
  description: 'final-cutover: bench the j/k and triage loop with the flag off and
    on, then delete the Off branches and the ace_final_deck registry entry and close
    its flag bead. Add deterministic goldens for every glance and deck state, re-baseline
    and inspect the changed subtitle and picker goldens, and inspect live captures.'
- id: final-docs
  title: User and plugin-author docs for finalizer visibility
  depends_on:
  - final-cutover
  size: small
  description: 'final-docs: document the FINALIZING row phase, chips, receipts, the
    FINAL deck, states, keys and live behavior in ace.md. Add the tail-delay key and
    picker letters to configuration.md, the step channel, typed evidence and operation
    records to plugins.md, and sase final status to the CLI docs. Record glossary
    follow-ups as PROPOSED FOLLOW-UP notes.'
proposed_by: bbugyi200.athena.0sr
create_time: 2026-09-27 05:49:26
status: done
bead_id: sase-1b2
---

- **PROMPT:** [prompts/202609/agents_tab_final_deck.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agents_tab_final_deck.md)
- **BEAD:** [sase-1b2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1b2/README.md)

# Plan: Finalizers on the Agents tab

## 1. Context

Finalizers are how a SASE turn lands its work: commit, check, tasks, and plugin
finalizers run after the model turn under host control. None of this shows on the Agents
tab. The research for this plan measured the following:

- **Post-turn window.** Between the model finishing and the finalizer result there is a
  median of **~108 s** (p90 265 s), and the row says `RUNNING` the whole time.
- **Rejected declarations.** 41% of runs had at least one rejected `sase final submit`.
- **Invisible warnings.** 96% of successful commit stitches print `⚠` warnings that only
  exist in finalizer stdout.
- **Unexplained skips.** In 15 days, 102 plan shells sealed a finalizer plan that was
  correctly skipped because of a pending handoff, and they left no record explaining
  why.

**Research.** Read the consolidated report with
`sase artifact read research:202609/agents_tab_finalizer_visibility/agents_tab_finalizer_visibility.md "<why>"`.
It holds the evidence, the four-researcher disagreements and how they resolve, the ASCII
mockups, and the risk table. This plan adopts its §10 recommendation and turns its §6
delivery table into phases. Where this plan and the report differ, the plan wins (§2
records each refinement).

**What changed since the research (verified on master `ee40b14447`).**

- **Card blocks shipped** (epic `sase-19x`). `CardBlock`/`BlockMeta`/`BlockSpreadOnly`,
  the block cursor, the rail and `[`/`]` exist under `src/sase/ace/tui/widgets/decks/`.
  Run blocks are therefore in scope now as `final-run-blocks`, not a deferred phase 4.
- **Card-block machinery is Main-only.** `panel_blocks.py` gates on `DeckId.MAIN` in
  about six places and reads `self.main_view`/`self._main_document` directly.
- **One preference slot.** `DeckPanelState.preferred_card` is a single slot shared by
  all decks (`widgets/decks/model.py`).
- **Adding a deck is not safe today.**
  - `panel_chrome._FALLBACK_ACCENTS` raises `KeyError` for any deck without an entry.
  - Several chains treat "not Main or Files" as Tools: `_agent_detail_decks.show_deck`,
    `panel._deck_is_empty`, `panel_chrome.refresh_chrome` tabs, and the footer card
    count.
- **No live finalizer state.** Nothing is written while finalizers run.
  - `finalizer_result.json` is written only at terminal points.
  - The skip branch (`controller.py` `_should_skip_finalizers` → return) writes nothing.
  - `finalizers_drift` is written to `agent_meta.json` and never read.
- **Executor records differ:**
  - Commit writes `attempt-N.<label>.outcome.json`.
  - Command writes `attempt-N.{stdout,stderr,diagnostics.json}` with no duration.
  - Plugin writes no timing or JSON outcome, and its `describe`/`validate` outputs are
    overwritten non-exclusively.
  - Conflict repair hard-codes the instance id `"commit"` (`commit_repair_conflict.py`).
- **The scan wire drops unknown keys.** The Agents tab reads `agent_meta.json` through
  the Rust scan wire `AgentMetaWire`, which drops unknown keys. A new summary key needs
  the Rust wire, the scanner's exhaustive struct literal, an index-schema bump, the
  Python mirror, and both enrichment paths (`_meta_enrichment_wire.py`,
  `_meta_enrichment_filesystem.py`).

**Rust boundary** (core memory `rust_core_backend_boundary`):

| Concern                                                                                                                                                                                                                                 | Owner                                                                                                             |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Reconciling artifacts into "what happened, in what state, and why" (source precedence, disposition, statuses, selection explanation, typed evidence, headline choice, dedupe, warning counts, live-tail cleanup, multi-run composition) | `sase-core` (`finalizer::run_view`). Any frontend (TUI, `sase final status`, a future web view) must agree on it. |
| The row summary's scan shape                                                                                                                                                                                                            | `sase-core` (`agent_scan::wire`)                                                                                  |
| Producers: journal, operation records, step channel, live sink, summary writer                                                                                                                                                          | Python `src/sase/finalizers/` (they run inside the controller and executors)                                      |
| Glyphs, colors, layout, cards, keys, loaders                                                                                                                                                                                            | Python TUI (presentation only)                                                                                    |

## 2. Binding decisions

The research's §9 open questions are settled here with its recommendations. A few
details are refined, each marked **(refined)** with its reason. The user can override
any of them during plan review.

| #   | Decision                                                                                                                                                                                                                                                                                                                                                                                     | Why                                                                                                                                                  |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | **Four zoom levels:** glance (row), in context (Reply receipt), diagnose (FINAL deck), author (Overview + `sase final status`)                                                                                                                                                                                                                                                               | Each question lives on a different surface, so only diagnose needs new deck/card/block structure                                                     |
| D2  | **Deck identity:** name `FINAL`, glyph `⊛` (U+229B, single-cell), picker key `n` (`N` = other panel), appended **after Tools**, count noun `finalizer`/`finalizers`. Accent candidate is rose `#FF87D7`; `final-deck-shell` confirms it stays distinct from the Main/Files/Tools accents and the clan magenta in a live capture, adjusts it if not, and records the value in its close note. | Matches `%final`/`sase final`/`final_context.json`. `⛭` is rejected because its width is ambiguous.                                                  |
| D3  | **Cards:** `overview` (tab `Overview`) plus one per selected instance, id `instance:<instance_id>` (tab `<id> <glyph>`)                                                                                                                                                                                                                                                                      | Instance ids are stable config keys, so the chosen card sticks across j/k. The `instance:` prefix means an instance named `overview` cannot collide. |
| D4  | **Card block = one run = one concrete shell's finalizer execution.** Block ids use `card_block_id(agent.identity)`, the same ids Reply uses.                                                                                                                                                                                                                                                 | Keeps the single card-block meaning. Attempts are sections inside a run.                                                                             |
| D5  | Configured-but-unselected instances **appear dimmed in Overview with the reason** (`%final:!lint`, `%final:none`, not default)                                                                                                                                                                                                                                                               | This is the most useful single answer to "why didn't my finalizer run?"                                                                              |
| D6  | A sticky FINAL card **wins over attention** while pressing j/k. The status strip and the subtitle carry the signal instead.                                                                                                                                                                                                                                                                  | Consistent with Main.                                                                                                                                |
| D7  | The chronic stitch warnings are **out of scope**; they already have owners (`sase-10x` and a note on `sase-yy.8.6`). Warnings never color rows: a dim `⚠N` suffix in the receipt, detail only in FINAL.                                                                                                                                                                                      | 96% of successes carry warnings, so a warning chip would be permanent noise                                                                          |
| D8  | Handoff-skipped shells get **no Reply receipt**. They appear as `○ skipped · plan handoff` in the Overview and the runs ledger.                                                                                                                                                                                                                                                              | A plan shell never lands work; its successor's receipt tells the story                                                                               |
| D9  | **(refined)** The receipt appears only once finalization **has begun** (summary phase `declaring`/`executing`/`settled`, or derived `interrupted`), never during `planned`. FINAL itself does show the planned state.                                                                                                                                                                        | A `◌ planned` receipt under every streaming reply is noise, and the deck already answers "what will run"                                             |
| D10 | **(refined)** A session row chip considers only runs **after the newest successful settled run**. Among those it picks the highest severity (failed > refused > interrupted > deferred > running) and adds `k of n runs` when n > 1.                                                                                                                                                         | "Severity first" over all runs would pin `⊛✗` on a session whose later feedback round landed cleanly                                                 |
| D11 | **(refined)** `FINALIZING` is a **presentation overlay** from a dedicated helper. `agent.status`, `display_status` (≈90 consumers), status buckets, ordering, filters, capacity, `agent_row_is_in_flight` and actions are all untouched.                                                                                                                                                     | A raw status change would ripple through runtime ticking, mirroring and capacity                                                                     |
| D12 | v1 is **read-only**: no retry, cancel, or bypass controls                                                                                                                                                                                                                                                                                                                                    | Those carry authorization and fixed-point semantics                                                                                                  |
| D13 | The per-agent run view is **`sase final status`** (`show` is taken by the configured-instance view)                                                                                                                                                                                                                                                                                          | Settled under `cli_rules.md`                                                                                                                         |
| D14 | **Additive data only.** There is no finalizer execution-wire bump, and the strict provider protocol is unchanged. Every new artifact is best-effort observability that never changes a verdict.                                                                                                                                                                                              | Observability must never break landing                                                                                                               |
| D15 | One **`ace_final_deck` beta flag** gates every user-visible surface until `final-cutover` deletes it                                                                                                                                                                                                                                                                                         | Phases land piecemeal                                                                                                                                |
| D16 | Structured progress comes **only** from the step channel. **Never scrape emoji** from stdout.                                                                                                                                                                                                                                                                                                | Scraping breaks silently when wording changes                                                                                                        |

Naming hygiene: help text says **"finalizer attempt"**, because "attempt" already means
agent retries (`D`, `↻2`). A gate's "finalize proc" is not a finalizer and must never
render with `⊛`.

## 3. Design specification (all phases implement against this)

### 3.1 Vocabulary

| Term                  | Meaning                                                                                                                                                         |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Run**               | One concrete shell's finalizer execution (an agent turn, or a monitor turn doing host completion). It is keyed by the artifacts dir passed to `run_finalizers`. |
| **Instance**          | One selected finalizer (`commit`, `check`, `acme@open-pr`, …) inside a run                                                                                      |
| **Finalizer attempt** | One budgeted try of an instance (`max_attempts`)                                                                                                                |
| **Operation**         | One unit of work inside an attempt: `subprocess`, `model_turn`, `validation`, or `internal`                                                                     |
| **Step**              | Optional structured progress inside an operation (the step channel)                                                                                             |
| **Node view**         | The projection for one selected Agents-tab node: one run for a lone turn, or every member shell's run for a session container                                   |

### 3.2 State vocabulary

Every state shows glyph + word + color together. The mapping lives in one shared module,
`src/sase/finalizers/view_vocabulary.py` (created by `glance-surfaces`), so the TUI and
`sase final status` agree. Colors reuse the existing palette.

| State                                    | Glyph                                            | Word                                                   | Color                                                     | Level    |
| ---------------------------------------- | ------------------------------------------------ | ------------------------------------------------------ | --------------------------------------------------------- | -------- |
| planned / waiting on `after`             | `◌`                                              | `planned` / `after X`                                  | dim                                                       | instance |
| declaring (declaration or recovery turn) | `▶`                                              | `declaration`                                          | `#FFD700` (RUNNING_COLOR)                                 | run      |
| running / retrying                       | `▶`                                              | op or step label, plus `attempt n/m` only when `m > 1` | `#FFD700`                                                 | instance |
| success                                  | `✓`                                              | `success`                                              | `#5FD75F`                                                 | both     |
| success with warnings                    | `✓` + `⚠N`                                       | `success`                                              | green + amber `#FFAF5F` count (**deck and receipt only**) | instance |
| failed                                   | `✗`                                              | `failed`                                               | `#FF5F5F`                                                 | both     |
| refused                                  | `⊘`                                              | `refused`                                              | `#D75FFF`                                                 | both     |
| deferred                                 | `⏸` (fall back to `‖` if a golden shows it wide) | `deferred`                                             | `#FFAF5F`                                                 | both     |
| not triggered                            | `○`                                              | `not triggered`                                        | dim                                                       | instance |
| skipped · handoff                        | `○`                                              | `skipped · <kind> handoff`                             | dim                                                       | run      |
| not run · blocked                        | `–`                                              | `not run · blocked by X`                               | dim                                                       | instance |
| not reached                              | `–`                                              | `not reached`                                          | dim                                                       | run      |
| interrupted                              | `!`                                              | `interrupted`                                          | `#FFAF5F`                                                 | both     |
| unavailable                              | `⚠`                                              | `unavailable · <why>`                                  | dim amber                                                 | run      |

Additional rules:

- The deck accent is a single hue (D2).
- `▶` matches the roster's running glyph, so rails and tabs agree with Reply.
- An `interrupted` or `unavailable` state never spins.

### 3.3 Data contracts (producers and consumers are built in parallel against these)

**C1. Progress journal: `<artifacts_dir>/finalizers/progress.jsonl`**

- **Writer.** The controller process running `run_finalizers` is the single writer. The
  file is append-only UTF-8 JSONL, one object per `\n`-terminated line, and each line is
  written with one `write` plus a flush.
- **Record fields.** Every record carries
  `{"v": 1, "seq": <int, 1-based, monotonically increasing across the file>, "t": <unix float>, "event": "<name>", …}`.
- **Ceiling.** `PROGRESS_JOURNAL_MAX_BYTES = 256 KiB`. When the next record would cross
  it, write one `{"event": "observability_truncated"}` record and stop journaling for
  the rest of the process.
- **Segments.** A `phase_started` record begins a new segment. Readers use the **last**
  segment and report how many earlier segments there were.
- **Best-effort.** Every I/O or serialization error is swallowed (logged at debug) and
  never changes control flow, verdicts, or result bytes.

| Event                                              | Extra fields                                                                                                                                                                      | Written                                                                                                         |
| -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `phase_skipped`                                    | `reason: "handoff:<plan\|questions\|monitor\|gate\|pipe>"`                                                                                                                        | At the skip branch, only when `artifacts_dir` exists and a handoff marker matched. There is no `phase_started`. |
| `phase_started`                                    | `run_id` (uuid4 hex), `plan_digest` (str\|null), `mode` (`normal`\|`no_model`), `runner: {pid, identity}` (`identity` = `core.process_identity.process_identity_token()` or null) | Right after the skip check                                                                                      |
| `cycle_started`                                    | `cycle`                                                                                                                                                                           | Each controller cycle                                                                                           |
| `declaration_started` / `declaration_finished`     | finished: `status` (`accepted`\|`rejected`\|`recovered`\|`failed`), `code?`                                                                                                       | Around `ensure_current_declaration`                                                                             |
| `recovery_turn_started` / `recovery_turn_finished` | finished: `ok`, `code?`                                                                                                                                                           | Around the declaration-recovery model turn                                                                      |
| `instance_started` / `instance_finished`           | `instance_id`; finished: `status`, `code?`                                                                                                                                        | Per instance per cycle                                                                                          |
| `attempt_started` / `attempt_finished`             | `instance_id`, `attempt`, `max_attempts`; finished: `status`, `code?`                                                                                                             | When the ledger allocates and settles an attempt                                                                |
| `op_started` / `op_finished`                       | `instance_id`, `attempt` (int\|null), `op`, `kind`, `label`; finished: `returncode?`, `duration_seconds`, `timed_out`                                                             | Around every operation (`operation-records`)                                                                    |
| `phase_finished`                                   | `status` (aggregate), `cycles`                                                                                                                                                    | In the controller's `finally`, after the aggregate result write                                                 |
| `observability_truncated`                          | —                                                                                                                                                                                 | At the ceiling                                                                                                  |

**C2. Operation record: `finalizers/<instance_id>/attempt-<N>.<op>.outcome.json`**

The file is write-once and exclusive, exactly as today's commit record. Operations that
run before an attempt exists (plugin `describe`/`validate`) write
`preflight.<op>.outcome.json` instead, replaced atomically, with `attempt: null`.

```json
{
  "schema_version": 1,
  "op": "<filename-safe label, unique within the attempt>",
  "kind": "subprocess | model_turn | validation | internal",
  "label": "<short human label, e.g. 'stitch main', 'just check', 'execute'>",
  "attempt": 1,
  "argv": ["…"],
  "started_at": 1727440000.0,
  "duration_seconds": 12.3,
  "returncode": 0,
  "timed_out": false,
  "stdout_truncated": false,
  "stderr_truncated": false,
  "logs": {
    "stdout": "attempt-1.run.stdout",
    "stderr": "attempt-1.run.stderr",
    "live": "attempt-1.run.live"
  },
  "steps": "attempt-1.run.steps.jsonl",
  "prompt": "…",
  "response": "…"
}
```

Field rules:

- `argv`, `returncode`, `logs.*`, `steps`, `prompt` and `response` are optional.
- `prompt`/`response` appear on `model_turn` only.
- `logs` values are **filenames relative to the instance dir**. For example, command
  keeps its existing `attempt-N.stdout`/`.stderr` names and points at them.
- Commit's existing keys (`message_file`, …) stay.
- Readers treat a record with no `schema_version` as a legacy commit record.

**C3. Step channel**

- **Env var.** `SASE_FINALIZER_STEPS_FILE` holds the absolute path of
  `attempt-<N>.<op>.steps.jsonl`. It is set for every attempt-scoped finalizer
  subprocess: command, plugin execute/verify, and stitch.
- **Record.**
  `{"v": 1, "t": <float>, "step": "<≤120 chars>", "state": "start|ok|warn|fail", "detail": "<≤500 chars>"?}`.
- **Ceiling.** 64 KiB per file; writers stop silently at the ceiling.
- **Helper.** `sase.finalizers.steps.emit_step(step, state="start", detail=None)`. It
  does nothing when the variable is unset and never raises. The SDK re-exports it as
  `sase.finalizers.sdk.step`.
- **Plugin workers** write steps only to the file. Stdout remains the JSON result
  channel.

**C4. Live sink: `attempt-<N>.<op>.live`**

- **Content.** Output chunks exactly as received (stdout and stderr interleaved; plugins
  tee **stderr only**).
- **Rotation.** At 512 KiB the file is moved to `.live.1` (replacing it) and a new file
  starts.
- **Failure handling.** A sink error disables the sink for that op and is swallowed.
- **Cleanup.** On normal completion, after the terminal `stdout`/`stderr` artifacts are
  written, both `.live` files are removed. They are **retained** on timeout, kill, or a
  terminal-write failure.
- **Unchanged.** The terminal exclusive artifacts are unchanged. The sink never creates
  their names.

**C5. Row summary: `agent_meta.json["finalizer_status"]`**

It is written through `update_meta_field`, followed by
`touch_turn_refresh_pulse(project_name_from_artifacts_dir(artifacts_dir))`.

```json
{
  "schema_version": 1,
  "phase": "planned | skipped | declaring | executing | settled",
  "reason": "handoff:plan | <aggregate diagnostic code> | null",
  "status": "success | failed | refused | deferred | null",
  "plan_digest": "…",
  "run_id": "…",
  "started_at": 1727440000.0,
  "updated_at": 1727440012.5,
  "runner": { "pid": 1234, "identity": "…" },
  "instances": [
    {
      "id": "commit",
      "status": "planned | waiting | running | success | failed | refused | deferred | not_triggered | skipped | not_run",
      "attempt": 1,
      "max_attempts": 1,
      "op": "stitch main",
      "step": "before hook: just fix",
      "started_at": 1727440001.0,
      "finished_at": null,
      "headline": "commit 8bb7e55",
      "warnings": 0,
      "reason": null
    }
  ],
  "instance_count": 1
}
```

**Write rules:**

- **When.** At plan seal (`planned`, every instance `planned`); at `phase_started`
  (`declaring`); when the first instance starts (`executing`); at every instance,
  attempt and op transition; at `phase_finished` (`settled` with `status`); and at the
  skip branch (`skipped` with `reason`).
- **Step updates.** A change to `step` alone is throttled to at most one write every 2
  s. The last value is always flushed at op end.
- **Instances.** At most 16 entries, in plan order; `instance_count` holds the true
  count.
- **String caps.** `op`, `step`, `headline` and `reason` are capped at 120 characters.
- **`reason` per instance status:**

  | Status   | `reason`                                                              |
  | -------- | --------------------------------------------------------------------- |
  | failed   | the first error diagnostic message, or the last non-blank stderr line |
  | deferred | `"<deferral reason> · <n> paths"`                                     |
  | refused  | the refusal reason                                                    |
  | not_run  | `"blocked by <id>"`                                                   |

- **`headline`** is chosen generically: SHA (`commit <short>`), then URL, then bead id,
  then `exit <code>`.
- **`warnings`** counts warn steps plus warning diagnostics in the latest attempt.

**Derived by readers, never written:**

- `interrupted`: the owning turn is terminal while `phase` is `declaring`/`executing`.
- `not reached`: the turn is terminal while `phase` is still `planned`.

**Readers** keep every field as a (capped) string or number so unknown future values
survive. Unknown values render as a neutral `•`.

### 3.4 The shared projection (`sase-core`: `finalizer::run_view`)

- **Binding.** `project_finalizer_node_view(request) -> FinalizerNodeViewWire`, bound in
  the `bead_decisions` binding domain (which already binds `finalizer`). The projection
  runs under `py.allow_threads`.
- **Caller.** Python calls it only off the event loop.

**Request.** `{schema_version: 1, runs: [RunInput…], tail_lines: 12}`. Each `RunInput`
carries:

| Field                         | Content                                                                                                                                                                                                                                                                                                                                                                                                  |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Identity (supplied by Python) | `run_id` (the block id string), `number`, `label`, `kind` (`agent`\|`monitor`)                                                                                                                                                                                                                                                                                                                           |
| Liveness facts                | `turn_terminal: bool`, `runner_live: bool\|null`                                                                                                                                                                                                                                                                                                                                                         |
| Text inputs                   | each `{text: str\|null, size: int, too_large: bool}`, decoded UTF-8 with replacement: `agent_meta`, `plan` (`finalizer_plan.json`), `authority_plan` (`finalizer_plan.authority.json`), `context` (`final_context.json`), `submission` (`final_submission.json`), `submission_attempts` (`final_submission_attempts.jsonl`), `journal` (`finalizers/progress.jsonl`), `result` (`finalizer_result.json`) |
| `instances[]`                 | `{instance_id, files: [{name, size, mtime_ns, text?: TextInput, line_count?: int, tail?: str}]}`. Operation records, steps and `diagnostics.json` carry `text`. Logs carry `size`/`line_count`. Active `.live` files carry a ≤16 KiB `tail`. Failed ops' stderr carries a ≤4 KiB `tail`.                                                                                                                 |
| `recovery_files[]`            | Names present among `final_declaration_recovery_{evidence,prompt,response}.md`                                                                                                                                                                                                                                                                                                                           |

**Ceilings.** Every JSON input over **4 MiB** is `too_large` and never parsed. It yields
an `unavailable` sub-state whose "too large" reason names the raw path, so `v`/`E` still
reach it.

**Source precedence** (the research's §5.3, verbatim in intent):

1. A plan integrity, digest, or scope mismatch → run `unavailable`, never a best-effort
   success.
2. A valid terminal result with matching identity is authoritative for outcomes. The
   result is a schema-v1 artifact with `cycles`, parsed by its own tolerant decoder,
   **not** the strict aggregate wire.
3. A journal `phase_skipped` → run `skipped · handoff:<kind>`.
4. The journal plus `runner_live == true` is authoritative for the active phase,
   instance, attempt and op.
5. `final_context.json` supplies trigger and submission facts. Submission attempts
   supply the declaration timeline.
6. Operation records, steps and live tails supply content only, never status.
7. The `agent_meta` summary is a row hint, never detail truth. The projection reads only
   `finalizers_drift` and the plan-seal `finalizers` block from `agent_meta`.
8. An open journal with a dead or mismatched runner, or an open journal on a terminal
   turn → `interrupted`.
9. A plan with no journal and no result on a terminal turn → `not reached`: calm, never
   `interrupted` or `failed`.
10. A planned instance never reached after an upstream terminal outcome →
    `not run · blocked by X`.

**Response** (`FinalizerNodeViewWire`):

- **Node aggregate:** status, glyph key, duration, `attention_instance_id`, and
  `run_level_trouble` (a recovery turn, drift, an integrity failure, or a non-success
  run-level diagnostic).
- **`instances[]`:** the union across runs in DAG order (plan `resolved_index` +
  `after`). Each carries its provider ref, its selection reason (`default`, `required`,
  `%final:<id>`), `after` ids, and per-run appearances.
- **`unselected[]`:** configured-but-unselected instances, each with `instance_id`,
  `provider_ref` and reason (`%final:!<id>`, `%final:none`, `not default`). These are
  read from the authority plan's `config_snapshot` instance ids, provider refs,
  `defaults` and `required` **only**.

  > **Secrecy rule.** The projection never outputs config values.

- **`runs[]`:** per run:
  - disposition (`active`, `ran`, `skipped`, `not_reached`, `interrupted`,
    `unavailable`) and its reason
  - `run_id`/`number`/`label`/`kind`, plan digest, timing, cycles
  - the declaration timeline (time, accepted/rejected, code, first message line, payload
    count) and the recovery turn
  - drift, run-level diagnostics
  - per-instance detail:
    - status, trigger (kind, `submission_required`, obligation count), and the declared
      payload summary (top-level keys → short values, capped)
    - attempts: number, status, timing, code
    - operations: op, kind, label, argv, timing, exit, `timed_out`, truncation, log
      metadata and relative paths, steps, live tail lines
    - typed evidence:
      `{kind, value, type: sha|url|bead|path|duration|exit_code|text, display}`,
      classified from `*_sha`, `*_url`, `bead_id`, `*_path`, `*_seconds`, `exit_code`
    - headline evidence (SHA → URL → bead → exit code)
    - deduped diagnostics with **attempt-scoped severity**: errors from superseded
      attempts of an eventually successful instance are downgraded to `superseded` and
      never paint red
    - warnings count, refusal reason, deferral `{reason, paths}`, the one failure reason
      line, and protocol-envelope file names for plugins

**Live tails.** Collapse `\r`-separated segments to the last segment of each line, keep
the last `tail_lines` lines, and preserve ANSI (Python renders it). There is **no
commit-specific field anywhere** in the request or response.

### 3.5 Glance surfaces (flag-gated)

**Row state** comes from pure helpers over the in-memory `Agent` (no I/O), in
`src/sase/ace/tui/models/finalizer_row_state.py`:

- `FINALIZING` (bold `#FFD700`) replaces the `RUNNING` word only when the row would
  render `RUNNING` and the summary phase is `declaring`/`executing`. The bucket stays
  Running (D11).
- The chip renders after the status parenthesis, beside `⚑`, only for running,
  declaring, failed, refused, deferred or interrupted:

  ```text
  ▶ sase-1ab--code    (FINALIZING)  ⊛ commit · just fix 1:42
  ✗ sase-1c9--code    (FAILED)      ⊛✗ check
  ✓ research.b.cld    (DONE)        ⊛⏸ commit
  ✓ sase-1d2--code    (DONE)                       ← success: silence
  ```

  - The label is the instance id, then the step label, else the op label.
  - Elapsed time is shown only if the row's existing per-second runtime tick re-renders
    the left segment. Never add a timer for it.

- **Session containers** aggregate their members in memory by the D10 rule
  (`⊛✗ check · 1 of 2 runs`).
- The render key and `_runtime_signature` gain a compact summary token
  (`phase, status, updated_at, instance_count`). The docstring rule applies: adding a
  visible field is a deliberate key edit.

**Identity header.** While FINALIZING, the compact header's existing activity-chip slot
shows `⊛ finalizing · commit · just fix` (or `⊛ declaration`). No new header lane is
added.

**Reply receipt** (`prompt_panel/_agent_finalizer_receipt.py`):

- **Construction.** It is built only from the member shell's summary and returns plain
  `Text`, so the hint flatteners accept it.
- **Placement.** It is appended to the **end of each shell's phase renderables**, so it
  belongs to that shell's card block.
- **Content:** a
  `render_phase_divider("FINAL", started_at, glyph="⊛", accent=<deck accent>)` line (no
  block meta), then one line per selected instance in plan order:
  - glyph, id, detail, and a right-aligned duration
  - a dim `⚠N` suffix when there are warnings
  - exactly one indented reason line on failure
  - a dim `p n  open FINAL deck` hint after a non-success, once the deck exists

  ```text
  ─── ⊛ FINAL ─── 07:31:40 ────────────────────────────────
    ✗ check    command_failed · attempt 2/2            3m40s
               FAILED tests/ace/tui/test_final_deck.py::test_receipt
    – tasks    not run · blocked by check
               p n  open FINAL deck
  ```

- **Where no receipt appears:** handoff-skipped shells, zero selected instances, no
  summary (legacy), and the `planned` phase (D9).

**Assembly sites** (all of them, including hint twins):

1. Session `render_phase`, non-hint and hint, in
   `_agent_display_agent_session_render.py`
2. The legacy `render_legacy_phase` in `_agent_display_render.py`, and its hint twin in
   `_agent_display_hint_body.py`
3. Lone-turn `AGENT CHAT`/`AGENT REPLY` in `_agent_display_render.py`, and their hint
   twins
4. Monitor phases that did host completion

Add one-line calls only; the logic lives in the new module (toobig).

### 3.6 The FINAL deck

- **Registration.** `DeckId.FINAL = "final"` is registered through the DeckSpec
  registry: name `FINAL`, glyph `⊛`, key `n`, blurb "how this node's turns landed", and
  the D2 accent. It is part of `active_deck_cycle()` only while `ace_final_deck` is on.
  A persisted `final` panel decodes to Main while the flag is off.
- **Availability** is a no-I/O probe over the Agent model's summaries.
  - It has content when the node's runs have **≥ 1 selected instance**, including a
    handoff-skipped run, whose Overview explains the skip. Its count is the number of
    distinct selected instance ids.
  - It is unavailable (dim) for clans, tribes, proc nodes, and legacy runs with no
    summary.
- **Subtitle.** `final <glyph>` in the status color (`final ✓`, `final ▶`, `final ✗`)
  replaces a count, so a landing failure shows in the border while you read Main or
  Files. `_build_switcher` gains styled-segment support.
- **Tabs double as a status strip:** `Overview │ commit ✓ │ check ✗`. The compact title
  tier applies when there are many instances, like Files over 4 tabs.
- **Default card**, in order:
  1. the panel's sticky FINAL preference, if that card exists for this node
  2. else `attention_instance_id`
  3. else `overview` when `run_level_trouble`
  4. else the first instance card
- **Stickiness.** `Ctrl+J/K` choices stick **per deck** (`per-deck-preferred-cards`), so
  a FINAL sticky card never overwrites Main's sticky Reply.
- **Document.** FINAL is a card document shown by a `CardDocumentView` subclass
  (`FinalDeckView`), so it inherits:
  - spread/paged with its own measurement namespace
  - anchors and heavy `━━ ⊛ commit ━━` spread separators
  - search (`,/`), export (`E`) of the active card's text, zoom, and split
  - card blocks (`final-run-blocks`)
- **Loader.** It models the LLM Calls panel:
  - it paints any cached projection instantly, otherwise a loading line
  - then it runs `run_worker(thread=True)`, which collects inputs, projects through the
    binding, and builds Rich renderables
  - stale results are rejected by subject identity **and** detail generation
  - a stat-only `(mtime_ns, size)` signature cache skips re-projection when nothing
    changed
  - it reloads when the agents surface drifts (the summary writer touches the pulse)
  - there is **no artifact I/O** in `compose`, `render`, keystroke or pump paths
- **Pinned attempts.** With `D`-pinned prior agent attempts, FINAL follows the pinned
  attempt's artifacts dir when it resolves, mirroring Tools.
- **Hint targets.** `v` offers operation logs (`stdout`/`stderr`/`live`), steps,
  protocol envelopes, recovery and conflict-repair prompts and responses, raw artifacts
  for `unavailable` states, and a `↗ commit view` link. Reuse the existing Main hint
  machinery (`prompt_panel/_agent_display_hints.py`, `_file_path_hints.py`) and the
  commit viewer (`prompt_panel/_agent_commits.py`).
- **Read-only** (D12).

### 3.7 Live behavior

- **Tail gate.** No tail renders until an op has run for
  `ace.agent_decks.final_tail_delay_seconds` (default `5.0`; `0` = immediately). Fast
  ops go straight from `▶` to `✓`.
- **Tail content.** At most 12 lines in the card, sanitized through
  `render_axe_output(…, "ansi")`, never raw Rich markup, and never copied into
  notifications or logs. Full logs open through `v`.
- **Following** reuses card-block semantics:
  - FINAL follows the newest run and the latest attempt while the reader is on them.
  - Otherwise the rail's arrival dot / `● 1 newer ›` marker appears, and the view never
    yanks the reader.
  - Within a tail, scrolling up pauses follow and returning to the bottom resumes it.
- **Scope.** The 1 Hz tick (elapsed time plus tail refresh) runs **only** while some
  panel shows FINAL for the selected node **and** that node has an active finalization.
  - The timer callback is thin and spawns a pump-free, coalesced off-thread collection.
  - It respects `NavigationGate` and the typing gate.
  - It cancels on hide, subject change, settle and teardown.
  - A background agent that is finalizing costs nothing.

### 3.8 Module map

| Module                                                                                                                                                                                                                      | Owner phase                               |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| sase-core `crates/sase_core/src/agent_scan/{wire,scanner}.rs`, `index/*`                                                                                                                                                    | core-status-wire                          |
| sase-core `crates/sase_core/src/finalizer/run_view/` (`mod.rs` facade, `wire.rs`, `decode.rs`, `precedence.rs`, `selection.rs`, `detail.rs`, `evidence.rs`, `tests/`); binding in `crates/sase_core_py/src/bead_decisions/` | core-run-view-model, core-run-view-detail |
| `src/sase/finalizers/progress.py` (journal), `status_summary.py` (tracker)                                                                                                                                                  | journal-and-summary                       |
| `src/sase/agent/pending_handoff.py` (`pending_handoff_kind`)                                                                                                                                                                | journal-and-summary                       |
| `src/sase/finalizers/operation_record.py`                                                                                                                                                                                   | operation-records                         |
| `src/sase/finalizers/steps.py`; `bounded_subprocess.py` sink and tick                                                                                                                                                       | step-channel-and-live-sink                |
| `src/sase/core/agent_scan_wire_*.py`, `models/_loaders/_meta_enrichment_*.py`, `models/_dedup.py`                                                                                                                           | status-summary-adapter                    |
| `src/sase/finalizers/view_vocabulary.py`; `ace/tui/models/finalizer_row_state.py`; `prompt_panel/_agent_finalizer_receipt.py`; flag helper `ace/tui/widgets/decks/final/flag.py`                                            | glance-surfaces                           |
| `ace/tui/widgets/decks/spec.py`                                                                                                                                                                                             | deck-spec-registry                        |
| `ace/tui/widgets/decks/document_view.py`, block-host mixin module(s)                                                                                                                                                        | card-document-view                        |
| `src/sase/core/finalizer_run_view.py`, `src/sase/finalizers/run_view_inputs.py`                                                                                                                                             | run-view-adapter                          |
| `src/sase/finalizers/cli_status.py`, `main/parser_final.py`                                                                                                                                                                 | final-cli-status                          |
| `ace/tui/widgets/decks/final/{view,loader,document}.py`                                                                                                                                                                     | final-deck-shell                          |
| `ace/tui/widgets/decks/final/overview_card.py`                                                                                                                                                                              | final-overview-card                       |
| `ace/tui/widgets/decks/final/{instance_card,enrichers}.py`                                                                                                                                                                  | final-instance-cards                      |
| `ace/tui/widgets/decks/final/live.py`                                                                                                                                                                                       | final-live                                |

New logic goes in new modules so `toobig` (tiers 700/850/1000) stays green. The
following files gain only thin delegations:

- `panel.py`, `panel_blocks.py` (already over 700 lines), `main_view.py`
- `_agent_detail_decks.py`
- `_agent_display_agent_session_render.py`

### 3.9 Feature flag scaffolding

`glance-surfaces` creates the flag with
`sase flag new ace_final_deck -k beta --when-enabled … --when-disabled … --remove-when …`,
following `sase memory read sase_flags.md`. Do not hand-add a registry entry.

- **On:** the FINALIZING row word, `⊛` chips, the header activity chip, Reply receipts,
  and the FINAL deck (cycle, picker, subtitle, keys and live tails).
- **Off:** rows, headers and Reply render exactly as before, and FINAL is absent from
  the deck cycle, picker, subtitle, palette and help.
- **Remove when:** the `final-cutover` phase of this epic has verified goldens, live
  captures and the j/k bench with the flag on.

Data phases (`journal-and-summary`, `operation-records`, `step-channel-and-live-sink`,
`status-summary-adapter`), the deck refactors, and `sase final status` are
unconditional; they have no visible TUI change. Every flag-gated behavior gets on and
off tests. `final-cutover` deletes the Off branch and the flag helper, removes the
registry entry, and closes the flag bead in the same change.

### 3.10 Cross-repo and lint protocol

- **sase-core phases** open the checkout with `sase repo open sase-core -r "<why>"`,
  read its `AGENTS.md`, follow its "Add a core function and expose it to Python" recipe,
  and pass `sase tool run check` **inside that checkout**. The host finalizer commits
  and pushes that repo.
- **Pin moves.** The first sase phase that needs new core behavior moves
  `sase-core-revision.txt` past the landed sase-core commit
  (`just ratchet-core-revision`), rebuilds the local extension (`just rust-install`),
  and registers new binding names in `tools/validate_sase_core_rs` (`REQUIRED_BINDINGS`)
  and `src/sase/_symvision_static_refs.py`. Binding names stay string literals for
  `tools/check_sase_core_rs_bindings`. The pin movers are `status-summary-adapter` and
  `run-view-adapter`.
- **Symvision.** A public symbol consumed only by a later phase gets an `--epic-symbol`
  whitelist entry (see `sase memory read symvision.md`). The consuming phase removes the
  entry. Prefer wiring a real consumer in the same phase.
- **Config.** Any new config key updates `src/sase/default_config.yml`, the schema, the
  settings parser, and their parity tests (the core-memory gotcha).

## 4. Phases

### 4.1 `core-status-wire`: finalizer_status summary field on the Rust agent-scan wire

Work in the sase-core checkout (§3.10).

1. In `crates/sase_core/src/agent_scan/wire.rs`, add `FinalizerStatusSummaryWire`,
   `FinalizerStatusInstanceWire` and `FinalizerStatusRunnerWire`, matching C5.
   - Use strings and numbers only (no enums), and `#[serde(default)]` everywhere.
   - Add `AgentMetaWire.finalizer_status: Option<FinalizerStatusSummaryWire>` with
     `#[serde(default, skip_serializing_if = "Option::is_none")]`, so existing scan
     payloads stay byte-stable.
2. In `scanner.rs::agent_meta_from_object`, add a hand-written tolerant
   `finalizer_status_from_value`:
   - A non-object value, or a missing or empty `phase` string, gives `None`.
   - Strings are capped at 120 characters on char boundaries.
   - At most 16 instances are kept; entries without a string `id` are dropped.
   - Negative, NaN or non-numeric numbers are dropped.
   - Unknown keys are ignored.
   - A malformed summary must never fail the whole meta.
3. Bump `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` to the next value and add the no-op
   `migrate_record_json_refresh_vNN` in `index/storage.rs`, following the latest
   precedent (`queue_capacity_multiplier`, v32).
4. Do **not** add the field to the fleet contract projection. If `sase tool run check`
   flags a gateway snapshot, the field leaked; remove the leak rather than regenerating
   the snapshot.

**Tests:**

- a scanner tolerance table: a valid summary, extra keys, a malformed nested value,
  oversize strings, 17 instances, a non-object value, and a missing phase
- serde skip-when-`None` round trip
- the `tests/python_wire_parity.rs` field order
- the index migration

### 4.2 `core-run-view-model`: FinalizerNodeView projection - decoders, precedence, and selection

Work in the sase-core checkout. Implement §3.4's request/response wires in
`finalizer::run_view` using free functions over `*Wire` structs and `thiserror` errors,
with no `macro_rules!`. The `mod.rs` facade holds only `mod`/`pub use` lines, and every
file stays at or under 1,500 lines.

1. **Tolerant decoders.** Each input must parse on its own.
   - **Plan:** the strict `FinalizerPlanWire`; a failure makes the run `unavailable`.
   - **Authority plan:** read only `config_snapshot.config` instance ids, provider refs,
     `defaults` and `required`, plus the entry digests needed for rule 1.
   - **Context:** tolerant; a failure means trigger facts are unknown.
   - **Submission attempts** and the **journal** (C1): tolerant JSONL that skips blank,
     malformed and unsupported-version rows, like `artifact_consumption.rs`.
   - **Result:** its own schema-v1 decoder with `cycles`.
   - **`agent_meta`:** `finalizers_drift` and the plan-seal `finalizers` block only.
   - **Ceilings:** enforce the 4 MiB pre-parse ceiling.
2. **Source precedence.** Encode rules 1–10 as a pure function from decoded facts plus
   `turn_terminal`/`runner_live` to the run disposition and per-instance statuses. Use
   the last journal segment and record the earlier-segment count.
3. **Selection.** Build the selection explanation from plan `selectors`/`required`,
   snapshot `defaults` and entry `selector_index`, and fill `unselected[]`. Enforce the
   secrecy rule with a test that asserts a sentinel config value never appears in the
   serialized response.
4. **Ordering and timeline.** Compute DAG order from `resolved_index`/`after`. Build the
   declaration timeline, the recovery-turn fact, cycles and reactivations, drift,
   run-level diagnostics, `attention_instance_id` and `run_level_trouble`.
5. **Binding.** Add `project_finalizer_node_view` in
   `crates/sase_core_py/src/bead_decisions/` (`#[pyo3(name = …)]`, request dict → wire,
   error mapping, `serialize_to_py`, `py.allow_threads`).
   - Register it in `register_bead_decisions`.
   - Add a round-trip test in that domain's tests, plus the name-presence loop.
   - Detail fields may be empty until the next phase.

**Fixtures** (hand-written artifact text under `tests/`), one per state this layer owns:

- planned (turn running)
- `skipped · handoff:plan`
- not reached (killed and legacy)
- interrupted (open journal with a dead runner; truncated journal)
- unavailable (digest mismatch, oversized result)
- zero attempts / not triggered
- a 2-cycle reactivation
- `not run · blocked by check`
- unselected via `%final:!lint`, `%final:none` and not-default

### 4.3 `core-run-view-detail`: FinalizerNodeView detail - attempts, operations, evidence, and runs

1. **Attempts.** Build them from result attempts plus journal attempt events.
2. **Operations.** Build them from C2 records (schema v1 and legacy commit records) plus
   journal op events. Add steps (C3, tolerant, ≤ 64 KiB), log metadata, and live-tail
   lines (carriage-return collapse, last `tail_lines`, ANSI preserved).
3. **Evidence.** Classify typed evidence by the conventions in §3.4 and select the
   headline. The ambiguous commit `result` kind stays `text` and never wins the
   headline.
4. **Diagnostics and reasons.**
   - Dedupe diagnostics with attempt-scoped severity (`superseded`).
   - Count warnings from warn steps plus warning diagnostics in the latest attempt.
   - The failure reason line is the first error diagnostic, else the last non-blank line
     of the supplied stderr tail.
   - Surface refusal and deferral details, and protocol-envelope names for plugins.
5. **Node composition.** Merge multiple runs into the node view: per-instance
   appearances by run, the runs ledger order (by `number`), and the node aggregate. The
   supersede rule matches D10: runs after the newest successful settled run decide the
   aggregate.

**Fixtures:**

- `builtin@commit`: a successful stitch with warn steps and a deferral variant
- `builtin@command`: a failure on attempt 1 and on attempt 2/2
- a hypothetical plugin `acme-sase@open-pr`:
  - execute/verify ops with timing
  - steps
  - typed `pr_url` evidence
  - `describe`/`validate` preflight records

  Its response must be renderable with **no** plugin-specific fields.

- a session of three runs: skipped, failed, then success

### 4.4 `journal-and-summary`: Controller progress journal, handoff skips, and the row summary writer

1. **`finalizers/progress.py`.** `ProgressJournal(artifacts_dir)` implements C1: append,
   `seq` continuation (read the last line's `seq` once at open; fall back to a line
   count), the ceiling, and swallowing errors.
2. **`finalizers/status_summary.py`.** `FinalizerStatusTracker` holds C5 state in
   memory. It writes through `update_meta_field` plus
   `touch_turn_refresh_pulse(project_name_from_artifacts_dir(…))` on transitions,
   applies the step throttle, and fills `reason`/`headline`/`warnings` per C5. It is
   best-effort.
3. **`agent/pending_handoff.py`.** Add
   `pending_handoff_kind(artifacts_dir) -> str | None` (`plan`, `questions`, `monitor`,
   `gate` or `pipe`). `has_pending_handoff` becomes a wrapper around it.
4. **Controller wiring** (`controller.py`, `controller_context.py`,
   `declaration_recovery.py`, ledger hooks):
   - **Skip branch:** `phase_skipped` plus the `skipped` summary, only when
     `artifacts_dir` exists and a marker matched. When `SASE_AGENT_TIMESTAMP` is unset
     or there is no dir, write nothing.
   - **Run events:** `phase_started` (with the runner identity from
     `core.process_identity`), cycles, declaration and recovery events, instance/attempt
     events, and `phase_finished` in the `finally` after the aggregate write.
   - **Integrity failure:** the plan-integrity-failure path still journals
     `phase_finished`.
5. **Plan seal.** Write the `planned` summary at plan seal (`llm_provider/_invoke.py`,
   beside the existing `finalizers` projection). Do the same where the monitor path
   reseals (`monitor/host_completion_state.ensure_finalizer_plan`). `%final:none` writes
   `instances: []`.

**Tests:**

- **Controller end-to-end** with fake executors: success, failure, deferral, refusal,
  empty plan, integrity failure, recovery turn, and a no-model monitor run.
- **Handoff:** writes `phase_skipped` for each marker kind.
- **Fault injection:** a raising journal or summary writer (monkeypatched) leaves
  verdicts, `finalizer_result.json` bytes and the exclusive artifacts unchanged.
- **Mechanics:** the ceiling and `observability_truncated`; the step throttle; pulse
  touches.

### 4.5 `operation-records`: One uniform operation record across every executor

1. **`finalizers/operation_record.py`.** `OperationRecorder` (a context manager) owns
   `started_at`/duration, writes the C2 record exclusively (atomic temp file +
   `os.replace` + exclusive create, like today), emits `op_started`/`op_finished`, and
   updates the tracker's instance `op`.
2. **Command.** Add op `run` with `label` = the configured command, and `logs` pointing
   at the existing `attempt-N.stdout`/`.stderr`. `attempt-N.diagnostics.json` stays.
3. **Plugin.**
   - `execute` and `verify` get attempt-scoped records with timing.
   - `describe` and `validate` get `preflight.<op>.outcome.json`. The
     `describe.stdout|stderr` / `validate.stdout|stderr` paths that `sase final submit`
     also writes stay unchanged.
4. **Commit.** Extend `record_stitch_artifacts` with `schema_version`, `op`, `kind`,
   `label` (`stitch <repo>` / `resume <repo>` / …), `started_at` and `logs`, keeping
   every existing key.
5. **Conflict repair.** It becomes a `model_turn` op
   (`attempt-N.<repo>-conflict-repair-turn.outcome.json`) that references its prompt and
   response files. Plumb the executing instance id through `commit_repair_conflict.py`
   in place of the hard-coded `"commit"` (lines ~81, 120, 275, 351), so a differently
   named commit instance writes to its own dir.

**Tests:**

- per-executor record shapes
- exclusivity and immutability errors unchanged
- legacy readers (`axe/runner_reporting.py`, `monitor/host_completion_state.py`)
  unaffected
- a renamed commit instance's conflict-repair artifacts land in its own dir

### 4.6 `step-channel-and-live-sink`: Step channel, stitch steps, and bounded live output

1. **Steps module.** `finalizers/steps.py` holds the env constant, `emit_step` and the
   ceiling (C3). Re-export `step` from `finalizers/sdk.py`, documented in its docstring.
2. **Environment.** Inject `SASE_FINALIZER_STEPS_FILE` per op:
   - For command and plugin, pass it as an explicit extra to `sanitized_env` /
     `executor_support.run_subprocess`, not by allowlisting the parent environment.
   - For stitch, add it to the environment built in `commit_repair_stitch.py`.
3. **`sase stitch create`** emits explicit steps at:
   - the before-commit hook (`start` with the command as detail, then `ok`/`fail`)
   - the VCS dispatch (the method name)
   - push
   - publication
   - the after-commit hook

   Warnings become `warn` steps through the typed warning path (`output.print_status` at
   its warning level), never by parsing printed text.

4. **Live sink.** Give `run_bounded_subprocess` optional `live_path` and `live_streams`
   parameters. `_reader` tees each chunk; rotation and cleanup follow C4. Plugins tee
   stderr only.
5. **Progress tick.** Add an optional `progress_tick` callback to
   `run_bounded_subprocess`, invoked about every 2 s from the waiting thread while the
   process runs. The executors use it to read the latest step from the op's steps file
   (tail ≤ 64 KiB) and update the tracker's `step` and `warnings`, throttled per C5.
6. **Record references.** Op records reference `steps` and `live` when present.

**Tests:**

- a fake subprocess that emits steps slowly
- sink rotation and cleanup
- sink failure swallowed
- a hard-timeout kill leaves `.live`
- stitch emits steps only when the env var is set
- plugin stdout is never teed
- the tracker picks up steps from the tick

### 4.7 `status-summary-adapter`: Python mirror and Agent model field for finalizer_status

1. **Pin.** Move the pin past `core-status-wire` and rebuild (§3.10).
2. **Python wire.** In `core/agent_scan_wire_markers.py`, add frozen
   `FinalizerStatusSummary`/`FinalizerStatusInstance`/`FinalizerStatusRunner`
   dataclasses and `AgentMetaWire.finalizer_status`.
   - Add an explicit tolerant nested converter in `agent_scan_wire_conversion.py`.
   - Bump the index schema constant in `agent_scan_wire_records.py`.
   - Update `tests/test_core_agent_scan_wire_agent_meta.py` (field order, tolerance) and
     `tests/test_core_agent_scan_wire_schema.py`.
3. **Scan probe.** In `tools/validate_sase_core_rs`, add a behavior probe modeled on
   `_validate_capacity_only_scan_contract`: write an `agent_meta.json` with a summary,
   scan it, and assert the field survives.
4. **Agent model.** Add `Agent.finalizer_status` (the appropriate state mixin under
   `ace/tui/models/`).
   - Populate it in **both** `_meta_enrichment_wire.py` and
     `_meta_enrichment_filesystem.py`, using the same tolerant parse.
   - Carry it in `_dedup._merge_agent_fields`.
   - Containers are **not** mirrored: session views aggregate members in memory. Each
     concrete turn has its own `agent_meta.json`, and mirroring would double-count (see
     `_root_represents_member`).
5. **No visible change.**

**Tests:** snapshot-path and filesystem-path parity (the same fixture produces an equal
`Agent.finalizer_status`), dedup carry, and container non-duplication.

### 4.8 `glance-surfaces`: FINALIZING rows, ⊛ chips, header chip, and Reply receipts behind ace_final_deck

1. **Flag.** Create `ace_final_deck` (§3.9) and add the helper `final_deck_enabled()` in
   `ace/tui/widgets/decks/final/flag.py`.
2. **Vocabulary.** Add `finalizers/view_vocabulary.py` for §3.2: `FinalizerStateStyle`
   with glyph, word and color, and a lookup from summary and view states.
3. **Row state.** Add `ace/tui/models/finalizer_row_state.py` with pure
   `finalizer_row_state(agent)` and `session_finalizer_row_state(container)` (D10),
   deriving interrupted and not-reached per C5.
4. **Row rendering.**
   - In `_agent_list_render_agent_status.py`, add the `FINALIZING` word and the chip
     after `)` (§3.5).
   - Add the summary token to `agent_render_key` and `_runtime_signature`, with a
     deliberate key edit.
   - Status buckets, `_agent_ordering`, filters, capacity, `agent_row_is_in_flight` and
     row actions are untouched; assert that in tests.
5. **Header.** Add the activity-chip override in `_identity_header_compact.py`.
6. **Receipt.** Add the receipt module and wire it into every assembly site in §3.5,
   non-hint and hint. Confirm that the hint cache invalidates through `Agent` repr
   (`_agent_display_hint_cache._digest_parts`). The `p n` hint line stays hidden until
   `final-deck-shell` provides the deck; guard it on deck availability.
7. **Flag off.** Output is byte-identical to today.

**Tests:**

- pure state tables for every C5 phase × instance status × turn terminality
- D10 supersede cases
- row render tests: FINALIZING stays in the Running bucket, ordering and filters are
  unchanged, chip text per state, success is silent, and warnings never produce a chip
- header chip
- the receipt for running, failed (reason line), deferred / not triggered, not run, and
  success with `⚠N`
- no receipt for skipped, planned, legacy and `%final:none`
- the hint twins return `Text`
- flag-on/flag-off parity
- `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` p95 unchanged (record the numbers)
- flag-on `sase screenshot` inspection of a real finalizing agent

### 4.9 `deck-spec-registry`: DeckSpec registry and explicit per-deck dispatch

1. **Specs.** Add `widgets/decks/spec.py` with a frozen
   `DeckSpec(deck_id, name, glyph, picker_key, blurb, count_noun, fallback_accent, accent_class, picker_class)`,
   `DECK_SPECS`, `deck_spec(deck)` and `active_deck_cycle()` (for now, the three decks).
2. **Derived tables.** Derive these from the specs: the `titles.py` tables
   (`DECK_GLYPHS`, `DECK_NAMES`, `DECK_PICKER_KEYS`, `DECK_BLURBS`, `DECK_COUNT_NOUNS`,
   and the dead `DECK_ACCENTS`), `panel_chrome._FALLBACK_ACCENTS`,
   `panel._DECK_ACCENT_CLASS` and `deck_picker_modal._DECK_CLASS`.
3. **Explicit dispatch.** Replace every fall-through that treats an unknown deck as
   Tools with an explicit per-deck branch that fails loudly (an assertion in tests,
   logged fallback to Main at runtime):
   - `_agent_detail_decks.show_deck`
   - `panel._deck_is_empty`
   - `panel_chrome.refresh_chrome` tabs
   - `_display_detail_footer.deck_card_count`
   - `_deck_refresh_views`, `_deck_show_empty` and `_deck_show_tribe_summary`
4. **Accessor routing.** Route every `DECK_CYCLE` iteration through
   `active_deck_cycle()`: subtitle, picker rows, `commands/catalog.py` deck commands,
   `layout.choose_new_panel`, `cycle_deck_id`, `_panel_detail.action_show_deck_at*`
   bounds, and persistence decoding (an inactive deck becomes Main).

**Tests:** existing deck tests unchanged; a spec-consistency test (every `DeckId` has a
spec, picker keys are unique and not in `{j, k, q, p}`); and no golden changes.

### 4.10 `per-deck-preferred-cards`: Per-deck sticky preferred cards

1. **Model.** Replace `DeckPanelState.preferred_card` with an immutable per-deck mapping
   plus `preferred_card_for(deck)` and `with_preferred_card(state, deck, card)`.
   `with_panel_deck` keeps the whole mapping.
2. **Scoping.**
   - `cycle_focused_deck_card` and block moves save under the focused panel's deck.
   - `DeckArea.set_preferred_card` re-shows only that deck's document.
   - `resolve_active_card`, `_preferred_card_from_area` and `new_panel_for_deck` take
     the deck into account.
3. **Persistence** (`models/agent_deck_persistence.py`). Add an optional v1 field
   `preferred_cards: {deck: card}` (validated: known decks only, non-empty strings of at
   most 256 characters) and keep writing `preferred_card` = Main's preference.
   - Decoding prefers the map and falls back to the legacy key as Main's.
   - There is no schema bump, so older builds still read the file.

**Tests:** Main's sticky Reply behavior is unchanged; round trip; legacy-file decode; a
file with the new key read by the legacy decoder path; and the restore path in
`actions/agents/_deck_persistence.py`.

### 4.11 `card-document-view`: A generic card-document view and block host beyond Main

1. **View extraction.** Extract `CardDocumentView` (`widgets/decks/document_view.py`),
   parameterized by `DeckId`, from `MainDeckView`. It covers paged/spread show,
   render-key dedupe, anchors, `show_card`, `pin_to_bottom`, the block mixin, and the
   deck accent for spread separators (`card_separator_for(deck, card)` replaces
   `main_separator_for`). `MainDeckView` becomes a thin subclass.
2. **Spread decision.** Generalize `panel_spread`:
   - `_render_mode` for every card-document deck
   - `_decide_main_mode` → `_decide_document_mode(deck)`
   - measurement cache keys namespaced by deck (`f"{deck.value}:{digest}"`)
3. **Block host.** Generalize `DeckPanelBlocksMixin` through one host accessor that
   returns `(view, document, deck)` for card-document decks:
   - replace the Main-only gates
   - make the rail accent per deck
   - add a per-deck scroll watcher
   - generalize `panel_transitions`

   Move code into new mixin modules so `panel_blocks.py` does not grow.

4. **Result.** At the end of the phase the only card-document deck is still Main.

**Tests:** the whole existing deck and card-block pilot suite plus the
`bench_tui_jk_blocks.py` cases unchanged; unit tests for the deck-parameterized helpers;
and no golden changes.

### 4.12 `run-view-adapter`: Python run-view facade, artifact collector, and end-to-end proof

1. **Pin.** Move the pin past `core-run-view-detail` and rebuild (§3.10).
2. **Facade.** Add `src/sase/core/finalizer_run_view.py`: the
   `project_finalizer_node_view(request)` facade over `require_rust_binding`, with
   frozen dataclasses mirroring the response and tolerant `from_dict` converters (the
   `finalizer_wire.py` style).
3. **Collector.** Add `src/sase/finalizers/run_view_inputs.py` (no TUI imports):
   - `RunTarget(run_id, artifacts_dir, number, label, kind, turn_terminal, runner)`
   - `collect_run_input(target) -> dict`: capped reads (at most 4 MiB + 1 bytes), a
     listing of `finalizers/<id>/`, and tails and line counts per §3.4
   - `runner_live` from `core.process_identity.process_identity_matches`
   - `run_inputs_signature(targets)`: stat-only
   - `build_node_request(targets, tail_lines)`
4. **Targets.** Add the TUI-side helper `node_run_targets(agent, attempt_number)`:
   - a lone turn gives one target
   - a session container gives its concrete turns in roster order
     (`concrete_agent_session_turn_rows`, `number` = roster index, matching Reply's
     rail)
   - a pinned prior attempt gives that attempt's dir

**End-to-end test** (real controller and executors, fake commands/providers):

- success
- fail-then-retry-then-pass
- deferral
- handoff skip
- a kill mid-op (live file retained → `interrupted`)
- a plugin fixture with steps

Each goes collector → binding → typed view, with assertions on the §3.2 states.

### 4.13 `final-cli-status`: Read-only sase final status run view

1. **Command.**
   `sase final status [<agent>] [-d/--artifacts-dir DIR] [-f/--format {pretty,json}]`.
   - Keep `status` alphabetical in `sase final -h` (`context`, `defer`, `doctor`,
     `list`, `prepare`, `show`, `status`, `submit`), and write excellent help with
     examples.
   - `<agent>` accepts a session, turn or monitor-turn name, resolved through the
     existing agent-name resolution used by `sase chats` / `--from-name`
     (`scripts/_agent_chat_from_name_sources.py`).
   - Inside a SASE agent turn with no argument, default to the calling turn. Outside
     one, a missing argument is a usage error.
2. **Pretty output** (Rich, colored, shared vocabulary):
   - a header line
   - per run: disposition, the plan list with statuses, the declaration timeline, and
     per-instance attempts/ops/evidence/diagnostics
   - relative log paths
   - the unselected list
3. **JSON output** is the projection response verbatim.
4. **Exit codes:** 0 when a view was produced (whatever the finalizer outcome), 1 when
   the agent is unknown or has no finalizer data, 2 for usage errors.

**Tests:** parser and help ordering, pretty and JSON output on the e2e fixtures, the
in-agent default, and unknown-agent handling.

### 4.14 `final-deck-shell`: Register the ⊛ FINAL deck with its loader, availability, and chrome

1. **Spec.** Add `DeckId.FINAL` and its `DeckSpec` (§3.6). `active_deck_cycle()`
   includes it only with the flag on. Add the `styles.tcss` rules for the deck border,
   `-unfocused` and the picker row, and confirm the accent (D2).
2. **Registration.** Register FINAL at every site from the deck inventory:
   - `panel.compose` (a `…-final-scroll` `VerticalScroll`) and the `set_deck` class and
     refresh branches
   - `_deck_is_empty`, a visibility message handler, `panel_interaction` view accessors,
     `set_availability` and resize
   - `_agent_detail_decks` refresh / availability / empty / tribe summary / `show_deck`
   - `_agent_detail_deck_layout` split and duplicate handling, and
     `_agent_detail_deck_targets.focused_final_view`
   - `_agent_detail_state.get_editor_file_info` (export of the active card)
   - the footer card count, pin-to-bottom (`actions/navigation/_basic.py`), the search
     corpus, and `empty_state` messages
   - the startup notice text listing the decks
3. **View and loader.** Add `widgets/decks/final/{view,loader,document}.py` per §3.6.
   The document builder assembles `overview` plus `instance:<id>` cards from the typed
   view. Card bodies are minimal here (a status header line each); the next two phases
   fill them.
4. **Chrome and selection.**
   - `probe_final_deck(agent, *, attempt_number)` in `availability.py` (no I/O)
   - the subtitle's `final <glyph>` in the status color, and tabs as the status strip
   - the default-card rule
   - sticky per-deck FINAL cards
   - `D`-pinned attempts
   - once FINAL is available, the receipt shows its `p n` hint
5. **Keys.** `p n` / `p N` come from `DECK_PICKER_KEYS` through the catalog. Update:
   - the help rows (`p M/F/T/N`)
   - the picker hint (`M/F/T/N`) and the legend's width tier
   - the palette labels
   - the tests that hard-code the deck set (`tests/test_keymaps_display_help_agents.py`,
     `tests/test_command_catalog_build.py`, `tests/test_command_availability_scope.py`,
     `tests/test_command_execution.py`, `tests/ace/tui/test_agents_deck_picker.py`,
     `tests/ace/tui/modals/test_deck_picker_modal.py`,
     `tests/ace/tui/widgets/decks/test_deck_model.py`)

   `default_config.yml` needs no key, because the picker letters are fixed in the spec.
   Verify this.

**Pilot tests** (model on `tests/ace/tui/widgets/decks/test_deck_block_paged_pilot.py`
and the LLM Calls panel tests):

- landing on the default card
- a sticky FINAL `instance:check` does not disturb Main's sticky Reply
- a Reply-above / FINAL-below split via `p N`
- availability and the dim picker entry for a legacy agent
- stale-result rejection under fast j/k
- an idle tick reloads nothing (the signature cache is hit)
- persistence of a FINAL panel, and its decode to Main with the flag off
- flag-off parity: cycle, picker, subtitle, palette and help unchanged

### 4.15 `final-overview-card`: The Overview card

Implement `widgets/decks/final/overview_card.py`, following the research mockups in
§4.4:

```text
OVERVIEW · 3 selected · ✗ failed · 2 cycles · 6m02s · plan 84a91c2d
  ✓ commit   builtin@commit     default                        2m04s
  ✗ check    builtin@command    %final:check · after commit    3m40s
  – tasks    builtin@tasks      default · not run (blocked by check)
  ○ lint     builtin@command    configured · not selected (%final:!lint)
DECLARATION   07:30:58 ✗ rejected  commit_bead_action_invalid
                        bead_action is required when a bead is assigned; use keep…
              07:31:05 ✓ accepted  2 payloads (commit, tasks)
CONTROLLER    2 cycles · commit reactivated after check
⚠ config drifted since this plan was sealed: check.max_attempts 2 → 3
RUNS          0 --plan ○ skipped · plan handoff · 2 --code ✗ · 4 --code ✓
sase final status <agent> · sase final list · sase final doctor
```

- **Visibility rules.**
  - `CONTROLLER` appears only when cycles > 1.
  - `RUNS` appears only for more than one run, or when any run was skipped or not
    reached.
  - Unselected rows are dimmed.
- **Calm run states:**
  - a skipped run shows
    `○ skipped · plan handoff — this shell handed off; its successor lands the work`
  - a not-reached run shows `– not reached`
  - an unavailable run says why and offers raw-artifact `v` targets
  - a recovery turn gets a declaration line with prompt/response `v` targets

**Tests:** renderer unit tests per state and width (120/80/60 columns); the secrecy rule
(no config values reach the card).

### 4.16 `final-instance-cards`: Generic instance cards with commit and command enrichers

1. **Generic renderer** (`widgets/decks/final/instance_card.py`), provider-neutral, per
   the research §4.4 mockups:
   - **Header:** `<id> · <provider_ref>` with the right-aligned status and attempt
     count.
   - **Context lines:** `why` (selection + `after`), `trigger` (kind words,
     `submission required`, obligation count), and `declared` (a generic payload
     summary).
   - **Attempt sections:** `─── attempt n ─── HH:MM:SS ─── 3m40s ─── ✗ exit 1 ───`.
     - Non-latest attempts collapse to one line.
     - The latest attempt, and a failing latest attempt, expand automatically.
     - Reuse the existing responsive-section fold machinery so an explicit user fold is
       never overridden. If per-attempt folds cannot fit it, render the collapsed line
       with `v` targets instead.
   - **Operations:** glyph + label + right-aligned exit and duration, with indented
     steps; warn steps are summarized as `⚠ N warnings (…) ›`.
   - **Evidence line:** typed rendering (short SHA with a `↗ commit view` target, links,
     bead refs, openable paths, durations, colored exit codes).
   - **Remaining lines:** deduped diagnostics;
     `logs  stdout N lines · stderr M lines   v to open`; `protocol …   v to open` for
     plugins; and calm refused and deferred blocks (deferral paths capped with
     `+N more`).
2. **Enrichers** (`widgets/decks/final/enrichers.py`): a registry keyed by
   `provider_ref` that returns **additional** renderables from typed data only.
   - `builtin@commit`: a per-repo table (repo, decision, short SHA, pushed, deferral
     paths, warnings) with commit-viewer targets through
     `prompt_panel/_agent_commits.py`.
   - `builtin@command`: an argv line and the failing lines from the stderr tail.
   - Plugins get none.
3. **Integration.** `v` hint targets for every file reference, `E` export, the search
   corpus, and spread separators `━━ ⊛ <id> ━━`.

**Tests:**

- The plugin fixture renders completely through the generic path. Assert that no
  provider-specific branch ran.
- commit and command enrichers
- attempt folding rules
- typed evidence rendering
- success with warnings shows `⚠N` only inside the deck
- width tiers

### 4.17 `final-run-blocks`: One card block per run on session containers

1. **Blocks.** On session containers, every FINAL card, `Overview` included, holds one
   `CardBlock` per run whose disposition is `active`, `ran` or `interrupted`.
   - The block id is `card_block_id(identity)`, the same id Reply uses.
   - `BlockMeta` carries the roster number, label, monitor glyph, and a status bucket
     from the run aggregate.
   - The block header is `render_phase_divider(label, start, glyph=…, block_id=…)`, with
     the run duration appended.
2. **Instance cards** get blocks only for runs where that instance ran. Not-triggered
   and skipped runs appear only in the ledger line.
3. **Blocks in FINAL.** Rail numbers match Reply's rail for the same session, so gaps
   are informative. `[`/`]`, newest landing and following come from the generalized
   block host. Block navigation makes the card the FINAL sticky card.

**Tests:**

- rail parity with Reply for the same session fixture
- `[`/`]` in FINAL
- newest landing
- block + sticky interplay
- a single-run node shows no rail

### 4.18 `final-live`: Live tails, following, and the 1 Hz tick for the selected agent

1. **Config key.** Add `ace.agent_decks.final_tail_delay_seconds` (default `5.0`) in
   `default_config.yml`, the schema and `agent_decks_settings.py`, with the parity
   tests.
2. **Tail.** Add the in-card live tail (§3.7) for the active op, sanitized through
   `render_axe_output(…, "ansi")`.
3. **Tick.** `widgets/decks/final/live.py` owns the 1 Hz tick:
   - It is thin and spawns a pump-free, coalesced collection (`spawn_pump_free_task`).
   - It runs only while FINAL is visible for the selected node **and** that node's
     summary is active.
   - It cancels on hide, subject change, settle and teardown.
   - It respects `NavigationGate` and the typing gate.
   - It updates elapsed counters and tails in FINAL only; the receipt stays static
     between summary writes.
4. **Following** (§3.7): the arrival marker, and scroll-up pause / bottom resume inside
   the tail.

**Tests:**

- An end-to-end fake finalizer that emits steps slowly, warns, retries, and then passes,
  fails or dies.
- No tail bleeds across agents on j/k.
- No tick when FINAL is hidden or the node is idle.
- Timer cancellation.
- A `SASE_TUI_TRACE=1` capture shows no file I/O on the pump. Follow `tui_perf.md` rules
  1, 2, 4, 13 and 14.

### 4.19 `final-cutover`: Remove the flag, add goldens, inspect live, and bench

1. **Bench first** (flag still present): run the j/k bench with the flag off and on in
   SINGLE, LEFT_RIGHT, and with FINAL pinned (the triage loop). p95 must be < 16 ms. Fix
   any regression, and record both numbers in the phase close note.
2. **Remove the flag:**
   - delete every Off branch and `decks/final/flag.py`
   - make `active_deck_cycle()` include FINAL unconditionally
   - remove the registry entry and collapse on/off tests to single-state
   - close the flag bead in the same change
   - `sase flag show ace_final_deck` and `tools/check_feature_flags` must be clean
3. **Goldens.** Use deterministic fixtures with fixed times, in a new
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_final.py` modeled on
   `test_ace_png_snapshots_llm_calls.py`:
   - row states (FINALIZING, `⊛✗`, `⊛⏸`, `⊛!`, success silence) at 120/80/60 columns
   - receipts (running, failed with reason, deferred + not triggered)
   - FINAL spread (a single successful commit with declaration rejections)
   - FINAL paged (a failed check across 2 attempts)
   - the plugin fixture
   - an Overview with an unselected instance and drift
   - a session container with run blocks and the rail
   - the Reply/FINAL split
   - the picker with `n`
   - the narrow title tiers

   Generate them with targeted `just fix-tui-screenshots -- <selectors>` and inspect
   **every** new PNG. Then re-baseline the existing goldens that legitimately change
   (the subtitle's `final` entry and the picker legend). Expect broad but uniform
   subtitle churn, and inspect it group by group in the retained visual report.
   Generation is not approval.

4. **Live inspection** with `sase screenshot`:
   - a real agent finalizing (row, header chip, receipt, FINAL live tail)
   - the j/k triage loop with FINAL pinned
   - a plan-handoff shell showing `skipped`
   - a deferred commit

   Fix any defects found.

5. **Check.** `sase tool run check` is green. Do not run `just check-full` unless
   explicitly instructed.

### 4.20 `final-docs`: User and plugin-author docs for finalizer visibility

Describe only shipped behavior.

- **`docs/ace.md`:**
  - the FINALIZING phase and `⊛` chips, and the Reply receipt
  - the FINAL deck: identity, cards, run blocks, the §3.2 state table, default-card and
    sticky rules, keys (`p n`/`p N`, `Ctrl+N/P`, `Ctrl+J/K`, `[`/`]`, `v`, `E`, `,/`,
    `Z`), live behavior, and the **triage loop** tip (Reply above, FINAL below)
  - the picker letters `m/f/t/n` in the Deck Picker section
- **`docs/configuration.md`:** the `final_tail_delay_seconds` row next to the other
  `ace.agent_decks` keys, and the picker letter list.
- **`docs/plugins.md`:** the step channel (`SASE_FINALIZER_STEPS_FILE`,
  `sase.finalizers.sdk.step()`), typed-evidence conventions, operation records, and how
  plugin output renders with no TUI code.
- **`docs/cli.md`** (or wherever the `sase final` subcommands are documented):
  `sase final status`.
- **Memory.** This plan does not authorize memory edits, so **do not edit
  `sase/memory/`**. Record a `PROPOSED FOLLOW-UP:` note on this phase's bead with
  ready-to-apply text for:
  - the `glossary:agent-data-deck` strand (the built-in decks now include FINAL)
  - a possible new strand, **Finalizer Run**

## 5. Verification

**Per phase:**

- targeted pytest (or targeted `just test -p` in sase-core) for the touched areas
- `just fix`, then `sase tool run check` in every repo the phase changed
- UI phases also inspect a flag-on `sase screenshot`

**Goldens** change only in `final-cutover`. Every earlier phase must leave existing
goldens pixel-identical (flag off, or no visual change).

**The epic is done when:**

- every §3.2 state is reachable from an end-to-end fixture and renders correctly in the
  row, the receipt, FINAL and `sase final status`
- a handoff-skipped plan shell reads `skipped · plan handoff`
- a killed finalizer reads `interrupted`, never a forever-spinner
- the plugin fixture renders with no plugin-specific code
- the goldens are committed and inspected
- live captures are inspected
- the j/k bench p95 is < 16 ms, with flag-off and flag-on numbers recorded
- the `ace_final_deck` flag bead is closed
- `sase tool run check` is green in both sase and sase-core

## 6. Out of scope (candidate follow-ups; record them as PROPOSED FOLLOW-UP notes, do not build)

- Provider `describe` presentation hints (`summary`, `headline_evidence`,
  `evidence_labels`).
- Notifications on `failed`/`refused` (never `deferred`).
- Clan and tribe finalizer roll-ups; FINAL stays unavailable for them in v1.
- Retry, cancel and bypass controls (D12).
- Attempt navigation inside run blocks, until some finalizer retries routinely.
- Fixing the chronic stitch warnings (D7; owned by `sase-10x` / `sase-yy.8.6`).
- Exposing `finalizer_status` through the mobile/fleet gateway contract.
