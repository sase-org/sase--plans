---
tier: epic
title: Script-only chops with structured launch proposals
goal: 'Chops become script-only, are referenced by full script name, emit a versioned
  structured JSON result that the axe runner turns into supervised sase agent launches,
  and gain declarative triggers/guards, fail-closed validation, keyed composition,
  and generic multi-target fan-out. Personal chop scripts move to the bbugyi200/bugyi-chops
  PyPI plugin, and the chop-owned xprompt workflows are retired in favor of shared
  sase functionality.

  '
phases:
- id: core-engine
  title: Rust core chop engine and wire types
  depends_on: []
  description: '''Rust core chop engine and wire types'' section: add chop result/proposal
    wire types, result and proposal validation, trigger/guard decision logic, checkpoint
    and once-per bookkeeping transforms, target expansion, and strict axe-config validation
    to ../sase-core, exposed through sase_core_rs with a thin Python facade.'
- id: script-only
  title: Script-only chops with full-name resolution
  depends_on:
  - core-engine
  description: '''Script-only chops with full-name resolution'' section: delete the
    agent-chop variant, resolve chop scripts by their full configured name with no
    sase_chop_ prefixing, and make axe config loading fail closed through the new
    core validation.'
- id: result-protocol
  title: Structured chop results and runner-executed launches
  depends_on:
  - script-only
  description: '''Structured chop results and runner-executed launches'' section:
    introduce the versioned chop result document, parse proposed agent launches from
    it, execute launches in the runner with scaffold injection and workflow-reference
    rejection, extend the run-history lifecycle beyond launch, and add verbose/dry-run
    manual-run debugging.'
- id: chop-sdk
  title: Chop SDK and builtin script consolidation
  depends_on:
  - result-protocol
  description: '''Chop SDK and builtin script consolidation'' section: ship a small
    chop-author SDK, collapse the cloned builtin chop scripts onto it via a registry,
    migrate builtins to structured results, and retire the cl_submitted_checks legacy
    alias.'
- id: triggers-guards
  title: Declarative triggers, guards, and dedupe keys
  depends_on:
  - chop-sdk
  description: '''Declarative triggers, guards, and dedupe keys'' section: wire inhibit_if
    guards, trigger providers (always, git commits-since with runner-owned checkpoints),
    and once_per event dedupe into the runner, with skips recorded in run history
    and doctor coverage.'
- id: composition-targets
  title: Keyed composition, env parity, and target fan-out
  depends_on:
  - triggers-guards
  description: '''Keyed composition, env parity, and target fan-out'' section: accept
    a keyed map form for chops with enabled/patch/disable semantics, add lumberjack-level
    env and secret references, and add generic for_each target fan-out with an enabled-projects
    source and stable per-target chop instances.'
- id: refresh-docs-builtin
  title: Builtin refresh-docs chop and xprompt workflow retirement
  depends_on:
  - composition-targets
  description: '''Builtin refresh-docs chop and xprompt workflow retirement'' section:
    replace the refresh_docs xprompt workflow with a builtin proposal-emitting chop
    script driven by the commits-since trigger and target fan-out, delete the four
    chop-owned workflows from sase/xprompts/, migrate tests off those fixtures, and
    update all chops documentation.'
- id: bugyi-chops
  title: bugyi-chops plugin package
  depends_on:
  - refresh-docs-builtin
  description: '''bugyi-chops plugin package'' section: build the bbugyi200/bugyi-chops
    repo into a published PyPI package of proposal-emitting chop scripts (toobig_split,
    fix_just, and the two recent-commit audits) with tests, an excellent README, a
    release workflow, and the sase--plugin repository topic.'
