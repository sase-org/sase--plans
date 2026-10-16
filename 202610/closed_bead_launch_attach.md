---
tier: epic
title: Attach agent launches to beads that were already closed
goal: 'An agent launched with `%id(..., bead=<id>)` against a bead that was already
  closed when the launch was submitted (the common `,x` / `sase agent restart` recovery
  case) runs attached to that closed bead instead of failing: the bead stays closed
  with its close metadata and ancestors untouched, the host appends one timestamped
  `CLOSED-BEAD LAUNCH:` audit note authored by the launching agent, and the agent
  is told its bead is already closed. A bead closed after the launch was submitted
  still refuses the launch, now with a clear `closed_while_queued` error.

  '
decisions:
  healer_done_only:
    ask: Should update-skew auto-restarts refuse to attach to beads closed as canceled
      or superseded?
    default: false
    why: every launch path behaves alike; the host notice already says not to redo
      canceled work
    answer: false
  beads_memory:
    ask: Update the generated sase_beads memory so it no longer promises every launched
      bead is in_progress?
    memory:
    - sase_beads.md
    corrects: A bead you were launched to work on is already `in_progress` before
      you read your prompt, because an epic launch preassigns every phase bead and
      the land bead.
    default: true
    answer: true
phases:
- id: core
  title: sase-core launch claim attaches to visibly closed beads
  depends_on: []
  size: medium
  description: 'core: teach sase-core''s run_claim_launch an optional submission time
    and run ID; attach to a bead closed at or before submission (status untouched,
    one CLOSED-BEAD LAUNCH: note by the launching agent, same mutation), refuse later
    closes as closed_while_queued, keep the legacy error when no submission time is
    given, and expose the new kwargs on the Python binding with tests.'
- id: runner
  title: Runner wiring, agent metadata, and the host notice
  depends_on:
  - core
  size: medium
  description: 'runner: move sase''s sase-core CI pin, pass the UTC submission time
    and run ID from the runner''s single launch-claim checkpoint through the facade,
    use a distinct SDD commit subject, record the attach in agent_meta, prepend a
    host notice telling the agent its bead is already closed, and invert/add tests.'
- id: guidance
  title: Prompt templates, docs, and memory
  depends_on:
  - runner
  size: small
  description: 'guidance: add the closed-bead sentence to the bd/work_phase_bead and
    bd/work_task prompts, document the attach and closed_while_queued rules in the
    bead claim lifecycle, %id, and ,x docs, correct the generated sase_beads memory,
    and record the out-of-scope follow-ups for the land agent.'
proposed_by: bbugyi200.athena.research.4n.linker.w0
decided_by: auto
create_time: 2026-10-16 13:03:44
status: done
bead_id: sase-1li
---

- **BEAD:** [sase-1li](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1li/README.md)

# Plan: Attach agent launches to beads that were already closed

## Background

Today an agent launched with `%id(<name>, bead=<id>)` fails when `<id>` is closed. The
only fatal check is sase-core's launch claim: `run_claim_launch` in
`crates/sase_core/src/bead/mutation/claims.rs` returns
`closed: cannot claim closed bead for agent launch: <id>`. It runs at the runner's
single promotion checkpoint (`_promote_bead_claim` in
`src/sase/axe/run_agent_runner_launch.py` → `claim_bead_for_agent_launch` in
`src/sase/axe/run_agent_runner_bead.py` → `BeadProject.claim_for_agent_launch` →
`bead_mutation_facade.claim_for_agent_launch` → the `bead_claim_for_agent_launch`
binding). That checkpoint runs after workspace preparation, so the failure also wastes a
workspace. Every other stage already tolerates a closed bead:

- the `%id` parser only checks that `bead=` is non-empty;
- the wait-time claim (`run_claim_wait`) declines quietly on a closed bead;
- force-reuse retention (`issue_retains_force_reuse_owner`) only matches `in_progress`,
  so a closed bead falls through to the launch claim;
- the finalizer's `bead_action: close` treats a closed bead as `CloseIdempotent`, and
  `keep` leaves the status alone.

