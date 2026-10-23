---
tier: epic
title: Flag stranded epic work in ACE
goal: "An in_progress phase or epic bead that nothing will ever resume can no longer sit
  unnoticed. sase-core classifies it once. ACE flags it with a single amber ⚐ vocabulary
  on the stuck row, on everything waiting on it, on its epic clans, and in the Beads
  pane, and any flagged row resolves it with one keystroke.

  "
decisions:
  flag_tasks:
    ask:
      Also flag in_progress task beads whose `sase bead work` worker ended without
      closing them?
    default: false
    why: The request targets epic phases; a stranded task blocks no waiter or land agent
    answer: true
  doctor_report:
    ask:
      Make `sase doctor` report stranded beads through the shared classifier, replacing
      its in_progress-gone warning?
    default: true
    why:
      One definition everywhere; the old check misfires during follow-up turns and
      ignores successors
    answer: true
phases:
  - id: core
    title: Stranded-work reducer in sase-core
    depends_on: []
    size: medium
    description:
      "core: add the pure Rust reducer that maps bead and session-member facts to
      owned/delegated/settling/stranded/blocked/unknown verdicts. It includes causes,
      ancestor propagation, wires, unit tests, and the `stranded_work_reduce` PyO3
      binding."
  - id: collect
    title: Python fact collector, facade, presentation vocabulary, and doctor
    depends_on:
      - core
    size: medium
    description:
      "collect: move the core pin. Add the binding facade and the cheap, dismissal-proof
      fact collector, and the shared ⚐ presentation vocabulary that ACE and doctor both
      use. Add the doctor report and the bead-lifecycle reference docs."
  - id: agents-rows
    title: Agents tab rows, clans, waiters, and info strip
    depends_on:
      - collect
    size: medium
    description:
      "agents-rows: add the off-thread stranded-view cache and its warmup lane. Render
      the STRANDED overlay, waiter and clan ⚐ tokens, breadcrumbs, the ▲ wave fold, and
      the info-strip count, with row-glyph docs, help legend, and PNG goldens."
  - id: agents-detail
    title: Agents detail callouts, toast, and query field
    depends_on:
      - agents-rows
    size: small
    description:
      "agents-detail: add the AGENT TURN header callout, the [beads] wait-lane and WAVES
      markers, a transition toast that never fires at startup, and the `stranded`
      live-query field, with docs and PNG goldens."
  - id: beads-pane
    title: Beads pane stranded badges
    depends_on:
      - collect
    size: small
    description:
      "beads-pane: add a deferred stranded pass and the phase/epic ⚐ badges, the legend
      entry, the status-bar count, and the detail property row, with docs and a PNG
      golden."
  - id: resolve
    title: One-keystroke resolution from any flagged row
    depends_on:
      - agents-detail
      - beads-pane
    size: medium
    description:
      "resolve: add a `stranded` Enter target and a resolve modal offering Continue (a
      prefilled `%id(bead=…) #fork:<last turn>` takeover) or Close bead (the shared
      BeadCloseModal). Wire it to `w` on Beads-pane rows, with help, docs, and a PNG
      golden."
proposed_by: bbugyi200.athena.13e
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-23 14:22:46
status: wip
---

- **PROMPT:**
  [prompts/202610/stranded_epic_work.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/stranded_epic_work.md)

# Plan: Flag stranded epic work in ACE

## Why

The `sase-1n6` E2 release-train epic made no critical-path progress for about 68 hours.
The research report `research:202610/sase_1n6_e2_stall_culprits.md` traced the stall to
one leaf: phase bead `sase-1n6.5.1.5.2.1.2`. Its third turn (`--3`) committed the fix
and then ended with `bead_action: keep`, no monitor, and no successor. The bead stayed
`in_progress` with no owner. The epic's land agent, and every ancestor land agent above
it, waited on that bead forever. Nothing in ACE showed it:

- The phase row read a calm green `DONE`.
- The waiting land row showed an ordinary `◐` bead token.
- The clan wave pips folded the phase to `●` done.

The report's first "close the hole" recommendation is to flag any IN_PROGRESS phase bead
whose session has ended with `bead_action: keep` and no live monitor or successor. This
epic delivers that flag. It goes slightly further where the flag would otherwise be
unreliable, as explained below.

## The concept: a stranded bead

A **stranded** bead is an `in_progress` work bead that nothing will ever resume:

- no member of its owning agent session is active;
- no monitor turn, gate turn, handoff, queued or waiting successor, or auto-restart
  recovery will continue it;
- no launched child epic carries its work.

The owning session is the bead's `assignee`. Epic launches preclaim each phase to its
phase agent and the epic bead to its land agent, and any `%id(bead=…)` launch reassigns
the bead.

