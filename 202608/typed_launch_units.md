---
tier: epic
title: Conditional launch admission and stand-alone proc launch units
goal:
  SASE accepts typed Agent and stand-alone Proc launch units with durable %if admission,
  first-class %proc execution, shared prompt-widget and LSP authoring assistance, and an
  attractive Agents-tab proc-shell experience without conflating procs with agents.
phases:
  - id: launch-code-contract
    title: Gated code directives and shared fenced-code contract
    depends_on: []
    size: medium
    description:
      "launch-code-contract: create the typed_launch_units beta gate and one Rust-owned
      CodeValue, directive grammar, fence scanner, and code-input wire shared by runtime
      and editor surfaces."
  - id: typed-launch-graph
    title: Typed mixed-unit planning and wait graph
    depends_on:
      - launch-code-contract
    size: medium
    description:
      "typed-launch-graph: replace agent-shaped fanout planning with a pure,
      schema-versioned Agent-or-Proc launch graph whose waits, conditions, identifiers,
      validation, and preview are fixed before approval."
  - id: durable-launch-admission
    title: Durable launch admission coordinator
    depends_on:
      - typed-launch-graph
    size: medium
    description:
      "durable-launch-admission: persist and supervise approved launch-unit outcomes,
      resolving waits before conditions and resources while preserving the existing
      agent launch path."
  - id: conditional-runtime
    title: Sandboxed conditional admission runtime
    depends_on:
      - durable-launch-admission
    size: medium
    description:
      "conditional-runtime: evaluate approved Bash or Python %if programs with bounded
      resources and typed context, and settle false predicates as durable skipped
      outcomes without allocating an agent, proc, workspace, or runner."
  - id: standalone-proc-runtime
    title: Native stand-alone proc runtime
    depends_on:
      - durable-launch-admission
      - conditional-runtime
    size: medium
    description:
      "standalone-proc-runtime: dispatch %proc units through native proc-shell identity,
      deferred operational workspaces, private scripts, sanitized environments,
      responsive cancellation, and crash-safe settlement."
  - id: directive-authoring-experience
    title: Prompt-widget and LSP authoring experience
    depends_on:
      - typed-launch-graph
    size: medium
    description:
      "directive-authoring-experience: expose snippets, clause completion, hover,
      diagnostics, navigation, and code-input assistance from the shared contract with
      prompt-widget/LSP parity and no keystroke-path I/O."
  - id: agents-proc-shell-experience
    title: Beautiful stand-alone proc shells in the Agents tab
    depends_on:
      - standalone-proc-runtime
    size: medium
    description:
      "agents-proc-shell-experience: project native proc-store records into responsive,
      visually distinct Agents-tab rows and details without creating agent artifacts,
      occupying agent slots, or corrupting agent counts."
  - id: mixed-launch-verification
    title: Integrated rollout, documentation, and verification
    depends_on:
      - conditional-runtime
      - standalone-proc-runtime
      - directive-authoring-experience
      - agents-proc-shell-experience
    size: medium
    description:
      "mixed-launch-verification: exercise the complete mixed-unit matrix, both
      feature-flag states, recovery and performance contracts, public documentation,
      approved memory regeneration, and full cross-repository checks."
proposed_by: bbugyi200.athena.0b8
bead_id: sase-s6
create_time: 2026-09-09 19:52:00
status: wip
---

- **PROMPT:**
  [prompts/202608/typed_launch_units.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/typed_launch_units.md)
- **BEAD:**
  [sase-s6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-s6/README.md)

# Plan: Conditional launch admission and stand-alone proc launch units

## Sources and verified current state

This epic implements the contracts in these audited research artifacts from the
`sase--research` sidecar:

- `research:202608/standalone_proc_launch_units/standalone_proc_launch_units.md`
- `research:202608/conditional_launch_admission/conditional_launch_admission.md`

