---
tier: epic
title:
  "SASE Goals G1: the goal ledger, the manual sase goal CLI, and the goal: artifact"
goal: "Goals are durable, conflict-free, cross-machine records that a person can create,
  list, show, edit, drop, reopen, merge, and cite as @goal:<id>. Hot reads stay fast
  however much settled history piles up, and every surface says honestly how fresh it
  is.

  "
phases:
  - id: core-model
    title: Goal domain model in sase-core
    depends_on: []
    size: medium
    description: "core-model: add the pure sase-core goal module. It covers ids, the
      frozen event vocabulary and status machine, the total deterministic reducer with
      concurrency tie-breaks, action validation, publish classes, and
      presentation-neutral card/row view models.

      "
  - id: ledger-io
    title: On-disk ledger, hot projection, doctor scan, and bindings
    depends_on:
      - core-model
    size: medium
    description: "ledger-io: add ledger file I/O in sase-core: STORE.json fence,
      marker-superset write ordering, append, O(unsettled) hot read, stat-signature
      projection, history scan, doctor scan and repair, and an I/O probe. Expose them as
      PyO3 bindings with a thin Python facade and move the core pin.

      "
  - id: ledger-root
    title: Ledger root resolution and the hidden-clone write lane
    depends_on:
      - ledger-io
    size: medium
    description: "ledger-root: resolve each project's ledger (goals.visibility /
      goals.host_role config, shared in the hidden beads clone or local-only), and add
      the locked write transaction that commits only goals/. Publish synchronously
      through the existing managed sync worker, and keep bead commits and readers clear
      of goals/.

      "
  - id: publish-sync
    title: Publishing, convergence, and honest freshness
    depends_on:
      - ledger-root
    size: medium
    description: "publish-sync: after each integration, reconcile live markers for
      touched goals. Mint ids only after a fetch and detect collisions. Add the
      unpublished outbox, a push leg on the sidecar auto-sync tick, a push-retry
      counter, a single-flight TTL background fetch, bounded fresh fetches, and the
      synced-ago watermark.

      "
  - id: cli
    title: The sase goal command
    depends_on:
      - publish-sync
    size: medium
    description: "cli: add sase goal (list default, show, new, edit, drop, reopen,
      merge, doctor) with human-only verbs refused inside agent runs. Render terminal
      output through a Rust renderer, add JSON output, and serve list/show from a lean
      entry.py fast path under the 50 ms budget. Register completion spec, run policy,
      and CLI docs rows.

      "
  - id: artifact-kind
    title: The goal artifact kind and @goal citations
    depends_on:
      - ledger-root
    size: medium
    description: "artifact-kind: make goal: a first-class builtin artifact kind across
      the sase-core catalog, parser, resolver, and editor/LSP kind lists. Wire it
      through Python builtin-entry dispatch, artifact read/show/path, one-line @goal
      prompt expansion, staging, and ACE @goal payload completion, with zero golden
      churn.

      "
  - id: acceptance
    title: Acceptance fixtures, benchmark, docs, and memory
    depends_on:
      - cli
      - artifact-kind
    size: medium
    description:
      "acceptance: prove the epic end to end with two-clone concurrency and marker-race
      fixtures, crash repair, fail-closed schemas, offline publishing, the agent refusal
      matrix, and the 100k-settled benchmark with file-open proof. Finish docs/goals.md
      and land the authorized memory updates."
proposed_by: bbugyi200.athena.0tb.w0
create_time: 2026-09-27 19:03:14
status: wip
---

- **PROMPT:**
  [prompts/202609/goal_ledger.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/goal_ledger.md)

# SASE Goals G1: the goal ledger, manual `sase goal` CLI, and the `goal:` artifact

## Context

This is the first of six sibling epics that deliver SASE Goals. The program is described
in two research notes:

- the design of record, `research:202609/sase_goals_design/sase_goals_design.md`;
- the delivery roadmap,
  `research:202609/sase_goals_epic_roadmap/sase_goals_epic_roadmap.md`.

G1 is the **ledger epic**. It builds durable, conflict-free, cross-machine goal records
and a manual CLI so that a person can use them. Nothing binds agents to goals yet (G2),
nothing claims them (G3), and there are no drafts (G4), no Goals tab (G5), and no
attention cutover (G6).

After G1 lands, a person can:

- create a goal on athena and see it on apollo;
- list unsettled goals in under 50 ms, even with 100k settled goals behind them;
- edit, drop, reopen, and merge goals;
- cite a goal in any prompt as `@goal:<id>`.

Every later epic builds on the contract this epic freezes. **The event vocabulary and
status machine below are frozen here**, including events that only later epics produce.
The ledger is immutable, so later epics can add producers but never make breaking schema
changes.

### Architecture at a glance

```text
 sase-core (Rust, owns the domain)              sase (Python, owns side effects)
 ─────────────────────────────────────          ──────────────────────────────────────────
 goal::model    ids · wire · reducer · rules    sase.goals.store    root resolution, config
 goal::ledger   files · markers · hot read      sase.goals.write    locked append → commit
                projection · doctor · probe     sase.goals.publish  integrate → reconcile → push
 goal::render   terminal (cli) · markdown       sase.goals.fetch    TTL single-flight fetch
                (artifact-kind)                 sase.main.goal_*    parser · handlers · fast path
 artifact_ref   goal: kind                      artifact_* + TUI    @goal: read/expand/complete
```

The Rust boundary follows the `rust_core_backend_boundary` core memory and the
`rust-core-required` decision:

- **In sase-core:** anything a CLI, TUI, gateway, or editor must agree on. That covers
  ids, events, reduction, validation, file layout, reading, rendering the goal card, and
  the kind.
- **In Python:** git, config, locking of the git clone, subprocesses, argparse, and TUI
  glue.

## Decisions this plan makes

Taken as design lead; each one is stated so the approver can overrule it at
PlanApproval.

1. **Where the ledger lives.** It is co-hosted in the project's beads sidecar repository
   as a top-level `goals/` directory, and written only through the host-owned hidden
   clone (`hidden_sidecar_clone_dir(key, "beads")`). This follows the
   `machine-link-writes-off-primary` decision.
   - A `goals.host_role` config field (default `beads`) names the hosting sidecar role.
     Splitting goals into their own repository later is therefore a config change.
2. **Visibility.** A config field `goals.visibility: shared | local` (default `shared`,
   which the user confirmed while this plan was being drafted).
   - Shared goal titles, outcomes, criteria, and timelines are exactly as public as bead
     titles already are, because the beads repo is public.
   - `local` keeps a project's ledger machine-local and never publishes it.
   - Projects without a usable host sidecar fall back to local-only automatically, and
     are labeled `local only`.
3. **No feature flag.** G1 is purely additive: a new command group, a new artifact kind,
   and new files in the beads repo. Every landed phase leaves master coherent, and none
   changes existing behavior. Per the `sase_flags` memory, a beta flag would only add
   both-state tests and a removal chore. No umbrella program flag is created.
4. **Rendering belongs to core.** The terminal list and card are rendered by sase-core,
   because the fast path cannot import rich or argparse. Slow and fast paths call the
   same renderer, so their output is byte-identical by construction.
