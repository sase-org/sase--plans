---
tier: epic
title: Stop in-flight +1s from reopening a task the instant its worker closes it
goal: 'A task bead that an assigned agent is actively working, or has just finished
  working, is never pushed back into the triage queue by corroboration that was already
  in flight before the close. Corroboration is still recorded, still visible to the
  owner, and a genuinely fresh reproduction still reopens the bead.

  '
phases:
- id: core
  title: Observation-window freshness rule in the bead core
  depends_on: []
  size: medium
  description: 'core: carry each +1 reporter''s observation-window start on the evidence
    wire, and reopen a closed task only when that window starts after the close, in
    both the mutation path and the event reducer.'
- id: cli
  title: Supplying and overriding the observation window from Python
  depends_on:
  - core
  size: small
  description: 'cli: resolve the reporter''s observation-window start from its own
    agent metadata, thread it through the mutation facade, add the explicit post-close
    override flag, and report a withheld reopen accurately.'
- id: surfaces
  title: Making withheld corroboration visible
  depends_on:
  - cli
  size: small
  description: 'surfaces: render post-close corroboration on the closed bead across
    the CLI, ACE, generated pages, and gate previews, and teach the task-filing skill
    when the override applies.'
- id: verify
  title: End-to-end race regression and store audit
  depends_on:
  - surfaces
  size: small
  description: 'verify: reproduce the original race end to end as a regression exercise,
    audit the live store for beads reopened by this race, and reconcile the documented
    contract.'
proposed_by: bbugyi200.athena.x9
create_time: 2026-08-10 10:49:19
status: wip
bead_id: sase-ix
---

- **PROMPT:** [prompts/202608/plus_one_post_close_reopen_race.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/plus_one_post_close_reopen_race.md)
- **BEAD:** [sase-ix](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ix/README.md)

# Plan: Stop in-flight +1s from reopening a task the instant its worker closes it

## Background: what was suspected, and what the evidence shows

The reported suspicion was that `sase bead +1` marks `in_progress` task beads as `open`
again. Taken literally, that is **not** what happens, and three independent checks
agree:

1. **Code.** `add_task_plus_one` in the Rust core promotes only
   `StatusWire::Open | StatusWire::Closed` to `Ready`; `in_progress` and `claimed` fall
   through untouched. The event reducer's `TaskPlusOneRecorded` arm applies the
   identical condition, so replay cannot diverge from the mutation.
2. **Live reproduction.** Creating a task, setting it `in_progress` with an assignee,
   and recording a +1 from a second reporter returns `changed=True` with the status and
   assignee unchanged at `in_progress`.
3. **Whole-store replay.** Replaying every event stream in the live bead store and
   diffing the result against `issues.jsonl` produced **0 status differences across
   3,145 beads**. Tabulating every status transition in that history shows
   `task_plus_one_recorded` has only ever produced `closed -> ready` (33 times) and
   `open -> ready` (once). It has never produced a transition out of `in_progress`, and
   the only two `in_progress -> open` transitions in the entire store were manual
   `sase bead update` calls by the owner on phase beads.

So the mechanism is not a status bug. **The symptom is real, though, and its cause sits
one step away.**

## Root cause: the `in_progress` guarantee expires exactly when it matters

The documented contract is that a +1 on an `open` or `closed` task promotes it to
`ready`, "while `claimed`, `ready`, and `in_progress` tasks retain their status." That
check is evaluated **only against the bead's status at the instant the +1 is written**.

Corroboration in SASE is produced asynchronously and in bulk: many agents hit the same
defect on trees that predate its fix, then file `/sase_new_task` corroboration minutes
or hours later. While the assigned worker holds the bead at `in_progress`, every one of
those +1s is correctly absorbed as evidence with no status change. The moment that
worker closes the bead, the remaining in-flight wave stops being absorbed: the very next
+1 sees `closed`, promotes the bead to `ready`, archives a `reopened_via: plus_one`
close record, and raises a fresh `TaskTriage` gate — for work that was just completed
and verified.