The research report `closed_bead_agent_relaunch.md` in the research sidecar
(`202610/closed_bead_agent_relaunch/`) looked at 11 real failures. 8 were exactly this
recovery case, and in 5 of the 6 that could be checked, the original run's work had
already landed on `master` before the relaunch. 3 of the 11 were deliberate closes made
while the launch was still queued (a superseded phase, a phase another agent finished, a
sub-epic closed by hand). Refusing those was correct. The user accepted every
requirement in that report. This plan implements them.

## Requirements (accepted from the research)

- **R1 Attach, don't reopen.** A launch against a closed bead proceeds attached to it.
  Status, assignee, resolution, close reason, `closed_at`, close history and every
  ancestor stay byte-identical. Nothing is reopened. `sase bead open` stays the
  deliberate path for a premature close.
- **R2 Visible-close rule.** Attach only when the close happened at or before the
  launch's submission time. A close after submission (made while the launch was queued)
  still refuses the launch, with a new `closed_while_queued` error that names the close
  and the remedy.
- **R3 Host-written audit note.** sase-core writes it in the same mutation and SDD
  commit as the launch checkpoint. Its author is the launching agent and it uses a fixed
  `CLOSED-BEAD LAUNCH:` prefix. Every launch adds one note; there is no dedup.
- **R4 Tell the agent.** The relaunched agent gets a host notice that its bead is
  already closed and how to proceed.
- **R5 Every launch path.** This covers `,x`, `sase agent restart`, retry handoff,
  update-skew auto-restart and typed `%id`. All of them reach the same claim call, so no
  TUI-only special case is needed.
- **R6 Kept out.** No reopen-on-launch, no new event kind or `BeadReopenCauseWire`
  variant, no feature flag, no confirmation modal, no `reopen=true` keyword. `bead=` is
  never dropped on relaunch.

## Design overview

`run_claim_launch` gains an optional launch context (`submitted_at`, `run_id`). Under
the existing mutation lock, the current status decides the result:

| Current status                                            | Result                                                               |
| --------------------------------------------------------- | -------------------------------------------------------------------- |
| `in_progress`, same agent                                 | No-op success (unchanged)                                            |
| `open` / `claimed` / `ready` / `snoozed`                  | Promote to `in_progress` and assign (unchanged)                      |
| `closed`, `submitted_at` absent                           | Today's `closed` error, byte-identical (older callers see no change) |
| `closed`, `closed_at` > `submitted_at`                    | **New** error, kind `closed_while_queued`                            |
| `closed`, `closed_at` ≤ `submitted_at`, or no `closed_at` | **New** attach: one `note_appended` event, status untouched          |

The note reuses the existing `note_appended` event, so the shared beads sidecar's wire
format does not change (older cores hard-fail on unknown event kinds). The runner passes
the launch's own submission timestamp, so a deliberate re-submission after a
`closed_while_queued` failure is, by construction, later than the close and attaches.

---

## Phase `core`: sase-core launch claim attaches to visibly closed beads

Repo: the linked `sase-core` repo. Open it with `sase repo open sase-core -r "<why>"`,
work only in the printed path, and read its `AGENTS.md` first. Follow its conventions:
no `macro_rules!`, import by module path, no new root `pub use` in `lib.rs`, no new
`core_*` alias in `prelude.rs`, Conventional Commit subjects, and never edit versions or
changelogs. This change is additive and non-breaking.

### Wire and signature

1. In `crates/sase_core/src/bead/mutation/mutation_wire.rs`, add:

   ```rust
   /// Launch provenance for `claim_for_agent_launch`'s closed-bead attach.
   #[derive(Clone, Debug, Default, PartialEq, Eq, Serialize, Deserialize)]
   pub struct BeadLaunchClaimContextWire {
       /// RFC 3339 time the launch was submitted; absent keeps the legacy closed error.
       #[serde(default, skip_serializing_if = "Option::is_none")]
       pub submitted_at: Option<String>,
       /// Run identifier quoted in the audit note (the run's artifacts timestamp).
       #[serde(default, skip_serializing_if = "Option::is_none")]
       pub run_id: Option<String>,
   }
   ```

   Export it from the mutation facade and `bead/mod.rs` the same way the neighbouring
   mutation wires are exported.

