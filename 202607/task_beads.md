---
tier: epic
title: Task beads — capture, triage, and work discovered follow-ups
goal: 'Discovered follow-up work has a first-class destination: phase agents record
  PROPOSED FOLLOW-UP notes, land agents file them as task beads and mark them ready,
  a builtin chop raises a per-bead triage gate whose default action launches a detached
  #bd/work_task agent, and the task type and ready status render beautifully on every
  bead surface.

  '
phases:
- id: core-model
  title: Rust core task type, ready status, and ready-query redefinition
  depends_on: []
  size: medium
  description: 'core-model: add IssueTypeWire::Task and StatusWire::Ready with validation,
    schema CHECKs and migrations, redefine the ready query to unblocked ready task
    beads, extend CLI parse/render/stats tables, and cover with parity and mutation
    tests in the sase-core linked repo.'
- id: py-model
  title: Python model mirror, parsers, and CLI text
  depends_on:
  - core-model
  size: medium
  description: 'py-model: mirror the task type and ready status through model.py,
    db.py migrations, jsonl/wire, parse_type_arg, argparse choices, the ready/stats/detail
    handlers, doctor branches, and the sase-core-rs pin.'
- id: presentation
  title: Shared bead type and ready status presentation
  depends_on:
  - py-model
  size: small
  description: 'presentation: create bead_type_presentation.py as the single type
    glyph/accent/chip authority, add the ready row to BEAD_STATUS_PRESENTATIONS, and
    extend the exhaustive presentation contract tests.'
- id: tui
  title: ACE TUI task surfaces and PNG goldens
  depends_on:
  - presentation
  size: large
  description: 'tui: add a Tasks section and task row kind to the Plans pane with
    filters and detail chips, make the agents-pane bead lane and plan-association
    role type-aware, extend the bead autocomplete and edit modal, and regenerate PNG
    snapshot goldens with open and ready task fixtures.'
- id: pages-mobile
  title: Bead pages, mobile wire, and remaining text surfaces
  depends_on:
  - presentation
  size: small
  description: 'pages-mobile: render the task type and ready status on published bead
    pages, admit task in the mobile bead type filters, and sweep remaining issue_type
    branches for task-awareness.'
- id: xprompts
  title: Remove bd/next, rewire capture, add bd/work_task
  depends_on: []
  size: medium
  description: 'xprompts: delete the bd/next xprompt and its doc/test references,
    redirect bd/work_phase_bead to PROPOSED FOLLOW-UP notes, teach bd/land_epic to
    file ready task beads, and add the bd/work_task xprompt with its tag, resolver,
    and tests.'
- id: task-launch
  title: sase bead work for task beads and detached submitter
  depends_on:
  - py-model
  - xprompts
  size: large
  description: 'task-launch: route task bead targets through sase bead work with single-commit
    in-progress marking, rollback, dry-run, model routing including a task_worker
    alias, single-segment prompt composition ending in bd/work_task, and a task_launch.py
    detached submitter mirroring epic_launch.py.'
- id: triage-gate
  title: TaskTriage gate kind end to end
  depends_on:
  - task-launch
  size: large
  description: 'triage-gate: register a task_triage gate kind with launch-default
    and close-with-reason branches, build the spec and command shims in task_gate.py,
    implement apply_side_effects to submit the detached launch or close with the feedback
    reason, and wire ACE and mobile surfaces.'
- id: triage-chop
  title: bead_task_triage builtin chop
  depends_on:
  - triage-gate
  size: medium
  description: 'triage-chop: add the bead_task_triage chop to the checks lumberjack
    lane with per-bead gate creation, stale-gate cancellation, state-file dedupe,
    counters, tests, and axe docs.'
- id: docs-memory-skill
  title: Memory template, sase_beads skill, and documentation
  depends_on:
  - tui
  - pages-mobile
  - triage-chop
  size: medium
  description: 'docs-memory-skill: append the user-authorized task-bead recommendation
    to the builtin memory template and regenerate via sase memory init, update the
    sase_beads generated-skill source, and document the type, status, gate, and chop
    in docs/beads.md and docs/notifications.md.'
create_time: 2026-07-30 18:55:14
status: done
bead_id: sase-bg
---