5. **`goal:` is a first-class kind.** It gets its own `ArtifactRefKindWire::Goal`
   variant, not a special case of `Document` (the pattern `tool:` uses). A goal is a
   resolvable record, not a reserved placeholder or a sidecar document.
   - That makes it a `feat!` in sase-core, per its AGENTS.md.
   - Moving sase's `sase-core-rs` PyPI window at the next sase release is the normal
     release follow-up. It is **not** part of this epic's definition of done.
6. **Five-character ids, minted after a fetch.** Goal ids are 5 Crockford base32
   characters, as the design specifies.
   - A mint checks for a local collision, and shared mode mints after the bounded fetch
     that every sync verb already does.
   - A cross-machine collision would need two machines to mint the same 1-in-33.5M id
     within one push window. It is detected deterministically and reported by
     `sase goal doctor`. It is never silently merged.
7. **Drafts are reserved, not built.** The `draft` status and `named` / `adopted` events
   are part of the frozen contract, and the draft root path is reserved. The draft store
   itself is G4.
8. **Coordinate with `sase-x7`** (canonical-only shared formats). The ledger is born
   canonical: no legacy decoders and no alias paths.
   - If sase-x7's compatibility inventory or regression enforcement exists when a phase
     lands, register the goal ledger there as a canonical shared format.

## The frozen contract

Phase `core-model` implements this section verbatim. Phase `ledger-io` implements the
layout and I/O rules. Later phases rely on both, and so do epics G2–G6.

### Identity

- **Goal id.** 5 characters from the Crockford base32 lowercase alphabet
  `0123456789abcdefghjkmnpqrstvwxyz`, the same alphabet as `PROC_ID_ALPHABET` in
  `crates/sase_core/src/procs/runtime.rs`.
  - Minted from `OsRng`.
  - Parsing folds uppercase to lowercase and rejects anything else.
- **Canonical ref.** `goal:<id>`. A cross-project citation is `goal:<project>@<id>`,
  mirroring `stitch:<repo>@<sha>`. The CLI shows `⌖<id>` (U+2316) as the visual form.
- **Event id.** A 26-character, ULID-shaped, lowercase Crockford string: a 48-bit
  millisecond timestamp plus 80 random bits.
  - It is monotonic within a process: equal milliseconds increment the random part.
  - Implement it in `goal::model::ids`. Do not add a `ulid` crate.
- **Actor.**
  `{principal: "<username>.<machine>", kind: human|agent|host, agent?: <agent name>}`.
  - Python builds the principal from `get_agent_owner_identity()`
    (`src/sase/config/_owner.py`).
  - The agent name comes from `discover_agent_identity()`
    (`src/sase/agent/identity.py`).

### Event envelope (`GoalEventWire`, one JSON file per event)

```json
{
  "schema_version": 1,
  "event_id": "01j8x9...26 chars",
  "goal_id": "7k2mq",
  "kind": "edited",
  "at": "2026-09-28T14:02:11.482Z",
  "actor": { "principal": "bryan.athena", "kind": "human" },
  "basis": "01j8x8...",
  "idempotency_key": "cli:3f1c...",
  "payload": {}
}
```

- `basis` is the head event id the writer reduced from. It is null only for `created`.
- `idempotency_key` is unique per logical action. When duplicates exist, only the first
  in reduction order counts; the rest are recorded as `duplicate` and have no effect.
- **Evolution policy:**
  - Unknown _fields_ are ignored everywhere (serde default, no `deny_unknown_fields` on
    envelope or payloads), so an older sase reads events from a newer one.
  - An unknown `kind`, or a `schema_version` above the supported one, makes **that
    goal** reduce to `readable: false` with a reason. It is listed with `⚠`, never
    dropped silently, and never affects other goals.
  - A `STORE.json` schema or layout the reader does not support makes the **whole
    ledger** fail closed with `run \`sase update\``.

### Event vocabulary

Each event is marked ● if G1 produces it, or ○ if it is reserved for the named epic.

| Kind              | Payload (all strings trimmed, lengths enforced by core)                                                                                                                                          | Producer                                                    |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------- |
| `created`         | `title` (≤60), `outcome` (≤280, one line), `criteria[]`, `origin`, `draft: bool`, `project`                                                                                                      | ● `new` (`draft: false`); ○ G4 drafts                       |
| `named`           | `title`, `outcome`, `auto: bool`                                                                                                                                                                 | ○ G4                                                        |
| `edited`          | `title?`, `outcome?`, `criteria_added[]`, `criteria_removed[]` (criterion ids), `note?`                                                                                                          | ● `edit`                                                    |
| `adopted`         | `draft_id?`, `agent`, `why` (≤280), `unit_prompt_digest?`                                                                                                                                        | ○ G4                                                        |
| `agent_attached`  | `agent`, `role_hint?`                                                                                                                                                                            | ○ G2                                                        |
| `progress`        | `note` (≤280), `agent`                                                                                                                                                                           | ○ G3                                                        |
| `plan_attached`   | `plan_ref`, `digest?`                                                                                                                                                                            | ○ G2                                                        |
| `claimed`         | `claim_no`, `claim` (≤280), `evidence[{ref, why}]`, `check[]` (1–3), `gaps[]`, `receipts[{tool, verdict, fingerprint, tree_sha, at}]`, `strength` (`tested`/`committed`/`documented`/`answered`) | ○ G3                                                        |
| `claim_retracted` | `claim_no`, `reason` (`rejected`/`followup`/`adopted`/`stale`), `feedback?`                                                                                                                      | ○ G3/G4                                                     |
| `settled`         | `flavor` (`verified`/`acknowledged`/`canceled`/`merged`/`superseded`), `into?` (for `merged`), `note?`                                                                                           | ● `drop` (`canceled`), `merge` (`merged`); ○ G3/G4 the rest |
| `reopened`        | `message` (≤280)                                                                                                                                                                                 | ● `reopen`                                                  |
| `merged`          | `from` (source goal id), `why?` (written on the **target**)                                                                                                                                      | ● `merge`                                                   |

- **Criteria.** Each criterion is `{id, text (≤200), source: user|plan|agent}`, with at
  most 10 per goal.
  - A criterion id is `<event_id>.<index>`, which makes it unique without coordination.
  - The reducer rejects an `agent`-kind actor removing a `user` or `plan` criterion.
    This guard against moving goalposts is frozen now, even though G1 has no agent
    writers.
- **Origin.**
  `{kind, principal, machine, at, via, agent?, unit_prompt_digest?, root_prompt_digest?}`.
  G1 writes `via: "cli"` with no digests.
- **Publish class** (`goal_event_publish_class`):
  - `sync`: every kind except `agent_attached` and `progress`.
  - `batched`: `agent_attached` and `progress`.
  - A local-only ledger never publishes. This closes the naming-push-policy gap raised
    in `research:202609/sase_goals_persistence.md`.

### Status machine and reduction

| Status    | Hot? | Reached by                                                     |
| --------- | ---- | -------------------------------------------------------------- |
| `draft`   | yes  | `created{draft:true}` (machine-local only, G4)                 |
| `active`  | yes  | `created{draft:false}`, `named`, `claim_retracted`, `reopened` |
| `review`  | yes  | `claimed` (G3)                                                 |
| `done`    | no   | `settled{verified \| acknowledged}`                            |
| `dropped` | no   | `settled{canceled \| merged \| superseded}`                    |

