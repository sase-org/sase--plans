---
tier: epic
title: New Project wizard — add or create projects from Admin Center ▸ Projects
goal: 'Pressing `n` on Admin Center ▸ Projects opens a beautiful three-stage New Project
  wizard (Source › Configure › Review). The user picks a provider, then either adds
  a repository SASE ranks as likely (local clones, unregistered repos in their GitHub
  owners, linked repos, disabled projects) or creates a brand-new repository through
  the provider plugin with every `gh repo create` option reachable. The project is
  registered enabled, set up so agents can launch in it, and ends on a Ready screen.
  The same behavior ships as `sase project suggest | add | create`, the CLI every
  frontend shares.

  '
decisions:
  human_only_record:
    ask: Add a decisions record that remote repository creation (sase project create)
      is human-only?
    memory:
    - decisions:project-create-human-only
    default: false
    answer: false
  glossary_project:
    ask: Update the Sase Project glossary strand to say sase project add/create and
      the wizard also create projects?
    memory:
    - glossary:sase-project
    default: false
    answer: true
phases:
- id: gh-hygiene
  title: sase-github traps the wizard would hit
  size: small
  depends_on: []
  description: 'gh-hygiene: in sase-github, bound `git clone` with a timeout, make
    `projects_base()` honor SASE_HOME, stop `ws_detect_workflow_type` from claiming
    non-GitHub hosts, and correct the stale `github_orgs` SSH/HTTPS docs.'
- id: provider-hooks
  title: Additive workspace-provider hooks and bare-git implementations
  size: medium
  depends_on: []
  description: 'provider-hooks: add the five onboarding hookspecs (parse, local candidates,
    probe, describe creation, create) with their frozen dataclasses, manager methods,
    registry wrappers and argument pins; extend the repo-candidate dataclasses and
    cache (host, truncated, remedy, repo id, push permission); implement every new
    hook for built-in bare git.'
- id: core-classify
  title: sase-core candidate classification and creation-form evaluation
  size: medium
  depends_on: []
  description: 'core-classify: add the sase-core `project_onboarding` module with
    `classify_project_candidates` and `evaluate_repo_creation_form`, their PyO3 bindings,
    table tests, a thin Python facade in sase, and the revision-pin bump.'
- id: core-plan
  title: sase-core onboarding plan derivation
  size: medium
  depends_on:
  - core-classify
  description: 'core-plan: add `plan_project_onboarding` (facts in, ordered steps,
    identity preview, setup resolution, blockers, warnings and intent fingerprint
    out) to the same module, with binding, exhaustive table tests, facade, and pin
    bump.'
- id: gh-provider
  title: sase-github implements the onboarding hooks
  size: medium
  depends_on:
  - gh-hygiene
  - provider-hooks
  description: 'gh-provider: implement parse, local candidates, probe, describe creation
    and create for GitHub, encoding the verified `gh repo create` rules exactly; enrich
    `gh repo list` fields, report truncation, remedies and the signed-in account.'
- id: cli-suggest
  title: Candidate discovery engine and `sase project suggest`
  size: medium
  depends_on:
  - provider-hooks
  - core-classify
  description: 'cli-suggest: create the `sase.project_onboarding` package with provider-neutral
    ref parsing and cache-first, per-owner discovery feeding core classification,
    plus the colored and JSON `sase project suggest` command.'
- id: cli-add
  title: Onboarding engine and `sase project add`
  size: large
  depends_on:
  - core-plan
  - gh-provider
  - cli-suggest
  description: 'cli-add: build facts gathering, plan documents, the convergent step
    executor with ownership journal and NDJSON events, the manage step, and the `sase
    project add` command with check, TTY confirm, durable-operation IO, init chaining,
    docs and skill updates.'
- id: cli-create
  title: '`sase project create` and the human-only guard'
  size: medium
  depends_on:
  - cli-add
  description: 'cli-create: add `sase project create` with strict provider-option
    validation, journaled remote creation and adoption, and the agent refusal with
    bypass and gate-option allowance; update docs and skill.'
- id: tui-source
  title: Wizard shell and the Source stage
  size: medium
  depends_on:
  - cli-suggest
  description: 'tui-source: build the NewProjectModal shell (stepper, provider strip
    with health chips, stage machine) and the Source stage (paste-anything filter,
    streamed classified groups, hidden-count footer, resume rows, live detail card),
    pushed directly in tests only.'
- id: tui-add
  title: Configure, Review, Progress, init hand-off and Ready for existing repos
  size: large
  depends_on:
  - tui-source
  - cli-add
  description: 'tui-add: complete the add-existing journey: identity and setup card,
    Review rendered from the CLI plan, durable apply with a live checklist, the reused
    init preview with its terminal valve, Ready, and recoverable failure.'
- id: tui-entry
  title: Projects-tab entry points and integration
  size: medium
  depends_on:
  - tui-add
  description: 'tui-entry: wire `n` (keymaps, default_config, schema, help), the palette
    entry and empty-state card, priority-truncated hints, and post-completion reload,
    selection and launch integration; document the wizard.'
- id: tui-create
  title: Create a new repository from the wizard
  size: large
  depends_on:
  - tui-entry
  - cli-create
  description: 'tui-create: show the pinned Create row, render the provider form with
    disclosure and live constraint greying, live name, availability and collision
    checks, danger-styled Review, create progress and the created-but-incomplete recovery.'
proposed_by: bbugyi200.athena.research.49.linker.w1
decided_by: reviewer
decided_via: tui
create_time: 2026-10-11 11:13:39
status: done
bead_id: sase-1jx
---

- **PROMPT:** [prompts/202610/new_project_wizard.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/new_project_wizard.md)
- **BEAD:** [sase-1jx](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jx/README.md)

# Plan: New Project wizard for Admin Center ▸ Projects

## Background

The user wants the **Projects** tab of the **SASE Admin Center** to create projects:

- New projects are enabled by default.
- The user picks the VCS (provider) the project uses.
- A menu then offers repositories the user is likely to add for that provider, such as
  repos in their configured GitHub organizations that are not yet SASE projects.
- The user can instead create a repository from scratch through the provider plugin. The
  wizard must offer every option the provider supports (public/private, description, …).
- It must be intuitive, reliable, and beautiful.

A research report precedes this plan. The user agreed with **every requirement it
recommends**, including all of its Requirement Adjustments A1–A11. Read it for the
evidence, mocks, and live measurements:

```bash
sase artifact read research:202610/projects_tab_add_project_wizard/projects_tab_add_project_wizard.md "<why>"
# fallback if the research sidecar is not cloned in your workspace:
sase artifact read file:explicit:4e988ce0ff2fb6be4c4b4336 "<why>"
```

