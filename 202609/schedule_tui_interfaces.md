---
tier: epic
title: Rename the public AXE and ACE interfaces to Schedule and TUI
goal: Make schedule, routine, job, and tui the consistent public interfaces across
  SASE while preserving scheduler behavior and existing internal identities.
phases:
- id: cli-contract
  title: Canonical commands and compatibility contract
  depends_on: []
  size: medium
  description: 'cli-contract: implement canonical CLI grammar, a sunset rollout flag,
    hidden compatibility inputs, help, completion, and lifecycle command forwarding.'
- id: config-core
  title: Shared configuration name translation
  depends_on:
  - cli-contract
  size: medium
  description: 'config-core: implement Rust-owned per-layer config normalization,
    public projections, exact provenance, and migration planning with binding tests.'
- id: config-surfaces
  title: Canonical configuration and migration interface
  depends_on:
  - config-core
  size: medium
  description: 'config-surfaces: connect the shared config contract to Python loading,
    schemas, defaults, config commands, keymaps, and an explicit migration preview/apply
    command.'
- id: job-authoring
  title: Public job script and SDK interfaces
  depends_on:
  - config-surfaces
  size: medium
  description: 'job-authoring: add job-named entrypoints, a thin public SDK facade,
    environment aliases, and version-aware script context presentation without moving
    implementation packages.'
- id: public-records
  title: Public status, references, and diagnostics
  depends_on:
  - job-authoring
  size: medium
  description: 'public-records: present jobs and routines in status, diagnostics,
    artifact references, producer metadata, and metrics while preserving stored identities
    and old wire readers.'
- id: tui-surfaces
  title: Schedule tab and TUI terminology
  depends_on:
  - config-surfaces
  - public-records
  size: medium
  description: 'tui-surfaces: update TUI titles, navigation, editors, help, notifications,
    and palette behavior, retaining internal widget and action identities and checking
    visual regressions.'
- id: integrations
  title: Plugin and operator-facing integrations
  depends_on:
  - cli-contract
  - config-surfaces
  - job-authoring
  - public-records
  - tui-surfaces
  size: medium
  description: 'integrations: update maintained plugin interfaces, operator config
    sources, shell helpers, and generated-completion inputs with a dependency-safe
    release order.'
- id: documentation
  title: Documentation, memory, skills, and current demos
  depends_on:
  - integrations
  size: medium
  description: 'documentation: update maintained documentation and media, glossary
    and memory sources, generated skill templates, and canonical site routes while
    preserving history.'
- id: acceptance
  title: Combined acceptance and rollout verification
  depends_on:
  - cli-contract
  - config-core
  - config-surfaces
  - job-authoring
  - public-records
  - tui-surfaces
  - integrations
  - documentation
  size: medium
  description: 'acceptance: audit every public surface, verify both compatibility
    states and existing state reuse, run combined checks, and record the coordinated
    rollout procedure.'
proposed_by: bbugyi200.athena.0kz
create_time: 2026-09-14 15:33:49
status: wip
bead_id: sase-113
---