The current launch path expands a prompt into agent-shaped segments in Python, assigns
agent names, applies agent-only normalization, and only then asks the Rust core for
fanout planning. Bare `%wait` is rewritten to the preceding agent name. That ordering
cannot represent a proc without inventing an agent, and it cannot prune a skipped unit
without changing dependency meaning.

SASE already has the important lower-level proc primitives: native `proc-shell` store
records, detached supervision, startup acknowledgement, bounded logs, timeouts,
operational workspace leases, resumable settlement, and a Procs pane. Existing monitor
rows in the Agents tab are projections of agent-owned proc shells, however; they are not
the first-class, family-independent proc launch unit required here.

The Rust core already owns the shared directive metadata consumed by the Python prompt
widget and `sase_xprompt_lsp`, but the Python expansion parser and the Rust editor
parser still have separate assumptions about ordinary fenced Markdown. Xprompt inputs
are scalar-only. This epic therefore starts with one canonical code/fence contract, then
uses it for `%if`, `%proc`, public `type: code` inputs, completion, diagnostics, and
runtime execution.

## Product contract and design decisions

### One typed launch model

Expansion produces a pure, immutable, schema-versioned launch plan before approval:

```text
prompt / xprompts
       |
       v
Rust fenced-code scan + typed LaunchPlan validation
       |
       v
approval preview: every unit, wait, condition, cwd, code digest, and resource intent
       |
       v
durable admission coordinator
       |
       +--> waits --> optional condition --> Agent dispatcher
       |
       `--> waits --> optional condition --> Proc supervisor --> child process