This plan adopts that design and turns it into contracts and phases.
[Refinements of the research](#refinements-of-the-research) lists the places where it
fills a gap or departs, each with a reason.

### What is true today (verified on master)

- **There's no create path.** `sase project` has
  `alias|current|disable|enable|list|set-current|set-state|show`
  (`src/sase/main/parser_project.py`). A project only appears as a side effect of a
  `#gh:` or `#git:` macro resolving through `ws_resolve_ref`, which clones, allocates
  the key (`gh_<owner>__<repo>`, `-2` on collision), and writes `WORKSPACE_DIR` and
  `PROJECT_NAME`.
- **Enabled is already the default.** A missing `PROJECT_STATE:` means enabled. Never
  write `PROJECT_STATE: enabled`.
- **Registered is not launchable.** sase-github uses the `separate_repo` SDD policy.
  `_resolve_sdd_creation_authorization` (`src/sase/sdd/_store_materialization.py`)
  refuses unless the repo's own `sase/sase.yml` has `is_sase_managed: true`. Creating a
  missing sidecar needs an interactive `y` that `--yes` can't grant (`docs/init.md`).
- **`n` is free** in `ace.keymaps.projects` (`src/sase/default_config.yml:716-739`).
  App-level `n` (`rename_cl`) is non-priority, so a pane-local binding wins.
- **The Projects hints row is already clipped.** `hints_text()` builds about 232 cells;
  `#projects-hints` is one line, so everything after `m mark` is cut at 120 columns.
- **Init is the gold-standard mutation flow.** It runs `sase init --check --json` in a
  session worker, shows `InitPlanModal` (glyphs `+ ~ − ●`, `d` diffs, `t` terminal
  valve), then submits apply (`projects_pane_init_actions.py`, `init_plan_modal*.py`).
- **Durable procs exist for long user operations.**
  `_submit_durable_proc(argv, operation=…, request=…, request_fingerprint=…, concurrency_keys=…)`
  (`src/sase/ace/tui/actions/_proc_action_submission.py`) needs a registered operation
  name (`src/sase/ops/names.py`). Every submit site must be registered in
  `src/sase/ace/tui/_proc_producer_sites_*.py` (enforced by
  `tests/ace/tui/test_proc_producer_inventory.py`). Durable rows expose `log_path`, and
  the observer reads tails with `read_proc_log_tail`.
- **Provider seams cover about 70% of discovery.** These exist today:
  `ws_get_workflow_metadata`, `ws_list_ref_namespaces` (github_orgs ∪ project owners),
  `ws_list_repo_candidates` (`gh repo list --limit 200`, 10 s, typed error kinds),
  `ws_peek_ref`, and `fetch_repo_candidates` (memo plus a 600 s disk cache keyed by
  `(workflow_type, namespace)`). Hook argument names are a frozen cross-repo boundary
  (`tests/test_workspace_provider_hookspec.py`).
- **sase-core has no project allocation, plan, or sidecar-role code.** The closest "host
  observes, core decides" precedents are `machine_setup/` and `git_object_sharing.rs`.
  `project_spec.rs` is already over the 1,500-line limit, so new code goes in a new
  module.
- **Sidecar visibility is handled elsewhere.** The in-flight plan
  `plan:202610/private_primary_sidecars.md` makes `sase repo init` create **private**
  sidecars whenever the primary repository is private or its visibility is unverified.
  That delivers research adjustment **A9**. This epic must not re-implement it; see
  [the prerequisite](#prerequisite-outside-this-epic).

### Prerequisite outside this epic

`cli-add` and `tui-add` route sidecar creation through the existing init flow. Before
they finish, confirm that the private-primary sidecar change has landed: look for
`src/sase/sdd/_sidecar_visibility.py` and the `private_primary` keyword on
`explicit_sidecar_config_update`. If it has not landed, the phase must not paper over
it. Record a blocker note on the phase bead and stop.

## Design principles

1. **The CLI is the product; the TUI is its most beautiful frontend.** Every decision
   ("which repos are likely", "what will happen") is computed once, below the TUI, so
   the CLI, the TUI, and future frontends agree.
2. **Plan, then apply.** Every mutation is previewed with the exact argv, in the init
   preview's glyph vocabulary, before it runs. Apply is one durable proc.
3. **Converge, don't replay.** Each step ensures a postcondition. Re-running the same
   argv re-observes the world, re-plans, and resumes at the first unmet step. Nothing
   remote is ever deleted.
4. **Each layer owns its concerns.** Providers own provider behavior (hooks). sase-core
   owns the shared rules (pure functions). Python owns effects and orchestration. The
   TUI owns presentation.
5. **Cached first, then revalidated.** Each owner refreshes independently. A failing
   owner never blanks the list, and nothing disappears silently.
6. **Never surprise.** Created repos, commits, pushes, visibility, and the
   current-project switch are all named before they happen. PUBLIC is shown in warning
   color.

## Refinements of the research

| #   | Research                                        | This plan                                                                                                                                                                                                      | Why                                                                                                                                                                    |
| --- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R1  | Paste "anything" into the filter.               | A fifth, **pure** hook `ws_parse_repo_ref(workflow_type, text)` turns pasted text into a canonical ref, identity, key/name bases and checkout path.                                                            | Someone has to own provider spelling (URLs, ssh, `gh:`). The hook is pure, so it is safe on keystroke paths and also drives the create form's live name normalization. |
| R2  | Steps `clone` then `register`.                  | One **`checkout`** step ("Clone and register" / "Register existing clone"), executed by `ws_resolve_ref`.                                                                                                      | Both providers clone and register in one call. Two steps would be a fiction.                                                                                           |
| R3  | `sase project create PROVIDER NAME … -o OWNER`. | `sase project create REF`, where REF is `gh:OWNER/NAME`, `OWNER/NAME`, or `git:NAME`.                                                                                                                          | Same grammar as `add` and the `#gh:`/`#git:` macros; no redundant owner flag.                                                                                          |
| R4  | Plan JSON "mirrors init".                       | Init's JSON has no `steps` or `apply_argv`. This epic defines its own `project_onboarding_plan` v1 document, reusing init's **operation names and glyphs** (create `+`, update `~`, delete `−`, validate `●`). | Review can reuse the init renderer's vocabulary without bending the init schema.                                                                                       |
| R5  | `add` is agent-callable.                        | `add` stays agent-callable, but **from an agent it never runs the commit-and-push setup step**: auto setup resolves off, and an explicit `--setup` is a blocker.                                               | Pushing to a shared repo's default branch is an outward-facing write. It needs the same human-only rule as remote creation.                                            |
| R6  | Constraint greying in the TUI form.             | A third pure core function, `evaluate_repo_creation_form`, with `strict` mode (CLI `-O` validation) and `lenient` mode (TUI greying).                                                                          | The CLI and TUI must agree on which option combinations are valid.                                                                                                     |
| R7  | `t` runs `gh auth login`.                       | Candidate, probe and form results carry a provider-supplied `remedy_argv`; `t` runs it in a suspended terminal.                                                                                                | Keeps the TUI provider-neutral.                                                                                                                                        |
| R8  | Owners are `github_orgs` ∪ project owners.      | `VcsRefNamespaces` gains `account`. sase-github reads the signed-in login from gh's local `hosts.yml` (no network), so personal repos always appear.                                                           | Discovery works even with an empty `github_orgs`, and the empty state can name the account.                                                                            |
| R9  | Journal at `~/.sase/projects/.onboarding/`.     | `$SASE_HOME/project_onboarding/journals/`, plus `preferences.json` beside it.                                                                                                                                  | The projects directory is scanned for project records.                                                                                                                 |
| R10 | Don't show the Create row until create lands.   | Same, and **no feature flag**. The `n` key is wired only in `tui-entry`, after add-existing is complete. Earlier TUI phases are reachable only from tests.                                                     | No dead affordances and no flag scaffolding.                                                                                                                           |
| R11 | A9 (private sidecars).                          | Delivered by `private_primary_sidecars`; this epic only _displays_ the resulting sidecar visibility in Review.                                                                                                 | Avoids two implementations of one invariant.                                                                                                                           |

## The experience

### Entry points (`tui-entry`)

- **`n` on Admin Center ▸ Projects ▸ PROJECTS.** It is gated like the other project-only
  actions and leads the hints row as `n new`.
- **Command palette:** _Projects: New project…_ opens Admin Center ▸ Projects and the
  wizard.
- **Empty state:** with no registered projects, the list's empty row reads
  `No projects yet — press n to add or create one`.

### Visual language

The wizard reuses the Admin Center's look, so it feels native:

- **Frame:** a `ModalScreen` with the thick teal frame (`#2BE7C7`) and border title
  `✚ New Project` in bold `#FFAF5F`, the Projects accent.
- **Stepper:** the existing stepper idiom
  (`save_location_picker_modal.py: "● Location › ○ Name"`):
  `● Source › ○ Configure › ○ Review`. Active steps are `●` in bold `#FFAF5F`, done
  steps `✓` in green, pending steps `○` dim.
- **Provider strip:** `PanelTabStrip` / `PanelTab` with a right-aligned health chip.
- **Chips:** `gh` in `#87D7FF`, `git` in `#00D7AF`, state `● enabled` in `#00D7AF`.
- **Accent swatch:** `■` in the color the new project will use in the Projects list:
  `project_accent(key, among=accent_among_keys(records) ∪ {key})`.
- **Review rows:** init's glyphs (`+` green, `~` yellow, `−` red, `●` cyan) and its
  `$ argv` line. Remote creation uses `ConfirmKind.DANGER` styling.
- **Untrusted text:** provider strings (names, descriptions, errors) render as literal
  `Text`, never Rich markup, with control characters stripped.

### Stage 1 — Source (`tui-source`, Create row in `tui-create`)

```
╭─ ✚ New Project ──────────────────────────────────────────────────────────────╮
│                     ● Source  ›  ○ Configure  ›  ○ Review                    │
│   GITHUB   Git (bare)                    ✓ bbugyi200 · 3 owners · cached 2m  │
│ ╭──────────────────────────────────────────────────────────────────────────╮ │
│ │ ⌕  Filter — or paste owner/repo, a GitHub URL, or a local path           │ │
│ ╰──────────────────────────────────────────────────────────────────────────╯ │
│ ╭─ Repositories ────────────────────────────────────────── cached 2m · R ──╮ │
│ │ ✚  Create a new GitHub repository…                                    n  │ │
│ │ RESUME                                                                   │ │
│ │ ↻  sase-org/forge            stopped at checkout · ⏎ finish setup        │ │
│ │ ON THIS MACHINE                                                          │ │
│ │ ⌂  bbugyi200/dotfiles        public     1d   cloned · not a project      │ │
│ │ SUGGESTED   bbugyi200 · sase-org · bobs-org                              │ │
│ │ ○  bobs-org/bob-plugins      public     4h   bob | Plugins Monorepo      │ │
│ │ ○  bobs-org/bob              private    4h   Bugyi's Obsidian            │ │
│ │ LINKED TO YOUR PROJECTS                                                  │ │
│ │ ⇄  sase-org/sase-core        public     0d   linked by sase              │ │
│ │ DISABLED PROJECTS                                                        │ │
│ │ ◐  bbugyi200/job             private    9d   ⏎ re-enables job            │ │
│ │ ┄  26 sidecars · 24 forks · 3 projects hidden          f forks · . all   │ │
│ ╰──────────────────────────────────────────────────────────────────────────╯ │
│ ╭─ bobs-org/bob ───────────────────────────────────────────────────────────╮ │
│ │ private · main · pushed 4h ago · you can push                            │ │
│ │ Checkout  ~/projects/github/bobs-org/bob/                   will clone   │ │
│ │ Project   bob   key gh_bobs-org__bob   ■   ● enabled                     │ │
│ │ SASE      ○ not yet managed — will be set up (sidecars private)          │ │
│ ╰──────────────────────────────────────────────────────────────────────────╯ │
│  ⏎ add · n new repo · [ ] provider · / filter · f forks · . all · R refresh  │
╰──────────────────────────────────────────────────────────────────────────────╯
```

**Behavior**

- **Opening is instant.** Rows come first from the cache only
  (`peek_cached_repo_candidates`). Then each owner revalidates in a worker with bounded
  parallelism (≤ 3). A not-yet-fetched owner shows a dim `◌ fetching bobs-org…` row. A
  failed owner shows a badge row (`⚠ bobs-org: gh auth expired — t runs gh auth login`)
  and keeps its stale rows.
- **Provider strip.**
  - Tabs come from `ws_get_workflow_metadata` for providers that implement the new
    hooks. `[`/`]` switch tabs.
  - The initial tab is the last-used provider (from preferences), else the current
    project's provider, else the first.
  - Health chips: `✓ bbugyi200 · 3 owners · cached 2m`, `◌ loading 2 of 3 owners`,
    `⚠ not signed in · t gh auth login`, or `✗ gh not installed`. With `gh` missing the
    tab is disabled, and Git (bare) still works.
- **Groups**, in order: RESUME, ON THIS MACHINE, SUGGESTED, LINKED TO YOUR PROJECTS,
  DISABLED PROJECTS. Empty groups are omitted.
- **Row glyphs:** `✚` create, `↻` resume, `⌂` local, `○` suggested, `⇄` linked, `◐`
  disabled, `↵` pasted ref, `◌` fetching, `⚠` owner error.
- **Pinned rows.** `✚ Create a new <Provider> repository…` is pinned first and never
  moves. It appears only from `tui-create` on, and stays present while loading or empty.
- **Hidden rows.** Enabled projects, sidecars, forks, archived repos, numbered clone
  copies and third-party local clones are hidden. A dim `┄` footer always states the
  counts and the reveal keys (`f` forks, `.` everything). Hidden rows still match the
  filter, so typing `sase` shows
  `● sase-org/sase · already the project sase — ⏎ show it`.
- **Filter.**
  - `/` focuses the filter, and typing filters in memory only.
  - `↑`/`↓`/`ctrl+n`/`ctrl+p` move the highlight from inside the filter. `⏎` chooses,
    and `esc` returns focus to the list.
  - A bracketed paste anywhere focuses the filter with the pasted text.
  - When the text parses as a ref (`ws_parse_repo_ref` for the selected provider), a
    synthetic `↵ Add sase-org/foo` row appears on top. This covers repos outside
    `github_orgs` or past the 200-row cap.
- **Truncation.** When a list is truncated the footer reads
  `showing 200 · type to filter or paste owner/repo`.
- **Detail card.**
  - It fills instantly from local data: parsed ref, checkout path, identity preview,
    accent.
  - It then fills from `ws_probe_repo`: debounced through `DetailPanelDebouncer`, run
    off-thread, and cached per ref for the modal's lifetime. Status goes `◌ checking…` →
    `✓ managed` / `○ will be set up` / `? unknown`.
  - The highlight itself is never debounced.
  - Below 30 rows of height the card collapses to one summary line.
- **Empty states.**
  - **No owners:** a card:
    `No GitHub owners yet — paste owner/repo, or add owners to github_orgs in Config Center`.
    If `account` is known, its repos already show, so this appears only when nothing is
    known.
  - **A provider plugin is not installed:** if the Plugins browser's catalog lists an
    uninstalled workspace-provider plugin, show it as a dim tab whose card explains how
    to install it (Admin Center ▸ Plugins). If that catalog isn't cheaply available,
    omit the tab.
- **Keys** (list focused): `j/k/↑/↓`, `⏎` choose, `n` new repo (from `tui-create`),
  `[`/`]` provider, `/` filter, `f` forks, `.` all, `R` refresh (bypasses the cache),
  `t` run the highlighted owner's `remedy_argv` in a suspended terminal, then refresh,
  `esc` close.
- **What `⏎` does on each row:**
  - **Already-enabled project:** dismisses the wizard and selects that project.
  - **Running resume row:** jumps to its proc in Admin Center ▸ Procs.
  - **Failed resume row:** opens Review with the resume plan.
  - **Anything else:** Configure.

### Stage 2a — Configure, existing repo (`tui-add`)

```
│ ╭─ Project ────────────────────────────────────────────────────────────────╮ │
│ │ Name       bob                                    ✓ unique               │ │
│ │ Aliases    —                                                             │ │
│ │ Checkout   ~/projects/github/bobs-org/bob/        will clone             │ │
│ │ Key        gh_bobs-org__bob      ■ accent         ● enabled              │ │
│ ╰──────────────────────────────────────────────────────────────────────────╯ │
│ ╭─ SASE setup ─────────────────────────────────────────────────────────────╮ │
│ │ [x] Make bob SASE-managed and initialize it                recommended   │ │
│ │     Needed before agents can launch here. Commits + pushes sase/sase.yml │ │
│ │     to main, then shows the init preview; creating the private sidecars  │ │
│ │     bob--plans · bob--beads asks you in a terminal.                      │ │
│ │ [ ] Make it the current project when done                                │ │
│ ╰──────────────────────────────────────────────────────────────────────────╯ │
```

- **Name and aliases are editable inline.** Live validation comes from core
  `plan_project_onboarding`, run with local facts only (records), debounced. Results
  read `✓ unique`, `✗ taken by sase — try bob_1`, or `✗ invalid name`.
- **The setup checkbox reflects the plan's setup resolution and reason.**
  - Already managed: `✓ already SASE-managed — sidecars connect on first launch`, and
    the checkbox is hidden.
  - No push rights: unchecked, with an amber line naming who must run `sase init -M`.
  - Bare git: the copy omits sidecars.
- **A disabled project** shows `bob is a disabled project — adding re-enables it`.
- **Keys and focus.**
  - `tab`/`shift+tab` move between fields, and `space` toggles a checkbox.
  - Advance with `⏎` on the focused `Review ›` button. Use the same advance gesture as
    the Config Center editors (check `config_edit_modal.py`).
  - `b` or `esc` goes back. Printable shortcuts never fire while a text input has focus;
    there, `esc` first blurs the input.
  - Going back and forth preserves drafts.

### Stage 2b — Configure, new repo (`tui-create`)

```
│ ╭─ Repository ─────────────────────────────────────────── GitHub options ──╮ │
│ │ ▸ Owner         sase-org                    ▾    ✓ you can create here   │ │
│ │   Name          sase forge  →  sase-forge        ✓ available             │ │
│ │   Description   Forge for SASE plugin scaffolds                          │ │
│ │   Visibility    ( ) Public  (•) Private  ( ) Internal  enterprise only   │ │
│ │   README        [x] Initialize with a README                             │ │
│ │   .gitignore    Python                      ▾                            │ │
│ │   License       MIT License                 ▾                            │ │
│ │ ▸ ADVANCED · 6  template · all branches · homepage · team · issues · wiki│ │
│ ╰──────────────────────────────────────────────────────────────────────────╯ │
│ ╭─ Project ────────────────────────────────────────────────────────────────╮ │
│ │ Name  sase-forge  ✓ unique   key gh_sase-org__sase-forge  ■  ● enabled   │ │
│ │ [x] Set up SASE (recommended for new repos)   [ ] Make it current        │ │
│ ╰──────────────────────────────────────────────────────────────────────────╯ │
```

- **Owner choices** are the namespaces, plus `account`. `can_create` comes from
  `ws_describe_repo_creation`.
- **Live name checks.**
  - Normalization via `ws_parse_repo_ref` (`sase forge → sase-forge`), on every
    keystroke, because the hook is pure.
  - Availability via `ws_probe_repo`, debounced 300 ms, off-thread.
  - SASE name/key/alias uniqueness via core plan.
  - Checkout path collision via an off-thread `exists()`.
- **Delight: a taken name becomes an offer.** It shows
  `✗ sase-org/sase-forge exists — ⏎ to add it instead`, which switches to 2a for that
  ref. Create intent is never silently converted.
- **Advanced** (`a`) reveals the advanced group with a count.
- **Live constraint greying** uses core `evaluate_repo_creation_form` (lenient mode).
  Choosing a Template greys README, .gitignore, License and Team, each with its reason.
  Internal visibility renders disabled (`enterprise orgs only`) unless the provider says
  otherwise.
- **Defaults.**
  - **Owner:** the last used, else the current project's owner, else `account`.
  - **Visibility:** **Private** the first time for each owner, then the last choice for
    that owner (`preferences.json`).
  - **README:** on.
  - **Make current:** off.
- **Large choice lists** (162 gitignore templates, 13 licenses) use
  `EnumChoicePickerModal`-style searchable pickers that also accept typed values.

### Stage 3 — Review (`tui-add`; danger variant in `tui-create`)

```
╭──────────────────── Review · Create private repo sase-forge ─────────────────╮
│  REMOTE   +  github.com/sase-org/sase-forge   private · README · MIT · Python│
│  LOCAL    +  clone → ~/projects/github/sase-org/sase-forge/ and register     │
│  PROJECT  +  sase-forge  (gh_sase-org__sase-forge)   ● enabled   ■           │
│  SASE     +  sase/sase.yml  is_sase_managed  commit + push to main           │
│           ●  then the init preview (private sidecars ask in a terminal)      │
│  ⚠ Creates a repository on github.com. A failed later step keeps it;         │
│    retrying resumes from the first incomplete step.                          │
│  $ sase project create gh:sase-org/sase-forge -v private -d … --yes --json   │
│              [ Create  y ]      [ Back  b ]      [ Cancel  esc ]             │
╰──────────────────────────────────────────────────────────────────────────────╯
```

- **Source of the content.** Review renders the plan document from
  `sase project add|create … --check --json`. It runs in a session worker like the init
  check, and the rows are built by a pure renderer.
- **Add mode** omits the REMOTE line and uses a neutral confirm.
- **Blockers and warnings.** Blockers render in red with their hint, and the confirm is
  disabled. Warnings render in amber.
- **Visibility.** PUBLIC always renders in warning color. The sidecar line states the
  visibility the init flow will use (private for a private or unverified primary, per
  `private_primary_sidecars`).
- **Keys:** `y` apply, `b`/`esc` back. Unlike `InitPlanModal`, `n` is not cancel.

### Stage 4 — Progress → init hand-off → Ready (`tui-add`)

- **Submission.** `y` submits `apply_argv` through `_submit_durable_proc`, with the
  `request_fingerprint` and `concurrency_keys` from the plan.
- **The checklist** is driven by NDJSON step events read from the durable proc's log
  tail. Rows go `◌` pending → spinner plus elapsed while running → `✓` with elapsed
  (green) → `✗` (red) with message and hint → `–` skipped (dim). There is never a fake
  percentage.
- **Closing never cancels.** A dim line says
  `closing won't stop setup — it continues in Procs`.
- **Init hand-off.** When setup is on and the result names an init hand-off, the wizard
  drives the **existing** init flow for the new key: check → `InitPlanModal` on top →
  apply or the `t` terminal valve for sidecar prompts → re-check. Then the checklist row
  shows `✓ SASE initialized`.
- **Init cancelled.** Ready shows an amber
  `⚠ Not initialized — agents can't launch here yet · i initialize`.

```
│                       ✓  sase-forge is ready              (in its accent)    │
│            github.com/sase-org/sase-forge · private · ● enabled              │
│                 key gh_sase-org__sase-forge   ■                              │
│                                                                              │
│      +  launch an agent in +sase-forge        c  make it current             │
│      ⏎  show it in Projects                   esc  close                     │
```

On failure the context is kept and the next move is named:

```
│  ✓ Created github.com/sase-org/sase-forge                            1.1s    │
│  ✗ Clone failed — ssh: Permission denied (publickey); https: auth required   │
│      hint: run `gh auth setup-git`, then retry                               │
│  The repository exists on GitHub and was kept.                               │
│        r finish setup (resumes at checkout) · t terminal · b back · esc close│
```

- **`r`** re-plans (Review) and resumes.
- **`t`** runs the same CLI command interactively in a suspended terminal, then
  re-plans.
- **Returning to Projects** (`tui-entry`):
  1. Reload records, then `_refresh_launchable_mru_snapshot`,
     `_refresh_options(preferred_project=key)`, `_notify_lifecycle_changed()`, and an
     off-thread `ensure_project_tag_catalog()`, so `+tag` resolves.
  2. If the active filter would hide the new row, the status offers `Show new project`
     instead of clearing the filter.

## Architecture

```
┌─ sase TUI (presentation only) ──────────────────────────────────────────────┐
│ NewProjectModal + pure models (stage machine, source rows, form state,      │
│ review rows from plan JSON, NDJSON progress parser)                         │
│  workers: discovery / probe / local plan preview (read-only, in-process)    │
│  session worker: `sase project add|create … --check --json` (Review)        │
│  durable proc:   `sase project add|create … --yes --json` (apply)           │
└───────────────┬─────────────────────────────────────────────────────────────┘
                ▼
┌─ sase (Python): src/sase/project_onboarding/ + CLI handlers ────────────────┐
│ refs · discovery · facts · planning · apply · journal · manage · events     │
│ → sase_core_rs: classify_project_candidates · plan_project_onboarding ·     │
│                 evaluate_repo_creation_form          (pure, table-tested)   │
└───────────────┬─────────────────────────────────────────────────────────────┘
                ▼
┌─ workspace-provider plugins ────────────────────────────────────────────────┐
│ existing: ws_list_ref_namespaces · ws_list_repo_candidates · ws_peek_ref ·  │
│           ws_resolve_ref (apply only)                                       │
│ new:      ws_parse_repo_ref · ws_list_local_repo_candidates ·               │
│           ws_probe_repo · ws_describe_repo_creation · ws_create_repo        │
│ sase-github (gh)  ·  built-in bare git                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Provider hook contract (`provider-hooks` defines; `gh-provider` implements for GitHub)

These are all `firstresult=True` and additive. Each hook returns `None` unless the
plugin owns `workflow_type`. The dataclasses are frozen, live in
`src/sase/workspace_provider/_hookspec.py`, and are re-exported lazily from
`sase.workspace_provider`. Pin every argument name in `_FROZEN_HOOK_ARGUMENTS` the day
the hooks land.

```python
@dataclass(frozen=True)
class ParsedRepoRef:
    workflow_type: str
    ref: str                      # canonical ref ws_resolve_ref accepts ("sase-org/sase", "forge")
    identity_key: str             # stable identity: "github.com/sase-org/sase", "git:/abs/forge.git"
    display: str                  # "sase-org/sase"
    host: str = ""
    owner: str = ""
    name: str = ""
    project_key_base: str = ""    # "gh_sase-org__sase"; bare git: the name
    project_name_base: str = ""   # "sase"
    checkout_dir: str = ""        # where the checkout lives or will be cloned
    web_url: str = ""
    normalized_from: str = ""     # original spelling when the provider normalized the name

def ws_parse_repo_ref(self, workflow_type: str, text: str) -> ParsedRepoRef | None:
    """Pure: no filesystem, network, or subprocess; may read cached config."""

@dataclass(frozen=True)
class LocalRepoCandidate:
    parsed: ParsedRepoRef
    path: str
    origin_url: str = ""
    numbered_copy: bool = False   # e.g. ~/projects/github/o/r_2

@dataclass(frozen=True)
class LocalRepoCandidates:
    entries: tuple[LocalRepoCandidate, ...] = ()
    message: str = ""

def ws_list_local_repo_candidates(self, workflow_type: str) -> LocalRepoCandidates | None:
    """Filesystem only (read .git/config directly), no subprocess or network, bounded."""

RepoProbeStatus = Literal["found", "not_found", "unavailable"]

@dataclass(frozen=True)
class RepoProbe:
    status: RepoProbeStatus
    parsed: ParsedRepoRef
    visibility: str = ""
    description: str = ""
    default_branch: str = ""
    is_empty: bool | None = None
    is_archived: bool | None = None
    can_push: bool | None = None
    sase_managed: bool | None = None   # remote default branch's sase/sase.yml
    repo_id: str = ""
    web_url: str = ""
    error_kind: VcsRepoCandidateErrorKind | None = None
    message: str = ""
    remedy_argv: tuple[str, ...] = ()

def ws_probe_repo(self, workflow_type: str, ref: str) -> RepoProbe | None:
    """Read-only, bounded network (≤10 s), non-interactive env, stdin=DEVNULL.
    Never clones or writes. Never call it from a keystroke path."""

RepoCreationFieldKind = Literal["line", "text", "bool", "choice", "search"]

@dataclass(frozen=True)
class RepoCreationChoice:
    value: str
    label: str = ""
    description: str = ""
    disabled_reason: str = ""

@dataclass(frozen=True)
class RepoCreationField:
    name: str                     # option key for -O and ws_create_repo
    label: str
    kind: RepoCreationFieldKind   # choice = small fixed set; search = searchable, accepts typed values
    group: Literal["basics", "advanced"] = "basics"
    default: str = ""             # "true"/"false" for bool
    choices: tuple[RepoCreationChoice, ...] = ()
    required: bool = False
    description: str = ""
    placeholder: str = ""
    excludes: tuple[str, ...] = ()  # when this field is set, these fields are unavailable
    requires: tuple[str, ...] = ()  # unavailable until every named field is set
    disabled_reason: str = ""       # statically unavailable (e.g. team on a personal account)

@dataclass(frozen=True)
class RepoCreationForm:
    workflow_type: str
    namespace: str
    fields: tuple[RepoCreationField, ...]
    can_create: bool | None = None
    message: str = ""
    remedy_argv: tuple[str, ...] = ()

def ws_describe_repo_creation(self, workflow_type: str, namespace: str) -> RepoCreationForm | None:
    """Read-only. May use bounded, cached network for choice lists. Owner and name are
    not fields; they are the ref."""

RepoCreateStatus = Literal["created", "exists", "failed"]

@dataclass(frozen=True)
class RepoCreateResult:
    status: RepoCreateStatus
    parsed: ParsedRepoRef
    repo_id: str = ""
    web_url: str = ""
    visibility: str = ""
    message: str = ""
    mismatches: tuple[str, ...] = ()  # requested settings the created repo does not reflect

def ws_create_repo(self, workflow_type: str, ref: str, options: dict[str, str]) -> RepoCreateResult | None:
    """Create only the remote (or bare) repository: never clone, register, or write
    project records. Always pass an explicit visibility, never fall back to public, and
    verify after creating. Report `exists` without deciding adoption."""
```

These dataclasses also gain fields (all additive with defaults):

- `VcsRepoEntry`: `repo_id: str = ""`, `viewer_permission: str = ""`,
  `is_empty: bool | None = None`.
- `VcsRepoCandidates`: `host: str = ""`, `truncated: bool = False`,
  `remedy_argv: tuple[str, ...] = ()`.
- `VcsNamespaceEntry`: `host: str = ""`.
- `VcsRefNamespaces`: `account: str = ""`.

`fetch_repo_candidates` gains `host: str = ""`. The memo and disk cache key becomes
`(workflow_type, host, namespace)`. Payloads record `host` and `truncated`, and
`VCS_REPO_CACHE_SCHEMA_VERSION` goes to 2. Existing completion callers keep working
unchanged.

### Core functions (sase-core `crates/sase_core/src/project_onboarding/`)

All three functions are pure, check `schema_version` first, and share one
`PROJECT_ONBOARDING_WIRE_SCHEMA_VERSION = 1`, exposed through a
`project_onboarding_wire_schema_version` getter. Follow the `machine_setup/`
conventions:

- string-constant vocabularies;
- `#[serde(default)]` on request fields;
- thiserror errors with `check_schema`;
- `deny_unknown_fields` where the house style uses it.

Bind them in a new `sase_core_py/src/project_onboarding/` domain, registered from
`lib.rs`, with round-trip tests. On the Python side, add the facade
`src/sase/core/project_onboarding_facade.py`, which mirrors the schema constant, uses
`require_rust_binding("<literal>")`, and validates returned dicts. Do not add to the
retiring `lib.rs` re-exports or `prelude.rs` aliases.

**`classify_project_candidates(input) -> list`** (`core-classify`)

Input `ProjectCandidateInputWire`:

- `workflow_type`
- `sidecar_roles: [str]`: passed from Python's role constants plus `research` and `sdd`.
  Rust owns no role list.
- `owned_namespaces: [str]`: github_orgs ∪ account ∪ owners of existing projects.
- `namespaces: [{name, kind_label, status: ok|error|loading, error_kind?, message, stale, truncated, fetched_at?}]`
- `remote: [{namespace, identity_key, ref, display, name, description, visibility, is_fork, is_archived, is_empty?, pushed_at?, repo_id, viewer_permission}]`
- `local: [{identity_key, ref, display, owner, name, path, numbered_copy}]`
- `projects: [{key, name, aliases, state, identity_keys: [str]}]`
- `inventory: [{identity_key, kind: sidecar|linked|external, project_key, open_count, last_open_at?}]`
- `journals: [{identity_key, ref, display, status: running|failed|complete, stopped_at?, updated_at}]`

Output `ProjectCandidateListWire`:

- `groups: [{id: resume|local|suggested|linked|disabled, rows: [Row]}]`, in fixed order,
  with empty groups omitted.
- `search_only: [Row]`: hidden rows, each with `hidden_reason`.
- `hidden: [{reason, count}]`
- `namespaces`: passed through.

A `Row` carries
`identity_key, ref, display, group, hidden_reason?, description, visibility, pushed_at?, is_empty?, viewer_permission, local_path?, project_key?, project_state?, linked_by: [key], journal_status?, stopped_at?`.

Identity compares casefolded `identity_key`. Remote and local entries with the same
identity merge into one row: the local group wins and remote metadata is kept.
Classification takes the first rule that matches:

1. A `running` or `failed` journal → `resume`. Complete journals are ignored.
2. A project match:
   - enabled → hidden `enabled_project` (search-only, offers "show it");
   - disabled → `disabled`;
   - sibling → hidden `enabled_project`.
3. Sidecar → hidden `sidecar`, when either:
   - the inventory has a `sidecar` entry with this identity; or
   - the name is `<base>--<role>` with `role ∈ sidecar_roles`. A user repo merely
     containing `--` is **not** hidden.
4. Archived → hidden `archived`. Fork → hidden `fork`.
5. A local entry:
   - `numbered_copy` → hidden `numbered_copy`;
   - owner ∉ `owned_namespaces` → hidden `third_party_local`;
   - otherwise → `local`.
6. Linked in the inventory → `linked`. Any row also carries a `linked_by` chip.
7. Everything else → `suggested`.

Ranking within a group: `open_count` desc, then `last_open_at` desc, then `pushed_at`
desc, then `display` casefolded asc.

**`evaluate_repo_creation_form(form, values, mode) -> state`** (`core-classify`)

`form` mirrors `RepoCreationForm`. `values: {name: str}`. `mode: strict|lenient`.

Output:

- `fields: [{name, available, reason, value, is_set}]`
- `effective_options: {name: value}`
- `errors: [{field, code, message}]`
- `valid`

Semantics:

- **Set.** A bool is set when it is true. Any other kind is set when non-empty. Accepted
  bool spellings are `true/false/yes/no/on/off/1/0`, normalized to `true`/`false`.
- **Availability.** A field is unavailable when any of these hold, and its `reason`
  names the cause:
  - it has a static `disabled_reason`;
  - a set field lists it in `excludes` ("not available with Template");
  - one of its `requires` is unset ("requires Template").
- **Effective options.** `effective_options` holds every available field whose value is
  set or equals its default. Unavailable fields are always dropped.
- **Error codes.**
  - `required`: a required field is empty.
  - `unknown_option`: a key in `values` that is not in `form`.
  - `invalid_choice`: a `choice`-kind value not in `choices`. `search` kinds accept
    typed values.
  - `disabled_choice`: the chosen choice has a `disabled_reason`.
- **Strict mode** adds `conflict` and `requires` errors when the caller explicitly
  supplied a non-default value for an unavailable field.
- **Lenient mode** silently drops those values, and the field state carries the reason.

**`plan_project_onboarding(request, facts) -> plan`** (`core-plan`)

Request:

- `mode: add|create`
- `actor: human|agent`
- `target`:
  `{workflow_type, provider_display, identity_key, ref, display, host, owner, name, web_url, checkout_dir, project_key_base, project_name_base, storage_policy}`
- `requested`: `{name?, aliases: [str], setup: auto|on|off, make_current: bool}`
- `creation?`: `{visibility, summary: [str]}`

Facts:

- `remote`:
  `{status: found|not_found|unavailable, visibility, is_empty?, is_archived?, can_push?, sase_managed?, repo_id, default_branch, message}`
- `journal?`: `{intent_recorded_at, remote_created, repo_id}`
- `checkout`:
  `{state: absent|matching|mismatch|not_git|empty_dir, origin_url, config_dirty, on_default_branch?}`
- `local_managed?`
- `record?`: `{key, name, aliases, state}`
- `projects`: `[{key, name, aliases, state}]`
- `creation_errors`: `[{field, code, message}]`
- `can_create?`

Plan:

- `mode`, `status: ready|blocked|current`
- `intent_fingerprint`: sha256 over the canonical JSON of
  `{mode, target.identity_key, target.ref, requested, creation}`. It deliberately
  excludes step states, so a resume of the same intent keeps the same fingerprint.
- `identity`:
  `{key, name, aliases, key_source: existing|allocated, name_source: existing|requested|allocated}`
- `setup`: `{enabled, code, reason}`
- `steps`:
  `[{id, operation, label, detail, state: pending|done|skipped, remote, requires_tty, handoff}]`
- `warnings`: `[{code, message}]`
- `blockers`: `[{code, message, hint}]`

Steps always appear in this order. Every rule below is a table-test case.

| Step                                                                         | Pending when                                                                                           | Done/skipped when                                                                                                                                                                        | Blockers                                                                                                                                                                                                                                            |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `create_remote` (create only; `+`, remote)                                   | remote `not_found`                                                                                     | found **and** the journal shows this intent created it: either `repo_id` matches, or the journal has no id and the remote is empty with the requested visibility. The step is then done. | `remote_exists` when found otherwise (hint: `sase project add REF`). `remote_unknown` when unavailable. `human_only` when `actor=agent`. `no_create_rights` when `can_create=false`. `invalid_option` for each creation error.                      |
| `checkout` (`+` "Clone into … and register" / `~` "Register existing clone") | checkout `absent` or `empty_dir` (the engine removes an empty dir first), or `matching` with no record | `matching` with a record                                                                                                                                                                 | `checkout_conflict` (names the path) for `mismatch`/`not_git`. In add mode, remote `not_found` → `not_found`; remote `unavailable` with no matching checkout → `remote_unknown`. With a matching checkout it is only a `remote_unverified` warning. |
| `reenable` (`~`)                                                             | record `disabled`                                                                                      | otherwise skipped                                                                                                                                                                        | —                                                                                                                                                                                                                                                   |
| `identity` (`~`)                                                             | requested name/aliases differ from the record or from the allocation                                   | otherwise skipped                                                                                                                                                                        | `invalid_name`, `name_taken`, `alias_taken`, each with an allocation suggestion.                                                                                                                                                                    |
| `manage` (`+`/`~`, remote)                                                   | setup enabled and not managed. Managed means `remote.sase_managed` or `local_managed`.                 | managed, or setup disabled (skipped, with the reason)                                                                                                                                    | `config_dirty` (sase/sase.yml has local edits), `not_on_default_branch`.                                                                                                                                                                            |
| `current` (`~`)                                                              | `make_current`                                                                                         | otherwise skipped                                                                                                                                                                        | —                                                                                                                                                                                                                                                   |
| `init` (`●`, `handoff`, `requires_tty` when `storage_policy=separate_repo`)  | setup enabled                                                                                          | otherwise skipped                                                                                                                                                                        | —                                                                                                                                                                                                                                                   |

**Setup resolution for `auto`:**

- `actor=agent` → off, code `agent`.
- `create` → on, code `new_repo`.
- Managed → off, code `already_managed`. Sidecars connect on first launch, so `init` is
  skipped.
- `can_push=true` → on, code `unmanaged_can_push`.
- `can_push=false` → off, code `no_push`, with warning `not_launchable` naming
  `sase init -M`.
- `can_push` unknown → off, code `unknown_push`, with a warning.
- `in_tree` providers (bare git) follow the same rules. Their probe reports
  `can_push=true` for local bare repos.

**Explicit setup:**

- `on` from an agent → blocker `human_only`.
- `on` with `can_push=false` → blocker `no_push`.
- `on` on an already managed repo → `manage` done, `init` pending.
- `off` on an unmanaged `separate_repo` target → warning `not_launchable`.

**Identity preview:**

- **Key.**
  - An existing record keeps its key.
  - Otherwise the key is `project_key_base` when it is free. Free means it doesn't
    casefold-match any key, name or alias, or `home`. If it is taken, try `-2`, `-3`, …
    (sase-github's rule).
  - Bare git (`workflow_type=git`) never gets a suffix: a collision is `name_taken`,
    because `init_bare_git_project` refuses claimed names.
- **Name.** The requested name, else the record's name, else `project_name_base`. If
  taken, try `_1`, `_2`, … (`allocate_project_name`'s rule). The target's own record is
  excluded from the collision checks.
- **Authority.** The provider stays authoritative at apply time. The result event
  reports the actual key.

**Status:**

- `blocked` if there are any blockers.
- `current` if no pending steps remain except `handoff` steps.
- Otherwise `ready`.

### The onboarding engine (`src/sase/project_onboarding/`)

The engine is split into modules by concern. Each module stays under the repo's toobig
limit.

| Module                                              | Owns                                                                                                                                                                                                                                                                                                                                                                                                                                           | Phase                               |
| --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| `refs.py`                                           | `parse_user_ref(text, *, workflow_type=None)`. A `gh:`/`git:` prefix selects the provider by metadata `workflow_type`. Otherwise every provider's `ws_parse_repo_ref` is tried and exactly one must match; the errors are `ambiguous — prefix with gh: or git:` and `not recognized`.                                                                                                                                                          | cli-suggest                         |
| `discovery.py`                                      | `load_candidate_inputs(workflow_type, *, cache_only, refresh, namespaces=None)` and `suggest(...)`. Namespaces and `account`; per-namespace `fetch_repo_candidates` (bounded parallelism); local candidates; record identities from each checkout's `.git/config` origin via `ws_parse_repo_ref`, falling back to parsing `gh_<owner>__<repo>` keys; inventory from `collect_repo_inventory(include_disabled=True)`; journals → core classify. | cli-suggest                         |
| `preferences.py`                                    | `$SASE_HOME/project_onboarding/preferences.json`: last provider, last owner per provider, last visibility per owner. Atomic writes, tolerant reads.                                                                                                                                                                                                                                                                                            | cli-suggest                         |
| `facts.py`                                          | `gather_facts(target, mode, *, actor)`: probe, checkout state (origin read from `.git/config`, `git status --porcelain sase/sase.yml` with a timeout), local managed marker, records, journal, creation errors.                                                                                                                                                                                                                                | cli-add                             |
| `planning.py`                                       | `build_plan_document(...)`: core plan plus provider display, accent, `apply_argv`, `next_argv`, and target summary, giving the plan document below.                                                                                                                                                                                                                                                                                            | cli-add                             |
| `journal.py`                                        | Ownership journals at `$SASE_HOME/project_onboarding/journals/<slug>.json`. The slug is a readable display stem plus `sha1(identity_key)[:12]`. Atomic writes; `list_incomplete()`; prune complete journals older than 30 days on write.                                                                                                                                                                                                       | cli-add                             |
| `apply.py`                                          | `apply_plan(...)`: re-gathers facts and re-plans first (converge). Refuses a `blocked` plan. Runs pending non-handoff steps in order, emits events, updates the journal, and stops at the first failure.                                                                                                                                                                                                                                       | cli-add                             |
| `manage.py`                                         | In the checkout: `enable_sase_management(config_path)`, then `git add sase/sase.yml`, commit `chore: mark repository as SASE-managed`, and push. Bounded timeouts and a non-interactive env. Push rejection fails the step with a hint (protected branch → open a PR or run `sase init -M` yourself).                                                                                                                                          | cli-add                             |
| `events.py`                                         | NDJSON event dataclasses: `emit_event()` plus a pure `parse_event_line()` the TUI reuses.                                                                                                                                                                                                                                                                                                                                                      | cli-add                             |
| `render.py`                                         | Rich renderers for suggest, plan, and progress.                                                                                                                                                                                                                                                                                                                                                                                                | cli-suggest, cli-add                |
| `human_only.py` (or `src/sase/agent/human_only.py`) | Actor detection. An agent is `SASE_AGENT`, `SASE_AGENT_NAME`, a discovered agent identity, or `SASE_MONITOR_ID`. It counts as human when `SASE_GATE_COMMAND=1` (an approved `/sase_gate` option) or when `SASE_PROJECT_CREATE_BYPASS` holds a non-empty reason (print a one-line note naming it).                                                                                                                                              | cli-add (actor), cli-create (guard) |

**How each step executes in `apply.py`:**

- **`create_remote`:**
  1. Write the journal intent (target, fingerprint, timestamp) **before** calling
     `ws_create_repo`.
  2. Record `repo_id` and `remote_created=true` on `created`.
  3. On `exists`, re-plan. The core rules decide adopt vs. blocker.
- **`checkout`:** `ws_resolve_ref(parsed.ref, workflow_type)` after an origin re-check.
  Clone timeouts come from `gh-hygiene`.
- **`reenable`:** `set_project_state_locked(key, "enabled")`.
- **`identity`:** the locked name and alias helpers in `sase.project_aliases`. Use
  `apply_project_name_update` under the spec lock for an explicit rename.
- **`manage`:** `manage.py`.
- **`current`:** the existing `set-current` implementation.
- **`init`:** never executed by `apply`. It appears in the result as
  `next_argv = ["sase", "init", "-p", KEY]`.

### CLI contract

The CLI follows `cli_rules`: sorted subcommands and options, a short alias for every
public long option, and colored output. Update the hard-coded `metavar` in
`parser_project.py` and the usage string in `project_handler.py`. Bare `sase project`
still delegates to `list`.

```
sase project suggest [PROVIDER] [-a] [-j] [-o OWNER]... [-r]
sase project add REF [-a ALIAS]... [-c] [-j] [-m] [-n NAME] [-S | -s] [-y]
sase project create REF [-a ALIAS]... [-c] [-d TEXT] [-j] [-m] [-n NAME]
                        [-O KEY=VALUE]... [-S | -s] [-v VISIBILITY] [-y]
```

- **`add REF`** accepts `owner/repo`, `gh:owner/repo`, `https://host/owner/repo[.git]`,
  `git@host:owner/repo.git`, `git:NAME`, or a path. It resolves through `refs.py`.
- **`create REF`** accepts `gh:OWNER/NAME`, `OWNER/NAME` (GitHub shorthand), or
  `git:NAME`.
- **Options:**

  | Option                                | Meaning                                                                                   |
  | ------------------------------------- | ----------------------------------------------------------------------------------------- |
  | `-a/--alias`                          | Repeatable. Set the project's aliases.                                                    |
  | `-c/--check`                          | Read-only plan. Exit 0 when ready or current, 1 when blocked.                             |
  | `-j/--json`                           | With `--check`, one plan document. With `--yes`, NDJSON events.                           |
  | `-m/--make-current`                   | Make the new project current. Off by default.                                             |
  | `-n/--name`                           | The project's display name.                                                               |
  | `-s/--setup`, `-S/--no-setup`         | Turn setup on or off. The default is auto.                                                |
  | `-y/--yes`                            | Apply without asking.                                                                     |
  | `-d/--description`, `-v/--visibility` | `create` only. Sugar for `-O description=…` and `-O visibility=…`.                        |
  | `-O/--option KEY=VALUE`               | `create` only. Validated strictly by `evaluate_repo_creation_form` before any subprocess. |
  | `suggest -a/--all`                    | Include hidden rows.                                                                      |
  | `suggest -o/--owner`                  | Repeatable. Restrict to the named owners.                                                 |
  | `suggest -r/--refresh`                | Bypass the cache.                                                                         |

  `create --check` without `--json` lists every provider option with its default,
  current value, and availability.

- **Confirmation:**
  - **No `--yes` on a TTY:** render the plan and ask `Apply? [y/N]`.
  - **No `--yes` without a TTY:** render the plan and exit 1 with `re-run with --yes`.
  - **`--yes`:** apply directly.
- **After a successful apply with setup on:**
  - **On a TTY without `--json`:** offer to run `sase init -p KEY` interactively now,
    defaulting to yes. Sidecar prompts happen there.
  - **Otherwise:** print the exact next command.
- **Durable-operation IO.** `add` and `create` accept the durable-operation IO flags
  (`add_operation_io_flags`: `-Q/--request-path`, `-R/--result-path`) and write a
  `DurableOperationResult` whose payload is the `result` event. Register `project.add`
  and `project.create` in `src/sase/ops/names.py` and wherever
  `coerce_operation_request` resolves request models.
- **Exit codes:**
  - 0: applied, or already current.
  - 1: blocked, failed, or partial.
  - 2: usage error, or the human-only refusal.

**The plan document** (`--check --json`):

```json
{
  "schema_version": 1,
  "kind": "project_onboarding_plan",
  "mode": "add",
  "status": "ready",
  "intent_fingerprint": "…",
  "provider": { "workflow_type": "gh", "display": "GitHub" },
  "target": {
    "ref": "bobs-org/bob",
    "display": "bobs-org/bob",
    "identity_key": "github.com/bobs-org/bob",
    "web_url": "…",
    "visibility": "private",
    "default_branch": "main",
    "checkout_dir": "…",
    "can_push": true,
    "sase_managed": false
  },
  "project": {
    "key": "gh_bobs-org__bob",
    "name": "bob",
    "aliases": [],
    "accent": "#…",
    "state": "enabled",
    "existing": false
  },
  "setup": { "enabled": true, "code": "unmanaged_can_push", "reason": "…" },
  "steps": [
    {
      "id": "checkout",
      "operation": "create",
      "label": "…",
      "detail": "…",
      "state": "pending",
      "remote": false,
      "requires_tty": false,
      "handoff": false
    }
  ],
  "warnings": [],
  "blockers": [],
  "apply_argv": ["sase", "project", "add", "bobs-org/bob", "--yes", "--json"],
  "next_argv": ["sase", "init", "-p", "gh_bobs-org__bob"]
}
```

**NDJSON events** (`--yes --json`, one object per line on stdout; human diagnostics go
to stderr):

```
{"event":"plan","plan":{…}}
{"event":"step","id":"checkout","status":"running","label":"…","at":"2026-10-11T14:00:00Z"}
{"event":"step","id":"checkout","status":"done","elapsed_ms":1834,"message":"…"}
{"event":"step","id":"manage","status":"failed","elapsed_ms":402,"message":"…","hint":"…"}
{"event":"result","status":"ok|partial|failed|blocked","project_key":"…","project_name":"…","next_argv":[…],"message":"…"}
```

**The journal:**

```
{"schema_version": 1, "identity_key", "ref", "display", "workflow_type", "mode",
 "intent_fingerprint", "status": "running|failed|complete", "proc_id"?,
 "intent_recorded_at", "remote": {"created": bool, "repo_id", "created_at"?},
 "steps": {"<id>": {"status", "finished_at", "message"}}, "project_key"?,
 "stopped_at"?, "updated_at"}
```

### Execution, concurrency, and recovery

- **Concurrency keys.** The TUI submits apply with
  `concurrency_keys=[f"project-onboard:{identity_key}", f"checkout:{checkout_dir}"]` and
  `request_fingerprint=intent_fingerprint`. A double press, two TUIs, or a TUI racing
  the CLI reattach or conflict instead of duplicating effects.
- **Read-only work stays in workers.** Discovery, probes and local plan previews run in
  cancellable `run_worker(thread=True)` workers. Only apply is durable: a modal-owned
  worker would die with its DOM node.
- **Hygiene for every subprocess.** Each one runs with `GH_PROMPT_DISABLED=1`,
  `GIT_TERMINAL_PROMPT=0`, `stdin=DEVNULL`, and a bounded timeout. Before apply, the
  selected identity is re-validated by re-planning.

| Situation                                    | Behavior                                                                                                                                                               |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gh` missing / not signed in                 | The tab is disabled, or a card offers `t` (`remedy_argv`), then refreshes.                                                                                             |
| One owner fails (403/429/network)            | Its stale rows stay with a badge; other owners are unaffected.                                                                                                         |
| No create rights / org policy                | The owner shows `⚠ can't create here`; Review is blocked. Visibility is never flipped to "fix" it.                                                                     |
| Name taken                                   | "⏎ add it instead" (2a). Intent is never silently converted.                                                                                                           |
| Remote created, then a later step fails      | "Repository created; setup incomplete". `r` reruns the same argv; core adopts it through the journal and resumes.                                                      |
| Lost response or killed runner during create | The outcome is unknown. A re-probe adopts only on journal evidence (repo id, or empty + requested visibility); anything else is `remote_exists` with "add it instead". |
| Checkout dir exists                          | A matching origin is adopted. Anything else is a blocker naming the path. A dirty matching clone is never reset.                                                       |
| Repo is already a disabled project           | `reenable` step. Never a duplicate record.                                                                                                                             |
| TUI closed mid-run                           | The proc finishes. The next `n` shows a RESUME row from the journal.                                                                                                   |
| Init or sidecar step fails or is cancelled   | The project stays registered and enabled; only init is retried (`i` on Ready, or `sase init -p KEY`).                                                                  |

Remote repositories are **never** deleted automatically.

## Phases

### gh-hygiene

Open sase-github with `sase repo open sase-github` and read its `CLAUDE.md`. Run checks
there with `sase tool run check`.

- **Clone timeout.** `remotes.clone_gh_repo` runs `git clone` without a timeout. Add one
  derived from core's network/git timeout helper, sized for a full clone (configurable,
  generous default). Keep the partial-target cleanup, and on timeout raise the same
  remediation error style.
- **SASE_HOME.** `workspace/projects.py:projects_base()` hard-codes `~/.sase/projects`.
  Use core `sase.core.paths.sase_projects_dir()`, which honors `SASE_HOME`. Update the
  tests' `_home_patches` accordingly.
- **Host detection.** `ws_detect_workflow_type` returns `gh` for any http/ssh origin.
  Return `None` when the origin's host is not one of `get_github_hosts()`. Keep today's
  behavior for no-origin and read-error cases.
- **Docs.** `github_orgs` does **not** choose SSH vs HTTPS; cloning always tries SSH,
  then HTTPS. It seeds owner completion and gates PR submission. Fix
  `docs/configuration.md`, `docs/architecture.md`, `README.md`, and core's
  `sase.schema.json` description of `github_orgs` (in the sase repo).
- **Tests:** timeout argv/kwargs, `SASE_HOME`-relative projects dir, and the non-GitHub
  host → `None` case.

### provider-hooks

- **Hookspecs.** Add the five hookspecs and dataclasses
  ([contract](#provider-hook-contract-provider-hooks-defines-gh-provider-implements-for-github))
  to `_hookspec.py`. Add matching `WorkspacePluginManager` methods (`parse_repo_ref`,
  `list_local_repo_candidates`, `probe_repo`, `describe_repo_creation`, `create_repo`)
  with identical parameter names.
- **Registry wrappers** in `_registry.py`:
  - `parse_repo_ref`/`probe_repo`/`describe_repo_creation`/`list_local_repo_candidates`
    swallow plugin exceptions into `None` or an `unavailable` result, with a logged
    warning.
  - `create_repo` propagates errors as `RepoCreateResult(status="failed")`.
- **Exports and pins.** Add lazy exports in `__init__.py`, and pin every argument name
  in `tests/test_workspace_provider_hookspec.py::_FROZEN_HOOK_ARGUMENTS`.
- **Field extensions.** Extend `VcsRepoEntry`, `VcsRepoCandidates`, `VcsNamespaceEntry`
  and `VcsRefNamespaces` as specified. Update `vcs_repo_completion.py` with the
  host-aware cache key, `truncated`, `host`, and the schema bump. Existing completion
  callers and tests must stay green.
- **Bare-git implementations** in `src/sase/workspace_provider/plugins/`:
  - **`ws_parse_repo_ref`:** `git:NAME`, `NAME` (valid project names only), and
    `~`/absolute paths. Identity is `git:<absolute bare path>`, defaulting to
    `sase_subdir("repos")/<name>.git`. Checkout is `~/projects/git/<name>/`. Key and
    name base are the name.
  - **`ws_list_local_repo_candidates`:** `sase_subdir("repos")/*.git` and
    `~/projects/git/*/`, reading origins from `.git/config` (or the bare path itself),
    without subprocesses.
  - **`ws_probe_repo`:** local stat and read only. Report `found` when the bare repo
    exists, `is_empty` from refs, `can_push=True`, visibility `local`, and
    `sase_managed` from the default branch's `sase/sase.yml` (read via a timed
    `git show`).
  - **`ws_describe_repo_creation`:** Advanced `bare_dir` and `checkout_dir` (line
    fields, with placeholders showing the defaults).
  - **`ws_create_repo`:** `git init --bare` at the bare path, idempotent for an existing
    empty bare repo, `exists` for one with refs. It never clones; registration happens
    later through `ws_resolve_ref`, whose fresh-init path seeds SDD, commits and pushes.
    Confirm that path is taken for an empty pre-created bare repo and adjust
    `bare_git_init.py` if not.
- **Docs.** Update the hook table in `docs/workspace.md` and the workspace-plugin
  section of `docs/plugins.md`.
- **Symvision.** New public symbols without a non-test consumer yet get `--epic-symbol`
  whitelists, per `symvision.md`, naming the consuming phase.
- **Tests:** the hookspec pins; the manager's 1:1 mapping; every bare-git hook,
  including "parse is pure" (patch `subprocess.run` and `Path.exists` to raise) and
  "local candidates spawn no subprocess"; the cache key and schema migration.

### core-classify

Open sase-core with `sase repo open sase-core` and read its `AGENTS.md`.

- **Module.** Create `crates/sase_core/src/project_onboarding/` (`mod.rs`, `wire.rs`,
  `classify.rs`, `form.rs`, `tests.rs`) implementing
  [`classify_project_candidates` and `evaluate_repo_creation_form`](#core-functions-sase-core-cratessase_coresrcproject_onboarding)
  exactly.
- **Bindings.** Add a `project_onboarding` binding domain in `sase_core_py` with the
  schema getter and round-trip tests.
- **Table tests:**
  - a sidecar by inventory, and a sidecar by the `--<role>` suffix;
  - a user repo with `--` that is not a role;
  - fork, archived, numbered copy, third-party local;
  - disabled and enabled projects, linked, resume journals;
  - casefolding and remote+local merge;
  - ranking ties.
  - For the form: each verified `gh repo create` constraint (template excludes
    license/gitignore/readme/team; include-all-branches requires template), unknown
    option, required, disabled choice, and strict vs. lenient.
- **Python side.** Add `src/sase/core/project_onboarding_facade.py`
  (`classify_project_candidates`, `evaluate_repo_creation_form`) following
  `src/sase/core/git_object_sharing.py`. Add the binding names to
  `tools/validate_sase_core_rs` `REQUIRED_BINDINGS` if that list governs new bindings,
  and add facade tests.
- **Pin.** Move `sase-core-revision.txt` past the core commit, per
  `docs/rust_backend.md`. Don't edit the `sase-core-rs` version window.

### core-plan

- **Plan function.** In the same module, add `plan.rs` implementing
  [`plan_project_onboarding`](#core-functions-sase-core-cratessase_coresrcproject_onboarding):
  setup resolution, identity preview, the step table, status, and the intent
  fingerprint. Keep any collision helpers it reuses from `project_spec.rs` `pub(crate)`,
  without growing that file.
- **Table tests:** every row of the step table and every setup-resolution branch,
  including:
  - resume convergence: each partial state resumes at the right step;
  - a found remote with no ownership evidence → `remote_exists`, never adoption;
  - the agent actor → `human_only` for create and explicit setup;
  - allocation suffixes for gh (`-2`) and bare git (`name_taken`);
  - fingerprint stability across step states.
- **Binding, facade, pin.** Add the binding, the facade function, facade tests, and the
  pin bump.

### gh-provider

Work in sase-github and verify against the installed `gh` (2.100 on athena). Never
create live repositories in tests; fake `gh` through the existing `subprocess.run`
patching style.

- **`ws_parse_repo_ref`** accepts:
  - `owner/repo` and `gh:owner/repo`;
  - `https://<host>/owner/repo[.git]` for `get_github_hosts()` only;
  - `git@<host>:owner/repo.git` and `ssh://git@<host>[:port]/owner/repo.git`.

  It produces:
  - identity `<host>/<owner>/<repo>`, casefolded;
  - key base `gh_<owner>__<repo>`;
  - checkout `github_workspace_dir(owner, repo, host)`;
  - normalized names, with characters outside `[A-Za-z0-9._-]` becoming `-`, matching
    GitHub, and `normalized_from` set.

  It must stay pure.

- **`ws_list_local_repo_candidates`:** walk `~/projects/github/<owner>/<repo>/` (and
  host-prefixed dirs) to a bounded depth. Read `origin` from `.git/config`, keep entries
  whose origin identity matches the directory, and mark `*_\d+` siblings
  `numbered_copy`.
- **`ws_probe_repo`:**
  - `gh repo view OWNER/NAME --json id,name,owner,visibility,isEmpty,isArchived,viewerPermission,defaultBranchRef,url,description`;
  - plus a raw read of `sase/sase.yml` on the default branch
    (`gh api -H "Accept: application/vnd.github.raw" repos/O/R/contents/sase/sase.yml`;
    404 → `False`, other errors → `None`);
  - `can_push` = `viewerPermission ∈ {ADMIN, MAINTAIN, WRITE}`;
  - error kinds via the `gh_cli.py` classifiers;
  - auth `remedy_argv = ("gh", "auth", "login", "-h", host)`.
- **`ws_list_ref_namespaces`:** add `account` from gh's local `hosts.yml`, with no
  subprocess, and include it as a namespace (`kind_label="you"`) when absent.
- **`ws_list_repo_candidates`:** add `id,viewerPermission,isEmpty` to `--json`. Set
  `host`, set `truncated` when the row count reaches the limit, and set the auth remedy.
  Confirm the extra fields don't slow `gh repo list` materially.
- **`ws_describe_repo_creation(namespace)`:**

  | Group    | Field                  | Kind                              | Default / rule                                                     |
  | -------- | ---------------------- | --------------------------------- | ------------------------------------------------------------------ |
  | basics   | `description`          | line                              |                                                                    |
  | basics   | `visibility`           | choice: public, private, internal | private. `internal` is disabled unless the org's plan supports it. |
  | basics   | `add_readme`           | bool                              | true                                                               |
  | basics   | `gitignore`            | search                            | `gh repo gitignore list`                                           |
  | basics   | `license`              | search                            | `gh repo license list`                                             |
  | advanced | `template`             | line, `OWNER/REPO`                | `excludes = [add_readme, gitignore, license, team]`                |
  | advanced | `include_all_branches` | bool                              | `requires = [template]`                                            |
  | advanced | `homepage`             | line                              |                                                                    |
  | advanced | `team`                 | search                            | `gh api orgs/{o}/teams`; disabled for personal accounts            |
  | advanced | `issues`               | bool                              | true; false → `--disable-issues`                                   |
  | advanced | `wiki`                 | bool                              | true; false → `--disable-wiki`                                     |
  - Cache the choice lists for 24 h under the vcs cache dir.
  - `can_create` is best-effort: true for `account`; for orgs, from the membership or
    org API, else `None`.
  - Excluded on purpose: `--clone` (SASE owns the checkout) and
    `--source/--push/--remote` (publishing a local dir is out of scope).
  - Never parse `gh --help`.

- **`ws_create_repo`:**
  - Run
    `gh repo create OWNER/NAME --<visibility> [-d] [-h] [--add-readme] [-g] [-l] [-p [--include-all-branches]] [-t] [--disable-issues] [--disable-wiki]`.
  - Always pass an explicit visibility. Use `non_interactive_gh_env()`, an explicit
    `GH_HOST`, `stdin=DEVNULL`, and a timeout.
  - On an "already exists" error, re-probe and return `exists`.
  - After creating, verify with `gh repo view --json id,visibility,…` and report
    `mismatches`. Never drop a setting silently, and never fall back to public.
- **Tests:**
  - argv-exact create, with conflicts rejected before any subprocess (via core
    `evaluate_repo_creation_form` strict, or a local mirror test of the same rules);
  - the error-kind table;
  - parse purity;
  - local candidates never spawn subprocesses;
  - probe never writes;
  - account parsing;
  - truncation.

  Update the README and configuration docs.

- **Version floor.** If sase-github now imports new core symbols, raise its `sase` floor
  to a version that ships them, matching repo conventions.

### cli-suggest

- **Package.** Create `src/sase/project_onboarding/` with `refs.py`, `discovery.py`,
  `preferences.py`, and `render.py` (suggest part), with `__init__.py` exporting lazily.
- **Command.** Add `sase project suggest [PROVIDER] [-a] [-j] [-o OWNER]... [-r]`
  (parser in `parser_project.py`, handler in a new `project_handler_onboarding.py`).
- **Human output.**
  - Grouped sections use the TUI's glyphs and colors; the provider header carries a
    health line.
  - Owner errors show their remedy.
  - The hidden-count footer reads e.g. `┄ 26 sidecars · 24 forks hidden — use --all`.
  - Truncation is noted.
  - A closing hint shows how to add (`sase project add REF`).
- **JSON:**
  `{schema_version: 1, providers: [{workflow_type, display, account, owners: [...], groups: [...], hidden: [...], search_only?: [...] (with --all)}]}`.
- **Agents may call `suggest`.**
- **Docs.** `docs/cli.md` project table and prose, and `docs/project_spec.md` (how
  projects come into existence).
- **Skill.** Update `src/sase/macros/skills/sase_project.md` with `suggest`. Do not
  deploy skills; that happens from a landed revision.
- **Completion and tests.** Run `just sync-completion-spec`. Add parser, sort-order, and
  help tests.
- **More tests:**
  - `refs.parse_user_ref`: prefix, ambiguity, unknown;
  - discovery with fake providers: cache-only first pass, then refresh; one failing
    owner keeps stale rows; local merge; inventory sidecar hiding; journals → resume;
  - preferences round-trip;
  - golden JSON;
  - a perf guard: `cache_only=True` spawns no subprocess.

### cli-add

Read the [prerequisite](#prerequisite-outside-this-epic) first.

- **Engine.** Add `facts.py`, `planning.py`, `journal.py`, `apply.py`, `manage.py`,
  `events.py`, the actor part of `human_only.py`, and plan/progress rendering in
  `render.py`, all per
  [the engine table](#the-onboarding-engine-srcsaseproject_onboarding).
- **Command.** Add
  `sase project add REF [-a ALIAS]... [-c] [-j] [-m] [-n NAME] [-S | -s] [-y]` with the
  durable-operation IO flags, the TTY confirm, init chaining, and exit codes per the
  [CLI contract](#cli-contract). Register the `project.add` operation.
- **Human output.**
  - The plan uses init's glyphs: a header naming the provider and target, then
    PROJECT/REMOTE lines, the step rows, warnings and blockers, and `Would run: …`.
  - Progress is a live checklist with elapsed times.
  - Ready shows a summary line in the project's accent.
- **Docs:**
  - `docs/cli.md`;
  - `docs/project_spec.md` (creation paths, enabled by default, identity allocation);
  - `docs/workspace.md:13-15` and the bare-git auto-init prose (`#` refs are no longer
    the only way to create a project);
  - `docs/getting_started.md` (mention `sase project add`);
  - `docs/init.md` (onboarding marks a repo managed before init).
- **Skill.** `sase_project.md` gains `add`: run `--check` first; agents never get the
  setup step; propose a human-run setup through `/sase_gate` when needed.

> [!decision] glossary_project
>
> Update the `glossary:sase-project` strand. It currently says a project "is created
> only when a new VCS macro argument resolves". It should say projects are also created
> by `sase project add`/`create` and the Admin Center New Project wizard, which never
> write `PROJECT_STATE`. Republish with `sase memory init`.

- **Tests:**
  - golden `--check --json` per scenario: fresh gh repo, existing clone, disabled
    project, already current, checkout conflict, no push rights, agent actor;
  - the NDJSON event sequence;
  - injected failure at each step, after which a rerun converges;
  - double submission (two applies with the same identity) produces no duplicate
    effects;
  - `manage.py` against a local bare remote: commit, push, and push rejection;
  - the non-TTY refusal without `--yes`;
  - the TTY confirm;
  - the operation result file.
- **Smoke.** Use a fake `gh` on `PATH` for end-to-end CLI tests, and run a read-only
  `sase project add <real repo> --check` smoke against the real `gh`.

### cli-create

- **Command.** Add
  `sase project create REF [-a ALIAS]... [-c] [-d TEXT] [-j] [-m] [-n NAME] [-O KEY=VALUE]... [-S | -s] [-v VISIBILITY] [-y]`
  and register `project.create`.
- **Validation before any subprocess.** Fetch the provider form
  (`ws_describe_repo_creation`). Map `-d`/`-v` into options, then run core
  `evaluate_repo_creation_form` in strict mode. Its errors become plan blockers
  (`invalid_option`).
- **`create --check`** works for agents and humans. Its human output lists every
  provider option with default, value and availability.
- **Guard.** Apply refuses inside agents and monitors (exit 2) with a message naming
  `/sase_gate` and `SASE_PROJECT_CREATE_BYPASS`. An approved gate option
  (`SASE_GATE_COMMAND=1`) and a non-empty bypass reason are allowed. The bypass prints a
  one-line note naming the reason.
- **Journaled creation.** The journal intent is written before `ws_create_repo`;
  adoption follows core rules on rerun.
- **Docs and skill.** Update `docs/cli.md`. `sase_project.md` gains `create`:
  human-only; agents propose the exact `sase project create … --yes` command through
  `/sase_gate`.

> [!decision] human_only_record
>
> Add the `decisions:project-create-human-only` strand in the decisions-web format
> (frontmatter `keyword`, `aliases`, `summary`, `metadata.status: accepted`,
> `metadata.decided`). It records:
>
> - **Claim:** `sase project create` refuses to apply inside SASE agents and monitors
>   unless it runs as an approved `/sase_gate` option or `SASE_PROJECT_CREATE_BYPASS`
>   holds a reason. `--check` stays available to agents. `sase project add` stays
>   agent-callable but never runs its commit-and-push setup step from an agent.
> - **Why:** minting a repository and pushing to a default branch are outward-facing.
>   Visibility can't be un-leaked, and deletion needs a scope most tokens lack. This
>   keeps parity with `--yes` being unable to approve sidecar creation.
> - **Cost:** agents can't self-serve new repos, and the guard is a guardrail, not a
>   security boundary.
> - **Reopens when:** SASE gains a host-verified typed approval for resource creation.
>
> Link [[decisions/global-install-is-human-only]], then republish with
> `sase memory init`.

- **Tests:**
  - strict validation goldens for each gh constraint;
  - the agent-env refusal, gate-option allowance, and bypass note;
  - journal intent written before the hook call;
  - a lost response (the hook raises after the remote exists) → rerun adopts only with
    evidence;
  - `exists` without evidence → `remote_exists` blocker;
  - bare-git create end to end with a temp `SASE_HOME`.

### tui-source

- **Modules** (Textual-free models first, following the `projects_pane_init.py`
  precedent):
  - `src/sase/ace/tui/modals/new_project_state.py`: the stage machine
    (Source/Configure/Review/Progress/Ready/Failed). Drafts are preserved on back
    navigation.
  - `new_project_source_model.py`: rows from the classification output, in-memory
    filtering (including search-only rows), the synthetic paste row, the hidden footer,
    health chip text, resume rows, and `peek` vs. revalidated merge.
  - `new_project_rendering.py`: pure Rich renderers for the stepper, rows, footer,
    detail card, and health chip.
  - `new_project_modal.py`: the `ModalScreen` shell (frame, title, stepper,
    `PanelTabStrip`, a `ContentSwitcher` per stage, the hints line).
  - `new_project_source.py`: the Source stage widget and its workers.
- **CSS.** Add a `NewProjectModal` block in `styles.tcss`.
- **Workers:**
  - **Discovery:** `discovery.load_candidate_inputs(cache_only=True)` for first paint,
    then per-owner refresh workers with ≤ 3 in flight. Each one marshals results back
    and re-runs core classification (fast).
  - **Probe:** debounced through `DetailPanelDebouncer`, cached per ref.
  - **`t` remedy:** runs in `app.suspend()`, then refreshes.

  Re-capture the selection after every await (`tui_perf` rule 4). Cancel workers and
  pump-free tasks at teardown.

- **Not reachable yet.** No key opens the wizard in this phase. Tests push it directly,
  like the init-plan visual tests. Selecting a row advances to a Configure placeholder
  that `tui-add` replaces. Whitelist not-yet-consumed public symbols with
  `--epic-symbol`.
- **Tests:**
  - pure-model tests: grouping, filter, paste row, footer counts, resume rows, health
    chips;
  - a perf guard: typing in the filter and moving the highlight spawn no subprocess and
    call no provider hook except the pure parse (patch `subprocess.run` and the probe to
    raise);
  - the highlight paints immediately while the probe is debounced;
  - a bracketed paste focuses the filter.
- **PNG goldens** (extending the config-center projects visual helpers), each at
  `120x40`, plus `80x24` for the narrow case:
  - loading (fetching rows);
  - populated (all groups, footer);
  - owner error badge;
  - empty / no owners;
  - narrow.

  Run `just fix-tui-screenshots` with selectors and inspect every golden.

### tui-add

- **Configure (existing).** The Project card and the SASE setup card (2a). Local
  identity validation runs through `planning` with local-only facts in a debounced
  worker.
- **Review.**
  - A session worker runs `sase project add … --check --json` (`sase_argv()`, home cwd,
    bounded timeout, return codes 0/1 accepted).
  - A pure parser turns the plan document into review rows; reuse
    `init_plan_modal_rendering`'s operation glyph table instead of copying it.
  - Blockers disable the confirm.
- **Apply.**
  - Submit
    `_submit_durable_proc(apply_argv, operation="project.add", request=…, request_fingerprint=intent_fingerprint, concurrency_keys=[…], cwd=home)`.
  - Register the new submit sites in `_proc_producer_sites_*.py`.
  - Route `on_complete` through an owner that outlives the modal (the pane or app), so
    closing the wizard mid-run still reloads Projects.
- **Progress.**
  - While Progress is visible, a 250 ms interval tails the durable proc's log off the
    pump thread, gated on `log.version`.
  - Lines go through `events.parse_event_line`; non-JSON lines feed a collapsible raw
    log.
  - The checklist renders in `new_project_progress.py`.
- **Init hand-off.** Factor the pane's init check, `InitPlanModal`, apply, and terminal
  flow into a reusable `projects_pane_init_flow.py` controller. Both
  `ProjectsPaneInitActionsMixin` and the wizard use it. Pane behavior and its existing
  tests stay unchanged. The wizard drives it for `InitScope.for_projects([key])` and
  updates its checklist on completion.
- **Ready and Failed** stages as mocked:
  - `+` closes the Admin Center and opens the prompt bar with `+tag `, via
    `_start_custom_agent_from_selection` after an off-thread
    `ensure_project_tag_catalog()`;
  - `c` makes it current, through the existing set-current worker;
  - `⏎` dismisses with an outcome `show(key)`;
  - `i` runs the init hand-off;
  - Failed has `r` re-plan, `t` terminal (`app.suspend()` +
    `subprocess.run(apply_argv minus --json)`), and `b` back.
- **Preferences.** Write them (last provider, last owner) off-thread on success.
- **Tests:**
  - plan-document → review-row tests;
  - the NDJSON → checklist reducer;
  - the stage machine through Ready and Failed;
  - submission signature and concurrency keys (mirroring
    `_projects_pane_init_flow_helpers.py`);
  - closing mid-run still triggers the reload;
  - the init controller refactor keeps the `test_projects_pane_init_flow_*` suites
    green.
- **PNG goldens:**
  - Configure existing;
  - Configure for an unmanaged repo without push rights (amber);
  - Review add;
  - Progress mid-run;
  - Failure;
  - Ready (accent);
  - narrow Review at `80x24`.

### tui-entry

- **Keymap.** Add `new_project: "n"`:
  - `ace.keymaps.projects` in `src/sase/default_config.yml` (the gotcha);
  - `ProjectsPaneKeymaps`;
  - `_PROJECTS_BINDING_META`;
  - `sase.schema.json` (keymaps `additionalProperties: false`);
  - `_PROJECT_ONLY_ACTIONS`;
  - `admin_center_projects_section` (help, ≤ 32 chars: `new project (add or create)`);
  - `docs/configuration.md` keymap table;
  - the tests enumerating projects keymaps.
- **The action.** `action_new_project` pushes `NewProjectModal` with the pane's records
  and current-project context. It handles the dismissal outcomes `show(key)`,
  `added(key)` and `none`, running the post-mutation sequence from
  [Stage 4](#stage-4--progress--init-hand-off--ready-tui-add). It never opens a second
  wizard when `n` is typed into a focused input.
- **Hints row.** Rebuild `hints_text()` as width-aware priority truncation. It leads
  with `n new`, drops the lowest-priority hints first, and ends with `? help` when
  anything was dropped. Pass the pane width at render time, using cached widths; no
  measuring on hot paths. Update the hint-substring tests.
- **Empty state.** The empty-state option reads
  `No projects yet — press n to add or create one` when no projects are registered. The
  filtered-empty text stays.
- **Command palette.** Add _Projects: New project…_: a keyless `CommandSpec` generator
  in `commands/catalog.py`, plus an app action that opens Admin Center ▸ Projects and
  then the wizard after mount.
- **Docs.** In `docs/ace.md`, add "Add or create a project" to the Projects tab section
  (stages, keys, setup, recovery) and the `n` key to its table.
- **Tests:**
  - binding and gating (active only on the PROJECTS sub-tab);
  - palette entry;
  - the outcome handling, selecting and offering "Show new project" when filtered;
  - the hints-row truncation at 80/120/200 columns.
- **PNG goldens:** the Projects tab with the new hints at 120x40 and 80x24, and the
  empty state. Refresh the existing projects goldens the hint change touches.

### tui-create

- **Create row.** Show the pinned `✚ Create a new <Provider> repository…` row (and `n`
  in Source) for providers whose `ws_describe_repo_creation` answers.
- **Form widget.** Add `src/sase/ace/tui/widgets/repo_creation_form.py`, rendering
  `RepoCreationForm`:
  - `line`/`text` reuse the existing single-line and multi-line editors;
  - `choice` is an inline radio row (`( ) Public (•) Private ( ) Internal`, with
    per-choice disabled reasons);
  - `bool` is a `[x]` toggle (`space`);
  - `search` is a searchable picker that accepts typed values;
  - a Basics/Advanced disclosure (`a`) shows a count.

  Availability, reasons and greying come from core `evaluate_repo_creation_form` in
  lenient mode, run on change (fast, pure). Do **not** change `TypedInputForm` behavior
  for its existing hosts; reuse its editor pieces.

- **Configure (new)** (`new_project_create.py`):
  - an owner selector with the `can_create` chip;
  - the name field with pure normalization preview and debounced availability probe (300
    ms), plus "exists → ⏎ add it instead";
  - the project identity row (name/key/accent, uniqueness via local plan);
  - setup (default on) and make-current (default off);
  - defaults from `preferences.py`, with visibility Private on first use per owner, then
    remembered.
- **Review (danger).** `sase project create … --check --json` in a session worker.
  `ConfirmKind.DANGER` styling with the visibility in the title, PUBLIC in warning
  color, and the "Creates a repository on <host>…" notice. Apply goes through
  `_submit_durable_proc(operation="project.create", …)` and the shared Progress/Ready
  path.
- **Recovery.** Created-but-incomplete failure: `r` resumes (the journal adopts the
  repo). The RESUME rows in Source cover a closed TUI.
- **Preferences.** Persist per-owner visibility on success.
- **Tests:**
  - form-state tests per gh constraint (template greying, all-branches requires
    template, internal disabled);
  - typing the name spawns no subprocess (the probe is debounced and off-thread);
  - "exists → add instead" switches the flow to 2a with the ref;
  - review-danger rows;
  - a stage-machine create path through Ready.
- **PNG goldens:**
  - Configure new (basics);
  - Advanced with a Template (greyed fields with reasons);
  - name taken;
  - Review danger (private, and public in warning color);
  - created-but-incomplete failure;
  - narrow `80x24`.

## Verification

- **sase:** `sase tool run check` for every phase that touches the sase repo, after
  `just fmt`, following `lint_and_test.md`. Do not run `just check-full` unless
  instructed.
- **Linked repos:** `sase tool run check` inside sase-core and sase-github for their
  phases.
- **Goldens:** TUI phases run `just fix-tui-screenshots` with targeted selectors through
  `/sase_monitor` when long, and inspect every created or updated golden before
  finalizing.
- **Live captures:** use `sase screenshot` (`tui_screenshot.md`) to confirm real
  behavior for each TUI phase, including the 80×24 layout and keyboard flow.
- **Before the epic lands:** a human runs one manual disposable-repo smoke
  (`sase project create gh:<you>/sase-wizard-smoke -v private --yes`, then delete it by
  hand). Agents never create live repositories.

## Out of scope (v1)

- Bulk add through marks.
- Publishing an existing local directory (`gh repo create --source --push`).
- A `＋ New project…` entry in the `+` launch picker.
- A template browser.
- Registering on other machines through fleet dispatch.
- Moving name/key allocation fully into sase-core: the core plan only previews it; the
  provider allocates.
- Changing the `t` terminal valve for sidecar creation, which stays the deliberate
  interactive `y`.