The store has no way to tell those two cases apart, because **a +1 records only when it
was written, never what the reporter observed**. `TaskPlusOneEvidenceWire` carries
`timestamp`, `reporter`, `note`, and `refs`; the write timestamp is by definition after
the close, so it can never distinguish stale corroboration from a fresh reproduction.

### Evidence from the live store

Of the 33 +1-driven reopens on record, **14 came off a close that arrived straight out
of `in_progress`** — a worker finishing its task. Four of those happened on a single
day, within minutes of the close:

| bead      | closed from   | closed at              | reopened by +1 | delta    |
| --------- | ------------- | ---------------------- | -------------- | -------- |
| `sase-ct` | `in_progress` | `2026-08-10T14:14:26Z` | `@wz`          | 0.7 min  |
| `sase-iq` | `in_progress` | `2026-08-10T14:25:32Z` | `@x2`          | 1.6 min  |
| `sase-is` | `in_progress` | `2026-08-10T14:21:49Z` | `@x2`          | 6.2 min  |
| `sase-ii` | `in_progress` | `2026-08-10T13:41:27Z` | `@sase-il.5`   | 48.5 min |

`sase-iq` is the sharpest case. It was preclaimed at `14:03:23`, and six separate +1s
landed while it was `in_progress` (`14:05:35`, `14:12:50`, `14:15:30`, `14:17:05`,
`14:21:45`, `14:22:04`), all correctly absorbed. Its worker closed it at `14:25:32`
after landing the fix. At `14:27:11` — 99 seconds later — a seventh reporter's +1
reopened it, and the bead now sits at `ready` with `assignee: sase-iq` still pointing at
the agent that finished it.

That trailing assignee is a second, smaller defect on the same path: `add_task_plus_one`
reopens without clearing `assignee`, so a reopened task advertises an owner that no
longer holds it. `launch_task_bead_work`'s live-assignee guard only fires for
`IN_PROGRESS`, so a reopened bead bypasses it entirely.

Note also that the same wave produces churn, not just one bad transition: six absorbed
+1s plus one reopening +1 is one bead crossing the triage boundary once for reasons that
have nothing to do with whether the defect still exists.

## The fix: give each +1 an observation window, and honor the close boundary

A +1 asserts "I saw this defect." The store must know **whether the reporter could have
seen the fix**. SASE already records exactly the right fact and never plumbs it into the
bead store: every agent's `agent_meta.json` carries `run_started_at`, the instant its
run began. An agent that started **after** the close cannot be carrying pre-close
observations; an agent that was already running when the close landed may well be.

The rule becomes:

- A +1 on an `open` (draft) task promotes it to `ready`. Unchanged — a draft has never
  been closed.
- A +1 on a `closed` task promotes it to `ready` **only when the reporter's
  observation-window start is later than that close**. Otherwise the evidence is
  recorded in full, the bead stays closed, and the withholding is stated in the command
  output and in an attributed note on the bead.
- `claimed`, `ready`, `in_progress`, and `snoozed` are untouched, exactly as today. The
  snooze +1-target wake path is untouched.
- When a +1 does reopen a closed task, `assignee` is cleared.

This is deliberately conservative in the direction the owner asked for: when the store
cannot prove the corroboration is fresh, it does not undo a close. A reporter that
genuinely re-reproduced the defect on a tree containing the fix says so explicitly and
still reopens the bead, and the owner can always reopen by hand with `sase bead open`.

### Why not the alternatives

- **A grace period after each close.** Any constant is arbitrary, and the observed
  deltas span 0.7 minutes to 48.5 minutes — there is no window that separates the
  classes.
- **Comparing repository revisions.** More precise in principle, but it drags VCS
  knowledge into a VCS-agnostic bead core and does not generalize across providers.
- **Never auto-reopen; raise an owner decision gate instead.** Clean, and worth
  revisiting if post-close corroboration turns out to need adjudication often, but it
  discards the 15 reopens on record that arrived a day or more after a close and were
  unambiguously valuable. Keep the automatic reopen; fix only the race.

## Phases