Work beads are phase beads and launched epic plan beads (owned by their land agents).
Epic beads are in scope by design, not only phases. A phase that authored a child epic
delegates its work to that epic, and the child's land agent closes the phase. If that
land agent ends without closing anything, the parent phase would look healthy
("delegated") while the chain is dead. Evaluating epic beads with the same rule is what
makes nested chains such as `sase-1n6`'s seven-level nesting correct.

> [!decision] flag_tasks When this decision is on, standalone task beads launched with
> `sase bead work` are classified too. Their rows and Beads-pane task rows get the same
> flags.

### Verdict states (one per candidate bead)

| State       | Meaning                                                                                          | Flagged?                 |
| ----------- | ------------------------------------------------------------------------------------------------ | ------------------------ |
| `owned`     | Some session member is active, or a monitor, gate, handoff, waiter, or recovery will continue it | no                       |
| `delegated` | No owner, but a launched (`in_progress`/`claimed`) child epic plan bead carries the work         | no                       |
| `settling`  | It would be stranded, but its last activity ended less than `settle_grace_seconds` (120 s) ago   | no                       |
| `stranded`  | Nothing will resume it. This is the leaf where a human must act                                  | **yes, ⚐ bold amber**    |
| `blocked`   | It is owned or delegated, but cannot progress because a stranded bead below it never closes      | **yes, ⚐ breadcrumb**    |
| `unknown`   | No session records could be found (pruned artifacts, remote-only owner, index unavailable)       | quiet (detail text only) |

### Stranded causes, in precedence order (first match wins)

1. `followup_failed`: the newest session member is a monitor or gate turn whose
   follow-up errored, launched degraded, or settled `lost`/`stopped` without launching a
   follow-up.
2. `failed`: the latest agent turn failed and no recovery is in flight.
3. `stopped`: the latest agent turn was killed or stopped.
4. `lost`: the latest agent turn is non-terminal, but its process is gone. This covers a
   record with no done marker and a dead process, and never a remote record.
5. `child_landed`: a child epic of this phase closed, and no newer agent turn followed,
   but the phase is still open.
6. `close_failed`: the latest turn declared `bead_action: close`, but the bead is still
   open (for example, the close was refused because of leftover epic symbols).
7. `kept`: the latest turn declared `bead_action: keep`. This is the case the report
   describes.
8. `unclosed`: the latest turn finished without closing the bead and without declaring
   any `bead_action`. This happens, for example, when it had no dirty repo, so the
   commit finalizer recorded nothing.

`kept` alone is not a reliable trigger. `bead_action` lives only in one turn's
`final_submission.json`, and a turn with no dirty repository records none. "No close
happened and nothing resumes it" is the real condition. The declared action only selects
the explanation.

### Invariants (tests enforce these)

- **Never contradict ACE.** Suppose any member of the owning session would render in ACE
  as `STARTING`, `RUNNING`, `WAITING`, `QUEUED`, a question or asking state, or
  `FINALIZING`. Suppose instead that a monitor's `next_action` or a gate's follow-up is
  still pending. In either case the bead is not stranded.
- **Positive evidence only.** "No records found" is `unknown`, never `stranded`. A
  non-terminal member owned by another machine counts as an owner.
- **Survives dismissal.** Dismissing a phase agent's rows does not hide the flag.
  Dismissal lives in the artifact index's separate `dismissed_agents` table, and the
  collector reads session records regardless of visibility.
- **Bead-anchored.** Every flag derives from the bead store plus session records, never
  from which rows happen to be loaded.

### Naming

ACE calls the state **stranded** and never "orphaned". `sase bead doctor` already
reports "orphan phase records" (a phase whose parent bead is missing), and ACE's
beads-navigation code uses "orphan phase" in the same sense. Doctor's own claim advisory
already names this condition `stranded` internally (`src/sase/doctor/checks_beads.py`).

## Design principles

**Intuitive.**

- One word (`STRANDED`), one glyph (`⚐`), and one hue: the amber `#FFAF00` that ACE
  already uses for "needs you" (the `S` chip, `▲` wave pips, and `⚑`).
- Flags appear where the user is already looking: the stuck row, the rows waiting on it,
  its epic clan, the Beads pane, and a global count.
- Ancestors carry a dimmer breadcrumb ⚐, so following the amber leads from the top-level
  epic clan down to the leaf.
- Enter on any flagged row opens the fix.

**Reliable.**

- One classifier in sase-core, fed by one Python collector, used by ACE and (decision)
  `sase doctor`.
- It only flags on positive evidence, has a settle grace, and stays quiet when evidence
  is unknown.
- Render paths never touch disk. The scan runs off the event loop on the existing bead
  warmup lane, behind a TTL cache with fingerprint invalidation.

**Beautiful.**

- It reuses ACE's existing grammar: the parenthetical status word (like the `FINALIZING`
  overlay), wait tokens, wave pips, and the header callout line used by the blocked
  epic-follow (`↪ ! … · resume: …`).