**Unsettled** means `draft`, `active`, or `review`. Running versus Idle is derived from
agent liveness in later epics and is never stored.

- **Order.** Events are reduced in causal order: every event comes after its `basis`
  when that basis is present. Ties are broken by ascending `event_id`.
  - An event whose basis is missing is ordered by `event_id` and gets a `missing_basis`
    diagnostic.
  - Reduction is a **total, deterministic function of the event set**. It never depends
    on directory listing order or on which clone integrated first. A permutation test
    proves this.
- **Concurrency.** Two events are _concurrent_ when neither is reachable from the other
  through `basis` links.
- **Content events** (`edited`, `progress`, `agent_attached`, `plan_attached`, and the
  `merged` target record) always apply to content, unless the writer's basis already
  showed the goal settled. In that case they have no effect and get an `on_settled`
  diagnostic.
- **Status transitions** follow the table. An illegal transition keeps the event on the
  timeline with `effect: ignored` and a reason.
- **Settlement races.** Among mutually concurrent settlements, a human `canceled` drop
  wins, because it expresses the person's intent. Otherwise the first in order wins, and
  the others become `superseded_settlement` diagnostics.
- **Claim races** (G3). Among concurrent `claimed` events, the first in order is the
  claim, and the rest become `superseded_claim`. A concurrent `canceled` beats a claim.
- **Late attachment.** An `adopted` or `agent_attached` concurrent with, or after, a
  settlement is recorded as a late attachment. The status stays settled.
- **Id collision.** A second `created` for the same goal id makes the goal
  `readable: false` with reason `id_collision`, which `doctor` reports. It is never
  merged silently.
- **Reduced state (`GoalStateWire`):**
  - `id`, `project`, `status`, `flavor?`, `readable`, `unreadable_reason?`;
  - `title`, `outcome`, `criteria[]`, `origin`, `revision`;
  - `head`, `created_at`, `updated_at`;
  - `merged_into?`, `merged_from[]`, `plan?`, `claims[]`, `contributors[]`,
    `last_progress?`;
  - `diagnostics[]`, and `timeline[]` of `{event_id, at, kind, actor, summary, effect}`.
  - `revision` counts applied `created`, `named`, and `edited` events.
- **Action validation.** `plan_goal_action(state, action, actor) -> Vec<GoalEventWire>`
  turns a requested action into events, or a typed refusal:
  - `new` → `created`;
  - `edit` → `edited` (refused on settled goals, or when nothing changed);
  - `drop` → `settled{canceled}`, with a `why` required;
  - `reopen` → `reopened` (settled goals only);
  - `merge` → `settled{merged, into}` on the source plus `merged{from}` on the target.
    Both goals must be unsettled and different.
  - Refusal codes are stable snake_case strings, e.g. `already_settled`, `not_settled`,
    `merge_into_self`, `title_too_long`.

### View models

`goal_row_view(state)` and `goal_card_view(state, now)` produce presentation-neutral
display structs. Every renderer consumes them: CLI terminal output, the markdown card,
prompt citations, and later the G5 TUI.

- **Row:** glyph, id, status word, title, age, `readable` flag, last-progress preview.
- **Card:** header (title, status badge, ref, project, opened-by, revision, mode label),
  then ordered sections: Outcome, Criteria, Merged, Plan, Claims, Timeline.
  - Sections with no content are omitted.
  - Relative ages are computed from `now`. Persisted surfaces pass absolute times, per
    the `bead_time_presentation` convention.

### Ledger layout (in the ledger root)

```text
goals/
  STORE.json                          {"schema_version":1,"layout":"sase-goal-ledger","created_at":…}
  live/<id>                           empty marker; the hot index
  items/<id>/events/<event_id>.json   immutable events, the only source of truth
```

- **Nothing is edited in place.** Writers add uniquely named event files and add or
  remove markers. Event files are never rewritten or deleted.
- **Marker-superset invariant.** `live/` always contains every unsettled goal, and may
  contain extras.
  - A marker is created _before_ the event that makes a goal unsettled (`created`,
    `reopened`).
  - A marker is removed _after_ the event that settles a goal.
  - Readers reduce each marked goal and omit settled ones, counting `stale_marker`. They
    never trust a marker alone.
  - Only `doctor --repair` and post-integration reconciliation remove extra markers.
- **Ledger roots:**
  - shared: `<hidden clone of goals.host_role>/goals`;
  - local-only: `~/.sase/projects/<key>/goals` (a plain directory, not a git repo);
  - reserved for G4 drafts: `~/.sase/projects/<key>/goal-drafts/goals` (same layout,
    never published).
- **Hot projection** at `~/.sase/projects/<key>/goals-hot.json`, machine-local:
  `{schema_version, project, mode, host_role, ledger_root, watermark_path, outbox_path, generated_at, goals: {<id>: {sig: [events_dir_mtime_ns, entry_count], row, state_digest}}}`.
  - A warm read is `readdir(live/)`, one stat per live goal's `events/` directory, and a
    re-reduce of only the goals whose signature changed.
  - The projection is rebuildable from `live/` alone.
  - It also carries the root pointer the fast path needs, so the fast path never loads
    config.
- **Performance contract.** Measured in `acceptance`; not asserted.

| Metric                                                               | Target                                                                              |
| -------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| Rust warm hot read, 1,000 unsettled goals                            | ≤ 5 ms                                                                              |
| `sase goal list` end to end (fast path, warm, 100 unsettled, athena) | p50 ≤ 50 ms, p95 ≤ 80 ms                                                            |
| Growing settled goals from 0 to 100k                                 | hot-list p95 changes by ≤ 10%                                                       |
| Cold rebuild                                                         | opens only `STORE.json`, `live/`, and live goals' `items/<id>/events/*` (I/O probe) |

## Rules for every phase

1. **Where work happens.** Work in sase-core happens in the linked checkout opened with
   `sase repo open sase-core`. Rust core, binding, pin, Python caller, and tests land
   together in the phase that needs them.
2. **The pin.** A phase whose sase code calls a new or changed binding moves
   `sase-core-revision.txt` past its sase-core commit with `just ratchet-core-revision`.
   The pin only moves forward, so after rebasing onto a sibling phase's pin move,
   re-ratchet.
3. **Verification.** Run `sase tool run check`, never `check-full`, in each repo the
   phase touched. Acceptance never depends on a green master.
4. **Golden churn.** Zero golden PNG churn. This epic changes no TUI layout.
5. **Blast radius.** A broken or missing goal ledger never breaks any existing command:
   bead, artifact, launch, or sync.
6. **File size.** Python files stay under the `toobig` 1,000-line cap, and Rust files at
   1,500 lines or fewer.
7. **Size guard.** If a phase turns out larger than its size, finish the
   contract-bearing part. Record the remainder as a `PROPOSED FOLLOW-UP:` note on the
   phase bead. Never widen the epic.

## Phase: core-model

- slug: core-model
- size: medium
- depends: []

Work in the linked sase-core checkout (`sase repo open sase-core`). Read its `AGENTS.md`
first.