2. Change `claim_for_agent_launch(beads_dir, issue_id, agent_name, now)` to also take
   `launch: &BeadLaunchClaimContextWire`, and thread it into `run_claim_launch`. Before
   `run_mutation`, validate that a present `submitted_at` parses as RFC 3339
   (`BeadError::validation` otherwise, in the style of the existing blank-agent-name
   check). Update every existing Rust caller to pass
   `&BeadLaunchClaimContextWire::default()`:
   `mutation/tests/{claims,claims_deps,links,proof_bounded,dual_mode,replay_golden_cases}.rs`
   and `crates/sase_core/tests/bead_read_model_mutation_proof.rs`. Existing goldens (for
   example `goldens/claim_launch_event.json`) must not change.

### The closed branch in `run_claim_launch`

Replace the unconditional `closed` error with the following:

- **`launch.submitted_at` is `None`:** return today's
  `BeadError { kind: "closed", message: "cannot claim closed bead for agent launch: <id>" }`
  unchanged.
- **`closed_at` is present and later than `submitted_at`:** return
  `BeadError { kind: "closed_while_queued", .. }` without writing anything. The message
  names the bead, its `closed_at`, its resolution (plus the close reason when present),
  the submission time and the remedy, for example:
  `bead sase-x.3 was closed at 2026-10-16T14:02:11Z (done) after this launch was submitted at 2026-10-16T13:58:40Z; relaunch to run it against the closed bead deliberately`.
  Compare by parsing both values with `chrono::DateTime::parse_from_rfc3339`, as
  `task_plus_one_reopen_decision` in `bead/events/reduction.rs` does. Mirror that
  function's validation error for an unparseable `closed_at`. Equal timestamps attach.
- **Otherwise (closed at or before submission, or no `closed_at` on legacy data):
  attach.**
  - Compose the note with a new
    `pub fn closed_bead_launch_note(issue: &IssueWire, agent_name: &str, launch: &BeadLaunchClaimContextWire) -> String`.
    It uses a `pub const CLOSED_BEAD_LAUNCH_NOTE_PREFIX: &str = "CLOSED-BEAD LAUNCH:"`,
    and its exact format is:

    ```text
    CLOSED-BEAD LAUNCH: agent <agent> launched against this closed bead; status left closed.
    Launch submitted <submitted_at>; run <run_id>.
    Prior close: <closed_at> · <resolution>.
    ```

    Drop `; run <run_id>` when `run_id` is absent. Print `unknown` for a missing
    `closed_at` or resolution. The closer is not included, because `IssueWire` does not
    carry it cheaply. The note's own event metadata supplies the attach timestamp and
    the author.

  - Mirror `run_append_note` in `notes_update.rs` inside this one mutation. Do not call
    the public `append_issue_note` while holding the lock. Stage
    `BeadEventOperationWire::NoteAppended` /
    `BeadEventPayloadWire::NoteAppended { entry, attachments: vec![] }` at `now`, with
    **actor = `agent_name`** (not `issue.created_by`, which the promotion branch uses).
    Push `BeadNoteWire::from_event(...)` onto the row, set `updated_at = now`,
    `validate()`, `stage_issue`, then `commit`. Return `MutationStep::NeedsReplay` on
    the same declines that the existing branches handle. The note reducer
    (`reduction.rs`, `NoteAppended` arm) pushes the note and sets `updated_at`, so the
    stored row and the row rebuilt from events stay identical.
  - Do **not** touch status, assignee, resolution, close reason, `closed_at` or close
    history. Do not call `archive_close_metadata`, reopen ancestors, or add a reopen
    cause or event kind.
  - Outcome: `outcome("claim_for_agent_launch", true, vec![id])` with
    `noted_ids = vec![id]`,
    `message = "attached to closed bead <id>; status left closed"` and
    `issue = Some(<the committed closed row>)`. Callers detect the attach because a
    successful claim returns a `closed` issue; no new outcome field is needed.

### Python binding

In `crates/sase_core_py/src/beads/mod.rs`, extend `py_bead_claim_for_agent_launch` to
`signature = (beads_dir, bead_id, agent_name, now=None, submitted_at=None, run_id=None)`.
Build the context and pass it through. Python callers pass these positionally after
`now`, so keep that order. Add a round-trip test in
`crates/sase_core_py/src/beads/tests.rs` covering the attach (closed issue returned,
note present) and `closed_while_queued` (a Python `ValueError` whose message starts with
`closed_while_queued:`).

