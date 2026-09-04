---
tier: epic
title: Initialize projects from the Admin Center Projects tab
goal: 'On the Admin Center Projects sub-tab, `i` initializes the marked or highlighted
  projects and `I` initializes every enabled project: each gesture plans off-thread
  via `sase init … --check --json`, shows a preview modal with the exact argv, per-planner
  rows, warnings, blockers, and full diffs, and on confirm streams exactly one `sase
  init … --yes` proc into the Procs tab — with an honest "Run in terminal" valve for
  TTY-only steps.

  '
phases:
- id: cli
  title: Project selector and structured check output for `sase init`
  depends_on: []
  size: medium
  description: 'cli: add the repeatable `-p/--project` selector beside `--all`, a
    `-j/--json` mode on `--check` with a schema version, per-planner `requires_tty`
    markers, and a status that distinguishes drift from blockers, lift the doctor
    plan serializer into `init_plan.py` without silent truncation, mirror the `--all`-with-subcommand
    dispatch guard for `-p`, and document both options.

    '
- id: flow
  title: The i/I gestures, the InitPlanModal preview, and the streaming apply proc
  depends_on:
  - cli
  size: large
  description: 'flow: wire `i`/`I` through the full Projects keymap chain, plan off-thread
    via a session-worker `sase init … --check --json` proc, show the `InitPlanModal`
    preview with per-planner rows, warnings, blockers, and full diffs, and on confirm
    submit exactly one streaming `sase init … --yes` session proc with a dedup key
    and an exclusive scope, then toast and refresh in place.

    '
- id: valve
  title: The run-in-terminal valve for TTY-only blockers
  depends_on:
  - flow
  size: small
  description: 'valve: when the plan reports TTY-only blockers, offer a "Run in terminal"
    button that suspends the TUI into interactive `sase init` for the blocked subset,
    handles unsupported suspend, and reloads on return without ever reporting held
    projects as initialized.

    '
- id: polish
  title: Hint line, key help, docs, and PNG snapshot goldens
  depends_on:
  - valve
  size: medium
  description: 'polish: surface the new keys in the Projects hint line and key help,
    document the flow in `docs/ace.md`, `docs/configuration.md`, and `docs/init.md`,
    and pin the modal''s visual states — single-project update, mixed all-projects,
    danger variant, TTY-blocked, diff expansion, and narrow terminal — with PNG snapshot
    goldens.

    '
- id: verify
  title: End-to-end verification of the init loop
  depends_on:
  - polish
  size: small
  description: 'verify: run the full verification gates and drive the real TUI through
    single, marked-set, and all-project inits, the no-op toast, the terminal valve,
    double-activation collision, and a mid-run kill, confirming refresh preserves
    selection and no half-written chezmoi state remains.

    '
proposed_by: bbugyi200.apollo.e
create_time: 2026-09-04 06:51:20
status: wip
bead_id: sase-wh
---