- **PROMPT:** [prompts/202609/schedule_tui_interfaces.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/schedule_tui_interfaces.md)
- **BEAD:** [sase-113](https://github.com/sase-org/sase--beads/blob/main/pages/sase-113/README.md)

# Rename the public AXE and ACE interfaces

## Outcome and scope

The user's names are final. The CLI root is `sase schedule`, its tab is **Schedule**,
its supervised groups are **routines**, its units of work are **jobs**, and the main
interactive application is launched with `sase tui`. Use “the SASE scheduler” and “the
SASE TUI” in prose, then “the scheduler” and “the TUI”. Do not substitute the research
report's proposed `scheduler` command, `Scheduler` tab, or `lane` noun.

Canonical explanation: the SASE scheduler runs jobs in independently supervised
routines; each execution is a job run; users operate it from the Schedule tab of the
SASE TUI. A routine retains its current interval, concurrent execution, timeout,
restart, and isolation behavior. Jobs can be triggered by time, events, or manual
execution. This work does not change that behavior or agent admission scheduling.

This is an epic because public configuration composition belongs to Rust, CLI and TUI
presentation belong to Python, extension contracts reach plugins, and current
documentation and generated interfaces require a coordinated audit. Each phase is
bounded direct implementation work. Authoring this epic is xlarge planning work; none of
its workers needs to invent another naming scheme or plan an internal rewrite.

Research context was read through the audited artifact interface:
`research:202609/scheduler_tui_naming/scheduler_tui_naming.md`. Re-read it with
`sase artifact read` if needed. Use its contract inventory and collision analysis; its
naming recommendations do not override this plan.

All paths below are relative to the named repository. Open every non-primary repo with
`/sase_repo` and use only its returned checkout. The implementation repos are `sase`,
`sase-core`, `sase-telegram`, `sase-github`, `sase-nvim`, and `chezmoi`. Inspect
`sase-research-artifacts` for current prompt text; change it only where an actual public
reference needs updating. Research and archived plan sidecars are context, not rename
targets.

## Public versus internal contract

| Surface                  | Canonical public spelling                               | Treatment of existing identity                                          |
| ------------------------ | ------------------------------------------------------- | ----------------------------------------------------------------------- |
| CLI roots                | `sase schedule`, `sase tui`                             | Retain hidden legacy input aliases through the rollout contract below   |
| Nested scheduler groups  | `routine`, `job`                                        | Keep existing handlers and argparse destination fields where useful     |
| TUI startup              | `--tab schedule`, `--no-schedule`, `--restart-schedule` | Normalize to internal `axe` tab and existing Python flags               |
| Job run options          | `--routine`, `--job-verbose`                            | Preserve `-L` and `-V`, normalize existing destination attributes       |
| Authored config          | `tui`, `schedule`, `routines`, `jobs`                   | Normalize each source layer to the existing internal model              |
| Job authoring            | `sase_job_*`, `sase.jobs`, `SASE_JOB_*`                 | Thin aliases/adapters to existing scripts, SDK, and execution machinery |
| Public references        | `job:<routine>/<job>`                                   | Resolve to the existing stored job identity; no duplicate artifacts     |
| Display text             | TUI, Schedule, scheduler, routine, job, job run         | Translate at the owner of the rendered result                           |
| Public structured output | Job/routine field names on canonical interfaces         | Keep old schemas readable and old-command compatibility output stable   |

Do not rename `src/sase/ace`, `src/sase/axe`, `src/sase/chops`, Python implementation
classes/functions, Rust modules or binding symbol names merely for vocabulary. In
particular, do not move `sase.ace.scheduler` or rewrite `HookJobRunner` because the
research suggested it. Those are independent architectural changes.

Preserve internal tab IDs, widget/CSS IDs, runtime action names, durable operation names
such as `axe.bgcmd`, process locks, PID files, state directories under `~/.sase/axe`,
`ace-run` workflow/storage paths, existing agent names, run IDs, stored tribe
associations, and stored `chop:` artifact/link keys. Historical paths and IDs may still
appear when showing the actual record or file; never manufacture a new path or rewrite
user-authored text to hide one. Rename new human-readable labels around them. Logs
already on disk, archived prompts/research/plans, released changelogs, historical blog
posts, and historical compatibility fixtures retain their original text.

User-editable config, SDK imports, executable names, documented environment controls,
and CLI JSON are public contracts, not automatically exempt “internals”. Give them
canonical accessors or boundary projections without moving their underlying storage. Do
not rename user-selected routine/job names, custom executable paths, arbitrary
dictionary keys, unrelated CI jobs, shell `jobs`, or Python coroutines.

## Rollout and compatibility policy

Create one sunset flag, provisionally `schedule_tui_names`, through `sase flag new`
after approval; first check for a semantically equivalent existing flag. Read
`sase_flags.md` before creation and use the generated registry/bead checklist.

- Enabled, the default: new names are canonical in help, completions, current docs,
  generated config, TUI, and public output. Existing command/option spellings remain
  hidden input aliases, old config loads through the compatibility normalizer, and old
  script protocols remain supported. The result is usable without first editing every
  installed user config or shell alias.
- Disabled: preserve the old public CLI/presentation contract as the explicit Off
  branch, while accepting the new inputs too so a partial operator migration is
  reversible. Existing public JSON remains byte-shape compatible where tested.
- The flag governs the transition away from the old primary interface. Its removal
  deletes that Off branch and makes the new primary interface unconditional. This epic
  does not remove accepted historical inputs or SDK compatibility contracts; retiring
  those would be a separate, explicitly flagged breaking change. Do not confuse hiding
  an alias with permission to delete it.
- The removal condition is that maintained integrations, configs, docs, skills, and
  supported rollout scenarios use the canonical interface and no longer require the old
  primary presentation. Record both-state tests and the normal removal thresholds on the
  generated flag bead. Keep this sunset flag after the epic; it is not beta scaffolding
  to remove at integration time.

Avoid noisy warnings in TUI refresh loops and repeated subprocess calls. Report legacy
config use once in config/doctor diagnostics with exact source paths and migration
instructions. Where CLI compatibility notices are useful, use stderr and keep JSON
stdout clean. Internal callers should emit canonical argv once the parser supports it.

Keep implementation phases off the canonical user release until the combined surface is
verified. If a phase must ship user-reaching unfinished behavior independently, follow
`sase_flags.md` for a temporary beta scaffold and remove that scaffold before the
complete epic lands. Do not deploy generated skills from an unlanded source tree.

## 1. Canonical commands and compatibility contract

Audit actual current sources before editing and record a concise surface inventory in
`docs/development/public_interface_naming.md`. Classify matches as public input, human
output, internal identity, or history, and give each public class an owner below. The
inventory is a finite migration contract, not an allowlist that exempts entire
implementation directories from review.

Primary entry points are `src/sase/main/parser_ace.py`, `parser_registry.py`,
`parser_root_help.py`, `parser.py`, `entry.py`, `ace_handler.py`, `ace_tmux.py`, and
`axe_handler.py`; shell completion has existing compatibility metadata helpers in
`src/sase/completion/compat.py`.

Implement these complete command families, preserving all existing non-renamed options,
exit behavior, and dispatch semantics:

```text
sase tui [QUERY] [OPTIONS]
sase schedule {ensure,job,maintenance,restart,routine,start,status,stop}
sase schedule job {doctor,list,run}
sase schedule routine {list,run,status}
```

Preserve the actual children of ensure, maintenance, start, stop, and status, including
hidden subprocess-only `bgcmd-launch`. Bare job and routine groups must use the central
default-list convention and name the canonical path in its delegation notice. Bare
`sase schedule` retains the existing root-group usage/exit behavior; do not make it
implicitly start a daemon or delegate to a nonexistent list child.

Register hidden `ace`/`axe`, `chop`/`lumberjack`, old long-option spellings, and
`--tab axe` input compatibility. Canonical help and completion must not advertise them,
including argparse usage metavars, errors, compact root help, full root help, and
default-delegation text. Accept mixed legacy/canonical aliases deterministically;
normalize to one handler without duplicate parser registration or changed short flags.
Set public positional metavars to JOB/ROUTINE while retaining internal attribute names.
Sort help as required by `cli_rules.md`.

Retain the original public command family separately from normalized dispatch so Section
5 can select the compatibility JSON format. The canonical `schedule` root selects the
new public JSON shape even when a nested legacy alias was typed; the legacy `axe` root
selects the old shape. With the rollout flag disabled, use the old shape for both roots.
Do not infer the output contract from an internal handler name.

Update internal command emitters that must traverse the CLI: TUI exec/restart and tmux
forwarding, daemon start/re-exec, routine process spawning, watchdog/service ExecStart
text, doctor remediation commands, update/restart notices, and background command
launching. Keep lifecycle locks and existing systemd unit identities so this cannot
create two schedulers. An installed old watchdog command must continue working through
the alias; a newly rendered unit uses `sase schedule ensure`.

Verify the feature-flag decision is available before lazy parser registration and
config-dependent default loading. Never make resolving the naming flag depend on a
configuration normalization step that itself needs to resolve that flag.

Acceptance: parser/dispatch tests for every family and relevant option, both root help
modes, default list, invalid input, completion metadata, query argument quoting,
tmux/re-exec forwarding, and captured subprocess argv. Lifecycle tests use isolated
fixtures/mocks, never the user's live scheduler. Existing implementation modules remain
importable. Add meaningful behavioral assertions, not a test per replacement string.

## 2. Shared configuration name translation

Implement shared semantics in `sase-core`, centered on `crates/sase_core/src/config/`
and `axe_chop/config.rs`, with `sase_core_py` bindings. The current `config/axe.rs`
composes exact `axe` / `lumberjacks` / `chops` segment paths and supports keyed entries,
legacy lists, sparse contributions, and provenance. Reuse it behind a boundary adapter.

Define one context-aware mapping for authored and displayed names:

```text
ace -> tui
axe -> schedule
axe.lumberjacks -> schedule.routines
...chops -> ...jobs
chop_timeout -> job_timeout
chop_script_dirs -> job_script_dirs
lumberjack_log_* -> routine_log_*
lumberjack_restart_backoff_max_seconds -> routine_restart_backoff_max_seconds
verbose_lumberjack_diagnostics -> verbose_routine_diagnostics
ace.axe_description_expanded -> tui.schedule_description_expanded
```

Inventory and explicitly map other public config fields containing these concepts,
including update retry settings and configurable keymap names/scopes. Rename keymap
names in the authored schema while mapping them to existing action methods. The
configured built-in `chop` tribe is exposed as `job`; arbitrary user tribe names remain
verbatim. Do not implement unscoped substring replacement.

If a user's existing `job` tribe and the built-in `chop` identity would collide, report
that conflict before migration and require an explicit resolution. Do not silently fold
independently authored tribes or rewrite their stored associations.

Normalize every layer before composition so a legacy user override still overrides
canonical bundled defaults, and a canonical overlay overrides a legacy base as before.
Keep the runtime dictionary/model shape stable for existing callers. Public projection
is separate from raw source paths and internal fields. Preserve source layer ordering,
list strategies, deletion/tombstone behavior, disabled entries, `for_each` identities,
and exact keys containing dots. Across separate layers, normal precedence wins.

Within a single source layer, combine non-overlapping contributions from synonymous
roots. Identical duplicate leaf values can collapse with a diagnostic; conflicting
values for the same semantic path must fail with both authored paths and a resolution
hint. Do not silently prefer a new spelling over an old value in the same layer. Define
list collisions explicitly: identical synonym lists can collapse; unequal lists for the
same semantic entity collection are conflicts rather than concatenation.

Carry original source segments through normalization so preview/reset/delete writes the
selected exact source entry, not a guessed canonical path. Provide shared public path
projection and migration planning from this mapping. Preserve existing Rust wire
contracts; extend/version additive APIs only as required and test Python bindings.
Python callers must not maintain a second ruleset.

Acceptance: old/new/mixed-layer parity; conflicting and identical aliases; keyed and
list forms; dotted names; per-target overrides; exact provenance; preview/set/reset/
delete; round trips from public paths to the same internal entity. Run the Rust repo's
full required check, including PyO3, not only `cargo test -p sase_core`.

## 3. Canonical configuration and migration interface

Connect Rust normalization in `src/sase/config/loading.py`, `layers.py`, `core.py`, the
config editor adapters, and `src/sase/axe/config_backend.py`. Keep ordinary internal
consumers of `load_merged_config()` working against the existing model. Ensure plugin,
user, machine, and project layers all use the same normalization.

Update `src/sase/config/sase.schema.json` and `src/sase/default_config.yml` together.
Advertise `tui`, `schedule`, routines, jobs, the renamed options above, and canonical
keymap actions/scopes. Compatibility schema paths remain accepted but do not dominate
the config catalog, completion, or new examples. Update schema descriptions, field
labels, built-in routine/job descriptions, and default tribe presentation. Preserve key
assignments and user-defined identifiers.

Make config init/show/catalog, validation, doctor, and exact editor previews display
canonical names. `sase config layers` still reports real authored keys and source paths,
with a separate canonical replacement diagnostic. A raw file preview must show the
actual file. New config writes use canonical keys; edits to existing unmigrated entries
target their actual paths and avoid creating a parallel alias subtree.

Add `sase config migrate-interface-names [PATH ...]`, preview-only by default, with
`-a/--apply` for explicit writes and `-j/--json` for machine-readable previews. With no
paths, inspect writable loaded layers and report skipped read-only/plugin sources. Make
this runnable against legacy config without requiring the TUI or daemon to start. Use
existing transactional/surgical write facilities and the shared migration plan; validate
all candidate documents before any write, detect changed source files, preserve
comments/order where supported, and clearly report partial I/O failure. Do not rewrite
free text, executable strings, arbitrary names, or historical records. Operator
shell/script migration belongs to Section 7.

Acceptance: preview has zero file or runtime effects; apply is idempotent; invalid or
conflicting input is not partially rewritten during preflight; values and effective
behavior match before/after; both config spellings work in both rollout states; source
selection and keymap action dispatch remain correct. Include the existing config
backend, config-edit, schema-keymap, and plugin-layer regression suites.

## 4. Public job script and SDK interfaces

Add `sase_job_*` console-script entries pointing at the existing implementation
functions in `pyproject.toml`. Preserve `sase_chop_*` executable compatibility. Script
discovery must prefer the canonical public name, avoid showing two rows for the same
built-in executable alias, and still find an old-only installed plugin or explicitly
configured custom executable. Do not deduplicate two distinct user executables merely
because their names resemble each other.

Add the public `sase.jobs` facade with `JobArguments`, `JobInvocation`, `JobLogger`,
`JobReport`, `JobResultBuilder`, `JobResultStatus`, `JobSummary`,
`JOB_RESULT_SCHEMA_VERSION`, and the corresponding job-named helpers currently exposed
from `src/sase/chops/__init__.py`. Re-export or minimally adapt existing objects; do not
copy execution/proposal/validation logic into a new implementation.

Define canonical documented environment names: `SASE_JOB_*` for script invocation,
`SASE_JOB_ROUTINE` for `SASE_CHOP_LUMBERJACK`, and `SASE_SCHEDULE_*` / `SASE_TUI_*` for
genuinely user-set scheduler/TUI controls. Keep private lifecycle FD transport, re-exec
guards, and test-only variables internal. The runner supplies equal values under both
script env families during compatibility. For user-supplied aliases, accept either; if
both differ, fail the relevant invocation/config validation with an actionable
diagnostic. Scrub both families from unrelated launches and preserve job ownership,
dedupe, and admission metadata behavior.

Audit context/result JSON separately from env names. Preserve the current result
protocol and existing script readers. If context fields containing old terminology are
documented public input, provide canonical read access through the new invocation facade
and additive aliases in the context, with conflict validation; never silently
reinterpret or rename a field under an unchanged required wire schema. Result tokens
unrelated to the rename remain identical. Update user-visible SDK errors and log copy.

Update canonical built-in script values in defaults only after entrypoints exist. Keep
internal file names and imported modules unchanged. Acceptance includes installed
entrypoint smoke tests, old/new SDK parity on one structured result, old-only script
execution, discovery de-duplication, env precedence/conflicts, env scrubbing, dry-run,
target expansion, and identical proposal/dedupe outcomes. No real agent launches.

## 5. Public status, references, and diagnostics

Update human output owned by `src/sase/axe/*render*`, `*status*`, `*doctor*`,
`src/sase/doctor/`, lifecycle/update handlers, log-pack renderers, and notifications.
Use Routine/Job/Job run headings, canonical commands in remediation text, and source
labels that describe the SASE scheduler. Change Rust-owned user errors in Rust,
including validation diagnostics emitted through Python adapters.

Include public diagnostic group selectors and exposed feature-flag names in the
inventory. For example, present `ace_refresh_tokens` as `tui_refresh_tokens` and accept
the canonical CLI/config selector while retaining the existing enum member, persisted
override key, and flag bead identity. Apply the same rule to other existing public
identifiers that name these interfaces. Keep aliases at their owning input/ presentation
boundary and verify they select exactly the same flag or diagnostic; do not create
duplicate flags or diagnostic checks.

Treat canonical CLI JSON as an explicit public projection of the existing status and job
records. Enumerate old/new field mappings and schema versions in the interface contract;
use an explicit new schema version wherever emitted field names change. Legacy command
aliases keep the old documented JSON schema for existing consumers; canonical commands
emit the new schema. TUI/internal readers continue using existing Rust/Python wire
models. Test both outputs against the same fixture, including nested routine/job objects
and error responses. Never change internal persisted status JSON just to make the CLI
result look new.

Add public `job:<routine>/<job>` input/display support wherever live `chop:` references
are supported, using shared Rust reference normalization where the semantics are core.
Resolve it to the same stored `chop:` identity before linking, lookup, dedupe, or audit
storage. Existing refs remain readable. CLI, artifact panels, generated link labels, and
prompt completion offer job vocabulary; they must not create a second artifact or orphan
old `launched` relations. Prove both refs resolve the same stored object.

Project stored `chop` producer/source/tribe values as Job or job on public views and
support canonical query/filter inputs that map to those identities. Preserve literal
agent IDs and stored associations. Ensure public selectors and snippets round-trip to
the same records. Do not repurpose the unrelated Proc concept as “job”.

For exported metric names and labels that explicitly name the renamed subsystem, add
canonical `sase_schedule_*` / routine/job equivalents and document compatibility
exports. Retain old metric series for existing dashboards during rollout; do not rename
internal counter fields or double-increment counters. Test both exports report the same
underlying observation. Keep raw historical log text intact.

Acceptance: current human output uses the new vocabulary, JSON schemas remain explicit,
references/queries preserve identity, existing history is readable, and migration or
error explanations are the only old prose on canonical surfaces apart from actual
paths/IDs. Cover notification/error rendering as well as happy-path tables.

## 6. Schedule tab and TUI terminology

Change `_TAB_DISPLAY_NAMES['axe']` in `src/sase/ace/tui/widgets/tab_bar.py` to
`Schedule` for the canonical surface; retain internal `axe` IDs and existing colors. Use
`SASE TUI` wherever the application title/header currently brands itself ACE. Inventory
header fallbacks as well as tribe-specific identities; do not overwrite a user's chosen
icon or title.

Audit visible strings in widgets, help/onboarding, refresh controls, jump-all labels,
palette categories, keymap metadata, daemon/routine/job sidebar rows, dashboards,
empty/loading/error states, manual-run notifications, add/edit dialogs, config commit
prompts, and settings. Relevant areas include `widgets/axe_*`, `widgets/_axe_*`,
`actions/axe*`, `modals/help_modal`, `commands`, and `keymaps`. Source directory names
are not a reason to skip their output strings.

Keep configuration editors wired to Section 3's shared exact-path model. Public actions
use schedule/routine/job names, mapped to existing runtime methods. Update defaults,
schema, registry metadata, palette, help, and completion consistently.

Remove or repoint the existing command-palette `jobs` search alias for the Procs panel:
it must find scheduler job operations/ Schedule navigation. Procs retains explicit
proc/process/background-command search terms. Keep background commands in their existing
location under a clear “Commands” label; do not move panels or redesign the workflow as
part of the rename.

Preserve async dispatch, refresh coalescing, cached data access, focus, and selected
identities. Read `tui_perf.md`; no config normalization or file IO in rendering or
keypress paths. The longer Schedule label must fit supported narrow and standard
terminal widths without losing tab selection indicators or header controls.

Acceptance: launch `sase tui --tab schedule` in an isolated fixture, navigate tabs, open
help and palette, add/edit a routine/job, and exercise manual job run and scheduler
controls with mocked effects. Verify labels and retained behavior in both rollout
states. Inspect actual/expected PNG diffs and update affected goldens through the
existing visual test workflow. Global header/tab changes may affect many snapshots; do
not rename test directories or regenerate unrelated images without inspecting why.

## 7. Plugin and operator-facing integrations

Reopen relevant repos through `/sase_repo` and read their AGENTS instructions. Update
only current public interfaces and callers:

- `sase-telegram`: add `sase_job_tg_inbound`/`sase_job_tg_outbound` entrypoints pointing
  to existing functions, retain old entrypoints, update the receiver's own executable
  command, publication smoke tests, current docs, and job-facing log/help output.
  Preserve receiver state and its independence from scheduler stop.
- `sase-github` and `sase-nvim`: update installation docs and user-facing command
  examples. Internal imports/comments referring to `sase.ace` implementations do not
  require edits. Update real command emitters if the inventory finds them.
- `sase-research-artifacts`: update maintained prompt/template terminology only where it
  actually occurs. Do not rewrite the research report or archived artifacts.
- `chezmoi`: migrate authored `home/dot_config/sase/sase*.yml` and project config
  sources through the verified config mapping; update script values to installed job
  entrypoints. Add canonical `tui` and `schedule` shell helpers and rename the existing
  `acei`/`aceii` helper interfaces to `tuii`/`tuiii`, preserving their provider install
  behavior. Keep old shell names forwarding during migration. Update snippets and their
  required LuaSnip counterparts, custom prompts, and completion generation inputs. Do
  not hand-edit generated SKILL.md or generated completions.

Inventory any configured third-party job plugin before switching its executable
references: migrate only to entrypoints that package actually provides. An old-only
external plugin remains executable through compatibility discovery; it must not be made
a hidden prerequisite for this epic by guessing an unavailable new name.

Use this delivery order: release additive core bindings before a host that requires
them; make plugin job entrypoints available before operator config points at them; land
the complete host interface; then activate authored operator config and generated
skills/completions from the landed host. Declare minimum dependency versions if a caller
requires a new API. Do not manually edit release-owned crate versions.

Do not apply the global chezmoi destination from a phase's unlanded workspace. After the
host-owned finalizer lands the relevant source, follow the chezmoi repo's required
`chezmoi update -a --force` step and the generated-skills deployment rule. Record that
post-landing operation for the owning host continuation so it cannot be forgotten. Do
not claim deployment completed merely because source changes are ready.

Acceptance: plugin entrypoints resolve in installed-package tests; shell helpers emit
canonical argv with correct quoting; canonical configs preserve effective settings; an
old installed plugin/config still works; each changed repo passes its required checks.
Do not send Telegram messages or restart the real scheduler to test naming.

## 8. Documentation, memory, skills, and current demos

Update README, INSTALL, current CLI/configuration/plugin/development/performance/
troubleshooting guides, user examples, alt text, navigation titles, and prompt/skill
sources. Use `docs/tui.md` and `docs/schedule.md` as canonical guide pages with `/tui/`
and `/schedule/` routes. Update `mkdocs.yml`, inbound links, and the site index;
preserve old `/ace/` and `/axe/` URLs with redirects/compatibility pages using the
existing site facilities. Keep external historical backlinks working. Explain legacy
spellings once in a migration guide, rather than repeating old acronyms throughout
current docs. Technical descriptions of actual retained storage paths stay accurate.

Update current demo tape commands/captions and regenerate currently promoted README/
landing-page media whose visible title/tab text still teaches ACE/AXE. Read demo
instructions first. Preserve existing asset URLs/file names where practical; names of
media files are not a reason to break links. Historical blog media is exempt unless it
is reused as current product guidance, in which case replace the current placement or
regenerate that promoted asset. Verify both text and visible media.

Memory changes are limited to the requested terminology, covered by the user's request
to rename all user-facing references. Before edits use `/sase_memory_write`, and read
each affected body through `/sase_memory_read`. Update the existing glossary
Chop/Lumberjack definitions to Job/Routine with old words as lookup aliases, add concise
Scheduler/TUI/Job Run definitions only as needed to explain these interfaces, and fix
the Proc definition's current TUI reference. Update active reference-note prose and
command examples such as `tui_perf.md`, `lint_and_test.md`, and current notification
guidance where these names describe the renamed functionality. Preserve accepted
decision bodies, historical statements, and valid internal paths/identifiers.

Read remaining affected memory via batched audited selectors discovered from its
catalog; do not grep/read canonical bodies directly. Prefer modifying existing notes,
not adding a new core-memory essay. If a generated note contains the old wording, change
its generator template. Run `sase memory init` to publish AGENTS and provider shims; do
not hand-edit generated instruction files.

Update source templates in `src/sase/xprompts/skills/`, including current references in
`sase_patches`, `sase_notify`, `sase_agents_status`, `sase_questions`, `sase_run`, and
`sase_var`. Their skill identifiers stay the same. Check metadata/descriptions as well
as bodies. Preview with `sase skill init --diff` or `--dry-run`; deploy only after the
source is clean and landed, then apply via the supported chezmoi workflow.

Acceptance: a reader can install, open the TUI, configure a routine, author/run a job,
and troubleshoot the scheduler entirely through canonical examples. Build current docs,
verify redirects/local links and runnable examples, inspect promoted media, validate
generated memory and skill previews, and run required primary-repo checks. Do not modify
historical research recommendations to make them agree with the decision.

## 9. Combined acceptance and rollout verification

Reconcile the surface inventory from Section 1 against the combined tree. Search
case-sensitive and case-insensitive whole words, plural forms, command argv arrays,
config keys, public env/script prefixes, schema strings, and templates. A zero-match
whole-repository grep is not the goal: classify remaining matches individually or in
narrow, justified groups. Current rendered product text must not retain old names except
migration explanations and actual retained identifiers. Internal imports and history
must not be churned to satisfy a token count.

Exercise this matrix in isolated test environments, without real provider launches,
network notifications, or control of the user's running services:

| Scenario                                       | Required outcome                                                                   |
| ---------------------------------------------- | ---------------------------------------------------------------------------------- |
| New CLI, new config, new script interface      | Canonical help/output and unchanged scheduler semantics                            |
| New CLI with legacy user/plugin config         | Same effective overrides and actionable migration diagnostics                      |
| Legacy CLI/options and old scripts             | Still usable; existing JSON consumers retain their contract                        |
| Mixed roots across layers and dotted job names | Correct precedence, provenance, and exact editing                                  |
| Conflicting aliases in one source              | Clear preflight error, no silent choice or partial config mutation                 |
| Old state, refs, tribes, and run history       | Same records remain addressable; no second daemon or artifact                      |
| New canonical refs, filters, SDK, JSON         | Correct public names and round trips to the existing backend                       |
| TUI at narrow and standard widths              | Schedule is visible; help/palette/editor/actions remain usable                     |
| Naming flag disabled                           | Old primary interface remains functional; new inputs do not strand migrated config |
| Watchdog, tmux, updater, plugin packaging      | Canonical emitted commands resolve with compatibility installed                    |

Reuse and extend existing parser/completion, config/schema/exact-editor, status/doctor,
chop SDK/runner/env, artifact projection, TUI navigation/palette/config editor,
lifecycle, tmux, and visual tests. Add boundary regression tests where renaming can
change behavior; do not add one-test-per-string assertions across all prose.

Every implementation phase changing tracked primary-repo files reads `lint_and_test.md`
and runs `just check`. Follow each linked repo's required checks. On the combined epic
tree run `just check-full` only through `/sase_monitor`, using TESTING/TESTED, and run
the affected/full visual suite as appropriate for the shared tab/header change. Resolve
caused failures before declaring acceptance. Do not replace required Rust workspace/PyO3
checks with a library-only cargo test.

Record the tested host/core/plugin revisions, compatibility matrix results, intentional
remaining internal names, and post-landing generation/apply order in the migration
document and final handoff. The plan is complete only when the full current public
surface is covered, existing supported state/config/extensions work, tests pass, and the
host-owned rollout has an explicit owner for every required generation/apply step. Do
not perform unrelated internal cleanup or defer a missed public rename as optional
follow-up. Agents declare completed work to host finalizers; they do not manually create
commits, branches, or PRs.