### Observation-window freshness rule in the bead core

All of this phase is in the sibling Rust core repo (`crates/sase_core`), reached with
the `/sase_repo` skill. It is core backend behavior: any frontend recording a +1 must
get the same answer.

- Add an optional `observed_since: Option<String>` to `TaskPlusOneEvidenceWire` in
  `bead/wire.rs`, with `#[serde(default, skip_serializing_if = "Option::is_none")]` so
  every existing evidence record in `issues.jsonl` and in the event streams still
  deserializes. Validate it as an RFC 3339 instant when present.
- Give `add_task_plus_one` in `bead/mutation.rs` an `observed_since: Option<String>`
  parameter, stored on the evidence it appends. Replace the bare
  `matches!(issue.status, Open | Closed)` promotion test with a helper — mirroring the
  existing `plus_one_wake_note` shape, so there is one place that answers "does this +1
  reopen" — that returns:
  - `true` for `Open`;
  - for `Closed`, `true` when `observed_since` is `None` (no provenance: preserve
    today's behavior) or when `observed_since` sorts strictly after the bead's live
    `closed_at`;
  - `false` otherwise.
- When the promotion is withheld, keep every other effect of the +1 (evidence appended,
  refs merged, `updated_at` bumped, `task_plus_one_recorded` event appended) and report
  it on the mutation outcome as a `reopen_withheld` flag alongside the `closed_at` it
  was measured against, so callers can explain themselves without recomputing the rule.
- When the promotion does happen from `Closed`, clear `issue.assignee` in addition to
  the existing `archive_close_metadata` call.
- Apply both changes identically in the `TaskPlusOneRecorded` arm of `apply_event` in
  `bead/events.rs`. Because the decision input now lives on the evidence itself, replay
  stays faithful; assert that in the event-parity suite (`tests/bead_event_parity.rs`).
- Expose the new parameter through the `bead_plus_one` binding in `crates/sase_core_py`,
  keeping it optional and last so the current call shape stays valid, and update the
  binding's documented signature.
- Tests: a closed task whose reporter's window opens before the close records evidence
  and stays closed; the same reporter with a window after the close reopens it and lands
  `assignee` empty; `observed_since: None` reopens (legacy behavior); an `open` draft
  promotes regardless; `in_progress`, `claimed`, `ready`, and snoozed-with-target beads
  all behave exactly as before; a stream replayed through the reducer matches the
  mutation result byte for byte.
- Cut a `sase-core` version bump so the sase repo can pin it.

### Supplying and overriding the observation window from Python

- `sase.core.bead_mutation_facade.plus_one` and
  `sase.bead._project_mutations.BeadProject.plus_one` gain an `observed_since` keyword
  and forward it to the binding.
- Add a small resolver — near `sase.agent.identity`, which already owns identity
  discovery — that returns the caller's observation-window start: `run_started_at` from
  the agent's own `agent_meta.json` under `SASE_ARTIFACTS_DIR` when the caller is a SASE
  agent, and the current time otherwise. A human running `sase bead +1` therefore always
  reopens, which is today's behavior and the right one: a human filing corroboration is
  asserting it now. Missing or malformed metadata falls back to the current time rather
  than failing the command, and the fallback is worth a debug-level log.
- `handle_bead_plus_one` in `sase/bead/cli_crud.py` resolves the window, passes it, and
  branches its output on the outcome's `reopen_withheld` flag: on a withheld reopen,
  print that the evidence was recorded, name the close it was measured against, and
  point at both `--verified-after-close` and `sase bead open`.
- Add `--verified-after-close` to the `+1` parser in
  `sase/main/parser_bead_lifecycle.py`. It asserts "I reproduced this on a tree that
  already contains the close" and sets the observation window to the current time.
  Reject it with a clear error when the bead is not closed, so it cannot become a reflex
  flag.
- When a reopen is withheld, append one attributed note to the bead recording the
  reporter, the close it postdates, and the fact that the bead was left closed. This is
  the durable trace the owner reads later; the snooze paths already set this precedent.
- Keep the `+1` fast path in `sase/main/bead_fast_path.py` working with the new flag.
- Pin the new `sase-core-rs` version in `pyproject.toml`.
- Tests: `tests/test_bead/test_plus_one_contract.py` for the domain and persistence
  round trip including `observed_since`; `tests/test_bead/test_cli_plus_one.py` for the
  withheld-reopen output, the override flag, the non-closed rejection, and the human
  fallback.

### Making withheld corroboration visible

Withholding a reopen must not make corroboration invisible; the owner still needs to see
that a closed bead is being re-reported.

- Add helpers to `sase/bead/plus_one_presentation.py` for "corroboration recorded since
  this bead's last close": the count, and a compact badge to sit beside the existing
  `+N` and `↺N` badges.
- Render it on the four surfaces `sase/bead/reopen_presentation.py` already names as the
  consumers of close-history presentation: the CLI (`sase/bead/cli_detail.py` detail
  view and the compact list row), the `TaskTriage` gate preview
  (`sase/bead/_task_gate_preview.py`), ACE (`sase/ace/tui/widgets/artifacts/`), and the
  generated bead pages (`sase/bead_pages/`). Mark the individual evidence entries that
  postdate the close in the detail view, next to the existing `evidence_reopened_bead`
  marker.
- Include `observed_since` in the JSON detail output (`sase/bead/cli_detail_json.py`)
  and in the SQLite mirror codec so `sase bead search`/`list` consumers see the same
  field.
- Update `src/sase/xprompts/skills/sase_new_task.md`: after a `sase bead +1` on a closed
  bead, the reporter must check whether the command reported a withheld reopen, and must
  pass `--verified-after-close` only when it actually reproduced the defect on a tree
  that contains the close. Regenerate the deployed skills as `generated_skills.md`
  describes.
- Tests: `tests/test_bead/test_plus_one_presentation.py`, the task-gate presentation
  tests, and the ACE bead-rendering tests.

### End-to-end race regression and store audit

- Add a regression exercise that reproduces the original failure end to end against a
  temporary store: create a task, preclaim it to `in_progress` with an assignee, record
  several +1s from reporters whose windows opened before the preclaim, close it as the
  assignee, then record one more +1 from a pre-close reporter and one from a post-close
  reporter. Assert the bead stays closed for the first and reopens with a cleared
  assignee for the second. Keep it close to
  `tests/test_bead/test_snooze_close_regression.py` in shape so the two race regressions
  read alike.
- Audit the live store for beads currently sitting at `ready` because of this race — a
  `close_history` record with `reopened_via: plus_one` whose `reopened_at` is close to
  its `closed_at` and whose close came out of `in_progress`. Report the list to the
  owner with a recommendation for each; do not mutate them without the owner's decision.
- Run `just check-full`, since this epic touches the bead store, the Rust binding pin,
  and four presentation surfaces.
- The `## Task Beads` section of `sase/memory/sase_beads.md` states the old promotion
  rule and will be wrong once this lands. Memory files may only be edited with the
  owner's explicit permission in the conversation, and plan text does not grant it. Ask
  for that permission; if it is granted, make the edit and run `sase memory init`. If it
  is not, leave the note untouched and say so plainly in the phase's completion report.

## Risks

- **A long-running agent's window opens early.** An agent that has been running for
  hours will have an observation window predating most closes, so a genuine post-close
  reproduction it files is withheld. That is the intended conservative direction, and
  `--verified-after-close` is the documented escape hatch — but it does mean the flag
  has to be discoverable, which is why the withheld-reopen output names it directly.
- **Silent suppression.** A withheld +1 that was actually fresh leaves the defect
  closed. Mitigated by the attributed note, the post-close corroboration badge on every
  surface, and the fact that the next reporter with a post-close window reopens the bead
  anyway.
- **Wire compatibility.** `observed_since` is optional on read, absent on write when
  unknown, and `None` deliberately preserves legacy behavior, so historical evidence and
  event streams replay to exactly the statuses they hold today. The whole-store replay
  described above is the baseline to re-run after the core phase.