### Tests (`crates/sase_core/src/bead/mutation/tests/claims.rs` unless noted)

- Split `claim_for_agent_launch_rejects_missing_closed_and_blank_requests`. Missing ID
  and blank agent keep their errors. Closed with a default context keeps the legacy
  `closed` error.
- **Attach:** a closed phase under a closed epic, launched with `submitted_at` after its
  `closed_at`. Status, assignee, resolution, close reason, `closed_at` and close history
  are unchanged, and the parent epic row is byte-identical. Exactly one new note starts
  with `CLOSED-BEAD LAUNCH:`; its author is the agent and its timestamp is `now`. The
  outcome has `changed == true` and `noted_ids == [id]`.
- **Two launches** of the same closed bead produce two notes. There is no dedup.
- **`closed_while_queued`** when `closed_at > submitted_at`. Nothing is written (event
  count and projection unchanged).
- **Equal timestamps** attach. An invalid `submitted_at` gives a validation error.
- **Note format:** a unit test for `closed_bead_launch_note`, with and without `run_id`.
- **Parity:** after an attach, the mutation projection equals the rebuild from events on
  both backings. Extend the existing dual-mode / read-model mutation-proof coverage
  (`dual_mode.rs`, `tests/bead_read_model_mutation_proof.rs`) with the attach case.

### Verify

Run `sase tool run check` in the sase-core checkout (it wraps `just check`; give it ≥10
min). A suggested commit subject is
`feat(bead): attach launch claims to beads closed before submission`.

---

## Phase `runner`: runner wiring, agent metadata, and the host notice

Repo: sase. Before finishing, read the `lint_and_test` memory with `/sase_memory_read`.

### Move the CI pin

Run `just ratchet-core-revision` so `sase-core-revision.txt` names a sase-core commit
that contains the `core` phase. Verify this with
`git merge-base --is-ancestor <core-phase-commit> <pinned-sha>` in the sase-core
checkout. Leave the `pyproject.toml` `sase-core-rs` window alone: the release lane owns
it (see `docs/contribute/rust-backend.md`).

### Thread the launch context

1. `src/sase/core/bead_mutation_facade.py::claim_for_agent_launch(beads_dir, bead_id, agent_name, *, now=None, submitted_at=None, run_id=None)`.
   Pass both values positionally after `now` through `_call_issue_operation`. Errors
   keep flowing through its existing `ValueError` mapping, so `closed_while_queued: ...`
   reaches the caller verbatim.