```

`LaunchUnit` is a tagged `AgentUnit | ProcUnit`, not an agent record with optional proc
fields. Every unit receives a stable logical ID during pure planning. Explicit and bare
waits resolve to those IDs before conditional pruning; a skipped predecessor remains a
terminal outcome, and its dependents never retarget to some other unit. Agent names and
proc IDs remain runtime identities layered on top of logical IDs.

The planner validates the complete expanded graph before any approval or child spawn:
all directive legality, body cardinality, aliases, IDs, project contexts, wait targets,
cycles, fanout, resource clauses, and condition/proc code must be valid together. An
approval authorizes exactly the displayed plan, identified by a content digest. Runtime
launch can still partially succeed after approval, but every terminal result is recorded
per logical unit and reported as eligible, launched, skipped, condition-error, or
launch-error rather than implied by an agent count.

### `%if` syntax and semantics

V1 accepts only a directive-owned fenced block:

````markdown
%if::

```bash
test -f pyproject.toml
```

#work
````

- Exactly one closed fence follows `%if::`; intervening blank lines are allowed.
  Unlabelled and `bash` fences mean Bash; `python` means Python. Unknown info strings,
  multiple fences, trailing condition text, and `%if:`/parenthesized forms fail with an
  actionable diagnostic.
- The block is an opaque `CodeValue`. Directive-looking text, xprompt references,
  frontmatter delimiters, Jinja, command substitutions, and nested prose inside it are
  never expanded or scanned as launch syntax.
- `%if` attaches to the one following logical launch unit after xprompt expansion and
  fanout. Repeated or ambiguous attachment is rejected. `%repeat` and `%alt` are
  expanded by the typed planner, so every resulting conditioned unit has a stable,
  previewable logical ID.
- Admission order is fixed: reserve the logical outcome, wait for dependencies, run the
  predicate, and only if eligible acquire a runner or workspace and launch. Exit 0 is
  eligible, exit 1 is skipped, and any other exit, signal, timeout, malformed result, or
  execution failure is a condition error.
- Predicates default to a ten-second wall timeout, execute in a fresh process group,
  receive bounded stdout/stderr, and are killed as a group on timeout or cancellation.
  They use a minimal sanitized environment and a private `0600` script. Python uses the
  SASE interpreter; Bash uses `/bin/bash --noprofile --norc` and does not inject strict
  mode.
- The approval preview names the recorded source cwd and exposes only a documented
  `SASE_CONDITION_CONTEXT` JSON file. Its versioned payload contains the logical unit,
  selected project, safe expanded inputs, and terminal waited outcomes plus any
  explicitly shareable output/workspace references. It contains no secrets, raw agent
  artifacts, implicit success policy, or unapproved environment inheritance. Success
  requirements belong in the predicate itself.
- A false predicate creates no agent directory, proc reservation, runner allocation,
  workspace lease, model request, finalizer, or fake completion artifact. It is still a
  durable terminal outcome for ordering and UI/CLI summaries.

### `%proc` syntax and semantics

V1 accepts these equivalent body styles:

````markdown
%proc("just check") %proc(bash="just check", timeout="20m", label="Scoped verification")
%proc(python="print('ready')", workspace=false, cwd="/tmp/project") %proc(timeout="20m",
idle_timeout="5m")::

```bash
just check
```
````

- A proc has exactly one body: one positional string, one `bash=` or `python=` value, or
  one directive-owned fence after `::`. Duplicate bodies, empty code, unknown
  language/info strings, residual prompt prose, multiple `%proc` directives in one
  logical unit, and unknown/repeated/conflicting options are hard errors.
- Options are `timeout`, `idle_timeout`, `cwd`, `workspace`, and `label`. Durations use
  the existing SASE duration grammar; `workspace` is a Boolean. `label` is descriptive
  and never identity.
- In project context, `workspace` defaults true. The proc supervisor acquires an
  operational lease only after waits and `%if` pass; an optional relative `cwd` is
  resolved beneath that leased checkout. `workspace=false` opts out and validates an
  ordinary cwd. Without project context, no lease is taken and an explicit ordinary cwd
  is required. `workspace=true` without a project fails; workspace 0 or another fallback
  is never guessed.
- `%id:name`/`%id(name)` is optional and becomes the proc's validated bare `shell_name`;
  the canonical proc ID is still allocated by the proc store. `%wait` may target
  `proc=<id-or-shell-name>`. Proc names cannot use the agent-family `--` convention.
- Agent-only directives (`%model`, `%effort`, `%auto`, `%final`, `%clan` and its
  aliases, agent runners/priority, and `%hide`) are rejected on a proc with a targeted
  explanation. Project references, `%wait`, `%if`, `%repeat`, `%alt`, and the optional
  proc `%id` work through the typed plan rather than agent normalization.
- A stand-alone proc is stored as lifecycle `proc-shell`, origin `xprompt-proc`, with
  native proc status, phase, identity, request fingerprint, code digest, safe preview,
  selected project, cwd/workspace intent, waits, condition outcome, timing, supervisor,
  and settlement metadata. It never creates agent artifacts, `done.json`, a synthetic
  agent name, family membership, or a finalizer obligation.
- The proc supervisor materializes the approved source as a private `0600` script in
  proc-owned runtime storage and executes an argv vector, never shell interpolation:
  `/bin/bash --noprofile --norc <script>` or `<sys.executable> <script>`. Its
  environment is sanitized and adds only documented proc context such as `SASE_PROC_ID`,
  project, project file, workspace, and a SASE-interpreter path prefix; it never sets
  `SASE_AGENT` or agent-artifact variables.

### Dependency and recovery contract

The wait target is a tagged value: logical launch unit, external agent, proc, bead, or
time. Bare `%wait` binds to the immediately preceding typed logical unit during
planning. Explicit `%wait(agent=...)`, `%wait(proc=...)`, bead/time waits, and mixed
forward/backward references have deterministic validation and cycle rules. Waiting
consumes neither agent runners nor workspace leases.

Approval starts one detached, durable launch-admission coordinator whose receipt lives
with the existing launch request rather than as an agent or proc row. It journals each
unit through reserved, waiting, checking, skipped/error, dispatching, and launched
states. Restarts replay idempotently: conditions are not silently re-run after a
persisted terminal result, proc IDs are not duplicated, and agent/proc dispatch uses a
stable request fingerprint. Settlement ownership is recorded before child execution.
Cancellation is responsive in wait, condition, workspace-acquisition, startup, child,
and settlement phases.

For procs, workspace order is wait, condition, lease, prepare script, start child,
settle, release. The supervisor must bind its PID and durable settlement policy before
starting the child so no crash window can orphan a lease. Execution timeouts begin when
the child starts, not while dependencies or the lease are pending.

### Feature rollout

Create an off-by-default beta flag named `typed_launch_units` only with `sase flag new`,
including its mandatory removal bead. The disabled state hides `%if` and `%proc` from
completion and rejects explicitly authored uses with a precise message; it must never
pass the directive text through to a model. The enabled state turns on planning and
execution for both directives as one coherent feature. Editor and TUI processes consume
immutable, startup-resolved flag state—no feature-flag I/O occurs on the keystroke path.

Retire the flag only after mixed Agent/Proc launches, skipped admission, recovery, and
both editor surfaces have operational evidence. Tests cover both flag states throughout
the epic.

## Phase 1: Gated code directives and shared fenced-code contract

Work in `sase` and the linked `sase-core` repository, opening the latter through
`/sase_repo`.

- Run `sase flag new typed_launch_units` with a beta description covering `%if`,
  `%proc`, and typed launch units. Accept only the CLI-generated registry/test/bead
  changes; do not hand-author the registry. Thread the immutable decision through launch
  entry points, ACE editor assistance, helper payloads, and LSP startup config.
- Add a reusable Rust `CodeValue { source, language, info_string }`, supported-language
  enum, normalized digest, safe one-line preview, and additive versioned wire. Define
  one CommonMark-compatible directive-owned fence scanner with exact ownership,
  indentation, closure, info-string, and source-span rules. It must run before ordinary
  literal-zone protection on every recursive xprompt expansion pass.
- Extend the shared directive registry with gated `%if` and `%proc` entries, legal
  forms, attachment/body rules, option metadata, value roles, synopsis, examples, and
  documentation text. The registry is descriptive and side-effect free; execution code
  must not duplicate grammar tables.
- Teach the Python directive parser to consume Rust-scanned spans and structured code
  wires instead of heuristically stripping the fence first. Preserve exact source for
  approval and diagnostics while removing owned code from subsequent xprompt/Jinja/
  directive scans. Reject incomplete or multiply-owned fences deterministically.
- Add `type: code` to the Rust xprompt frontmatter/catalog schema and Python `InputType`
  transport, but keep it gated/internal until prompt widget, LSP, binding, validation,
  rendering, and helper parity land in Phase 6. A code input is structured source plus
  language, not a plain string with a convention.
- Test literal `%`, `#`, `---`, `{{ }}`, `$()`, backticks, indentation, blank lines,
  CRLF, missing/extra fences, unknown languages, nested expansion, schema compatibility,
  feature-off explicit rejection, and absence of raw directive leakage. Run core
  formatting/lints/tests and focused Python parser/flag tests.