- **PROMPT:** [202607/prompts/task_beads.md](prompts/task_beads.md)
- **BEAD:** [sase-bg](https://github.com/sase-org/sase--beads/blob/main/pages/sase-bg/README.md)

# Task Beads: Capture, Triage, and Work Discovered Follow-Ups

## Context

The 2026-07-30 research report `202607/sase_beads_close_integrity_and_capture/` (research sidecar) found that the
bead-capture prohibition in the `bd/*` xprompts leaves discovered work with no destination, and that `sase bead ready`
plus the `open` status lost their meaning once epic launch preassignment landed (F2). This epic implements a variant of
the report's recommendation 2 (with parts of 3), as directed by the user:

- Epic **phase agents** never create beads; they record discovered work as `PROPOSED FOLLOW-UP:` note entries on their
  own bead.
- The epic **land agent** triages those entries and may file each as a bead of a **new `task` type**, then mark it
  **`ready`** (a **new status**).
- `ready` gets a new, precise meaning: _a task bead proposed to the user for triage_. A new built-in lumberjack chop
  detects ready task beads and raises a **new first-class gate** per bead that lets the user either **launch an agent**
  to work it (default) or **close it with a reason**.
- `open` remains the draft state while an agent is still fleshing a task bead out.
- The `#bd/next` xprompt is removed; a new `#bd/work_task` xprompt drives launched task workers.
- The builtin `sase/memory/sase.md` template gains a recommendation telling agents when they can and SHOULD file task
  beads themselves.
- The `task` type and `ready` status get first-class visual treatment on every bead surface (TUI, pages, CLI, mobile,
  gate notification).

### Explicit user authorizations carried by this epic

- The user explicitly requested the change to the builtin memory template that generates `sase/memory/sase.md`. Editing
  `src/sase/main/init_memory/templates/memory-sase.template.md` and then running `sase memory init` (which regenerates
  this repo's `sase/memory/sase.md`, `AGENTS.md`, and the provider shims) is authorized and REQUIRED; do not ask again.
- The user explicitly requested removing the `#bd/next` xprompt and editing `bd/work_phase_bead` / `bd/land_epic`.

### Cross-repo boundary

Bead storage, mutation, query, and validation live in the Rust core (`sase_core` crate); Python is a facade plus CLI and
launch orchestration. Phase `core-model` is implemented in the **sase-core linked repo**: the implementing agent MUST
open it with the `/sase_repo` skill (`sase repo open sase-core -r "..."`) and work only in the printed path. The dev
build (`just install`) builds `sase_core_rs` from the linked checkout (`Justfile:22-24`), so later phases in this repo
pick the change up locally. Follow sase-core's existing release conventions for the version bump; if the released
version leaves the `sase-core-rs>=0.16.0,<0.17.0` window pinned in `pyproject.toml:46`, phase `py-model` must adjust the
window.

## Design

### The `task` bead type

A `task` bead is a standalone, launchable unit of discovered work. Constructor grammar: `sase bead create -T task` (no
parentheses — unlike `plan(<file>)` / `phase(<parent>)`).

Validation rules (enforced in BOTH the Rust validator `wire.rs::IssueWire::validate` and the Python mirror
`src/sase/bead/model.py::Issue.validate`, plus both SQLite CHECK schemas):

| Field                          | Rule for `task`                                                                                       |
| ------------------------------ | ----------------------------------------------------------------------------------------------------- |
| `parent_id`                    | MUST be empty (tasks are top-level; putting them under an epic would trip the descendant close guard) |
| `tier`                         | MUST be unset                                                                                         |
| `size`                         | OPTIONAL (drives model routing at launch, same vocabulary as phases)                                  |
| ChangeSpec metadata / `bug_id` | Disallowed at creation (mirror the existing phase restriction)                                        |
| `is_ready_to_work`             | Disallowed (epic-launch flag stays plan-tier-epic only)                                               |
| dependencies                   | Allowed (task→task blockers use the existing dep verbs unchanged)                                     |
| provenance                     | Free text in the description (e.g. "Discovered while landing sase-xy"); no new link types             |

### The `ready` status

New `Status.READY` / `StatusWire::Ready` between `open` and `in_progress`:

- Meaning: _drafted and proposed to the user; awaiting triage_.
- Restricted to task beads: CHECK `status != 'ready' OR issue_type = 'task'` (both schemas) plus validator rules.
- Transitions: `open → ready` (author marks done drafting, via `sase bead update <id> -s ready`), `ready → open`
  (retract to draft), `ready → in_progress` (launch), `ready → closed` (triage-close with reason). `claimed` never
  applies to task beads (`claim_for_agent_launch` is type-agnostic and already no-ops on `in_progress` + matching
  assignee — verified at `sase_core/src/bead/mutation.rs:375-440`; no core change needed for the launch claim).

### `sase bead ready` redefined

The current derived predicate (`open` phases / epic-tier plans without blockers — `read.rs:660-679`,
`cli.rs:394-433,2153-2156`) has zero traffic since launch preassignment: an `open` phase exists only during the
compile→launch window. Redefine the command (BOTH the Rust fast path and the Python fallback) to list **task beads with
`status == ready` and no active blocker** (reuse `has_active_blocker`). New empty-state message:
`No ready task beads (epic work is preassigned at launch).` The epic-launch flag machinery (`is_ready_to_work`,
`mark_ready_to_work`, `ready_marked` events) is untouched.

### Capture flow

1. Phase agents append `sase bead note <own-bead> 'PROPOSED FOLLOW-UP: <one-line summary — detail>'` (append-only
   `note_appended` events merge safely; this is the existing `sase bead note` verb, no new machinery).
2. The land agent collects `PROPOSED FOLLOW-UP:` entries from child bead notes while verifying (it already runs
   `sase bead show` on every child), files each worthwhile one as
   `sase bead create -T task -t "<title>" -d "<details + provenance>"` then `sase bead update <id> -s ready`, and
   records in its close note why any entry was NOT filed.
3. Guardrail preserved: phase agents _propose_, the land agent _files_.

### Triage gate (new first-class gate kind, NOT `kind: custom`)

Constraint discovered during design: gate option commands receive only the gate _input_ JSON on stdin
(`executor.py:_run_owned_command`); the user's feedback text (our close reason) lands in `response.json` and is only
available to `GateAdapter.apply_side_effects` — which today acts only for `plan`/`epic_plan`
(`src/sase/notification_gates/adapters.py:62-169`). Also `custom` gates forbid nothing-but-neutral presentation.
Therefore add a first-class kind:

- Kind `task_triage`, action `TaskTriage`, sender `bead-task-triage`, registered in `_ADAPTERS` (`adapters.py:213-275`),
  `auto_policy="forbidden"` (a human must triage).
- **One gate per ready task bead** (independent decisions, per-bead close reasons, clean per-bead dedupe). The "as few
  commits as possible" requirement is satisfied because a launch marks status+assignee in ONE mutation commit, pushed
  once. (Alternative considered and rejected: one batch gate per cohort — a single close reason would then span
  unrelated beads and decisions could not be split.)
- Query: `launch OR close`; `primary_branch: ["launch"]` (launch is the default). `launch` feedback `optional` (optional
  guidance text forwarded to the worker prompt); `close` feedback `required` (the close reason).
- Payload carries `bead_id`, `project` (key), and bead title; presentation shows the bead id + title in the notes, a
  `preview` resource rendering the bead detail (description/notes), icon `✦`, tags `["bead", "task"]`.
- Option commands are thin `#!<python>` shims (the `plan_gate.py:222-228` + `gate_command_entrypoint` pattern) that
  validate and echo JSON; the real work happens in `apply_side_effects`:
  - selection `("launch",)` → submit a **detached background task** (the epic-approval pattern: `adapters.py:127-169` →
    `prepare_epic_launch` → `submit_detached_task`), argv `sase bead work <bead-id> --yes-to-all`, cwd resolved for the
    bead's project (reuse the `epic_launch_cwd` resolution approach), origin mapped from the gate source; write the
    background task id back into `response.json`.
  - selection `("close",)` → close the bead in-process with `--resolution canceled --reason "<feedback>"` semantics
    (reuse the programmatic close path the CLI close handler uses).
- ACE: route `TaskTriage` through the existing custom-gate modal machinery (`_notification_modal_flow.py:190-218`
  dispatch + `_notification_custom_gate.py` loader work off the bundle's branch structure); add
  `ACTION_BADGES`/`ACTION_ICONS` entries (`notification_modal_constants.py:7-40`). Mobile/Telegram: add the kind to
  `_MOBILE_GATE_ACTION_KINDS` (`src/sase/integrations/_mobile_notification_actions.py:23-29`).

### Task work launch

Extend `sase bead work` (help: "Create or launch work for an epic plan **or task bead**") to accept a task bead id:

- Accept status `ready` (normal), `open` (manual power-user launch), or `in_progress` with a dead assigned agent
  (relaunch, mirroring epic retry semantics — skip if the assigned agent is alive); reject `closed`.
- Sequence: mark `status=in_progress` + `assignee=<agent name>` in ONE mutation, commit, push (best-effort like other
  bead pushes); compose the prompt; launch; roll status/assignee back on launch failure (mirror `rollback_work_launch`).
  `--dry-run` prints the composed prompt without writes.
- Agent name: the bead id itself (e.g. `sase-xy`), giving `%id(!<bead_id>, bead=<bead_id>)` — the runtime launch claim
  then no-ops cleanly.
- Prompt composition (single segment, and it MUST NOT contain `---` — the fast-launch adapter
  `launch_cwd_bead_work.py:42` assumes no fan-out; update its docstring which currently names only the two existing bead
  xprompts): `#<gh|git>:<project>` VCS prefix via the same vcs-context helpers the epic path uses, with `#commit` as the
  rollover method; `%id(...)`; `%m:<model>`; then `#bd/work_task:<bead_id>`. Optional launch feedback from the gate is
  appended as a trailing prompt line.
- Model routing: bead `model` field if set; else the size-based `<size>_phase_worker` alias when `size` is set; else a
  new builtin `task_worker` model alias (default `"@default"`, registered wherever the builtin aliases like
  `epic_lander` are defined in code, and documented in the commented `model_aliases` block at
  `default_config.yml:725-742`).
- New module `src/sase/bead/task_launch.py` mirroring `src/sase/bead/epic_launch.py`: build argv, idempotency lock (an
  active launch task for the same bead is reused, not duplicated),
  `submit_detached_task(label="Task launch · <id>", tags=("task", "launch"), ...)`.

### `#bd/work_task` xprompt (and edits to its siblings)

All in the keep-sorted `xprompts:` block of `src/sase/default_config.yml` (block: 880-984). Draft bodies (final wording
may be polished in-phase, semantics fixed):

New entry (sorted after `bd/work_phase_bead`, line ~955):

```yaml
bd/work_task:
  tags: work_task_bead
  input:
    bead_id: word
  content: |
    %wait(priority=15)
    Can you complete the work for task bead {{ bead_id }}? The bead is already reserved for you and assigned to your
    agent name: it was set to status=in_progress by the launch that started you; do not set the status by hand. Run
    `sase bead show {{ bead_id }}`, read the description and notes, do the work, and close the bead with
    `sase bead close {{ bead_id }} --note "<what you verified>"`. If you discover genuinely distinct follow-up work,
    do not expand this bead's scope: file a new task bead (`sase bead create -T task ...`), refine it while it is
    `open`, and mark it ready to triage with `sase bead update <id> -s ready`.
```

`bd/work_phase_bead` (945-955) — replace the final sentence `Do NOT create new beads.` with:

> Do not create beads yourself: record discovered follow-up work as a `PROPOSED FOLLOW-UP:` entry via
> `sase bead note {{ bead_id }} 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`; the epic's land agent triages these
> into task beads.

`bd/land_epic` (883-910) — extend step 1 with: _"While reviewing child beads, collect every `PROPOSED FOLLOW-UP:` note
entry."_ and extend step 3 (before the close) with: _"File each collected follow-up you judge worthwhile as a task bead:
`sase bead create -T task -t '<title>' -d '<details incl. which bead proposed it>'`, then
`sase bead update <id> -s ready`. Record in your close note why any entry was not filed."_

Remove `bd/next` (911-924) plus its references: the Built-in XPrompts table row `docs/xprompt.md:943`, the sentence at
`docs/troubleshooting/runner-slots.md:22` (reword to name only `#bd/land_epic`), and the `bd/next` parametrize case in
`tests/test_bead_xprompt_tags.py:65-95`. String-only incidental uses of the name (fabricated catalogs in
`test_xprompt_catalog_structured.py`, `test_xprompt_processor_workflow_flatten.py`,
`tests/ace/tui/test_xprompt_config_insert.py`, comments in `config_yaml.py`/`workflow_runner.py`) stay as they are.

Tag machinery: add `work_task_bead` to the `XPromptTag` enum (`src/sase/xprompt/tags.py:12-28`), a
`resolve_work_task_xprompt` helper in `src/sase/bead/xprompts.py`, a stub in the `fake_cli_work_xprompts` fixture
(`tests/test_bead/conftest.py:36-49`), and the tag-name list in `tests/test_xprompt_tags_parsing.py:58-59`.

### Triage chop

New builtin chop `bead_task_triage` in the `checks` lumberjack lane (interval 300 — triage cadence; the 10s `waits` lane
is too hot for user-facing notifications):

- `src/sase/scripts/sase_chop_bead_task_triage.py` with `@builtin_chop("bead_task_triage")` + `main()`, console script
  in `pyproject.toml` `[project.scripts]` (alphabetical, near line 123), config entry under `axe.lumberjacks.checks` in
  `default_config.yml` (636-660).
- Each tick: enumerate enabled projects with bead stores (the `sase_chop_bead_claim_checks.py:257-261` shape); list
  ready task beads (status filter via the read facade); reconcile against a state file in `runtime.context.state_dir`
  mapping `bead_id → gate request_id`:
  - ready bead with no live gate → create a `task_triage` gate (in-process `create_gate`, the `plan_gate.py` pattern —
    NOT by shelling to `sase gate create`); record the mapping.
  - pending gate whose bead is no longer `ready` (closed/relaunched/retracted out of band) → `cancel_gate` and drop the
    mapping (no stale notifications).
  - bead still ready with a live pending gate → skip (no re-notification spam).
- Gate creation is idempotent by `request_id`; derive the id deterministically from bead id + a generation counter so a
  canceled gate can be re-raised if the bead becomes ready again.
- Emit a chop summary with counters (`gated`, `canceled`, `skipped`) via `ChopResultBuilder`.

### Memory recommendation (builtin `sase/memory/sase.md`)

Append a new section to `src/sase/main/init_memory/templates/memory-sase.template.md` (static prose; the template's
strict Jinja2 vars are unaffected). Draft:

```markdown
## File Discovered Work As Task Beads

Unless your prompt explicitly forbids creating beads (epic phase workers, for example, must record `PROPOSED FOLLOW-UP:`
notes on their own bead instead), you can and SHOULD capture discovered follow-up work as sase task beads:

- A linter or test is flaky or failing and you did not cause it: file a task bead instead of ignoring the failure.
- A sase memory file or skill contains out-of-date information that should be updated: file a task bead proposing the
  update.
- A tool, command, or script this project is responsible for has a bug or a clear, objective improvement that would help
  future agents: file a task bead to fix or improve it.

Create the bead with `sase bead create -T task -t "<title>" -d "<details>"`, refine it while it is `open` (draft), and
mark it ready to triage with `sase bead update <id> -s ready`. Ready task beads are proposed to the project owner, who
either launches an agent to work them or closes them with a reason.
```

After editing the template, run `sase memory init` in this repo and commit the regenerated `sase/memory/sase.md`,
`AGENTS.md`, and provider shims together with the template change (explicitly user-authorized; see Context).

### Visual language

New shared module `src/sase/bead_type_presentation.py` (sibling of `bead_status_presentation.py` /
`phase_size_presentation.py` — today type renders everywhere as bare text; this module becomes the single authority):

| type  | glyph | accent    | chip style              |
| ----- | ----- | --------- | ----------------------- |
| plan  | `▸`   | `#FFD700` | `bold black on #FFD700` |
| phase | `↳`   | `#87D7FF` | `bold black on #87D7FF` |
| task  | `✦`   | `#D787FF` | `bold black on #D787FF` |

`ready` status row in `BEAD_STATUS_PRESENTATIONS` (`src/sase/bead_status_presentation.py:33-62`): cli_glyph `◇`,
tui_glyph `◇`, ANSI `\x1b[96m`, rich `#00D7AF`, label `Ready` — and the mirrored Rust tables (`cli.rs:2158-2165`
`status_icon`, `cli.rs:1930-1949` ANSI map). If Fira Code renders `✦` or `◇` as tofu in the pinned PNG environment, fall
back to `❖` / `◈` respectively and note the substitution in the phase commit.

Surfaces to update (details in phases): ACE Plans pane (new top-level **Tasks** section between Proposals and Epics,
`kind:task` + `status:ready` filters, type chip in the detail header, counts line), agents-pane bead lane, `@bead:`
autocomplete detail, bead edit modal label, published pages (roster Type column + identity facts), CLI
(`list/show/stats/ready`, both Rust fast path and Python), mobile wire, PNG snapshot fixtures/goldens, and the
`TaskTriage` notification badge/icon.

## Phases

### Phase core-model — Rust core: `task` type + `ready` status + ready-query redefinition

**Repo:** sase-core (open via `/sase_repo`). **Dependencies:** none.

- `wire.rs`: add `IssueTypeWire::Task` (16-22) and `StatusWire::Ready` (6-14); extend `IssueWire::validate` (278-321)
  with the task rules and the ready-requires-task rule from the Design section.
- `schema.rs`: extend the CHECK constraints (33-46) for the new type/status and ready-requires-task; add migrations
  following the existing `needs_issue_type_migration` / `issue_type_migration_sql` pattern (108+).
- `read.rs`: type/status parse+render (768-808); redefine the ready query (660-679):
  `status == Ready && issue_type == Task && !has_active_blocker`; keep `has_active_blocker` treating `Ready` as
  non-blocking-resolved? No — a `ready` dependency is still unresolved work: extend the active-blocker status set
  (703-720) to `Open | Claimed | Ready | InProgress`.
- `mutation.rs`: allow `Ready` in `update` status transitions for task beads only; keep `set_ready_to_work` /
  `preclaim_epic_work_plan` guards untouched.
- `cli.rs`: `parse_status` (2128-2135) + `parse_issue_type` (2137-2141) + `--type` constructor grammar `task` (836-859);
  `status_icon`/ANSI tables (1930-1949, 2158-2165) with the `◇` ready glyph; `handle_ready` (394-433) new predicate +
  empty message `No ready task beads (epic work is preassigned at launch).`; `handle_stats` (475-495) adds `Ready:` and
  `Tasks:` rows; `is_ready_surface_issue` (2153-2156) replaced by the task predicate.
- `search.rs`: type/status values (120-151).
- Parity/tests: extend `bead_read_parity.rs`, `bead_storage_parity.rs`, `python_wire_parity.rs`, and the mutation unit
  tests (create task, mark ready, update transitions, ready query, stats). Run the crate's checks per sase-core
  conventions; bump the crate version per its release process.

### Phase py-model — Python mirror: model, storage, parsers, CLI text

**Dependencies:** core-model.

- `src/sase/bead/model.py`: `IssueType.TASK`, `Status.READY`, `Issue.validate` mirror rules (16-23, 9-13, 75-98).
- `src/sase/bead/db.py`: CHECKs + migration (28-29, 54-61, 233-300); `src/sase/bead/jsonl.py:38,83`;
  `src/sase/core/bead_wire.py:46,63`.
- `src/sase/bead/cli_crud.py`: `parse_type_arg` grammar accepts bare `task` (41-78); create-time validation +
  `Created task: ...` message (82-138).
- Argparse: `parser_bead_lifecycle.py` update `--status` choices (255-262) + create `--type` help text (116-126);
  `parser_bead_queries.py` list/search `--status`/`--type` choices (110-131, 173-205). Follow `cli_rules.md`
  (alphabetical, short aliases, excellent help).
- `src/sase/bead/cli_query.py`: `handle_bead_ready` (180-190) mirrors the new predicate/empty message (drop the
  hard-coded `○`, use the ready presentation glyph); `handle_bead_stats` (205-215) adds `Ready`/`Tasks` rows.
- `src/sase/bead/cli_detail.py` (152-267): render type via the value (chip module arrives next phase), show `Size` for
  tasks when set.
- `src/sase/doctor/checks_beads.py:231-233` and any other `issue_type` branches found by grep — make them task-aware.
- Bump/adjust the `sase-core-rs` pin (`pyproject.toml:46`) if the sase-core release left the current window.
- Tests: mirror-validation tests, migration tests, CLI parse tests, ready/stats output tests; `just check`.

### Phase presentation — shared type/status presentation modules

**Dependencies:** py-model.

- New `src/sase/bead_type_presentation.py` per the Visual language table (glyph, accent, chip helper mirroring
  `phase_size_presentation.py:37-56`), consumed in later phases; exhaustive contract test (the
  `tests/test_bead_status_presentation.py:15-49` style).
- Add the `ready` row to `BEAD_STATUS_PRESENTATIONS` (`src/sase/bead_status_presentation.py:33-62`); update
  `tests/test_bead_status_presentation.py` and `tests/test_bead/test_claimed_status.py:90` style assertions.

### Phase tui — ACE TUI surfaces

**Dependencies:** presentation.

- Plans data: new `tasks` bucket in `plans_data_models.py:66-77` + snapshot collection in `plans_data.py:109-169`
  (top-level `IssueType.TASK` beads, all non-closed statuses plus recent closed consistent with sibling sections).
- Plans list: `PlanRowKind` + `"task"` (`plans_list.py:34`); a **Tasks** section between Proposals and Epics
  (`build_plan_options`, `_section_option` 339-362) with a `◇ ready` hint in the header legend.
- Row renderer `task_text()` in `plans_rendering.py` following the sibling row grammar (208-286): `✦ ` in
  `bold #D787FF`, status glyph, id `bold #FFD700`, title `white`, age `dim`; counts line adds tasks
  (`build_plans_status` 53-124).
- Detail pane `plans_detail.py`: type rendered as a chip via `bead_type_presentation` (70-77), size chip for tasks when
  set (74-75), preview markdown branch (288-310).
- Filtering: `_PlanFilterRecordKind` + `"task"` (`plans_filtering.py:16`, `_issue_record` 58-68), filter bar completions
  `kind:task` + ready status (`plan_filter_bar.py:38-66` — statuses flow from `bead_status_display_order()`
  automatically once the presentation row exists; verify).
- Actions: `actions/artifacts_plans.py` kind gates (165-180, 259, 422-436) — task rows open the bead edit modal /
  preview like phases; `bead_edit_modal.py:38` label gains the type glyph.
- Agents pane: `_agent_bead_section.py` lane header hard-codes `"phase"` (71-82) — make it type-aware (`task` lane
  label + labels `Task Title` etc.); `agent_associated_plan` role derivation (298-340) and
  `_agent_associated_plan_types.py:19` gain a `"task"` role.
- `@bead:` autocomplete detail kind (`_artifact_ref_entity_catalogs.py:105-117`) gains `task`.
- Visual snapshots: extend `tests/ace/tui/_artifacts_plans_helpers.py` and
  `tests/ace/tui/visual/test_ace_png_snapshots_artifacts_plans.py:39-90` fixtures with an `open` task and a `ready` task
  so every new glyph/chip/status appears in a frame; regenerate goldens with `--sase-update-visual-snapshots`
  (`just test-visual`); verify no tofu (see Visual language fallback).
- Read `sase/memory/tui_perf.md` via `/sase_memory_read` before implementing (required for TUI changes).

### Phase pages-mobile — published pages, mobile wire, remaining text surfaces

**Dependencies:** presentation.

- Pages: roster Type column (`src/sase/bead_pages/roster.py:19-49`) and identity facts (`rendering_identity.py:112-133`
  — size fact for tasks) render the new type/status; the ready status glyph flows through `status_icon`.
- Mobile: `_mobile_helper_beads.py` type filters (44-47, 416-425) admit `task`; `bead_type` passes through (261).
- `src/sase/main/plan_search_render.py:98-110` and `tests/test_plan_search_render.py:83-87`: extend the shared icon
  vocabulary with `◇` if plan-search surfaces bead statuses (verify; skip if bead-only).
- Sweep remaining `issue_type` string branches (`src/sase/agent/bead_display.py:387-397`, `src/sase/bug_links.py:85`)
  for task-awareness.

### Phase xprompts — remove `bd/next`, rewire capture, add `bd/work_task`

**Dependencies:** none.

- Apply every edit in the Design section "`#bd/work_task` xprompt" verbatim-or-polished: remove `bd/next`
  (`default_config.yml:911-924` + `docs/xprompt.md:943` + `docs/troubleshooting/runner-slots.md:22` +
  `tests/test_bead_xprompt_tags.py` parametrize case), rewrite `bd/work_phase_bead` + `bd/land_epic`, add `bd/work_task`
  inside the keep-sorted block.
- Tag machinery: `XPromptTag.work_task_bead` (`src/sase/xprompt/tags.py`), `resolve_work_task_xprompt`
  (`src/sase/bead/xprompts.py`), conftest stub (`tests/test_bead/conftest.py:36-49`),
  `tests/test_xprompt_tags_parsing.py:58-59`.
- Add a `#bd/work_task` row to the Built-in XPrompts table in `docs/xprompt.md`; new/updated cases in
  `tests/test_bead_xprompt_tags.py` (resolves, priority-only wait, config-sourced).

### Phase task-launch — `sase bead work <task-bead>` + detached submitter

**Dependencies:** py-model, xprompts.

- Dispatch: `cli_work_entry.py` routes `issue_type == TASK` targets to a new task path (status acceptance/relaunch rules
  per Design); epic path untouched. Update the `work` subcommand help text.
- Compose + launch per Design (single segment, no `---`; VCS prefix via the existing vcs-context helpers with `#commit`;
  `%id(!<bead_id>, bead=<bead_id>)`; model routing incl. the new builtin `task_worker` alias; `--dry-run`; single-commit
  in-progress marking with push; rollback on launch failure). Update the `launch_cwd_bead_work.py:29-57` docstring that
  enumerates bead xprompts.
- New `src/sase/bead/task_launch.py` (mirror of `epic_launch.py:24-160`): argv builder, idempotency, origin mapping,
  `submit_detached_task` wiring.
- Tests: rendering tests (the `tests/test_bead/test_work_rendering*.py` patterns), CLI launch/dry-run/relaunch tests
  (the `test_cli_work_epic_*.py` patterns), submitter idempotency tests.
- Read `sase/memory/xprompts.md` via `/sase_memory_read` before composing launch prompts.

### Phase triage-gate — `task_triage` gate kind end to end

**Dependencies:** task-launch.

- Adapter: `task_triage` entry in `_ADAPTERS` (`adapters.py:213-275`) + kind-validation hook (`kind_validation.py`) +
  `expected_primary` (`validation.py:169-182`) = `("launch",)`.
- Spec builder + side effects + command entrypoints in new `src/sase/bead/task_gate.py` (the `plan_gate.py` shape:
  `create_task_triage_gate`, `_build_task_triage_gate_spec`, `execute_task_triage_gate_command`,
  `translate_task_triage_response`), per the Design section (launch → detached task via `task_launch.py`, writing the
  task id into `response.json`; close → in-process close with `--resolution canceled` + feedback reason).
- `apply_side_effects` branch for the new kind (`adapters.py:62-169`), including cwd/project resolution for the detached
  launch.
- ACE: dispatch `TaskTriage` to the custom-gate modal flow (`_notification_modal_flow.py:190-218`), badge/icon
  (`notification_modal_constants.py`), gate-kind tab (`notification_modal_tags.py:22-27`). Mobile:
  `_MOBILE_GATE_ACTION_KINDS`.
- Tests: spec validation, executor round-trip with feedback, side-effect launch submission (detached task recorded),
  close-with-reason path, ACE dispatch test, mobile kind test.

### Phase triage-chop — `bead_task_triage` builtin chop

**Dependencies:** triage-gate.

- Implement the chop per the Design section (script + `@builtin_chop` + `main()` + `pyproject.toml` console script +
  `default_config.yml` `checks`-lane entry with a sensible `timeout`).
- State-file reconciliation (gate/cancel/skip) + deterministic regeneration-safe request ids + counters summary.
- Tests: tick behavior with fake stores/gates (gates created once, canceled when beads leave `ready`, no re-notification
  while pending); config schema coverage (`tests/test_config_schema_automation.py`) if it enumerates chops.
- Docs: chop row/description in `docs/axe.md` (chop tables at 330-357, 439-478).

### Phase docs-memory-skill — memory template, skill, documentation

**Dependencies:** tui, pages-mobile, triage-chop.

- Memory template edit + `sase memory init` regeneration + commit, per the Design section (user-authorized).
- `src/sase/xprompts/skills/sase_beads.md` (generated-skill SOURCE — never edit provider copies): document the `task`
  type, `ready` status, the create→ready workflow, `sase bead work <task-id>`, and the triage gate. Deployment note:
  after this epic lands on the canonical branch, run `sase skill init --force` + `chezmoi apply` from the clean merged
  tree (generated_skills.md rules) — the land agent should do this, not a mid-epic phase.
- `docs/beads.md`: task type + ready status in the lifecycle section (104-160), `ready` command section (559-562),
  capture flow (PROPOSED FOLLOW-UP → lander files task beads), triage gate + chop, `work` for task beads.
- `docs/notifications.md`: `TaskTriage` in the types table (106-122).
- Final sweep: `sase doctor` clean; `just check` green.

## Success criteria

- `sase bead create -T task -t X` creates an `open` task bead; `sase bead update <id> -s ready` marks it ready;
  `sase bead ready` lists exactly the unblocked ready task beads (both dispatch paths agree), and marking a phase bead
  `ready` is rejected by both validators.
- Within one chop interval a ready task bead produces exactly one `TaskTriage` gate; answering **launch** (the default)
  marks the bead `in_progress` in one pushed commit and a detached task launches an agent whose prompt ends with
  `#bd/work_task:<id>`; answering **close** closes it with the typed reason and `resolution=canceled`; the gate never
  re-fires while pending and is canceled if the bead leaves `ready` out of band.
- `#bd/next` is gone from config, docs, and tests; phase agents' xprompt directs follow-ups to `PROPOSED FOLLOW-UP:`
  notes; the land xprompt directs filing them as ready task beads.
- A fresh `sase memory init` in a sase-managed repo emits the task-bead recommendation section.
- Task beads render with the `✦`/`#D787FF` identity and `◇` ready glyph in the ACE Plans pane (new Tasks section,
  filters, detail chip), bead pages, CLI list/show/stats/ready, and the gate notification; PNG goldens cover an open and
  a ready task bead.