- id: chezmoi-cutover
  title: Chezmoi config migration and end-to-end verification
  depends_on:
  - bugyi-chops
  description: '''Chezmoi config migration and end-to-end verification'' section:
    rewrite the chezmoi axe.lumberjacks config onto the new schema, delete the migrated
    chop executables and their bash tests from chezmoi, and verify the full system
    end to end with doctor, manual runs, and dry runs.'
create_time: 2026-07-18 15:41:26
status: wip
---

# Plan: Script-only chops with structured launch proposals

## Context

Axe "chops" are the periodic jobs run by lumberjacks. Today a chop is either a **script chop** (external executable
resolved as `sase_chop_<name>`, invoked with `--context <ctx.json>`, judged by exit code) or an **agent chop** (a raw
prompt string passed verbatim to the agent launcher, with a terminal `agent_launched` status). The
`lumberjack_chop_config_improvements` research report (in the research sidecar repo) audited this surface and found that
everything interesting — "should this fire?", "what should it run across?", "did the work succeed?" — has escaped into
imperative code: copy-pasted marker/threshold dances in xprompt workflows, a 158-line guard script wrapping a one-line
launch, self-launching scripts calling `sase run` directly, and config that silently ignores typos and invalid
durations.

This epic implements the redesign Bryan requested:

1. **Script chops only.** The agent-chop variant is removed entirely.
2. **Full script names in config.** The `sase_chop_` naming constraint is dropped; config references the executable's
   real name.
3. **Structured JSON results.** Scripts write a versioned result document; when it contains proposed sase agent
   launches, the _runner_ validates and executes them. Scripts never launch agents themselves, which makes manual/debug
   runs side-effect free by design.
4. **No more `#!` xprompt workflow execution from chops** — not even from proposals. The logic in the chop-owned
   workflows (`sase/xprompts/refresh_docs.yml`, `audit_recent_bugs.yml`, `audit_recent_improvements.yml`,
   `fix_just.yml`) is either translated into shared sase functionality requested via chop configuration (refresh-docs,
   commit-threshold triggers) or migrated into chop scripts in the new **bbugyi200/bugyi-chops** repo (audits, fix_just,
   toobig_split).
5. **Config improvements from the research report:** P0-1 (declarative triggers/guards), P0-2 (end-to-end action
   lifecycle + structured outcomes), P0-4 (fail-closed validation + env parity), P1-6 (keyed composition, activation,
   secret references), P2-10 (builtin registry and shared state helpers), and a generic target fan-out in the spirit of
   P1-5 that makes "run this chop against every enabled sase project" a one-liner. P0-3 (keyed concurrency/resource
   caps) is explicitly **out of scope**.
6. **bugyi-chops** becomes a real Python package published to PyPI, carries the `sase--plugin` GitHub repository topic
   (the topic is SASE's canonical plugin-registry mechanism; the request to "add the label" is satisfied by the topic),
   and gets an excellent README.
7. **Debuggability is a first-class goal:** a verbose mode for chop scripts plus dry-run manual runs that show what a
   chop _would_ launch.

Current state that shapes the work (verified against the live config, not the older research snapshot): the chezmoi
overlay configures seven agent chops (two recent-commit audits and five `refresh_docs` rows differing only in
project/threshold/cadence) and one custom script chop (`toobig_split`, which self-launches agents via `sase run`). The
`gh_actions_fix` chop was already removed, and `sase_fix_just` survives only as an unconfigured executable in chezmoi.
The sase-telegram plugin's `sase_chop_tg_inbound`/`sase_chop_tg_outbound` console scripts stay in that plugin and keep
working; only the config that references them changes.

## Design overview

### The new chop contract

A chop is an executable named by config. Resolution order: exact name inside each `axe.chop_script_dirs` entry, then the
interpreter's bin directory, then `$PATH` — always by the configured name, never with an implicit prefix. Each chop
entry gains an optional `script:` field (defaults to the chop's `name`) so display identity and executable name can
differ, e.g. the builtin `hook_checks` chop declares `script: sase_chop_hook_checks` (builtin executables keep their
existing console-script names; the prefix just stops being magic).

Invocation stays `<script> --context <ctx.json>` from the lumberjack state dir, with stdout+stderr streamed to the
per-run log and SIGKILL-on-timeout semantics preserved. The runner additionally passes `SASE_CHOP_RESULT_FILE` (a path
inside the run directory, also mirrored in the context JSON) and `SASE_CHOP_VERBOSE` when verbose diagnostics are
enabled. Exit-code-only scripts remain fully supported: no result file means today's success/failure semantics.

A script that wants to do more atomically writes a **versioned chop result document** to `SASE_CHOP_RESULT_FILE`:

- `schema_version`, `status` (`ok` | `no_op` | `check_error`), `summary`, `reason`
- `counters` (string→int) and optional `evidence` file references
- `proposed_launches`: an ordered list of structured launch proposals — prompt text, workspace ref (e.g.
  `gh:sase-org/sase`), optional agent-name template, tribe, model/effort, extra env, an optional `dedupe_key`, and an
  optional `wait_on` reference to an earlier proposal in the same list for sequencing (replacing today's hand-built
  `---`/`%wait` multi-prompts).

The **runner** (never the script) validates and executes proposals sequentially, preserving the current same-tick
workspace-allocation safety. Validation is fail-closed: malformed documents and proposals whose prompt contains a `#!`
standalone-workflow reference are rejected with actionable errors. The runner auto-injects the boilerplate scaffold
today's configs hand-assemble in three layers: agent name derived from the chop (and target) when the proposal does not
name one, `%tribe:chop`, and the workspace ref block. Inline `#name` prompt-part xprompts remain legal inside proposal
prompts; only standalone `#!` workflow launches are banned.

### Lifecycle beyond "launched" (P0-2)

Run history grows past terminal-at-launch. Persisted statuses keep `running`, `success`, `failure`, `timeout`, and
`missing_script`, and add: `skipped` (guard/trigger said no — with a recorded reason, fixing the "silent no-op" UX gap),
`no_op` (healthy nothing-to-do result), `check_error` (the probe itself degraded), and a launch lifecycle (`launched` →
terminal `action_succeeded` / `action_failed`). The existing chop-agent registry (durable `agent_chops.json` +
`SASE_CHOP_*` env stamped into launched agents and their `agent_meta.json`) is repurposed as the linkage that lets a
lumberjack housekeeping pass finalize `launched` runs once every proposal's agents reach terminal state, using the
agent-completion artifacts that already exist rather than a second completion channel. CLI, dashboard, and TUI render
the new statuses.

### Declarative triggers, guards, and dedupe (P0-1)

Evaluated by the runner before dispatch (and per-proposal where noted), using context it already builds:

- `inhibit_if:` — keyed guards. v1 providers: `changespec` (name prefix + non-terminal statuses, evaluated against the
  snapshot the runner already writes — retiring the `sase_fix_just` guard script) and `agent_hood` (active agents in a
  named hood, retiring `toobig_split`'s occupancy check).
- `trigger:` — v1 providers: `always` (default) and `git.commits_since` (project, threshold, checkpoint policy
  `on_observation` | `on_action_accepted` | `on_action_success`), with the watermark owned by the runner in per-chop
  state — replacing the marker dance triplicated across the retired workflows, including its recover-from-missing-marker
  behavior.
- `once_per:` — a per-proposal event-dedupe key (config template or script-supplied `dedupe_key`) backed by a bounded
  runner-owned seen-store.

Guard/trigger skips write `skipped` run entries with the provider's reason. The typed-result `no_op`/`check_error`
distinction plays the `script_probe` role from the research; no shell-string gates are added (that design was
deliberately retired).

### Fail-closed configuration (P0-4) and composition (P1-6)

Axe config parsing stops degrading silently: unknown keys, duplicate chop identities, invalid durations, and
non-positive intervals become errors with config-path and provenance information; `agent:`/`xprompt:` keys are rejected
with a specific migration message. Duration parsing learns days and compound forms (`1d`, `1h30m`). The JSON schema and
the runtime parser agree.

`chops:` additionally accepts a map keyed by chop name (the legacy list form normalizes into it), so overlays can patch
one field, set `enabled: false`, or override a packaged entry without duplicating it, with per-field provenance.
Lumberjacks gain a lumberjack-level `env:` merged under each chop's `env:` (de-duplicating the Telegram credentials),
and env values accept secret _references_ (resolve from an env var, a file, or `pass` — the Telegram doctor's existing
discovery order) so plaintext values can leave config. With agent chops gone, `env:` and `timeout:` apply uniformly to
every chop, closing the env-parity hole.

### Generic target fan-out (P1-5-inspired, generalized)

A chop may declare `for_each:` with either a literal list of target objects (each an arbitrary key/value map, able to
override `run_every` and other per-chop knobs) or a named **target source**; v1 ships `projects` (enabled sase projects,
with optional name/vcs filters) so "run this chop against all enabled projects" is one config stanza. Expansion produces
stable per-target instance IDs (`refresh_docs[sase-core]`) with per-instance timestamps, run history, checkpoints, and
dedupe state. Target fields flow to the script via the context JSON (`target` object) and `SASE_CHOP_TARGET_*` env. The
source interface is provider-shaped so plugins can add sources later; nothing about it is specific to the all-projects
use case.

### Rust core boundary

Per the core-backend boundary rule, the deterministic domain logic lives in `../sase-core` (`crates/sase_core`) behind
`sase_core_rs`, following the established wire + facade pattern (e.g. the agent-cleanup planner): chop result/proposal
wire types with a schema version, result and proposal validation, guard/trigger decision functions (JSON-in/JSON-out
over snapshot + checkpoint state), checkpoint/once-per bookkeeping transforms, target expansion, and strict axe-config
validation building on the existing `config_validate` machinery. Python keeps IO, YAML decoding, subprocess management,
agent launching, and state-file persistence, calling through a thin facade in `src/sase/core/`.

### What happens to each existing custom chop

| Today                                                                                  | After                                                                                                                                                                      |
| -------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 5× `refresh_docs` agent chops + `sase/xprompts/refresh_docs.yml`                       | One builtin proposal-emitting `sase_chop_refresh_docs` script + `git.commits_since` trigger + `for_each` projects source (sase shared functionality, requested via config) |
| `sase_recent_bug_audit`, `sase_recent_improvement_audit` agent chops + audit workflows | Small bugyi-chops scripts emitting one audit-agent proposal each; threshold/marker logic replaced by the shared trigger                                                    |
| `sase_fix_just` executable + `fix_just.yml` workflow                                   | bugyi-chops script emitting one fixer-agent proposal; the ChangeSpec guard becomes declarative `inhibit_if`                                                                |
| `toobig_split` executable (self-launches `sase run`)                                   | bugyi-chops script emitting per-file proposals with `wait_on` sequencing; hood occupancy check becomes declarative `inhibit_if`                                            |
| `tg_inbound`/`tg_outbound` (sase-telegram)                                             | Unchanged scripts; chezmoi config references them by full name (`script: sase_chop_tg_inbound`) with creds via lumberjack-level env/secret refs                            |

## Phases

### Rust core chop engine and wire types

Greenfield module in `../sase-core` (`crates/sase_core/src/axe_chop/` or similar) plus `sase_core_py` bindings and a
thin facade module in this repo's `src/sase/core/`:

- Wire types (serde, `schema_version` consts, PyO3-free): chop result document, launch proposal,
  guard/trigger/once-per/`for_each` config blocks, decision results (fire / skip+reason / check-error), checkpoint and
  seen-store documents.
- Pure functions exposed via `sase_core_rs`: parse+validate a chop result document (actionable diagnostics, `#!`
  workflow-reference rejection inside proposal prompts, scaffold-derivation helpers for agent names); evaluate
  guards/triggers given config + changespec snapshot + agent snapshot + checkpoint state + now; compute
  checkpoint/seen-store updates for each checkpoint policy; expand `for_each` targets (literal + source-provided rows)
  into stable instance IDs; strictly validate the new `axe:` config shape (unknown keys, duplicates, `agent`/`xprompt`
  rejection with migration hints, enriched duration parsing incl. `d` and compound forms, positive-int intervals) with
  provenance-aware diagnostics, reusing the existing config validation seam.
- Rust unit tests plus Python-side facade tests in this repo (parity style, like existing facades). Cargo versioning
  stays release-plz-owned; do not hand-edit workspace versions.

This phase defines the contracts every later phase consumes; land it first.

### Script-only chops with full-name resolution

Remove the agent-chop code path and the prefix magic in this repo:

- Delete `chop_runner_agent.py` and the `chop.agent` dispatch branch in `chop_runner.py`; drop `agent`/`xprompt` from
  `ChopConfig`, the parser, and `sase.schema.json`; remove the `agent-backed` inventory status,
  `agent_launched`/`agent_failed` outcome statuses, and their CLI/TUI renderings. Keep `chop_agents.py`'s registry (it
  becomes proposal-launch linkage in the next phase) but remove its role as an _input_ discriminator.
- Replace `sase_chop_<name>` resolution with full-name lookup of the new `script:` field (default: `name`) across
  `chop_script_dirs` → interpreter bin dir → `$PATH`. Update `default_config.yml` builtin entries with explicit
  `script: sase_chop_*` values, renaming the `cl_submitted_checks` chop entry to `pr_submitted_checks`. Rework the
  "available scripts" scan: enumerate `chop_script_dirs` contents by bare name and keep a legacy `sase_chop_*` PATH scan
  as a discovery convenience only.
- Route axe config loading through the new core validation so it fails closed, surfacing errors through CLI/doctor with
  config-path + provenance details; `sase axe chop doctor` gains checks for unresolvable `script:` names.
- Migrate the affected tests (runner/find/script-runner/inventory/doctor/config/CLI/TUI, plus the axe dashboard
  snapshots) and update the agent-chop sections of `docs/axe.md` and `docs/configuration.md` enough to stay truthful;
  the full docs rewrite lands with the refresh-docs phase.

### Structured chop results and runner-executed launches

The heart of the redesign, in `run_script_chop_once` and friends:

- Create the per-run result-file path, export `SASE_CHOP_RESULT_FILE` + context field, and parse the document (via the
  core facade) after the process exits. Missing file → legacy exit-code semantics; invalid file → `check_error` with
  diagnostics in the run entry.
- Execute validated proposals sequentially in the runner: scaffold injection (derived agent name, `%tribe:chop`,
  workspace ref), extra env pass-through, `wait_on` sequencing via the existing wait-directive machinery, launch via the
  existing agent launcher, and registry linkage (`agent_chops.json` + `SASE_CHOP_*` env) per proposal.
- Extend persisted run statuses (`no_op`, `check_error`, `launched`, `action_succeeded`, `action_failed`) and add the
  housekeeping pass that finalizes `launched` runs from agent-completion artifacts. Update status renderings across CLI
  (`sase axe chop list/run`), the axe dashboard, and TUI notify paths, including exit-code mapping.
- Debug UX: `sase axe chop run` gains `-n|--dry-run` (run the script, render the parsed result and a proposals table,
  launch nothing) and `-V|--chop-verbose` (sets `SASE_CHOP_VERBOSE`, prints the full result document and per-proposal
  validation detail). Follow the CLI rules memory (short aliases, sorted help, colored output).
- Structured results are stored alongside the run entry so the TUI can distinguish a healthy no-op from a degraded check
  from real action — with tests at each layer.

### Chop SDK and builtin script consolidation

P2-10, plus the shared plumbing external chop authors need:

- New small `sase.chops.sdk` (name at implementer's discretion) covering: `--context`/`--verbose` arg handling, context
  loading, leveled logging that honors `SASE_CHOP_VERBOSE` (debug lines to stderr), the shared `key=value … reason=…`
  summary emitter, and an atomic result-document writer/builder with proposal helpers. Public enough for bugyi-chops to
  depend on.
- A registry/decorator for builtin chops that folds the cloned preamble (argparse, context read, `log` closure,
  `HookJobRunner` construction) so a new builtin needs one module + one entry-point line; migrate the 13
  `src/sase/scripts/sase_chop_*.py` modules onto it and have them emit structured results (counters/reason) alongside
  their existing summaries.
- Retire the `sase_chop_cl_submitted_checks` console-script alias (config already renamed in the script-only phase).
  Update the chop output-contract tests to the SDK-emitted format.

### Declarative triggers, guards, and dedupe keys

Wire the core decision engine into the lumberjack tick and manual runs:

- Config: `inhibit_if` (changespec, agent-hood providers), `trigger` (`always`, `git.commits_since` with checkpoint
  policies), `once_per` (per-proposal key templates + script-supplied `dedupe_key`), all validated fail-closed in the
  core engine from phase 1.
- Runner-owned checkpoint and seen-store state under the chop state dir, updated per the configured checkpoint policy
  (`on_action_success` finalizes from the lifecycle housekeeping pass), replacing today's ad-hoc markers with a single
  bookkeeping path.
- Skips write `skipped` run entries with provider reasons, visible in CLI/TUI history; manual runs bypass triggers but
  honor guards unless `--force` is passed (flag design per CLI rules memory).
- Doctor: validate trigger/guard config against known providers, project refs, and checkpoint policies; per-chop "why
  didn't this fire?" is answerable from `sase axe chop list -v` output.

### Keyed composition, env parity, and target fan-out

P1-6 plus the generalized P1-5 mechanism:

- Map-form `chops:` (normalized with the legacy list), `enabled: false` soft-disable, single-field overlay patches,
  duplicate-identity errors, and per-field provenance surfaced through the config inventory.
- Lumberjack-level `env:` merged beneath chop `env:`; secret-reference values (env var, file, `pass`) resolved at
  dispatch by shared code with the Telegram doctor updated to recommend them.
- `for_each:` fan-out: literal target lists and the `projects` source (enabled projects via the project registry, with
  optional filters), core-owned expansion into stable instance IDs, per-instance scheduling/state/history, `target` in
  context JSON + `SASE_CHOP_TARGET_*` env, and per-target overrides for fields like `run_every`. Inventory/doctor/TUI
  list instances under their parent chop.

### Builtin refresh-docs chop and xprompt workflow retirement

Translate the last workflow into shared functionality and finish the retirement:

- New builtin `sase_chop_refresh_docs` script (SDK-based): given a target project, emit the update+polish agent-proposal
  pair (polish `wait_on` update) with a sensible default docs-refresh prompt template overridable via config vars;
  cadence/thresholds come from `run_every` + `git.commits_since` + `for_each` targets, not from the script.
- Delete `sase/xprompts/refresh_docs.yml`, `audit_recent_bugs.yml`, `audit_recent_improvements.yml`, and `fix_just.yml`;
  migrate the workflow-loader, launcher- barrier, config, and notification-fixture tests that used them as fixtures onto
  synthetic fixtures.
- Documentation sweep for the whole redesign: rewrite the chops sections of `docs/axe.md` and `docs/configuration.md`
  (new contract, result document, proposals, triggers/guards, targets, map-form chops, debugging), update
  `docs/xprompt.md`'s scheduled-workflow table, `docs/plugins.md`'s chop-script guidance (full-name console scripts,
  SDK), and the JSON schema.

### bugyi-chops plugin package

Work happens in the bbugyi200/bugyi-chops repo (opened through the repo-access skill):

- Python package `bugyi-chops` depending on `sase` for the SDK; console scripts (full names, e.g.
  `bugyi_chop_toobig_split`, `bugyi_chop_fix_just`, `bugyi_chop_recent_bug_audit`,
  `bugyi_chop_recent_improvement_audit`) that scan/decide and **emit proposals only**:
  - `toobig_split`: port the oversized-file scan; per-file proposals with `wait_on` chaining and `%auto`; drop the
    self-`sase run`, the flock, and the hood-occupancy check (declarative `inhibit_if: agent_hood` + runner dedupe
    replace them).
  - `fix_just`: one fixer-agent proposal (the launched agent runs the just checks in its workspace and fixes
    fmt/lint/tests, using `#pr` rollovers); the ChangeSpec guard moves to `inhibit_if`.
  - The two audits: port the audit prompts from the retired workflows into one-proposal scripts (with `#pr(...)`
    rollovers); thresholds/markers come from the shared trigger config.
- Port the chezmoi bash regression tests into the package's own test suite (pytest preferred); add CI plus a tag-driven
  PyPI publish workflow (trusted publishing).
- Excellent README: what chops are, the result-document contract, per-chop docs with example lumberjack config stanzas
  (triggers/guards/targets), install via `sase plugin install`, and a debugging section (`sase axe chop run -n -V`,
  direct script invocation with `--verbose`).
- Add the `sase--plugin` repository topic (and a proper repo description) so the package appears in `sase plugin list`
  as a community plugin.

### Chezmoi config migration and end-to-end verification

Work happens in the chezmoi repo (opened through the repo-access skill); config files only — no memory or
agent-instruction files:

- Rewrite `home/dot_config/sase/sase.yml` + `sase_athena.yml` `axe:` sections onto the new schema: map-form chops with
  full `script:` names, `tg_*` rows on `script: sase_chop_tg_*` with lumberjack-level env/secret refs, `refresh_docs` as
  one builtin-chop stanza fanned out over enabled projects (preserving today's per-project thresholds/cadences via
  target overrides), the audits/fix_just/toobig_split rows pointing at bugyi-chops scripts with declarative
  triggers/guards, and no `agent:` keys anywhere.
- Delete `home/bin/executable_sase_chop_sase_fix_just`, `executable_sase_chop_toobig_split`, and the migrated bash tests
  (`sase_fix_just_chop_test.sh`, `toobig_split_chop_test.sh`, and the agent-chop barrier test if fully superseded);
  apply chezmoi per that repo's instructions after committing.
- End-to-end verification on the new stack: fail-closed config errors on a deliberate typo, `sase axe chop doctor`
  clean, `sase axe chop run -n -V` for each migrated chop showing sane proposals, one real manual run per family,
  trigger/guard skip reasons visible in history, and the lifecycle reaching `action_succeeded` on a completed launch.

## Testing strategy

Every phase lands with its own tests: Rust unit tests + Python facade parity tests (core-engine);
runner/config/inventory/doctor/CLI/TUI test migrations (script-only); result-parsing, proposal execution, lifecycle
finalization, and dry-run/verbose CLI tests (result-protocol); SDK + output-contract tests (chop-sdk);
trigger/guard/once-per decision + skip-history tests (triggers-guards); map-form normalization, provenance, secret-ref,
and target-expansion tests (composition-targets); loader-fixture migrations + builtin refresh-docs tests
(refresh-docs-builtin); the ported bash-regression suite (bugyi-chops); and scripted end-to-end checks
(chezmoi-cutover). The chezmoi bash tests are the richest acceptance suite for the migrated chops — port their cases,
don't drop them. `just check` gates each sase-repo phase; `cargo` fmt/clippy/test gate the core phase.

## Risks and mitigations

- **Scope breadth (three external repos + Rust core).** Mitigated by strict phase ordering, the contracts all being
  defined in phase 1, and sase-repo phases staying shippable behind backward-compatible config (legacy list form and
  exit-code-only scripts keep working until the cutover phase).
- **Fail-closed validation breaking existing configs at upgrade time.** The chezmoi cutover phase is sequenced last and
  the validation errors must carry migration hints (especially for `agent:`/`xprompt:` and prefixed script names); the
  runtime keeps running other lumberjacks when one lumberjack's config is invalid.
- **Lifecycle finalization correctness** (linking proposals → agents → terminal state) reuses the existing registry +
  done-marker artifacts rather than a new channel, and the housekeeping pass must tolerate dead PIDs and missing
  artifacts (statuses degrade to `action_failed` with a reason, never hang in `launched`).
- **PyPI naming/publishing** for `bugyi-chops` assumes the name is free; if taken, fall back to `bugyi200-chops` and
  record the choice in the repo README.
- **sase-core release coupling:** the sase repo's `sase-core-rs` version pin must move in step with the new bindings;
  the core phase should land wire types + bindings in one release train per that repo's release-plz conventions.