2. `src/sase/bead/_project_mutations_claims.py`: give `claim_for_agent_launch` and
   `claim_for_agent_launch_outcome` keyword-only `submitted_at: str | None = None` and
   `run_id: str | None = None`, and forward them. Update the docstring ("Atomically
   claim one non-closed bead…") to describe the attach.
3. `src/sase/core/time.py`: add
   `launch_timestamp_to_utc_iso(timestamp: str) -> str | None`. It parses the runner's
   `YYmmdd_HHMMSS` launch timestamp as wall time in the configured timezone
   (`get_timezone()`, the zone `generate_timestamp()` writes in) and returns
   `YYYY-MM-DDTHH:MM:SSZ` in UTC. It returns `None` when parsing fails; the claim then
   sends no submission time and keeps the legacy error.
4. `src/sase/axe/run_agent_runner_bead.py::claim_bead_for_agent_launch(...)`: add
   keyword-only `submitted_at: str | None = None`, `run_id: str | None = None` and
   `auto_restart: bool = False`, and forward the first two to
   `project.claim_for_agent_launch`.
   - Keep the store lock, force-reuse retention, the "already in_progress and assigned
     to another agent" refusal, the missing-issue fresh-clone retry, the required local
     SDD commit and the best-effort publish and convergence hint exactly as they are.
   - When the claimed issue comes back `Status.CLOSED`, use the commit subject
     `chore(beads): note closed-bead launch of {bead_id} for {agent_name}` instead of
     the claim subject.
   - The function still returns the `Issue`. Callers detect the attach by its `closed`
     status.

> [!decision] healer_done_only When `auto_restart` is true and the current issue (read
> under the lock as today) is closed with a resolution other than `done`, raise
> `RuntimeError("auto-restart will not relaunch against bead <id> closed as <resolution>; relaunch it by hand to attach deliberately")`
> before calling the claim. Add a test for each resolution. When this decision is off,
> do not add the `auto_restart` parameter; every launch path attaches alike.

### The promotion checkpoint (`src/sase/axe/run_agent_runner_launch.py::_promote_bead_claim`)

- Compute `submitted_at = launch_timestamp_to_utc_iso(state.timestamp)` and
  `run_id = state.artifacts_timestamp`, and pass both on both branches (with and without
  `force_reuse_bead_association`). `state.timestamp` is the launch's argv timestamp. It
  is allocated before any `%wait` and replayed verbatim across refresh re-execs, so it
  is the submission time the visible-close rule needs. If `healer_done_only` is on, also
  pass `auto_restart="auto_restart" in bootstrap.agent_meta`.
- Capture the returned issue. When it is closed:
  - set `bootstrap.agent_meta["bead_launch_disposition"] = "closed_attached"`,
    `["bead_launch_prior_closed_at"] = issue.closed_at` and
    `["bead_launch_prior_resolution"] = <resolution value or None>` before the existing
    `write_agent_meta`;
  - prepend the host notice to `state.prompt`. Make it idempotent the way
    `_maybe_prepend_continuation` in `run_agent_exec_retry.py` is: skip if the prompt
    already starts with it.
- Keep `bootstrap.agent_meta["bead_claim_promoted"] = True`. Its consumers are the
  shutdown release in `run_agent_runner_lifecycle.py`, the chop claim reconciler
  (`sase_chop_bead_claim_checks.py`) and the doctor (`checks_beads.py`). They only
  reconcile `claimed`/`in_progress` claims, which is a no-op for a closed bead, so the
  flag keeps meaning "past the launch checkpoint".

### Host notice

Add `closed_bead_launch_notice(issue: Issue) -> str` beside
`claim_bead_for_agent_launch`. Its text:

> Host notice: bead `<id>` was already closed (`<resolution>`, `<closed_at>`) when this
> run started. This run is attached to it without reopening it, and a
> `CLOSED-BEAD LAUNCH:` note on the bead records the relaunch. First check whether the
> earlier work already landed (`git log --grep <id>` and the bead's notes via
> `sase bead read <id>`). If it did, verify it and finish without redoing it; closing
> the bead again is a safe no-op. If the work is genuinely unfinished, run
> `sase bead open <id>` first and close it yourself with
> `sase bead close <id> --note "<what you verified>"` when done (the finalizer will not
> close a reopened `open` bead). If the resolution is `canceled` or `superseded`, do not
> redo the work unless this prompt explicitly asks for it.

### Tests

- `tests/test_bead/test_epic_jit_claim_integration.py::test_closed_rendered_bead_fails_before_model_callback`:
  rework it into three cases that keep the phase/epic parametrization.
  - **Closed before submission:** attaches. The bead stays closed with the same
    resolution, it gains one `CLOSED-BEAD LAUNCH:` note and the model callback runs.
  - **Closed after submission:** `closed_while_queued` is raised before the model
    callback.
  - **No submission time:** the legacy failure.
- `tests/test_core_facade/test_bead_mutation.py`: keep the legacy
  `closed: cannot claim closed bead` assertion for calls without `submitted_at`. Add
  facade cases for the attach and for `closed_while_queued`.
- `tests/test_run_agent_runner_bead.py`:
  - the attach uses the `note closed-bead launch` commit subject;
  - force-reuse with a closed bead whose assignee is the prior owner attaches;
  - an in_progress bead held by another agent is still refused.
- A `_promote_bead_claim` test. On attach, the agent_meta disposition keys are written,
  `bead_claim_promoted` is `True`, the notice is prepended exactly once and
  `submitted_at` / `run_id` reach the claim. On an ordinary open bead, no notice is
  added and no disposition keys are written.
- A `launch_timestamp_to_utc_iso` unit test: conversion under a non-UTC configured
  timezone, and `None` on garbage.

Run `just check` inside `sase tool run` as the `lint_and_test` memory directs.

---

## Phase `guidance`: prompt templates, docs, and memory

Repo: sase. Before editing anything under `docs/`, read the `docs_writing` memory with
`/sase_memory_read`. Before finishing, read `lint_and_test`. Copy the exact notice and
note text from the `runner` and `core` phases' landed code; do not paraphrase them.

### Prompt templates (`src/sase/default_config.yml`)

- `bd/work_phase_bead` currently says the bead "was set to status=in_progress before you
  started reading this". Add one sentence after it: if the host notice says the bead was
  already closed when the run started, the run is attached to the closed bead. Check
  whether the earlier work landed before redoing anything, and run
  `sase bead open {{ bead_id }}` only if the work is genuinely unfinished. The
  template's existing `sase bead close` step is then a safe no-op or the real close.
- `bd/work_task`: add the same sentence, worded for a task bead.

### Docs

- `docs/reference/work-units/bead-lifecycles.md`, **Bead Claim Lifecycle**. Extend the
  **Promote** bullet, or add a sibling **Attach** bullet. A launch whose bead was
  already closed when it was submitted attaches without reopening. The bead keeps its
  status, resolution and ancestors, and gains one `CLOSED-BEAD LAUNCH:` note authored by
  the launching agent. A close made after submission fails the launch with
  `closed_while_queued`; re-submitting the launch attaches deliberately. Name
  `sase bead open` as the path for a premature close.
- `docs/reference/tui/agents-tab-modes.md`, at the `,x` / `sase agent restart`
  paragraph: one or two sentences saying that a relaunch whose bead is already closed
  now runs attached to it with an audit note, rather than failing.
- The `%id` `bead=` documentation (`docs/reference/macros/directives.md` row, and the
  `%id` keyword text in `docs/reference/macros/directive-syntax.md`): mention that
  `bead=` may name a closed bead, with a link to the lifecycle section.
- Run the docs checks the `docs_writing` memory prescribes.

### Memory

> [!decision] beads_memory Edit
> `src/sase/main/init_memory/templates/memory-sase-beads.template.md`.
> `sase/memory/sase_beads.md` is generated from it and refuses direct edits. In the
> **Statuses** paragraph, change the claim that a launched bead is always `in_progress`
> to say it is _normally_ `in_progress`. Add one sentence: a launch against a bead that
> was already closed when the launch was submitted attaches without reopening it, the
> host adds a `CLOSED-BEAD LAUNCH:` note, and the prompt starts with a host notice. Then
> run `sase memory init` to regenerate `sase/memory/sase_beads.md`, `AGENTS.md` and the
> shims. Update any init-memory snapshot test that pins the template text.

### Follow-ups for the land agent

Phase workers do not create beads. Record each of these as a
`sase bead note <this-phase> 'PROPOSED FOLLOW-UP: …'` so the land agent triages them
with `/sase_new_task`:

1. **Fix premature closes at the source.** `bd/work_phase_bead` and `bd/work_task` close
   the bead mid-turn, before the host commits. Moving the actual close behind the
   finalizer's post-commit `bead_action: close` (`close_assigned_bead_after_commit` in
   `src/sase/workflows/commit/bead_hooks.py`) would leave failed commits `in_progress`,
   which `,x` already handles. Medium. This is the root cause of most closed-bead
   relaunches (case sase-xe.6 in the research).
2. A non-failure "stood down" outcome for `closed_while_queued`, instead of a red failed
   agent.
3. A `,x` banner and `sase agent restart` preview hint
   (`src/sase/agent/_restart_preview.py`) saying the bead is closed and the relaunch
   will attach.

## Out of scope

- Reopen-on-launch, new event kinds or reopen causes, feature flags, confirmation modals
  and a `reopen=` keyword.
- Changing the promotion event's existing `issue.created_by` actor.
- Epic preclaim (`preclaim_epic_work_plan`) and `sase bead work`, which keep skipping
  and rejecting closed beads.
- Two live agents on one closed bead (only a copied prompt can cause it, because `,x`
  and restart kill the old run), and artifact-retention protection for attached runs.