1. Add a flat top-level module `crates/sase_core/src/goal/`.
   - `mod.rs` is a facade of `mod` and `pub use` lines only.
   - Submodules: `ids.rs`, `wire.rs`, `reduce.rs`, `actions.rs`, `view.rs`, and
     `tests/`.
   - Keep every file at 1,500 lines or fewer. Use free functions over `*Wire` structs,
     `thiserror` errors, and no `macro_rules!`.
   - Import by module path. Never add to the root `pub use` in `lib.rs`.
2. `ids.rs`: goal-id minting, parsing, and validation, and ULID-shaped event-id minting
   with in-process monotonicity. Use `OsRng` and the existing `chrono`; add no new
   crate.
3. `wire.rs`: implement exactly the contract above.
   - Types: `GoalEventWire`, `GoalEventKindWire`, a payload per kind,
     `GoalCriterionWire`, `GoalOriginWire`, `GoalActorWire`, `GoalStatusWire`,
     `GoalSettleFlavorWire`, `GoalStateWire`, and `GoalDiagnosticWire`.
   - Constants: `GOAL_LEDGER_SCHEMA_VERSION = 1` and `GOAL_WIRE_SCHEMA_VERSION = 1`.
   - Payloads are tagged by the envelope's `kind`. Deserialize by dispatching on `kind`;
     do not rely on serde's untagged guessing.
   - Unknown fields are ignored. Unknown kinds and versions surface as a typed
     `Unsupported` value rather than an error, so the reducer can mark the goal
     unreadable.
4. `reduce.rs`: `reduce_goal_events(goal_id, events) -> GoalStateWire`.
   - Causal-then-id ordering, dedupe by idempotency key, and every rule in "Status
     machine and reduction".
   - It must be total: it never panics and never errors on well-formed input. It only
     emits diagnostics.
   - Also add `goal_event_publish_class(kind)`.
5. `actions.rs`: `plan_goal_action(state_or_none, action, actor, now, ids)`, returning
   planned events or a stable refusal code.
   - All length, criteria-count, and one-line checks live here.
   - Take `ids` as an injectable minting source so tests are deterministic.
6. `view.rs`: `goal_row_view` and `goal_card_view` as described above.
   - Also add `GOAL_GLYPH = "⌖"` and `GOAL_ACCENT_HEX = "#FF87AF"`.
   - `#FF87AF` is a rose that is absent from the existing pane palette: stitches
     `#FFD700`, beads `#D787FF`, agents `#0062FF`, patches `#00D7AF`, files `#FFAF5F`,
     plans `#AF87FF`. G5 may retune this single constant.
7. Tests in `goal/tests/`:
   - every transition in the status table, legal and illegal;
   - every race rule (settlement, claim, late attachment, content-after-settle);
   - id collision, missing basis, and idempotency dedupe;
   - forward-compatibility: unknown field ignored; unknown kind or newer version makes
     only that goal unreadable;
   - a permutation property test: shuffling a goal's event set in many orders yields an
     identical `GoalStateWire`;
   - a JSON round-trip of every event kind against committed fixture files under
     `goal/tests/fixtures/`. These freeze the vocabulary; later epics add fixtures and
     never edit these.
8. This phase adds no bindings and changes nothing in sase.
9. Verify with `sase tool run check` in sase-core.

## Phase: ledger-io

- slug: ledger-io
- size: medium
- depends: [core-model]

1. Add `crates/sase_core/src/goal/ledger/`: `layout.rs`, `append.rs`, `read.rs`,
   `projection.rs`, `doctor.rs`, `probe.rs`, and `tests/`.
2. **`layout.rs`:** path helpers, and `STORE.json` init and validation (fail closed on
   an unknown schema or layout). Name the reserved draft root as a documented constant.
3. **`append.rs`:** `goal_ledger_append(root, request)`.
   - The request carries an action, actor, `expected_head?`, and an idempotency key.
   - Steps:
     1. Take the caller-supplied lock path with `crate::store_lock` (the lock file lives
        outside the tracked tree).
     2. Load and reduce the target goal(s).
     3. Call `plan_goal_action`. If `expected_head` is stale, re-plan against the
        current state: commutative content edits proceed, and anything else returns
        `stale_basis` with the current state.
     4. Write in marker-superset order, with atomic temp-file + fsync + rename (reuse
        the pattern in `bead/jsonl.rs`).
     5. Return `{events, created_paths, removed_paths, states}` so Python can commit
        exactly those paths.
   - Add a test-only fault hook, a cfg-gated or explicit request field that is never
     reachable from the CLI, that stops after the event write and before the marker
     step. `acceptance` uses it for crash tests.
4. **`read.rs`:**
   - `goal_ledger_list(root, projection_path?, filter)`: the hot read. It reads `live/`
     plus signatures and reduces only goals that changed. Settled goals' directories are
     never opened.
   - `goal_ledger_show(root, id)`: O(events of one goal).
   - `goal_ledger_history(root, filter, limit)`: an explicit history scan for
     `done`/`dropped`/`all`, newest first. It is documented as outside the O(n)
     contract.
   - Return
     `GoalListWire {schema_version, project, mode, generated_at, stale_markers, goals: [GoalStateWire]}`.
5. **`projection.rs`:** read, refresh, and write `goals-hot.json` as specified.
   - Write atomically under a sibling `.lock`.
   - Skip the write when nothing changed.
   - Report `Missing`, `SchemaMismatch`, `Stale`, or `Fresh`.
   - Extract a shared stat-signature helper from `bead/touch_index.rs`
     (`scan_stream_signatures`, `mtime_ns`, `write_index_atomic` are private there) into
     a small `pub(crate)` utility rather than copying it.
6. **`doctor.rs`:** `goal_ledger_doctor(root, repair: bool)`.
   - Checks:
     - `STORE.json` exists and is supported;
     - full marker⇔unsettled reconciliation;
     - orphan markers;
     - unreadable goals, including `id_collision`;
     - stray non-event files under `items/`;
     - projection status.
   - `repair` only adds or removes markers and rebuilds the projection. It never touches
     an event file.
   - Return the changed paths so Python can commit them.
7. **`probe.rs`:** an I/O probe that counts file opens and directory reads per path
   class during a read. Tests use it to prove that settled items are never opened.
8. **Bindings.** Add a new domain `crates/sase_core_py/src/goals/mod.rs` with
   `register_goals`.
   - Bindings: `goal_ledger_init`, `goal_ledger_append`, `goal_ledger_list`,
     `goal_ledger_show`, `goal_ledger_history`, `goal_ledger_doctor`,
     `goal_projection_status`, `goal_mint_id`, `goal_ledger_wire_schema_version`.
   - Follow the AGENTS.md recipe (parse the dict into the request wire,
     `serialize_to_py`, and map errors to `ValueError` with a stable code prefix).
   - Add round-trip tests in the domain's `tests.rs`.
   - Bindings run inside `py.allow_threads`.