- Exactly one new glyph. Leaves are bold and breadcrumbs dim.
- Zero new chrome when nothing is stranded: every token is zero-suppressed.

## Visual vocabulary (defined once in `collect`, used by every surface)

| Element          | Rendering                                                                              | Where                                               |
| ---------------- | -------------------------------------------------------------------------------------- | --------------------------------------------------- |
| Leaf glyph       | `⚐` (U+2690, covered by the bundled DejaVu font) in `bold #FFAF00`                     | stranded rows, bead tokens, badges                  |
| Breadcrumb glyph | `⚐` in `#AF8700` (dim amber, not bold)                                                 | blocked rows, tokens, and clans                     |
| Overlay word     | `STRANDED` in `bold #FFAF00`                                                           | the owning session row when its bucket is Done      |
| Global count     | `⚐ N stranded`, zero-suppressed                                                        | Agents info strip, Beads status bar                 |
| Age              | compact age since the last activity (`31h`, `2d`), via the existing duration formatter | detail callouts, badges, modal (never on list rows) |
| Cause sentence   | short sentence per cause, e.g. `ended with bead_action keep; nothing resumes it`       | detail callouts, modal, toast, doctor               |

`⚐` (hollow flag) deliberately pairs with the existing amber `⚑` (filled flag, "the
follow-up failed"). Filled means a successor broke. Hollow means there was never anyone
to hold it.

## Mockups (the `sase-1n6` shape)

Agents tab, with the leaf epic clan expanded and its ancestors collapsed:

```
26 [5 running · 15 waiting · 6 done] · ⚐ 1 stranded · group: by project (o)
▸ sase-1n6 (WAITING) [W5 D4] ⚐1                       ← breadcrumb: something below is stranded
▸ sase-1n6.5.1.5 (WAITING) [W1 D3] ⚐1                 ← breadcrumb
▾ sase-1n6.5.1.5.2.1 (WAITING) [W1 D2] ⚐1  ●▲∎        ← bold: a direct phase is stranded; its wave pip is ▲
   1 ─ sase-1n6.5.1.5.2.1.1 (DONE)
   2 ─ sase-1n6.5.1.5.2.1.2 (STRANDED ⚐)
   ∎ ─ sase-1n6.5.1.5.2.1.land (WAITING ⚐ sase-1n6…2.1.2)
```

The AGENT TURN header of the stranded row:

```
sase-1n6.5.1.5.2.1.2 · CODEX(gpt-6-luna)
⚐ STRANDED 31h · --3 ended with bead_action keep; nothing resumes it · ⏎ resolve
```

The resolve modal:

```
╭─ ⚐ Stranded phase ───────────────────────────────────────────╮
│ sase-1n6.5.1.5.2.1.2 · Live proof                             │
│ in_progress · stranded 31h                                    │
│                                                               │
│ Last turn  sase-1n6.5.1.5.2.1.2--3 · ended Oct 22 06:28       │
│ Why        ended with bead_action keep; nothing resumes it    │
│ Blocking   6 beads · sase-1n6.5.1.5.2.1 … sase-1n6            │
│                                                               │
│  f ⏎  Continue: fork the last turn; the fork takes over       │
│  c    Close bead: the work is done; record a note             │
│  esc  Cancel                                                  │
╰───────────────────────────────────────────────────────────────╯
```

Beads pane:

```
— Epics (3/3) · ⊜ blocked ► ready ▶ launched ⚐ stranded
▾ ⚐ ▤ sase-1n6.5.1.5.2.1 Live proof  ◐ in_progress  1/2 phases
   ↳ sase-1n6.5.1.5.2.1.2 Prove the release  ◐ in_progress ⚐ 31h
```

## Architecture

```
bead store (read model) ─┐
artifact index (by agent_session, incl. dismissed) ─┼─► collector (Python, off-thread) ─► stranded_work_reduce (Rust, pure)
handoff/liveness/finalizer markers ─┤                       facts JSON                         verdicts + propagation
final_submission.json (latest turn) ─┘                                                              │
                     ┌──────────────────────────────────────────────────────────────────────────────┤
                     ▼                                                                              ▼
     ACE stranded view cache (TTL 30 s, fingerprint/roster keyed)                     sase doctor (decision)
     → rows · waiters · clans · info strip · detail · toast · Beads pane · resolve modal
```

The Rust reducer owns the domain rule: what counts as an owner, delegation, causes,
grace, and propagation. Per the Rust-core boundary, every frontend must agree on it.
Python owns only fact gathering: process liveness, marker files, and file reads that the
host already performs today. This mirrors the `wait_epic_follow` reducer split.

## Phase `core`: stranded-work reducer in sase-core

Open the linked repo with `sase repo open sase-core` and follow its `AGENTS.md` ("Add a
core function and expose it to Python"). Model everything on
`crates/sase_core/src/wait_epic_follow.rs`: a pure function with no I/O, `serde` wires
with `#[serde(deny_unknown_fields)]` and defaults, and deterministic output.

**Module.**

- `crates/sase_core/src/stranded_work/`. Its `mod.rs` is a facade holding only `mod` and
  `pub use` lines. It contains `wire.rs`, `classify.rs`, `propagate.rs`, and `tests.rs`.
- Register the module in `lib.rs`.
- Entry point:
  `pub fn reduce_stranded_work(input: &StrandedWorkInputWire) -> StrandedWorkReportWire`.

**Input wires.**

- `StrandedWorkInputWire { schema_version: u32 (=1), now: f64, settle_grace_seconds: f64 (default 120.0), beads: Vec<StrandedWorkBeadFactsWire> }`
- `StrandedWorkBeadFactsWire { id, kind: "phase"|"epic"|"task", status, candidate: bool, assignee: Option<String>, parent_id: Option<String>, closed_at: Option<f64>, records_available: bool, members: Vec<StrandedWorkMemberFactsWire> }`
  - Non-candidate beads, such as closed or open child plans, are context for delegation
    and `child_landed`.
- `StrandedWorkMemberFactsWire { name, artifact_dir, turn_kind: "agent"|"monitor"|"gate", lifecycle: "starting"|"running"|"asking"|"waiting"|"queued"|"done"|"failed"|"stopped"|"unknown", process_live: bool, remote: bool, finalizing: bool, pending_handoff: Option<String>, handoff_inflight: bool, recovery_in_flight: bool, turn_state: Option<String>, followup_pending: bool, followup_outcome: Option<String>, followup_agent: Option<String>, declared_bead_action: Option<String>, started_at: Option<f64>, ended_at: Option<f64> }`

**Output wires.**

- `StrandedWorkVerdictWire { bead_id, kind, state, reason: Option<String>, owner: Option<String>, delegates: Vec<String>, session: Option<String>, last_turn: Option<String>, last_turn_artifact_dir: Option<String>, declared_bead_action: Option<String>, since: Option<f64>, parent_id: Option<String>, blocked_by: Vec<String>, blocks: Vec<String> }`
  - `reason` holds the cause code for `stranded`, the owner kind for `owned`
    (`running`/`asking`/`waiting`/`queued`/`finalizing`/`handoff`/`monitor`/`gate`/`recovery`),
    and `no_records`/`records_unavailable` for `unknown`.
- `StrandedWorkReportWire { schema_version, verdicts (sorted by bead_id), stranded_ids, blocked_ids }`

**Rules, per candidate bead.**

1. **Owner.**
   - Any member that is non-terminal and has `process_live || remote`, or is
     `finalizing`, or has `pending_handoff` or `handoff_inflight`, or
     `recovery_in_flight`, makes the bead owned.
   - So does a monitor or gate member whose `turn_state` is still running, pending, or
     settling, or whose `followup_pending` is true.
   - A monitor or gate whose follow-up launched passes ownership to `followup_agent`,
     which is itself a member.
   - The owner is the newest qualifying member.
2. **Delegation.** Otherwise, if any child bead (`parent_id == id`, kind `epic`) has
   status `in_progress` or `claimed`, the state is `delegated` with `delegates`.
3. **Unknown.** Otherwise, no members means `unknown`: `records_unavailable` when
   `!records_available`, else `no_records`.
4. **Since.** Otherwise, `since` is the maximum of members' `ended_at` (falling back to
   `started_at`) and closed child `closed_at`. If `now - since < settle_grace_seconds`,
   the state is `settling`.
5. **Stranded.** Otherwise the state is `stranded`, with the cause chosen by the
   precedence list above. `last_turn` is the newest agent-kind member.

**Propagation.**

- For each stranded bead, walk `parent_id` upward through candidate beads. At each
  ancestor:
  - append the leaf to `blocked_by`;
  - append the ancestor to the leaf's `blocks`, nearest first;
  - switch `owned`/`delegated`/`settling` ancestors to `blocked`.
- Never overwrite a `stranded` or `unknown` ancestor's state. Keep walking through them,
  so the root sees every leaf.
- Deduplicate and sort all lists.

**Binding.**

- Add a new binding domain `crates/sase_core_py/src/stranded_work/` (`mod.rs`,
  `tests.rs`) exposing `stranded_work_reduce`.
- Use the same calling convention as the `wait_epic_follow` binding: a Python mapping
  in, a serialized mapping out.
- Register it via `register_stranded_work` in `crates/sase_core_py/src/lib.rs`. A missed
  registration only fails as `AttributeError` in sase, so the round-trip test must call
  it by name.

**Tests.** Table-driven reducer tests, one per row of the state and cause tables, plus:

- every owner kind (running, asking, waiting, queued, finalizing, each pending-handoff
  kind, monitor running, monitor completed with follow-up pending, monitor follow-up
  launched with a running successor, gate pending, recovery, remote non-terminal);
- the settle boundary (exactly at grace);
- delegation to a launched child epic versus an `open` (unlaunched) child, which does
  not delegate;
- `child_landed`;
- a stranded epic whose land ended, which leaves its parent phase blocked;
- a named `sase_1n6_shape` fixture:
  - members `--1`, `--mon` (follow-up launched `--2`), `--2`, `--mon-0` (follow-up
    launched `--3`), and `--3` done with `keep`;
  - parent epic, delegated parent phase, grandparent epic, and root all `blocked`;
  - expected: exactly one `stranded` leaf with cause `kept`, and a `blocks` chain to the
    root;
- determinism (shuffled input gives identical output).

Run `sase tool run check` inside the sase-core checkout.

## Phase `collect`: Python collector, facade, vocabulary, and doctor

**1. Pin.** Move `sase-core-revision.txt` past phase `core`'s sase-core commit with
`just ratchet-core-revision` (see `docs/contribute/rust-backend.md`). Do not edit it by
hand unless that recipe cannot reach the commit. "Check pinned core bindings"
(`tools/check_sase_core_rs_bindings`) must pass.

**2. Facade.** Add `src/sase/core/stranded_work_facade.py`:

- frozen dataclasses mirroring the verdict and report wires;
- `reduce_stranded_work(facts) -> StrandedWorkReport`, using
  `require_rust_binding("stranded_work_reduce")`, following
  `src/sase/core/wait_dependency_resolution/_epic_follow.py`.

**3. Collector.** Add `src/sase/bead/stranded.py`. Split facts into
`src/sase/bead/stranded_facts.py` to stay well under the `toobig` caps. It exposes
`load_stranded_report(project, *, now=None) -> StrandedReport`, which holds the facade
report plus `titles: Mapping[str, str]` and `generated_at`.

- **Beads.**
  - Locate the store with `canonical_beads_dir_for_project`.
  - Candidates come from `bead_read_facade.list_issues(…, statuses=[IN_PROGRESS], …)`:
    phase beads, plus plan beads with tier `epic`.
  - Context beads are the child plan beads of candidate phases, in any status. Use
    `get_epic_children`, the hierarchical ID prefix, or a parent-filtered list query;
    prefer whichever the read model serves without replay.

  > [!decision] flag_tasks When this decision is on, the candidate query also includes
  > `task` beads.

- **Members.**
  - Records whose `agent_session` equals the bead's `assignee`, plus any record named
    `assignee`. The `<bead>--plan` root shares the session.
  - Query the artifact index; the precedent is
    `project_records(project, agent_session=…)` in `src/sase/monitor/store.py`.
  - Do **not** apply the visible-roster or dismissed filter.
- **Member facts.** Use the exact predicates `sase agent list` and ACE already use, so
  the "never contradict ACE" invariant holds by construction:
  - `record_is_live` and `active_status_for_record` in
    `src/sase/agent/_running_listing_common.py`;
  - the pending-handoff markers in `src/sase/agent/pending_handoff.py`, and the
    handoff-inflight marker;
  - `recovery_in_flight` in `src/sase/agent/auto_restart/forward.py`;
  - monitor and gate `agent_session_turn` state, `next_action`, `followup_outcome`, and
    `followup_agent`, with the semantics of `turn_followup_handoff_agent` in
    `src/sase/core/wait_dependency_resolution/_artifact_state.py`;
  - the finalizer phase;
  - remote (fleet) ownership.
  - `process_live` means the record is active under the same rule `sase agent list` uses
    to list it as a live row. It is not a raw pid probe: pre-run `WAITING`/`QUEUED`
    records may have no runner pid, and they must still count as owners and never as
    `lost`.
- **Declared action.**
  - Factor `_commit_bead_actions` out of `src/sase/main/final_handler.py` into a
    reusable helper beside `load_latest_finalizer_submission` in
    `src/sase/finalizers/declaration_manifest.py`, and use it from both places.
  - Read only the latest agent turn's `final_submission.json`.
- **Budget.**
  - One index query per candidate session.
  - At most one small JSON read per candidate.
  - No full artifact scans.
  - About 100 ms for 30 candidates.
  - Never call it on the UI thread.
  - Callers wrap failures: collector errors yield an empty report plus a logged warning,
    never an exception into ACE.

**4. Vocabulary.** Add `src/sase/bead/stranded_presentation.py`, with no Textual imports
and Rich style strings only. It holds:

- the glyph, the word, and the leaf and breadcrumb styles;
- `stranded_cause_sentence(verdict)`, with one short sentence per cause (the
  followup-failed sentence names the member and its outcome);
- `stranded_age_label(since, now)`;
- `blocked_sentence(verdict)`, e.g. `blocked by stranded sase-…2 (+1 more)`.

Every surface in later phases imports from here, so wording never drifts.

**5. Doctor.**

> [!decision] doctor_report When this decision is on, `src/sase/doctor/checks_beads.py`
> replaces the `in_progress beads whose promoted agent is gone` message with one
> `WARNING` that:
>
> - lists stranded beads with their last turn, cause sentence, and age;
> - gives both remedies: fork the last turn with `%id(bead=<id>) #fork:<turn>`, or run
>   `sase bead close <id> --note "…"`.
>
> The claimed-bead and dependency advisories stay as they are. If the decision is off,
> doctor is untouched.

**6. Docs.**

- Add a short "Stranded beads" reference section to
  `docs/reference/work-units/bead-lifecycles.md`: the definition, states, causes, and
  remedies. Read `docs_writing.md` with `/sase_memory_read` first.
- Update the doctor reference if the decision is on.

**7. Tests.**

- Collector tests over fixture artifact directories:
  - the `sase-1n6` monitor chain ending in `keep`;
  - a turn with no `final_submission.json` (`unclosed`);
  - declared `close` with the bead still open;
  - a dismissed session (still stranded);
  - a remote non-terminal member (owned);
  - each pending-handoff marker (owned);
  - a `--plan` root;
  - a fork launched with `%id(bead=…)` that moved the assignee (owned);
  - child-epic delegation;
  - the invariant: one fixture per active ACE status.
- Facade round-trip.
- Presentation sentences.
- The doctor message, if the decision is on.

## Phase `agents-rows`: Agents tab rows, clans, waiters, and info strip

Read `tui.md` and `tui_perf.md` with `/sase_memory_read` before starting.

**1. View cache.** Add `src/sase/ace/tui/models/agent_stranded.py`, following
`agent_wait_beads.py` and `agent_epic_follow_progress.py`.

- `cached_stranded_view(project)` is memory-only, for render paths.
- `warm_stranded_views(projects)` runs off-loop.
- TTL is 30 s, recomputing early when the project's bead-store fingerprint or the agents
  roster generation changes.
- The view indexes verdicts by bead ID, maps each verdict to its **owner row** (the
  session or anchor node for `verdict.session`, or a root linked through
  `phase_bead_id`/`bead_id`, never a member turn row), keeps per-epic leaf sets, and
  exposes a stable signature for diffing and render keys.

**2. Scheduling.**

- Add a project-level lane to `AgentBeadWarmupMixin`
  (`src/sase/ace/tui/actions/agents/_loading_bead_warmup.py`), covering projects present
  in the loaded roster.
- Keep its scheduled, running, and pending coalescing and its navigation deferral.
- When the signature changes, patch only the affected rows with `patch_row`: owner rows,
  waiters of affected beads, and clans of affected epics. Then refresh the info strip.
- No full rebuilds and no disk I/O in render paths.

**3. Rows.**

| Row                                                               | Rendering                                                                                                                                                                                                                                                                        |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Owner row, stranded, Done bucket                                  | Status word becomes `STRANDED` plus ` ⚐` (leaf style). This is presentation only, like `FINALIZING`                                                                                                                                                                              |
| Owner row, stranded, other bucket (`FAILED`, `STOPPED`, …)        | Keeps its louder word and appends ` ⚐` (leaf style)                                                                                                                                                                                                                              |
| Owner row, blocked (for example a delegated phase showing `DONE`) | Word unchanged, plus a breadcrumb ` ⚐`                                                                                                                                                                                                                                           |
| `WAITING` row                                                     | Stranded bead tokens use `⚐` (leaf) instead of `◐`, blocked ones use breadcrumb `⚐`, and multi-target counts gain `⚐N`. If any target (bead or agent session) resolves to a stranded or blocked verdict and no token showed it, append a breadcrumb `⚐` inside the parenthetical |
| Epic clan row (clans are named after the epic bead ID)            | `⚐N` after the count chip and any `?N`. N is the distinct leaves in the epic's subtree. Leaf style when a leaf is a direct phase or the epic itself, breadcrumb otherwise                                                                                                        |
| Wave pips                                                         | In `_wave_state` (`src/sase/ace/tui/models/_agent_clan_waves.py`), stranded or blocked anchor rows fold to `NEEDS_YOU` (`▲`)                                                                                                                                                     |
| Info strip (`src/sase/ace/tui/widgets/agent_info_panel.py`)       | ` · ⚐ N stranded` after the status strip. Leaves only, bold amber, zero-suppressed                                                                                                                                                                                               |

Implementation notes:

- The main files are `_agent_list_render_agent_status.py`,
  `_agent_list_render_agent.py`, and `src/sase/ace/tui/wait_status_presentation.py`.
- Add the row's stranded signature to `agent_render_key`
  (`_agent_list_render_cache.py`); otherwise rows render stale.
- Status buckets, chip counts, and dismissal behavior are unchanged; the overlay is
  render-time only.

**4. Help and docs.**

- Add `⚐` entries, leaf and breadcrumb, to "Agent Row Glyphs", "Waiting Badges", and
  "Clan Waves" in `src/sase/ace/tui/modals/help_modal/agents_reference_sections.py`.
- Add the glyph table rows to `docs/reference/tui/agent-row-glyphs.md`.
- Describe the `STRANDED` overlay in `docs/reference/tui/agent-statuses.md`, next to
  `FINALIZING`, linking to the bead-lifecycles section.

**5. Tests.**

- Unit tests:
  - row text for every table row;
  - the wave fold;
  - TTL and fingerprint invalidation;
  - the lane patching only the affected rows;
  - the info strip.
- PNG goldens `agents_stranded_epic_clan_120x40` and
  `agents_stranded_nested_chain_120x40`:
  - seed the view cache the way `test_ace_png_snapshots_agents_waiting.py` seeds
    `_WAIT_BEAD_STATUS_CACHE`;
  - use the `epic_clan_agents`/`wave_clan_agents` fixtures;
  - pin time.
- Run `just fix-tui-screenshots -- <selectors>` through `/sase_monitor` and inspect the
  report before finishing.

## Phase `agents-detail`: Agents detail callouts, toast, and query field

**1. Header callout.** Add a callout line to the compact AGENT TURN header
(`src/sase/ace/tui/widgets/prompt_panel/_identity_header_compact.py`), styled like the
blocked epic-follow line in `_agent_wait_section.py`:

- stranded: `⚐ STRANDED 31h · <last-turn suffix> <cause sentence>`;
- blocked: `⚐ blocked 31h by stranded <leaf> (+N more)` in breadcrumb style.

**2. Wait lanes.** In the `[beads]` lane of `_agent_wait_section.py`, stranded or
blocked bead entries render `⚐ stranded 31h` or `⚐ blocked by <leaf>` in place of the
plain status glyph.

**3. WAVES section.** The waits-on lines in `_agent_display_clan_waves.py` mark stranded
members with `⚐` (leaf) and blocked members with a breadcrumb `⚐`.

**4. Toast.** Follow `src/sase/ace/tui/actions/agents/_epic_follow_toasts.py`:

- Diff successive views' leaf sets.
- The first view after startup is the baseline and never toasts.
- Emit one coalesced warning per refresh, e.g.
  `⚐ sase-…2.1.2 is stranded: --3 ended with bead_action keep; nothing resumes it.`

**5. Query field.**

- Add a `stranded` boolean on owner rows in
  `src/sase/ace/tui/models/agent_live_query.py`.
- Add it to `KNOWN_FALLBACK_FIELDS` in `agent_live_query_pushdown.py`; it is never
  pushed down.
- Leave the existing `attention` field unchanged.
- Document it in `docs/reference/tui/query-system.md`.

**6. Docs.** Add the header callout and lane markers to
`docs/reference/tui/agents-main-deck.md`.

**7. Tests.**

- Unit tests for the callout text, lane entries, toast diffing (including no startup
  toast), and the query field.
- PNG goldens `agents_stranded_detail_header_120x40` and
  `agents_stranded_blocked_detail_120x40`.

## Phase `beads-pane`: Beads pane stranded badges

**1. Data.**

- Add a deferred second pass in `src/sase/ace/tui/widgets/artifacts/beads_pane.py`, like
  the existing external-issue pass. It reads the shared stranded view for the pane's
  project, warming it off-thread when cold.
- It never delays the bead snapshot.
- It re-renders on a signature change during the existing 10 s tick.

**2. Rendering** (`beads_rendering.py`, `beads_list.py`):

- Phase rows: ` ⚐ 31h` after the status, leaf style. Blocked phase rows get a breadcrumb
  ` ⚐`.
- Epic rows: `_state_glyph` becomes `⚐` when the subtree holds a leaf, outranking
  launched and ready. It is bold when the leaf is a direct phase or the epic itself.
- The Epics header legend gains `⚐ stranded`.
- The status bar (`build_beads_status`) gains a zero-suppressed `⚐ N stranded` token.

> [!decision] flag_tasks When this decision is on, task rows in the Tasks section get
> the same badge, and the status-bar count includes them.

**3. Detail.** Add a `Stranded` property row in `beads_detail_properties.py`:

- stranded: `<last turn> · <cause sentence> · <age>`;
- blocked: `Blocked by stranded <leaf>`.

**4. Help and docs.**

- Add the legend line to the beads section of
  `src/sase/ace/tui/modals/help_modal/patches_artifact_bindings.py`.
- Update `docs/reference/tui/artifacts-tab.md`.

**5. Tests.**

- Rendering cases in `tests/ace/tui/_artifacts_beads_rendering_rows.py`.
- PNG golden `artifacts_beads_stranded_120x40`; monkeypatch the snapshot loader as
  `test_ace_png_snapshots_artifacts_beads.py` does, and seed the stranded view.

## Phase `resolve`: one-keystroke resolution from any flagged row

**1. Enter target.**

- Extend `EnterTargetKind` with `"stranded"` (and a matching source) in
  `src/sase/ace/tui/actions/agents/_agent_enter_models.py`.
- Teach `_agent_enter_resolver.py` to offer the target on:
  - stranded or blocked owner rows;
  - `WAITING` rows whose targets resolve to stranded or blocked verdicts;
  - epic clan rows with leaves. These replace the "Select an agent inside this clan"
    empty result.
- A pending gate target still sorts first.
- `enter_action_label_for_targets` yields `resolve stranded`, so the footer shows
  `<enter> resolve stranded` under the conditional-footer rule.

**2. Modal.** Add `StrandedResolveModal` in `src/sase/ace/tui/modals/`, matching the
mockup:

- an amber border and `⚐` title;
- the bead ID and title, `in_progress · stranded <age>`, the last turn and its end time,
  the cause sentence, and a `Blocking` line built from `blocks` (count plus nearest and
  root);
- a selectable leaf list when a blocked row has several leaves;
- actions: `f` or Enter for Continue, `c` for Close bead, `esc` to cancel;
- every fact comes from the cached view and report, so mounting does no disk I/O.

**3. Continue.**

- Open the home prompt bar prefilled with `%id(bead=<leaf>) #fork:<last turn> `,
  followed by an editable stub: "Continue stranded phase `<leaf>`: `<last turn>`
  `<cause sentence>`. Re-read `sase bead read <leaf>` (notes included), finish the
  phase, and close it with `sase bead close`. If it must stay open, arm a monitor or
  gate before ending."
- Reuse the existing fork prompt-target flow (`_fork_actions.py`) for the VCS tag and
  `%tab` when the last-turn row is loaded; otherwise prefill directly.
- Verify that `#fork:` resolves a dismissed turn name. If it does not, show Continue
  disabled, with the reason.
- Launching with `%id(bead=…)` makes `claim_for_agent_launch` reassign the bead to the
  new agent, so the flag clears as soon as the fork starts.

**4. Close bead.**

- Load the issue off-thread, then push the existing
  `BeadCloseModal(issue, unclosed_descendants=…)`.
- Submit it through a helper factored out of `_artifacts_beads_mutations.py`
  (`_submit_bead_close`) that takes the issue instead of a pane row, so the Beads pane
  and the Agents tab share one durable mutation path.

**5. Beads pane.**

- `beads_launch_work` (`w`, in `_artifacts_beads_launch.py`) on a stranded or blocked
  phase or epic row opens the same modal. Healthy phases keep "Phases launch with their
  epic".
- Update its help label to `launch / resolve stranded`.
- No new keymap entries.

**6. Freshness and hints.**

- Add `force_stranded_refresh(project)` to `agent_stranded.py`. It bypasses the TTL and
  schedules an immediate warmup. Call it after either action completes so the flags
  clear promptly.
- Append the configured `act_on_agent` key hint (`⏎ resolve`) to the agents-detail
  header callout.

**7. Help and docs.** Update the agents and beads binding help, and add a "Resolving
stranded work" entry to `docs/reference/tui/agents-tab.md` and
`docs/reference/tui/artifacts-tab.md`.

**8. Tests.**

- Resolver unit tests: target placement and ordering.
- Modal tests: prefill text, disabled Continue, the close path through the shared
  helper, and the forced refresh.
- PNG golden `agents_stranded_resolve_modal_120x40`.

## Verification (every phase)

- In sase, run `sase tool run check` before finishing. In sase-core, run the same
  command from that checkout.
- Phases that change rendered output must run `just fix-tui-screenshots -- <selectors>`
  through `/sase_monitor`. Inspect the report and golden diffs; generation is not
  approval.
- `just check-full` is not requested.
- Keep new Python modules under the `toobig` caps and Rust files at or under 1,500
  lines.
- Introduce each symbol in the phase that first uses it. When a phase must ship a symbol
  that only a later phase consumes (for example, presentation helpers when the doctor
  decision is off), whitelist it with an `--epic-symbol` entry keyed to the consuming
  phase. Read `symvision.md` with `/sase_memory_read`.
- Phase workers record discovered follow-ups as `PROPOSED FOLLOW-UP:` notes on their
  phase bead.

## Non-goals and follow-ups (for the land agent to triage)

- **No automatic relaunch.** The flag informs and offers one-keystroke remedies.
  Auto-resuming stranded phases is a separate policy decision.
- **No change to agent prompts, `bead_action` semantics, or finalizers.** The existing
  submit-time note ("bead_action keep leaves it open with nothing resuming it") stays.
- **Not in this epic:** Telegram, mobile, or inbox notifications for stranded beads, and
  a `sase bead` CLI listing. All of them can reuse `load_stranded_report`.
- **The report's second recommendation is separate:** warning when a long monitor
  handoff starts with a dirty workspace and a `--next` that waits on master.