## Phase 2: Typed mixed-unit planning and wait graph

Keep shared backend/domain behavior in `sase-core`; Python should be a thin adapter.

- Replace the agent-only planning wire with an additive schema-versioned `LaunchPlan`
  containing stable logical IDs, source order, selected project, typed waits, optional
  condition, and tagged `AgentUnit`/`ProcUnit` payloads. Preserve compatibility by
  translating old all-agent requests at the binding edge while callers migrate.
- Move classification immediately after recursive xprompt expansion and before agent
  names, model/provider defaults, VCS/artifact preparation, runner normalization, or
  workspace allocation. Each payload receives only its legal normalization pass.
- Parse `%proc` bodies/options and `%if` attachment into typed values. Resolve explicit
  and bare waits, `%repeat`, `%alt`, logical IDs, proc `%id`, agent names, projects, and
  fanout into the final static graph. Do not let skipped runtime outcomes change IDs or
  wait targets.
- Define `WaitTarget` and `LaunchOutcome` enums and a public receipt/result wire. Extend
  `%wait(proc=...)` validation and cached completion identifiers without making old
  agent, bead, or time forms ambiguous. Document terminal ordering separately from
  success requirements.
- Validate the entire graph in Rust: unknown directives/options, body cardinality,
  illegal cross-kind clauses, identifier collisions, name ambiguity, project/cwd/
  workspace policy, forward references, cycles, impossible targets, and unsupported
  fanout. Return stable codes and source spans suitable for CLI, ACE, and LSP.