9. **sase side.**
   - Add a thin `src/sase/core/goal_ledger_facade.py`:
     - typed wrappers over the bindings via `require_rust_binding("<literal>")`;
     - a Python mirror `GOAL_LEDGER_WIRE_SCHEMA_VERSION = 1` with a stale-core check.
   - Add every binding name to `REQUIRED_BINDINGS` in `tools/validate_sase_core_rs`,
     with a schema check and a validator test.
   - Keep binding names as string literals for `tools/check_sase_core_rs_bindings`.
   - Whitelist facade symbols consumed only by later phases with
     `--epic-symbol '<this epic bead>(<Symbol>)'`. Read `symvision.md` with
     `/sase_memory_read` first. `acceptance` removes every whitelist entry this epic
     adds.
10. Move `sase-core-revision.txt` past the sase-core commit with
    `just ratchet-core-revision`.
    - Parallel phases may have moved the pin meanwhile. The pin only moves forward, so
      re-ratchet after rebasing.
11. Tests in Rust:
    - marker-superset ordering under the fault hook;
    - `stale_basis` and re-plan;
    - projection invalidation by signature;
    - fail-closed `STORE.json`;
    - an unreadable goal isolated from its neighbors;
    - doctor repair idempotence;
    - an I/O probe test with 1,000 settled plus 10 live goals showing zero opens under
      settled `items/`.
12. Tests in Python: facade round-trips on a temp ledger.
13. Verify with `sase tool run check` in sase-core, then in sase.

## Phase: ledger-root

- slug: ledger-root
- size: medium
- depends: [ledger-io]

1. **Config.** Add a `goals:` block to `src/sase/default_config.yml`, following the
   `gotchas` core memory:

   ```yaml
   goals:
     visibility: shared # shared | local
     host_role: beads # sidecar role whose repository hosts goals/
     fetch_ttl_seconds: 60 # background freshness fetch triggered by `sase goal list`
     push_timeout_seconds: 20 # bound on synchronous publish for goal writes
   ```

   Add typed config accessors and validation.

2. **Resolution.** In a new `src/sase/goals/` package, add `store.py` with
   `resolve_goal_ledger(project) -> GoalLedger`.
   - Fields: `mode: shared|local`, `root`, `host_role`, `hidden_clone`,
     `watermark_path`, `outbox_path`, `lock_path`, `projection_path`, and `reason`.
   - Shared mode requires that the project has the `host_role` sidecar, that the sidecar
     has a push remote, and that its hidden clone can be materialized.
   - Otherwise, or when `visibility: local`, use local-only mode with a human-readable
     `reason`.
   - Materialize the hidden clone through a **public** wrapper around the existing
     `_ensure_hidden_document_root` / `machine_document_sidecar_roots` logic in
     `src/sase/sdd/_artifact_link_machine_store.py`. Symvision forbids importing the
     private name across files.
   - Resolution writes the projection header (root pointer) so the fast path can find
     the ledger without config.
3. **Write transaction.** Add `src/sase/goals/write.py` with
   `apply_goal_action(ledger, action, actor) -> GoalWriteOutcome`.
   - Shared mode steps:
     1. Take the hidden clone's `store_git_write_lock`
        (`src/sase/sdd/_git_contention.py`).
     2. Commit any leftover uncommitted `goals/` files from a crash first, as
        `chore(goals): recover uncommitted ledger files`.
     3. Initialize `STORE.json` on the first write.
     4. Call `goal_ledger_append`.
     5. Commit exactly the returned paths with pathspec `goals/`, as
        `chore(goals): <verb> goal <id>`.
     6. Release the lock and publish.
   - Local mode appends only.
   - Authorization goes through `authorize_store_mutation`: the path is under
     `repos/<host_role>/`, so the hidden-sidecar machine context applies.
4. **Basic publish.** For the `sync` publish class, run the existing managed sync worker
   synchronously for the hidden clone, bounded by `goals.push_timeout_seconds`.
   - The worker is `push_bead_work_launch` → `run_managed_sync_worker` in
     `src/sase/bead/_sync_publication.py` and `src/sase/bead/sync_worker.py`. It uses
     the same `.git/sase-bead-sync.lock`, so goals and bead-link publishing serialize.
   - Return `published | unpublished(reason)`. A failed publish is never a failed write:
     the event is durable locally. `publish-sync` adds the outbox and retries.
5. **Bead coexistence.**
   - In `normalize_sdd_commit_pathspecs` (`src/sase/sdd/_commit_store.py`), exclude
     `goals/` (`:(exclude)goals`) whenever a bead pathspec resolves to a split store's
     repo root. Bead commits must never sweep goal files.
   - Add tests proving that bead readers, bead pages, `sase bead doctor`-style integrity
     checks, and the semantic conflict resolver ignore a top-level `goals/` directory.
6. **Tests.**
   - Resolution matrix: shared, `visibility: local`, missing host role, no remote,
     unmaterializable clone, and a custom `host_role`.
   - Write transaction commits only `goals/` paths.
   - Crash-leftover recovery.
   - A split-store bead commit leaves dirty `goals/` files alone.
   - Local-mode writes never invoke git.
   - Use `tests/sdd/test_artifact_link_hidden_clone_e2e.py` and
     `tests/sdd_store/_helpers.py` as fixture models (bare remote plus `SASE_HOME=tmp`).
7. Verify with `sase tool run check`.

## Phase: publish-sync

- slug: publish-sync
- size: medium
- depends: [ledger-root]

1. **Post-integration reconciliation.** After the hidden clone integrates (fetch +
   rebase), list the goal ids touched in the incoming range and in the local unpushed
   range.
   - Reduce them with `goal_ledger_doctor` scoped to those ids (add an `ids` filter in
     sase-core if needed, with a pin ratchet).
   - Commit any marker fix as `chore(goals): reconcile live markers` before pushing.
   - Hook this into integrations of the **hidden host clone only**: the goal publisher,
     the TTL fetch worker, and the hidden-clone bead-link publisher. The ledger then
     converges no matter which of them integrates.
   - Primary and workspace clones only ever receive `goals/` through pull, fast-forward,
     or rebase. Nothing writes goal files there.
   - Reconciliation fails open. An error logs, records a diagnostic that `doctor`
     reports, and lets bead publication continue. A goal problem must never block beads.
   - Goal files never conflict, so a rebase containing goal commits must never enter
     semantic conflict repair. Add a test.
2. **Fetch before mint.** In shared mode, `new` performs the bounded integration first.
   Then it mints with `goal_mint_id`, checking `items/<id>` locally and retrying on
   collision.
   - Offline, it mints against local state and says so.
   - A post-integration `id_collision` (two `created` events under one id) is reported
     by the reconcile step and by `doctor`, with the remedy spelled out: drop one and
     recreate it.
3. **Outbox.** Add `~/.sase/projects/<key>/goals-outbox.json`
   `{pending, since, attempts, last_error, last_attempt_at}`.
   - Written when a publish fails or times out, and cleared on success.
   - `GoalWriteOutcome` carries it, so the CLI can print `↑ unpublished — will retry`.
4. **Retry legs.**
   - (a) Every mutating `sase goal` command, and `list --fresh`, first retries a pending
     outbox.
   - (b) Add a push leg to the `sidecar_auto_sync` chop
     (`src/sase/scripts/sase_chop_sidecar_auto_sync.py`). For each project with a
     pending goals outbox, run one bounded publish of the hidden host clone, sharing
     that chop's backoff and work budget.
   - Today that chop only pulls into primary clones (see
     `research:202609/sase_goals_persistence.md` §4).