- **BEAD:** [sase-wh](https://github.com/sase-org/sase--beads/blob/main/pages/sase-wh/README.md)

# Plan: Initialize projects from the Admin Center Projects tab

## What this delivers

1. On the Projects sub-tab of the SASE Admin Center, `i` requests initialization of the
   marked set — or the highlighted project when nothing is marked — and `I` requests
   initialization of every enabled project (the canonical `sase init --all` inventory,
   never narrowed by marks, filter, or highlight).
2. Either gesture immediately sets a status line ("Checking initialization for sase…"),
   then plans **off-thread in a child process** via
   `sase init [-p <names>…|--all] --check --json`.
3. When everything is current, no modal opens — a toast says so
   (`sase is initialized · config, memory, repos, and skills are current`; aggregate
   counts for `I`). The gesture's intent was "initialize", not "inspect".
4. Otherwise an `InitPlanModal` opens in the Updates tab's house style ("the
   confirmation **is** the dry run"): the exact argv verbatim, per-project sections with
   the four planner rows, warnings and blockers verbatim, `d` for full diffs, and a
   **specific** primary button ("Initialize sase", "Initialize 5 runnable projects") —
   never "OK".
5. Confirm submits **exactly one** session-worker proc running
   `sase init -p <names> --yes` or `sase init --all --yes`, streamed live into the Procs
   tab, then a completion toast (success / current / partial / failure) and an in-place
   refresh that preserves selection, filter, and sub-tab.
6. Plans whose blockers are TTY-only grow a "Run in terminal" button that suspends the
   TUI into the real interactive `sase init` for the blocked subset.

Init is never implicit, never eager, never in-process, and never N procs. Zero init
logic is re-implemented in the TUI.

## Three hard constraints (verified against source)

- **C1 — init planning is cwd-global.** The planners read `Path.cwd()` process-globally
  and `run_init_onboarding_all` scopes each project by `os.chdir`-ing the whole process
  (`_working_directory`, `src/sase/main/init_onboarding.py:429`). Running planners or
  the coordinator inside a TUI thread would plan the wrong directory or corrupt every
  other thread's relative paths. **The TUI must always shell out**, and after the `cli`
  phase it always passes an explicit scope flag (`-p` or `--all`) so it never manages
  cwd for project scoping.
- **C2 — parts of init are non-bypassable-interactive by design.** `--yes` skips the
  generic per-plan `[y/N/d]` prompt but explicitly cannot authorize creating a missing
  provider sidecar repository, and owner-identity setup gates on a TTY
  (`src/sase/main/_repo_init_sidecars.py:138`,
  `src/sase/main/config_init_handler.py:480`; `docs/init.md` documents this as
  intentional). A TUI-only flow can never be complete; the `valve` phase is its honest
  exit, not a workaround.
- **C3 — init is too slow for any implicit path.** Measured ~8.4 s of planning work per
  project (~92 % host-scoped) on top of ~1.2 s CLI startup. Per-project drift is never
  computed eagerly for the list, and "all" is always **one** proc: N per-project procs
  would redo the host-scoped work N times and race the chezmoi deploys the coordinator
  deliberately batches (`defer_chezmoi_deploy`, `src/sase/main/init_onboarding.py:515`,
  with `LockTimeoutError` handling for the whole batch).

Two more verified facts shape the design: `--check`'s exit code conflates drift and
blockers (both exit 1, `init_onboarding.py:364-371`), so the JSON payload — not the exit
code — must distinguish them; and the apply **re-plans fresh**
(`_run_init_onboarding_result` plans and applies in the same invocation), so a
TUI-submitted `--yes` proc never applies a stale plan. That is the CLI's own trust model
and is why v1 needs no preview-fingerprint machinery.

## Decisions already made — do not relitigate

These were settled by the consolidated research report (read it with
`sase artifact read research:202609/projects_tab_init_ux/projects_tab_init_ux.md "<why>"`
for full evidence trails):

- **`i` respects marks** via the pane's existing `_target_records()` contract
  (`src/sase/ace/tui/modals/project_list_controller.py:254`), with disabled and
  system-managed rows filtered out of the mark set with a status message. **`I` always
  means the canonical `--all` inventory** and the modal says so.
- **Session worker, not durable proc.** The closest-weight precedents (sase self-update,
  memory publish) are session workers; init from this UI is user-attended; and apply
  re-plans fresh, so a durable fingerprint-checked `init.apply` operation buys nothing
  v1 needs. It remains the documented escalation path if unattended init is ever wanted.
  No new operation name in `src/sase/ops/names.py`.
- **The CLI is the shared API — no sase-core port.** Init planning is irreducibly
  Python-side (chezmoi, Jinja skill rendering, xprompt loading). The
  `rust_core_backend_boundary` litmus is satisfied because the TUI re-implements
  nothing: any frontend can consume `sase init --check --json` and submit
  `sase init … --yes`. The JSON payload carries a `schema_version` so promoting the
  schema into `sase-core` later is contained.
- **No feature flag.** This is an additive opt-in keybinding, not unproven behavior or a
  deprecation.
- **Apply uses blunt `--yes` semantics.** The memory step may commit and push; the modal
  must carry that warning with equal prominence to the CLI prompt's wording
  (`init_onboarding.py:148-171`). A commit/no-commit control like `MemoryPublishModal`'s
  is an explicit non-goal for v1 (the plan reviewer can veto this here).
- **No-op means no modal** (toast only). **Never auto-switch to the Procs tab.**
- **Out of scope:** a cached INIT drift column (severable fast-follow); structured TTY
  approvals through the coordinator's `_init_stdin`/`_init_input_func` injection seam
  (future work the valve keeps room for); hoisting host-scoped planners out of the
  `--all` loop (already filed as task bead `sase-w4` — do not refile).

## Cross-cutting constraints

- **Never edit memory files.** No phase may add, edit, or remove anything under
  `sase/memory/`, or in `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, or
  `QWEN.md`. A phase that concludes a memory note should change records a
  `PROPOSED FOLLOW-UP:` note on its own bead instead.
- **Do not hand-edit `CHANGELOG.md`** — release-please generates it.
- **Read the reference memories for your domain first**, through `/sase_memory_read`:
  `cli_rules.md` for the `cli` phase; `tui_perf.md` before any TUI phase;
  `lint_and_test.md` before finishing any phase.
- **No blocking IO on the Textual event loop.** Every subprocess call and JSON parse
  goes through a thread worker or a session proc, exactly as the pane's existing workers
  do. A key action updates visible status synchronously before any IO.
- **Every new `_submit_session_worker` call site must be registered** in the
  proc-producer inventory (`src/sase/ace/tui/proc_producer_sites.py` aggregates
  `_proc_producer_sites_actions.py` and siblings), or
  `tests/ace/tui/test_proc_producer_inventory.py` fails.
- **Keymap changes update `src/sase/default_config.yml`** (core-memory gotcha) and the
  whole chain listed in the `flow` phase.
- **Verification is two-speed.** Workspaces are ephemeral: `just install` first, then
  `just check` while iterating; `just check-full` only in the `verify` phase and only
  through `/sase_monitor`. PNG goldens run through `just test-visual`.
- Phase workers never create beads; they record `PROPOSED FOLLOW-UP:` notes on their own
  phase bead.

## Vocabulary and visual language

- The user-facing verb is **initialize**; the modal is the **initialization plan**.
- The CLI's glyph vocabulary carries over verbatim
  (`src/sase/main/init_preview.py:28-35`): `+` green create, `~` yellow
  update/overwrite, `−` red delete, `●` cyan validate/deploy, plus a dim `✓` for a
  current planner row. Warnings render yellow and blockers red, verbatim from the
  payload.
- Target modal shape (single project; the `--all` variant adds one aggregate line — e.g.
  `8 enabled · 5 need attention · 2 current · 1 unavailable` — and one collapsible
  section per project, changed/held first, current projects summarized in one line):

```text
┌─ ↻ Initialize sase ──────────────────────────────────┐
│ Project  sase                      ENABLED · github   │
│                                                      │
│ This can write files and may commit, push, create    │
│ repositories, or deploy managed files.               │
│ $ sase init -p sase --yes                            │
│                                                      │
│ ┌─ Initialization plan ────────────────────────────┐ │
│ │ ✓ CONFIG  Current                                │ │
│ │ ~ MEMORY  1 update · +96 −0                      │ │
│ │     ~ sase/task_types.json                       │ │
│ │ ✓ REPOS   Current                                │ │
│ │ ✓ SKILLS  Current                                │ │
│ └──────────────────────────────────────────────────┘ │
│                                                      │
│  y run · d diff · t run in terminal · esc cancel     │
│       [ Cancel (n) ]   [ Initialize sase (y) ]       │
└──────────────────────────────────────────────────────┘
```

---

## Phase `cli`: Project selector and structured check output for `sase init`

### Gap

`sase init` scopes only to "the cwd project" or `--all`, so a TUI (or any caller) cannot
initialize a named subset as one process with one batched chezmoi deploy. And `--check`
speaks only Rich prose with an exit code that cannot distinguish "drift" from "blocked",
so no frontend can render a truthful preview without re-implementing the planners. Both
additions are independently useful CLI features.

### Work

Read `sase/memory/cli_rules.md` through `/sase_memory_read` before touching the parser
(short aliases, alphabetical ordering, options-are-optional).

**1. `-p/--project NAME` (repeatable).** In `register_init_parser`
(`src/sase/main/parser_init.py:99-245`), add to the existing mutually exclusive group
that holds `-a/--all` and `-M` (around L114), so `-p` excludes both. Accept a project
name, display name, or alias; resolve against `resolve_init_project_inventory()`
(`src/sase/main/init_project_scope.py:97-120`). An unknown or non-enabled name fails
fast with a message listing the valid names. Implementation: give
`run_init_onboarding_all` (`src/sase/main/init_onboarding.py:493-591`) an explicit
target list and have `-p` filter the inventory into it — reusing the per-target
`Project: <ref>` headings, the `os.chdir` scoping, the status aggregation, the single
deferred chezmoi deploy, and its `LockTimeoutError` handling unchanged. Mirror the
existing `--all`+subcommand dispatch error in `src/sase/main/entry.py:266-301` for
`-p`+subcommand.

**2. `-j/--json` on `--check`.** `--json` without `--check` is a usage error (v1
promises structured output only for the read-only plan). With it, stdout carries exactly
one JSON document (human Rich rendering suppressed); exit codes are unchanged, and
consumers are expected to parse the payload rather than the exit code.

**3. The shared serializer.** Lift `_plan_row` out of
`src/sase/main/doctor/checks_config_init.py:111` into `src/sase/main/init_plan.py` next
to `InitPlan`/`InitAction` (`init_plan.py:13-41`) and have doctor call it. The lifted
serializer must **not** silently truncate: doctor may keep its `MAX_DETAIL_ROWS` cap via
a parameter, but the `--json` consumer gets every action, or an explicit truncation
marker if a cap is ever applied. The payload carries:

- top-level `schema_version` (start at `1`) and a top-level `status` that distinguishes
  `current`, `drift`, and `blocked`;
- one entry per targeted project: name, display name, per-project `InitRunStatus`-style
  status, `unavailable_reason` when the inventory held it back;
- per planner (config/memory/repo/skills): label, summary, `has_changes`, `runnable`,
  `requires_tty`, warnings, blockers, and actions `{path, operation, detail}` plus
  whatever `InitAction.new_content` provides so a consumer can render **full unified
  diffs with no second pass**.

**4. Typed `requires_tty` markers.** Classify blockers at their creation sites — at
minimum the provider-sidecar consent (`_repo_init_sidecars.py:138`) and owner-identity
setup (`config_init_handler.py:480`) — so consumers never grep prose to learn a plan is
TTY-held.

**5. Docs.** `docs/init.md` (options table and prose) and `docs/cli.md` (init section)
document both options, including the `-p`/`-a`/`-M` exclusivity and the JSON contract at
a summary level.

**6. Tests.** Parser wiring and exclusivity; `-p` filtering, alias resolution,
unknown-name failure, `-p`+subcommand error; JSON payload shape for a current project, a
drifted project, and a blocked project (drift vs blocked distinguished despite identical
exit codes); `requires_tty` present on the classified blockers; no silent truncation;
doctor output unchanged through the shared serializer.

### Done when

`sase init -p <name>` initializes exactly the named enabled projects as one process with
one batched chezmoi deploy; `sase init -p a -p b --check --json` and
`sase init --all --check --json` emit one schema-versioned document that distinguishes
drift from blockers and marks TTY-only blockers; doctor still passes through the shared
serializer; docs cover both options; and `just check` passes.

---

## Phase `flow`: The i/I gestures, the InitPlanModal preview, and the streaming apply proc

### Gap

Nothing in the Admin Center can invoke init. The Projects pane has every needed part —
mark-set targeting, thread workers, session procs with streaming reporters, and a
confirm-with-preview modal family — but no gesture, no preview, and no submission path.

This phase is `large`: plan it before implementing, and read `sase/memory/tui_perf.md`
through `/sase_memory_read` first.

### Work

**1. Keybindings, through the whole chain.** Two new Projects actions,
`initialize_project` (`i`) and `initialize_all_projects` (`I`) — both keys are currently
free in the Projects scope:

- `src/sase/default_config.yml` `ace.keymaps.projects` block (~L386-409);
- `ProjectsPaneKeymaps` (`src/sase/ace/tui/keymaps/app_keymaps.py:262-286`) — field
  names must exactly match the YAML keys or `scopes.py` raises at startup;
- `_PROJECTS_BINDING_META` (`src/sase/ace/tui/keymaps/metadata.py:193-218`);
- `src/sase/config/sase.schema.json` `ace.keymaps.projects` properties (~L2020);
- `_PROJECT_ONLY_ACTIONS` (`src/sase/ace/tui/modals/projects_pane.py:140-160`) so both
  are inert on the Repos/Workspaces sub-tabs;
- the keymap tests that pin the full default map (`tests/test_keymaps_defaults.py`,
  `tests/test_keymaps_validation.py`, `tests/ace/tui/test_projects_pane.py`).

`Enter` keeps meaning enable; init is never implicit.

**2. Scope.** New module `src/sase/ace/tui/modals/projects_pane_init.py` with a frozen
scope dataclass (selected project names or the all-projects sentinel), an `init_argv()`
mapping (`["sase","init","-p",…,"--check","--json"]` / `…"--yes"` /
`["sase","init","--all",…]`), and typed parsing of the phase-`cli` JSON payload. `i`
targets `_target_records()` (`src/sase/ace/tui/modals/project_list_controller.py:254`):
marked set if any, else the highlighted row. Disabled and system-managed rows in the
mark set are filtered out with a status message naming them (matching `--all`'s own
inventory rule); if nothing survives, warn and submit nothing. `I` ignores highlight,
filter, and marks entirely.

**3. The check proc.** Model on the Updates tab's preview→confirm→run shape
(`_submit_update_preview_proc`, `src/sase/ace/tui/actions/update_run.py:79-158`): a new
actions mixin `src/sase/ace/tui/modals/projects_pane_init_actions.py` (mixed into
`ProjectsPane`) sets a synchronous status line, then submits a session worker
(`_submit_session_worker`,
`src/sase/ace/tui/actions/_proc_action_submission.py:199-298`) that runs the check argv
via the `SessionProcReporter` subprocess helpers
(`src/sase/ace/tui/session_proc_reporter.py:133-275`) with an **explicit** cwd (never
scoping-significant, since the argv always carries `-p`/`--all`), captured stdout for
parsing, and an explicit timeout scaled to the target count (~8.4 s per project +
startup, with generous headroom). A dedup key makes repeat activation warn instead of
stacking. Exit codes 0 and 1 both mean "parse the payload"; other exits are errors.

**4. The preview.** On completion, parse the payload on a worker thread, then:

- all targets current → toast per the no-op rule, no modal;
- otherwise push `InitPlanModal`, built on `PluginActionConfirmModal`'s bones
  (`src/sase/ace/tui/modals/plugin_action_confirm_modal.py:64-90`) — reuse
  `PluginActionPreviewSection` if it fits, else extract a neutral shared base; do not
  leak "Plugin" naming into Projects.

Modal rules (see the mock above): exact apply argv verbatim; the may-write warning
including the memory may-commit-and-push wording with prominence equal to the CLI
prompt's (`init_onboarding.py:148-171`); four planner rows per project using the
planner's own summary and the glyph vocabulary; warnings yellow and blockers red,
verbatim; the `--all` variant's aggregate line and per-project sections with
changed/held first and current projects in one summary line; `d` toggles full unified
diffs built from the payload's action content; `ctrl+d`/`ctrl+u` scroll; a specific
primary button counting only runnable projects, with copy noting that held projects
remain unchanged; the dangerous variant styling when any overwrite/delete action is
present (no name-typing ceremony); the plan's timestamp shown, with a note that confirm
re-plans fresh. Reserve the `t` key/button slot for the `valve` phase but ship this
phase without it.

**5. The apply proc.** Confirm submits **exactly one** session worker:
`dedup_key=f"sase-init:{scope_key}"`, `exclusive_scopes=("sase-init",)` (deliberately
global — selected and all-project scopes share chezmoi deployment; cross-process races
stay covered by the chezmoi lock in the coordinator, which the TUI must not
re-implement), `reporter.set_command(argv)`, then `reporter.run(argv, cwd=…, timeout=…)`
streaming live into the Procs tab. Parse the coordinator's own `Project: <ref>` headings
into `reporter.phase(…)` for a one-line status (`Project 3 of 8 · memory`) — extend
`reporter.run` with an optional line hook if needed; no fake percentages. Double
activation and selected-vs-all overlap produce one clear collision warning via the
dedup/exclusive machinery.

**6. Completion.** `_on_init_complete` maps the outcome to a toast that distinguishes
success / current / partial
(`Initialized 5 · 2 current · 1 needs attention — see Procs`) / failure, then reloads
the pane inventories through the existing reload path preserving selection
(`preferred_project`), filter, and sub-tab. Never auto-switch to the Procs tab.

**7. Registration and tests.** Register both new submit call sites in the proc-producer
inventory (`_proc_producer_sites_actions.py` or the fitting sibling). Unit tests:
scope→argv mapping for every shape; payload parsing including `requires_tty` and
drift-vs-blocked; mark filtering messages; no-op toast path; the timeout scaling. A
pilot test: `i` → modal → confirm submits exactly one proc with the right argv, dedup
key, and exclusive scope; a second activation while one is in flight warns and submits
nothing.

### Done when

`i` and `I` plan off-thread, preview truthfully or toast on no-op, submit exactly one
streaming apply proc on confirm, refresh in place preserving selection, filter, and
sub-tab; entering the tab performs no init work; the preview never writes; planning and
apply never touch the event-loop thread; the proc-producer inventory test passes; and
`just check` passes.

---

## Phase `valve`: The run-in-terminal valve for TTY-only blockers

### Gap

C2 means some plans are honestly not completable from the TUI: provider-sidecar creation
and owner-identity setup demand a real TTY. Without a valve the modal shows a red
blocker and a dead end; the user must guess that the fix is "quit and run `sase init`
yourself".

### Work

When the parsed payload carries any `requires_tty` blocker, `InitPlanModal` shows a "Run
in terminal (`t`)" button/binding. Activating it dismisses the modal and runs
interactive `sase init` under `with self.app.suspend():`, modeled exactly on
`action_edit_project_spec`
(`src/sase/ace/tui/modals/project_management_actions.py:178-220`) including its
`SuspendNotSupported`/`OSError` handling. Scope the interactive argv to the blocked
subset via `-p <blocked names>` (no `--yes` — the point is the real prompts); for a
single-project plan that is just `-p <name>`. On return, reload the pane inventories and
re-resolve state; the toast reflects only what actually happened, and a held or
still-blocked project is **never** reported as initialized.

Tests: the button appears only when a `requires_tty` blocker is present; activation
suspends with the right argv and never submits a proc; the unsupported-suspend path
notifies instead of crashing; the post-return reload runs.

### Done when

A TTY-blocked plan offers one honest exit that hands the user the interactive tool
scoped to the blocked projects, the TUI recovers cleanly whether or not suspend is
supported, no held project is ever reported initialized, and `just check` passes.

---

## Phase `polish`: Hint line, key help, docs, and PNG snapshot goldens

### Gap

The feature works but is not discoverable, documented, or visually pinned. The Projects
hint line does not mention `i`/`I`, the docs' Projects key tables are stale, and nothing
stops a future change from silently degrading the modal.

### Work

- **Hint line.** Add `i init  I init all` early in `hints_text()`
  (`src/sase/ace/tui/modals/project_management_rendering.py:279-330`). The line's own
  comment says it already overflows 120 columns, so deliberately abbreviate or drop a
  segment to make room and say which one in the phase notes.
- **Key help.** Add both keys to whatever scoped key-help the Projects tab surfaces (the
  Admin Center footer/help path the existing Projects keys use).
- **Docs.** `docs/ace.md` Projects Tab key table and a short prose subsection on the
  init flow (preview, no-op toast, the memory commit/push warning, the terminal valve);
  `docs/configuration.md` projects keymap table (~L1145-1178) for the two new fields; a
  one-line cross-reference from `docs/init.md` that the Admin Center Projects tab can
  drive init.
- **PNG goldens** (`just test-visual`, goldens under
  `tests/ace/tui/visual/snapshots/png/`; accept intentional changes only with
  `--sase-update-visual-snapshots`): refresh the existing Projects goldens broken by the
  hint line, and add new goldens for the modal's states — single-project update,
  all-projects mixed status, danger styling, TTY-blocked with the `t` button, full-diff
  expansion, and a narrow terminal.

### Done when

The keys are discoverable from the pane and the docs, both docs tables match the shipped
defaults, the six modal states plus the refreshed Projects goldens pass
`just test-visual`, and `just check` passes.

---

## Phase `verify`: End-to-end verification of the init loop

### Work

1. `just install`, then `just check-full` **through `/sase_monitor`** (never inline),
   plus `just test-visual`.
2. Drive the real TUI through each loop, confirming the visible state at every step:
   - **Single project with drift:** `i` → status line → modal shows the argv, the
     drifted planner row with glyphs, and the diff under `d` → confirm → one proc
     streams in the Procs tab → toast → row refreshes in place with selection preserved.
   - **Marked set:** mark several projects including a disabled one → `i` filters the
     disabled row with a message and previews the rest → confirm runs one proc with one
     batched chezmoi deploy.
   - **All projects:** `I` ignores marks and filter, shows the aggregate line and
     per-project sections, and the primary button counts only runnable projects.
   - **No-op:** on a current project, `i` toasts and opens no modal.
   - **TTY-blocked:** with a plan holding a `requires_tty` blocker, `t` suspends into
     interactive `sase init` scoped to the blocked subset and the TUI reloads on return;
     the completion messaging never calls the held project initialized.
   - **Collision:** activating `i` while a check or apply is in flight warns once and
     stacks nothing.
   - **Kill:** kill the apply via the Procs tab mid-run and confirm no half-written
     chezmoi state remains (`chezmoi status`/`chezmoi doctor` clean, re-running
     `sase init` converges).
3. Any gap found is fixed in this phase when small, or recorded as a
   `PROPOSED FOLLOW-UP:` note on the phase bead.

### Done when

`just check-full` and `just test-visual` pass, every loop above is confirmed by
observation, and remaining findings are recorded on the bead.