- Render one deterministic approval model from the plan. Show unit kind and logical ID,
  agent/proc identity intent, project, waits, condition language/source/digest/cwd/
  context fields, proc language/source/digest/options, runner/workspace intent, fanout,
  and every rejected warning. Planning and preview are pure: no proc reservation,
  workspace lease, agent directory, subprocess, or condition execution.
- Add Rust/property tests for mixed graphs, all wait directions, bare-wait binding,
  forward references, cycles, fanout stability, directive-order permutations, project
  rules, proc option conflicts, illegal agent directives, stable serialization, and
  golden approval previews. Add Python binding/parity tests and migrate all-agent
  callers without changing their public behavior.

## Phase 3: Durable launch admission coordinator

- Extend the existing launch-request/LaunchApproval payload and response with the plan
  digest, schema, typed preview, and per-unit result summary. Old all-agent approvals
  remain readable and dispatch through the compatibility adapter.
- After approval, start a detached launch-admission coordinator with an explicit startup
  acknowledgement and persisted request sidecar. It is infrastructure—not a proc shell,
  agent, or Agents-tab row—and must use the existing launch request's lifecycle and
  notification ownership.
- Journal per-unit state transitions and stable dispatch fingerprints atomically. On
  restart, reconcile persisted agent/proc identities and terminal outcomes before
  deciding whether to wait, evaluate, or dispatch; make duplicate coordinators and
  partial writes safe and observable.
- Implement typed dependency resolution for logical units and existing external agent,
  proc, bead, and time targets. Capture waited terminal outcomes for condition context.
  Waiting must not hold agent runners, priority slots, workspace leases, proc records,
  or provider capacity.
- Preserve the established agent launch path after admission: only an eligible
  `AgentUnit` gets name allocation, provider/model defaults, artifacts, VCS context,
  runner/priority handling, spawn, and finalizer behavior. Keep all-agent performance
  and results unchanged when the feature is disabled.
- Report batch summaries as total, eligible, launched, skipped, condition errors, and
  launch errors in CLI/ACE notifications. Partial runtime failure must not erase
  already-launched unit identities or collapse a condition error into generic dispatch
  failure.
- Test approval rejection/cancellation, daemon startup failure, coordinator crash at
  every journal boundary, replay, duplicate dispatch prevention, external wait targets,
  no-resource waiting, partial success, old request compatibility, and kill escalation.

## Phase 4: Sandboxed conditional admission runtime

- Implement one reusable condition evaluator around `CodeValue`. Materialize a private
  script, construct argv without interpolation, sanitize the environment, enforce the
  default/maximum timeout and output caps, supervise a new process group, and preserve a
  bounded diagnostic tail plus full approved source digest in the receipt.
- Generate the versioned `SASE_CONDITION_CONTEXT` JSON file only after waits settle.
  Include safe expanded xprompt inputs, logical/project identity, and normalized waited
  outcomes; expose workspace/output references only when they already exist and are
  explicitly allowed by the launch contract. Never serialize secrets or arbitrary agent
  environment/artifact content.
- Persist checking start, evaluator PID/process group, code digest, context digest, exit
  classification, timestamps, and bounded diagnostics before advancing. A crash after a
  terminal predicate result must not re-run arbitrary code; an ambiguous in-flight crash
  settles as a condition error unless recovery can prove the original process and
  result.