5. **Push-retry counter.** This is the contention signal that the roadmap uses to revise
   G2–G6 and to decide whether `goals.host_role` should split out.
   - Add an additive `push_attempts: int` field to `_ManagedSyncOutcome`
     (`src/sase/bead/sync_worker.py`) and to `PushOutcome`
     (`src/sase/bead/_sync_publication.py`), defaulting to 0 so bead callers are
     unchanged.
   - After each goal publish, update the machine-local
     `~/.sase/projects/<key>/goals-sync-stats.json`
     `{schema_version, publishes, push_retries, rejected_after_max, last_retry_at}`
     atomically. `push_retries` counts non-fast-forward rejections that were retried.
   - `goal_sync_status` includes these counters, and `sase goal doctor` prints them as
     one dim line. A missing or corrupt stats file reads as zeros and never fails a
     command.
6. **Freshness.**
   - The watermark is the mtime of the hidden clone's `.git/sase-bead-sync.integration`
     marker, which `integrate_sdd_repository` touches on every successful integration
     (`src/sase/sdd/_integration_marker.py`).
   - Expose
     `goal_sync_status(ledger) -> {mode, synced_at | never, unpublished, refreshing}`.
   - Add `src/sase/goals/fetch_worker.py`
     (`python -m sase.goals.fetch_worker <project>`): a detached, single-flight fetch
     that bails if another is running, holds a lock file next to the projection, and
     integrates, reconciles, and refreshes the projection.
   - `sase goal list` spawns it when the watermark is older than
     `goals.fetch_ttl_seconds`.
   - `--fresh` runs the same integration synchronously, bounded, before reading.
   - Local mode reports `local only` and never fetches.
7. **Tests.**
   - Push race retried, and `push_retries` is incremented exactly once for it.
   - Timeout → outbox → an auto-sync leg publishes → outbox cleared.
   - The watermark advances only on a successful integration.
   - The single-flight fetch never runs twice concurrently.
   - A reconcile after an injected concurrent settlement race (write synthetic events
     through the binding) fixes the marker.
   - `tests/test_bead/test_sync_remote_push.py` models the push race.
8. Verify with `sase tool run check`.

## Phase: cli

- slug: cli
- size: medium
- depends: [publish-sync]

### Command surface

Bare `sase goal` runs `list`. Subcommands and options are alphabetical, and every long
option has a short alias (`cli_rules` memory).

```text
sase goal doctor [-j/--json] [-r/--repair]                                   # --repair: human only
sase goal drop ID -w/--why TEXT                                              # human only
sase goal edit ID [-c/--criterion TEXT]... [-o/--outcome TEXT] [-t/--title TEXT]
                  [-x/--remove-criterion N]...                               # human only
sase goal list [-a/--all-projects] [-f/--fresh] [-j/--json] [-n/--limit N]
               [-s/--status {unsettled,active,review,done,dropped,settled,all}]
sase goal merge ID -i/--into GOAL [-w/--why TEXT]                            # human only
sase goal new -o/--outcome TEXT -t/--title TEXT [-c/--criterion TEXT]...     # human only
sase goal reopen ID -m/--message TEXT                                        # human only
sase goal show ID [-j/--json]
```

- `-s` defaults to `unsettled`. `done`, `dropped`, `settled`, and `all` run the history
  scan, newest first, with `-n` defaulting to 20. Help text says that these scan
  history.
- `ID` accepts `7k2mq`, `⌖7k2mq`, `goal:7k2mq`, and `goal:<project>@7k2mq`.

### Human-only refusal

`new`, `edit`, `drop`, `reopen`, `merge`, and `doctor --repair` refuse inside an agent
run.

- An agent run means `discover_agent_identity()` returns an identity, or `SASE_AGENT` is
  set. This matches the `sase service init` guard in `src/sase/service/platform.py`.
- On refusal, print one styled line to stderr and exit 2:
  `sase goal drop is a human verb: only a person creates, reshapes, or settles a goal. Agents may run \`sase
  goal list\` and \`sase goal show\`.`
- Model the tests on `tests/test_bead/test_bead_show_agent_guard.py`.

### Rendering

Rendering happens in sase-core, in `goal/render/terminal.rs`, consuming the view models.

- **List on a TTY:**

  ```text
  ⌖ Goals · sase                                  1 review · 2 active · synced 12s ago
    REVIEW
    ⌖ 7k2mq  Goals feature design                                            3m
    ACTIVE
    ⌖ 3fq9t  Tailnet dispatch mesh works on every machine                    2h
    ⌖ b81xd  Deck paging polish                                              8m
  ```

- **List off a TTY, or inside an agent run:** one compact line per goal,
  `⌖ 7k2mq  review  Goals feature design  · 3m`.
- **Footer chips:** `↑ 2 unpublished` (warning color), `local only`, `refreshing…`, and
  `⚠ 1 unreadable (run sase goal doctor)`.
- **Empty state:**
  `No active goals in sase. Start one: sase goal new -t "<title>" -o "<what will be true when done>"`.
- **The card (`show`):**

  ```text
  ⌖ Goals feature design                                              ACTIVE
  goal:7k2mq · sase · opened 2h ago by bryan.athena · rev 3

  OUTCOME    A critiqued, recommended design for Goals exists.
  CRITERIA   1  Covers storage and sync                                 user
             2  Names every open question                               user
  MERGED     ← goal:3fq9t "Old goals idea" · 1h ago
  TIMELINE   2h  created by bryan.athena
             1h  edited title
             3m  merged goal:3fq9t into this goal
  ```

- **Write confirmations** are one or two lines, with past-tense verb, ref, and title:

  ```text
  ⌖ Created goal:7k2mq  Goals feature design
    published · cite it as @goal:7k2mq
  ```

  or `saved locally · ↑ publish failed (timeout) — will retry automatically`.

- **Visual grammar:**
  - Status is always text plus glyph, never color alone.
  - The accent is `GOAL_ACCENT_HEX`, used on `⌖` and ids.
  - Review is bold accent. Done is green and dimmed. Dropped is grey. Unreadable uses
    the warning color.
  - Honor `NO_COLOR` / `FORCE_COLOR` / TTY via the flag the caller passes; Python uses
    `sase.core.term_color.should_colorize`.
  - Rows keep a stable order: lane, then most recent activity, then id.
  - Verify that `⌖` measures one cell wide.

### Output, fast path, and registration

- **JSON.** `-j` prints `GoalListWire` / `GoalStateWire` verbatim with `schema_version`.
  That shape is what G5 and the gateway will consume. Slow- and fast-path JSON must be
  identical (test).
- **Fast path.** Add `src/sase/main/goal_fast_path.py`, early-dispatched in
  `src/sase/main/entry.py` right after the bead fast path.
  - Handles: bare `goal`, `goal list` with only `-j` and unsettled `-s` values, and
    `goal show ID [-j]`.
  - How it resolves: it finds the nearest `.sase/checkout.json`, then the project name,
    then `goals-hot.json`, and calls one binding that returns
    `{handled, exit_code, stdout, stderr, spawn_fetch}`.
  - It must print the same bare-`goal` delegation notice that
    `default_list_delegation_notice()` prints.
  - Anything else falls back to argparse (`-h`, `-a`, `-f`, history statuses, a missing
    projection, or a non-canonical project).
  - It imports only the stdlib and the binding loader. An import-isolation subprocess
    test asserts that `argparse`, `rich`, and `sase.config` are not loaded; model it on
    `tests/main/test_parser_narrowing.py:194` and the `completion_fast_path` contract.
  - It spawns the TTL fetch worker when told to.