- Implement exact exit semantics: 0 eligible, 1 skipped, every other exit/signal/
  timeout/exec failure an error. A skipped outcome is terminal for dependency ordering
  and is never silently reclassified as successful execution.
- Ensure false/error/cancelled conditions acquire no downstream runner or workspace and
  create no agent/proc identity or artifact. Exercise signals during waiting and
  checking, timeout races, output truncation, missing interpreters, malformed context,
  source-cwd disappearance, and cleanup of scripts/context files.
- Add end-to-end tests for agent units first and generic coordinator tests that the proc
  dispatcher will inherit: all exit classes, Bash/Python parity, literal code, waited
  outcomes, fanout, external dependencies, full-plan validation before any spawn, and
  no-resource skip/error invariants.

## Phase 5: Native stand-alone proc runtime

- Extend proc request/store wires additively for origin `xprompt-proc`, code language/
  digest/safe preview, label, selected project, cwd/workspace intent, logical launch
  receipt, waits, condition result, and phases `waiting`, `checking`,
  `acquiring-workspace`, `preparing-script`, `running`, and `settling`. Keep native proc
  ID allocation authoritative and validate optional bare shell names independently of
  agent-family names.
- Give the coordinator a durable reserve/dispatch handshake: reserve the proc identity
  only after admission passes, write the exact approved request fingerprint, start the
  supervisor, and reconcile acknowledgement without duplicate reservations or children.
  The supervisor must own settlement before it acknowledges readiness to run.
- Refactor operational lease support so the proc supervisor—not the submitting ACE/ CLI
  process—waits and acquires after admission. Atomically bind supervisor PID, settlement
  policy, and lease before child spawn; release through existing resumable settlement on
  every terminal path. Never substitute workspace 0.
- Materialize the Bash/Python source as `0600` under proc runtime storage, validate the
  approved digest, set the sanitized documented environment, execute argv directly, and
  remove private inputs according to a documented retention policy while retaining the
  digest and bounded safe preview.
- Make stop/kill responsive in coordinator wait, workspace acquisition, script prep,
  startup, child execution, and settlement. Execution and idle timers begin at child
  start. Preserve bounded logs and phase timestamps for UI observation and recovery.
- Ensure proc launches do not allocate runners, agent artifacts, agent names, families,
  completion/finalizer files, or agent counts. Existing agent-owned monitor procs and
  ordinary Procs-pane launches must retain their semantics.
- Test every workspace matrix entry, relative cwd containment/symlink escape, no-project
  validation, shell-name ambiguity, interpreter argv/env, file modes, timeouts, output
  bounds, kill races, startup failure, crash windows around lease/policy/child,
  settlement replay, and mixed Agent/Proc dependency ordering.

## Phase 6: Prompt-widget and LSP authoring experience

The Rust directive contract is the single source of truth; ACE and the LSP render it for
their capabilities rather than maintaining parallel directive lists.

- Extend the contract with snippet/recipe alternatives, cursor/placeholder locations,
  legal forms, option ordering, mutual exclusions, repeatability, value roles, language
  choices, flag availability, and concise examples. Provide at least: `%if::` plus
  Bash/Python fence templates; positional, named, and fenced `%proc` templates; proc
  option clauses; `%wait(proc=...)`; and fenced `type: code` input authoring.
- In the prompt input widget, insert balanced, correctly indented templates with the
  cursor inside the code/body/value. Complete valid remaining proc options and values,
  suppress duplicates/conflicts, offer Bash/Python, durations/booleans, and present
  static syntax even when dynamic identifiers are unavailable. Use the existing cached
  helper/native inventories for proc IDs and shell names; helper refresh stays
  off-thread and never blocks typing.
- In `sase_xprompt_lsp`, expose equivalent snippet completions with placeholders,
  trigger characters, completion resolve/documentation, hover examples, semantic tokens
  for directive-owned code, and signature/option guidance. Respect clients that do not
  support snippets by emitting a useful plain-text insertion.
- Add grammar-aware diagnostics and source spans for incomplete/unclosed fences,
  misplaced/repeated `%if`, invalid proc bodies/options/languages/durations/booleans,
  illegal agent-only directives, ambiguous wait targets, residual prose, feature-off
  use, and unsupported `type: code` locations. Add safe code actions for completing a
  fence or converting a simple proc form, never for executing/enabling arbitrary code.
- Complete public `type: code` transport: frontmatter validation, prompt binding,
  preview, helper/native catalogs, syntax highlighting, external-editor completion, and
  lossless Bash/Python values. Do not expose the type in public completion until every
  surface handles it.
- Maintain prompt-widget/LSP parity tests generated from the shared contract, including
  every directive recipe and option. Add cursor-position snapshots, plain-text fallback,
  feature-on/off visibility, dynamic-proc helper outage, document invalidation, literal
  fence safety, and a keystroke-path test proving no disk scan, subprocess, provider
  call, or project discovery occurs synchronously.

## Phase 7: Beautiful stand-alone proc shells in the Agents tab

Stand-alone proc shells are visible alongside agents because that is where users watch
live work, but they remain a distinct presentation kind backed only by the proc store.

- Add a presentation-only `PROC_SHELL` row/detail adapter sourced from cached proc-store
  snapshots. Select stand-alone `xprompt-proc` records, preserve existing agent-owned
  monitor projections, deduplicate by native proc ID, and merge/sort/group off the UI
  event loop. Do not fabricate `Agent`, artifact, clan, or workflow records.
- Render a compact but unmistakable row: a gear/terminal glyph, shell name or short proc
  ID, optional label, Bash/Python badge, current phase/status, elapsed time, and
  project. Use the existing theme with calm cyan/blue identity, active amber motion,
  green clean completion, red failure, and subdued skipped/cancelled states; retain
  meaning in monochrome and via text/glyph, not color alone.
- Keep hierarchy intuitive: the proc is a top-level work item, never indented under a
  family. Group and filter by status/project/date using the Agents tab's existing
  interaction model. Header summaries report separate values such as
  `3 agents · 2 procs`; procs do not change agent runner, unread, clan, or family
  counts.
- Build a dedicated proc-shell detail composition with a `PROC SHELL` identity header,
  status/phase timeline, project/workspace/cwd, language, code digest and syntax-colored
  safe preview, waits and condition result, timeouts, timestamps, supervisor identity,
  settlement state, and a bounded live-log tail. Never expose private scripts, context
  files, secrets, or unbounded output.
- Route stop/kill through the native proc service with confirmation and tracked
  operation feedback; enable actions only for valid phases. Add copy-proc-ID, copy-log-
  path, and jump-to-Procs-pane actions where those concepts already exist, while keeping
  keybindings and help text consistent with neighboring rows.
- Reuse the proc observer/cache and patch only changed rows/details. Load/reconcile
  off-thread, preserve selection by stable proc ID, debounce detail refresh, pause work
  for hidden panes, and keep stale snapshots visible with a subtle freshness/error
  indicator instead of flashing empty.
- Add model/projection/action tests plus inspected PNG snapshots for mixed agent/proc
  groups, long labels, Bash/Python, every active/terminal state, hidden pane refresh,
  narrow and wide terminals, empty/error/loading states, selection stability, and log
  truncation. Measure refresh and navigation p50/p95 against an agent-only baseline and
  keep hot paths within the existing TUI performance budgets.

## Phase 8: Integrated rollout, documentation, and verification

- Exercise representative mixed submissions end to end: Agent→Proc, Proc→Agent,
  Proc→Proc, skipped predecessor→dependent, condition error, explicit external waits,
  forward references, project fanout, `%repeat`/`%alt`, cancellations in every phase,
  and a multi-unit plan where one runtime launch fails after another succeeds.