- **Registration.**
  - `_COMMAND_REGISTRARS` in `src/sase/main/parser_registry.py` and
    `COMMAND_REGISTRARS_BY_NAME` in `parser_full_registrars.py`.
  - A new `src/sase/main/parser_goal.py`, modeled on `parser_flag.py`, with an
    `examples:` epilog and the "defaults to `sase goal list`" sentence.
  - A `# --- goal ---` block in `entry.py`, a thin `goal_handler.py`, and domain
    handlers in `src/sase/goals/cli*.py`.
  - Add `"sase goal"` to `expected_groups` in
    `tests/main/test_parser_command_defaults.py`.
  - Add a short-alias/alphabetical test like `tests/main/test_artifact_handler.py:182`.
  - Regenerate `tests/completion/snapshots/cli_spec.json` with
    `just sync-completion-spec`.
  - Add the write verbs to `_WRITES_TRUE_OVERRIDES` in
    `src/sase/completion/run_policy.py`.
- **`-a/--all-projects`** (slow path): iterate enabled projects and render one section
  per project, skipping ledgers that don't exist yet with a dim `no goals yet` line.
- **Docs.** Add `sase goal` rows to `docs/cli.md`, including the "defaults to `list`"
  paragraph. `docs/goals.md` itself is written in `acceptance`.
- **Tests:**
  - every verb, in shared and local mode;
  - refusal in an agent run for each human verb, while `list`, `show`, and `doctor` are
    allowed;
  - renderer golden _text_ tests in Rust (not PNGs);
  - fast path vs. slow path byte equality for list and show, with and without color;
  - the empty state;
  - an unpublished footer;
  - an unreadable goal row.
- Verify with `sase tool run check` in sase-core (renderer) and sase. Ratchet the pin.

## Phase: artifact-kind

- slug: artifact-kind
- size: medium
- depends: [ledger-root]

### sase-core (`feat!`)

- Add `ArtifactRefKindWire::Goal` and
  `ArtifactRefPayloadWire::Goal { project: Option<String>, id }`.
- Add a `KindRegistration` in `crates/sase_core/src/artifact_ref/kinds.rs`:
  - `kind: "goal"`, `display_name: "Goal"`, `status: Live`, `reserved: true`;
  - `argument_summary: "goal:<id> or goal:<project>@<id>"`;
  - `offered_in_completion: true`, and no fragments.
- Parse and validate using `goal::model::ids`. Then render, and update
  `kind_rejects_fragments` and the reserved list in `provider_spec.rs`.
- `resolve_artifact_ref` returns a builtin-entry-required resolution for goals (Python
  supplies ledger context).
- `artifact_link/path.rs`: goals have no artifact markdown page. Return the "none" path
  like stitch/commit.
- Pager scanning (`scanner.rs`) picks up every catalog kind automatically. Test that
  YAML-style `goal: text` (plan frontmatter) stays a malformed candidate that is
  dropped. Plan bodies must never linkify `goal:` followed by a space.
- Append `goal` to `BUILTIN_ARTIFACT_REF_KINDS`
  (`crates/sase_core/src/editor/at_reference.rs`) and give it an LSP icon. Update the
  two exact-list tests. Editor and LSP payload completion stay kind-only in G1.
- Add a markdown renderer `goal/render/markdown.rs`:
  - `goal_card_markdown(card)` produces the full card for `sase artifact read`, with
    absolute times;
  - `goal_citation_line(card)` produces a single line of at most 400 characters for
    prompt expansion, e.g.
    `goal ⌖7k2mq "Goals feature design" in the sase project (active) — outcome: A critiqued, recommended design for Goals exists. (+2 criteria: sase goal show 7k2mq)`.
- Add bindings for both renderers. Ratchet the pin.

### sase

- Update the Python mirrors `src/sase/artifact_ref_wire.py` and
  `src/sase/artifact_ref_parsed_models.py`.
- Add `goal` to `BUILTIN_ENTRY_KIND_TYPES` and a new
  `src/sase/artifact_providers/builtin_entry_goal.py`, modeled on
  `builtin_entry_bead.py`.
  - It resolves through `resolve_goal_ledger` and `goal_ledger_show`.
  - It fills `ArtifactEntry.properties` with id, status, title, and outcome.
  - An unknown id is an unresolved entry with a helpful message.
- **`sase artifact read goal:<id>`** prints the markdown card: a card branch in
  `artifact_cli/read.py` `_prepare_body`, like `_stitch_body`.
- **`show`** prints metadata. **`path`** prints the goal's `items/<id>` directory.
  **`open`** pages the card.
- **Prompt expansion.** `@goal:<id>` expands to `goal_citation_line`, in
  `artifact_ref_prompt_rendering.py` and `artifact_ref_prompt_resolution.py`.
  - An unknown goal fails the launch, the same as other unresolved refs.
  - Citing a settled goal is allowed.
  - A citation never binds; binding is G2's `%goal`.
- Add `goal` to `_NON_FILE_REF_KINDS` in `src/sase/core/prompt_artifact_staging.py`.
  Otherwise it is silently not staged.
- **ACE `@` completion.**
  - The kind row appears automatically from the catalog.
  - Add `@goal:` payload rows (id plus title, unsettled goals only) read from the hot
    projection off the event loop, per the TUI performance rules in `tui.md`. Read it
    with `/sase_memory_read`.
  - Add a badge in `_prompt_input_bar_completion_rows_artifacts.py`.
- **Documentation.** Document the kind wherever the builtin kinds are documented under
  `docs/`. The artifact kind docs list `@goal` as a citation that never binds.
- **Zero golden churn.**
  - Run the targeted visual checks in check-only mode:
    - `just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_at_reference_completion.py`;
    - the plan-gate snapshot tests;
    - the pager snapshot tests.
  - Route them through `/sase_monitor` if they are long.
  - Any drift is a bug in this phase, not a golden update.
- **Tests:**
  - Rust: kind catalog, parse, render, fragments, scanner, editor and LSP lists.
  - Python: read, show, path, expansion, staging, completion rows, cross-project
    `goal:<project>@<id>`, and a settled-goal citation.
- Verify with `sase tool run check` in both repos.

## Phase: acceptance

- slug: acceptance
- size: medium
- depends: [cli, artifact-kind]

### Cross-cutting fixtures

Add these in `tests/goals/`, using bare remotes plus two hidden clones under separate
`SASE_HOME`s.

1. **Two-clone convergence.**
   - Concurrent `new`, `edit`, `drop`, `reopen`, and `merge` from both clones, on the
     same goal and on different goals.
   - Every rebase is conflict-free.
   - Both clones end with byte-identical `sase goal list -j -s all` output after
     integrating.
2. **Marker race.** Inject the concurrent settlement race (a verify-vs-reject pair
   written through the binding as `host` actors). Integration reconciles the marker on
   both clones.