- Assert that approval preview, coordinator receipt, CLI/notification summary, proc
  store, Agents tab, Procs pane, and editor diagnostics agree on logical IDs, native
  identities, code digests, condition outcomes, and terminal counts. Add recovery
  scenarios that kill/restart the coordinator and proc supervisor at each durable
  boundary and prove no duplicate child, proc, workspace lease, or predicate execution.
- Run security-focused tests for source/path traversal, symlink cwd escapes, script
  permissions, inherited environment, secret/context redaction, argv injection,
  decompression/size limits, output caps, timeout/kill escalation, malicious fence
  contents, and approval-plan digest mismatch.
- Document `%if`, `%proc`, `type: code`, wait semantics, approval behavior, workspace
  rules, exit classifications, feature enabling, recovery, CLI results, and Agents-tab
  presentation in public user and architecture docs. Include concise copyable examples
  and failure messages rather than relying on research artifacts as user documentation.
- The current glossary says a proc shell belongs to an agent and therefore conflicts
  with stand-alone proc shells. Canonical `sase/memory/*.md`, `AGENTS.md`, and provider
  shims must not be edited based only on this plan. Before changing the Proc/Proc Shell/
  launch-unit glossary and xprompt memory, obtain explicit user permission in that
  implementation conversation; once granted, edit the canonical notes and run
  `sase memory init`, review generated outputs, and require `sase memory init --check`
  to be clean. If permission is withheld, use `/sase_new_task` to record the stale
  memory with the required fields instead of silently changing it.
- Verify the disabled beta state preserves existing all-agent launches, rejects explicit
  `%if`/`%proc` before model dispatch, hides them from completion, and shows no proc
  rows from this origin. Verify the enabled state across CLI, ACE prompt input, external
  LSP clients, approval, runtime, Agents, Procs, recovery, and helper-outage fallbacks.
- In each repository run its formatter, lints, unit/integration tests, and
  binding/schema compatibility gates. In `sase`, run `just install`, then
  `just check-full` only through `/sase_monitor` because this touches the Rust binding,
  launch broadening set, TUI, and stable wires; run and inspect `just test-visual`. Run
  full `sase-core` checks and LSP harnesses. Revalidate documentation examples against
  the shipped CLI.

## Acceptance criteria

- `%if::` and all supported `%proc` forms parse from one Rust-owned
  fenced-code/directive contract, remain opaque through recursive expansion, and are
  rejected precisely while `typed_launch_units` is disabled.
- Every submission is completely and purely validated into stable typed Agent/Proc
  logical units before approval; preview displays the exact approved code, digests,
  dependencies, cwd/context, and resource intent, and bare waits never retarget after a
  skip.
- Conditions run only after waits and before runners/workspaces, exit 1 creates a
  resource-free terminal skipped outcome, all other abnormal results are condition
  errors, and durable recovery never silently re-executes a settled predicate.
- Stand-alone procs use native proc identity and supervision, private argv-executed
  scripts, deferred operational workspaces, sanitized environment, bounded logs,
  responsive cancellation, and crash-safe settlement—with no synthetic agent artifact,
  runner slot, family, or finalizer.
- The prompt input widget and LSP offer equivalent polished snippets, option/value
  completion, hover, diagnostics, and code-input assistance. Feature state and dynamic
  inventories are cached; typing never performs synchronous disk, subprocess, provider,
  or project-discovery work.
- The Agents tab displays stand-alone proc shells as beautiful, accessible, stable
  top-level work rows with rich proc-specific details and actions, separate counts, no
  double projection, inspected visual snapshots, and measured responsive refresh.
- CLI/ACE notifications, coordinator receipts, Agents and Procs views, and recovery all
  agree on unit identity and eligible/launched/skipped/error counts across the complete
  mixed dependency matrix.
- Both feature-flag states, security/recovery boundaries, old all-agent compatibility,
  editor parity, full repository checks, LSP tests, and visual/performance tests pass;
  docs ship with the feature and canonical memory is changed only with explicit user
  permission plus `sase memory init`.