3. **Id collision.** Two clones mint the same id through an injected minting source.
   `doctor` reports `id_collision` and prints the remedy.
4. **Crash repair.**
   - Use the fault hook to stop between the event and the marker step. `list` stays
     correct under the superset invariant.
   - A crash before commit leaves dirty `goals/` files. The next write recovers and
     commits them.
   - `doctor --repair` converges.
5. **Fail closed.**
   - `STORE.json` schema 2 makes every verb refuse with the update hint.
   - An unknown event kind makes one goal `⚠ unreadable` while its neighbors render.
6. **Offline.**
   - A remote that is unreachable during `new` produces a durable local goal, an
     `↑ unpublished` footer, and an outbox entry.
   - The auto-sync push leg publishes it once the remote returns.
7. **Agent refusal matrix.** Every human verb is refused, and `list`, `show`, and
   `doctor` succeed.

### Benchmark

Add `tools/goal_ledger_bench`.

- It seeds ledgers through the binding: 0, 1k, and 100k settled goals, and 10, 100, and
  1,000 unsettled goals.
- It measures the Rust warm and cold hot reads, and end-to-end `sase goal list` (fast
  path, warm) as p50 and p95 over 50 runs.
- It reports I/O-probe open counts by path class.
- It reports the push-retry counters from a contention run: two hidden clones against
  one bare remote, each publishing 50 interleaved goal writes.
- It writes a markdown report and registers it with
  `sase artifact create -p <report> -l "Goal ledger benchmark"`.
- The performance contract must hold.
  - A miss with a local cause is fixed in this phase.
  - A miss that needs structural change is recorded with its measurements as a `feature`
    task bead (via `/sase_new_task`) and named in the land notes. It is never planned as
    a child epic.

### Docs

Write `docs/goals.md` and register it in the `mkdocs.yml` nav under "Beyond the Basics",
next to Beads. It covers:

- what a goal is, and how it differs from a bead and from a plan's `goal:` field;
- the statuses;
- the CLI tour;
- citing goals;
- where goals live, what publishing discloses, and `goals.visibility: local`;
- freshness (`synced Ns ago`, `--fresh`, `↑ unpublished`);
- `doctor`;
- a short "coming next" paragraph naming binding, claims, drafts, the Goals tab, and the
  attention cutover without promising dates.

Link it from `docs/cli.md`. `just docs-check` must pass.

### Memory

The user chose all six memory changes below while this plan was being drafted, and
approving the plan authorizes them. Use `/sase_memory_write` first, then run
`sase memory init`. Together they add about 150 always-loaded tokens: two decision
roster lines and one glossary term.

- **New** `sase/memory/decisions/goals-host-binds.md` (D1):
  - Claim: the host binds every LLM turn to exactly one goal before spawn; agents only
    name or adopt their own draft and claim or keep open; only a human settles.
  - Author `[[...]]` links to `host-owned-completion`, `gates-never-block`, and
    `receipts-prove-before-they-skip`. The other new and edited notes also link every
    memory they name in prose.
  - Reopens when a launch path can be bound neither in the runner nor on the unit wire,
    or per-model claim precision stays under the floor and a per-model policy cannot
    contain it.
  - It holds invariants only: no fail-open, storage, or attention clauses.
- **New** `sase/memory/decisions/goal-ledger.md` (D2):
  - Claim: a goal is its own Rust-owned domain, not a bead type. State is immutable
    events plus live markers, written through the hidden clone. Hot reads are
    O(unsettled). Freshness is shown, never promised. Drafts stay local until named.
  - Cite the benchmark's measured list latency and push-retry behavior.
  - Reopens when push contention or hot-list p95 grows with settled history. Splitting
    `goals.host_role` is a config change, not a reopening.
- **New** `sase/memory/glossary/goal.md` (stage 1; aliases `goals`, `⌖`):
  - identity `goal:<id>`;
  - the statuses, with Running/Idle derived and not stored;
  - `@goal:<id>` cites and never binds;
  - "not to be confused with" a plan's `goal:` frontmatter field, a task bead, or agent
    status.
- **Edit** `sase/memory/glossary/artifact.md`: add "a goal" to the list of durable
  records.
- **Edit** `sase/memory/glossary/artifact-reference.md`: add `@goal` to the builtin
  kinds, noting that it cites a goal and never binds one.
- **Edit** the Model sentence in
  `src/sase/main/init_memory/templates/memory-sase-artifacts.template.md` to add goals,
  then regenerate with `sase memory init`.
- `sase memory init --check` is clean.

### Cleanup and verification

- Remove every `--epic-symbol` whitelist entry this epic added.
- Confirm that `sase-core-revision.txt` is at or past the last sase-core commit.
- Verify with `sase tool run check` in sase-core and in sase.

## Definition of done (frozen)

The land agent audits exactly these items:

1. The sase-core `goal` module implements the frozen contract. Fixture files cover every
   event kind, and `sase tool run check` passes in sase-core.
2. sase's `sase-core-revision.txt` is at or past the final sase-core commit, and
   `sase tool run check` passes in sase. A PyPI floor or window move is **not**
   required.
3. Every `sase goal` verb works in shared and local-only mode, and every human-only verb
   is refused inside agent runs.
4. The two-clone convergence, marker-race, id-collision, crash-repair, fail-closed,
   offline, and refusal fixtures pass.
5. The benchmark report exists as an artifact, includes the push-retry counters, and the
   performance contract holds. Otherwise each miss has a filed task bead with
   measurements.
6. `goal:` reads, shows, expands, stages, and completes. The targeted visual checks show
   zero golden PNG churn.
7. `docs/goals.md` is in the mkdocs nav, and `just docs-check` passes.
8. The memory items listed under the `acceptance` phase have landed, and
   `sase memory init --check` is clean.

**Out of scope: sibling epics, never child epics of this one.** The land agent must not
plan any of the following. Each is a separate sibling epic planned from what G1
measures.

- **G2, deterministic binding:** `%goal`, `SASE_GOAL_ID`, plan-derived goals,
  inheritance across launch paths, `agent_attached` / `plan_attached` producers,
  `pursues` / `defines` relations, and `sase goal audit`.
- **G3, claims and verification:** `builtin@goal`, the `GoalVerify` gate, `verify` /
  `reject` verbs, `progress`, `evidences`, `sase goal stats`, and the `/sase_goal`
  skill.
- **G4, drafts:** the draft store, `name` / `adopt`, the intake block, answer
  acknowledgement, `follows`, and standing goals.
- **G5:** the Goals tab and Agents-tab integration.
- **G6:** the attention cutover.

Manual athena↔apollo checks (below) are for the user, not landing blockers.

## Check it yourself (≈ 5 minutes)

1. On athena, run `sase goal new -t "Try goals" -o "I can see this goal from apollo"`.
   Then run `sase goal list --fresh` on apollo: the goal appears, with `synced 0s ago`.
2. `time sase goal list` finishes in under 50 ms.
3. `sase goal drop <id> -w "tried it"` removes the goal from the list, and
   `sase goal list -s dropped` shows it.
4. `sase artifact read goal:<id> "check the card"` renders the card, and a prompt citing
   `@goal:<id>` expands to the one-line goal citation.
5. Inside any agent, `sase goal new …` is refused with the human-verb message.
