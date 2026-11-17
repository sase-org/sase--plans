---
tier: epic
title: Keep workflow data out of host command substitution
goal: Fix sase-1tc.1 and sase-1sn while preserving authored commands, macro expansion,
  and Jinja value semantics.
parent_bead: sase-1tc.1
phases:
- id: source_policy
  title: Rust-owned command substitution source policy
  depends_on: []
  size: medium
  description: 'source_policy: implement typed provenance, command-span planning,
    Python binding, and core/binding tests. Keep shell execution in Python and publish
    the containing core revision through host completion.'
- id: preprocessing
  title: Evaluate authored commands before interpolating data
  depends_on:
  - source_policy
  size: medium
  description: 'preprocessing: implement source-aware Jinja rendering, explicit preprocessing
    provenance, and macro argument/recursive expansion integration. Prove generated
    data never reaches either command execution pass.'
- id: workflow_routes
  title: Preserve provenance across every workflow rendering route
  depends_on:
  - preprocessing
  size: medium
  description: 'workflow_routes: migrate all prompt rendering callers, remove context-value
    placeholders, cover workflow and invocation boundaries, update templating documentation,
    and produce the complete acceptance matrix.'
proposed_by: bbugyi200.athena.sase-1tc.1
proposed_with: codex/gpt-6-astra
create_time: 2026-11-17 11:31:47
status: wip
bead_id: sase-1tc.1.1
---

- **PROMPT:** [prompts/202611/cmdsub_source_provenance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/cmdsub_source_provenance.md)
- **PARENT:** [202611/backlog_epic_cmdsub_retries_launches.md](https://github.com/sase-org/sase--plans/blob/main/202611/backlog_epic_cmdsub_retries_launches.md)
- **BEAD:** [sase-1tc.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1tc/sase-1tc.1.1.md)

# Keep workflow data out of host command substitution

## Scope and evidence

This child epic implements only `sase-1tc.1`, the `cmdsub` phase of
`plan:202611/backlog_epic_cmdsub_retries_launches.md`. The underlying task is
`sase-1sn`. The parent epic's accepted decisions remain final; this work changes no
memory notes. It does not implement the parent's retry, chop, bead-store, or triage
phases.

The assigned phase had no earlier implementation notes on 2026-11-17. On sase
`e6ee9fd446`, the earlier fix `d0dc942c16` is present, but the defect remains. Using the
checkout's Python, `render_prompt_template`, the existing disabled/fenced-region
protection, and `process_command_substitution` produced:

| Template / context channel                                         | Actual result at the base commit           |
| ------------------------------------------------------------------ | ------------------------------------------ |
| `{{ review.summary }}` = `$(printf EXECUTED_FIELD)`                | `EXECUTED_FIELD`                           |
| `{{ review._data }}` = `$(printf EXECUTED_DATA)`                   | `EXECUTED_DATA`                            |
| `{{ loop_step }}` = `$(printf EXECUTED_JOIN)`                      | `EXECUTED_JOIN`                            |
| `{{ setup.branch_name }}` = `topic/$(printf${IFS}EXECUTED_BRANCH)` | `topic/EXECUTED_BRANCH`                    |
| `{{ review._raw }}` containing a command                           | Protected by a disabled-region placeholder |
| `{% if "LGTM" in review._raw %}` with `_raw = "LGTM"`              | Incorrectly takes the false branch         |

A second reproduction mocked only `_execute_cmd_cached`, returning `EXECUTED_EARLY`.
With a simple `#echo` macro, early preprocessing of `#echo({{ review.summary }})`, with
`summary = "$(printf EARLY)"`, called the executor with `printf EARLY` under **both**
`SASE_PROMPT_AUTHORSHIP=human` and `SASE_PROMPT_AUTHORSHIP=agent`. The early macro
argument resolver is therefore part of this fix, not a separate follow-up.

Relevant paths are `src/sase/file_references.py`,
`src/sase/llm_provider/preprocessing.py`, `src/sase/macro/_processor_engine.py`,
`_processor_render.py`, `processor.py`, and `workflow_executor_utils.py`.
`tests/test_prompt_authorship_safety.py` covers the earlier fix but lacks the four
channels above and macro-argument execution.

Three medium phases separate the Rust policy and binding, the shared Python
preprocessing mechanism, and the workflow adapters. The latter must cover copied steps
and repeated invocation, not merely change one late-preprocessing boolean. All
dependencies are sequential. Each phase includes its own meaningful tests.

## Chosen mechanism and invariants

Choose **authored-source substitution before interpolation**, as permitted by the parent
design. A whole-template `agent_authored=True` decision based on referenced variables
cannot preserve a human command next to model output in the same template.

1. Only complete command expressions present in authorized source may execute.
   Authorized source means the original human prompt or a macro/workflow definition
   admitted by the existing trust mechanism in a human-authored launch. Agent-authored
   launches cannot enable execution through either early or late preprocessing.
2. Context values are data regardless of their key or producer. This includes structured
   model fields, `_data`, `_raw`, `_response`, nested collections, loop items and joins,
   bash/python results, arguments, runtime globals, and repo-derived branch names. Never
   infer command authority from a field name, matching bytes, a macro-looking string, or
   the ambient human stamp alone.
3. Execute source commands before inserting runtime values. The resulting text,
   including command stdout, is already rendered for this purpose and must never be
   scanned again for commands. A command assembled across an interpolation boundary
   stays literal. Do not quote an entire rendered prompt and call that provenance.
4. Jinja sees the original typed values in comparisons, indexing, filters, loops, and
   conditionals. Substitute commands from literal template text only when that text is
   evaluated by Jinja: a false branch must not run a command. A small Jinja AST adapter
   for literal output nodes can call the source-substitution helper; interpolated
   expressions must not call it. Expressions producing a string that looks like a
   command remain data. Do not pre-execute every branch while compiling the template, or
   replace context values with opaque placeholders.
5. Keep source literal-zone rules: fenced code, disabled regions, escapes, and
   directive-owned code bodies remain protected. An escaped command must not become
   executable on a second pass. Runtime text containing marker syntax cannot grant
   authority to any command.
6. Macro expansion remains enabled for rendered text, including `render.combined` from
   `with_q_and_a.yml` and `with_feedback.yml`. Resolve human-written macro-argument
   commands at the source boundary; never substitute commands in arguments extracted
   from rendered data. Each admitted macro definition is its own source boundary, before
   argument/scope substitution, including local helpers and recursion.
7. Pass provenance explicitly, using an internal typed source/rendered distinction and
   the existing launch-authorship stamp. Do not change `os.environ` temporarily, use a
   process-global trust toggle, or rely solely on a `str` subclass that loses metadata
   during concatenation, copying, or serialization. An already-rendered prompt must not
   become source because another function accepts a plain string. Missing provenance on
   a generated/rendered route fails closed for cmdsub while preserving the rest of
   preprocessing.
8. This fixes the implicit prompt-substitution capability. Explicit, trusted `bash:` and
   `python:` workflow steps retain their existing quoting and trust contract. No new
   unsafe compatibility switch is introduced.

## Source policy

Work in `sase-core`, opened with `/sase_repo`. Read its `AGENTS.md` first.

Add a focused `prompt_command_substitution` domain with serde request/response types and
tests. The core owns the source-versus-rendered eligibility rule and balanced
command-span planning; it does not launch subprocesses or embed Jinja. The request must
distinguish authorized human source, admitted definition source in a human launch, and
generated/rendered data. If the Python adapter supplies protected ranges, validate their
offsets rather than trusting invalid ranges. Return ordered replacement actions
containing the original command bytes and spans. State whether offsets are UTF-8 bytes
and test non-ASCII prefixes.

Preserve the current supported scanner behavior for complete nested parentheses, escaped
`\$(`, malformed/unclosed substitutions, and multiple commands. Never create new command
candidates by rescanning replacement text. Put the policy in core rather than
duplicating it across Python entry points. Keep `_execute_cmd_cached` and shell I/O on
the Python side for the next phase.

Register the PyO3 binding in `crates/sase_core_py`, following the repository's existing
wire conventions, and add a round-trip binding test that proves registration and the
generated/rendered refusal. Keep exports module-scoped. This phase ships the tested core
API without partially switching Python callers.

Run the focused core and binding tests through the repository recipes, then
`sase tool run check` in sase-core. Host completion must publish the core change and
move sase's `sase-core-revision.txt` to a containing commit, or the next phase uses
`just ratchet-core-revision` once that commit exists. Never invent a SHA or call a new
binding before the pin and the local extension contain it. Do not globally reinstall
sase.

## Preprocessing

Add a thin Python adapter to the core policy and a source-aware prompt renderer. Keep
Jinja engine integration, subprocess execution, and orchestration in Python. Use the
source renderer in the shared early pipeline and macro-definition rendering; allow the
older workflow-rendering facade to remain until the next phase migrates its callers. Do
not expose a half-migrated return type to existing callers.

Resolve launch authorship **before** early expansion. Carry explicit rendering state
through `PreprocessResult`, macro processor entry points, and the ordinary
`preprocess_prompt` composition. Once the source pass completes, late preprocessing does
file/artifact references, formatting, and the other existing work without
command-substituting the rendered result. Make the early and late APIs agree on the same
policy; directly using an early or late entry point must not bypass it.

Integrate `_resolve_command_substitution_in_args`, `process_loaded_macro_catalog`,
`process_macro_references_with_catalog`, `expand_single_macro`, local macro helpers,
legacy placeholder substitution, and post-expansion recursive passes. Raw source
commands and admitted definition commands execute only at their own boundary. Expanded
values, command stdout, and concatenated expansion results never do. Keep macro input
validation, conditional segments, directives, trust refusals, and trace capture working.
Trace records are evidence, not authority to execute again.

Tests in this phase must cover:

- The four unsafe context channels through early + late preprocessing in a human launch,
  plus `_raw`/`_response`, repeated interpolation, and nested containers.
- One template with a human literal command next to an interpolated malicious value: the
  human marker is created; the data marker is absent.
- Plain, positional, named, shorthand, nested, and local-helper macro arguments under
  human and agent launches. Include a command assembled by adjacent data fragments.
- Human macro arguments still substitute; `#echo` and other ordinary macros from a
  rendered `combined` value still expand without executing embedded command text.
- Real Jinja comparisons and transforming filters, a command in a false Jinja branch,
  escaped commands, fences/disabled regions, and command stdout containing `$(...)`.
- Repeated preprocessing does not execute derived text or execute an authored command
  twice; concurrency does not share authorization through mutable global state.

Use temporary marker files and executor-call assertions, not production commands. Keep
existing safety assertions. Assertions that require the obsolete placeholder
representation may be replaced with stronger observable value and no-execution
assertions, with the reason recorded. Do not broadly delete the earlier regressions.

## Workflow routes

Replace `_wrap_model_output_values` and the workflow rendering facade only after the
source-aware path and macro protections exist. Migrate every caller and its downstream
consumer so rendered text cannot be mistaken for fresh source:

- `workflow_executor_steps_prompt_prepare.py`: source rendering, embedded expansion, the
  second macro-expansion pass, and late preprocessing.
- `workflow_executor_parallel.py`: pre-expansion and the copied `WorkflowStep.agent`
  passed to subsequent preparation. Preserve provenance even when text is unchanged.
- `workflow_executor_steps_embedded.py` and
  `workflow_executor_steps_embedded_expand.py`: tagged and embedded `prompt_part`
  rendering, including paths with no pre-steps.
- `workflow_runner.py`: simple-macro conversion to a synthetic agent step and
  `expand_workflow_for_embedding` argument rendering.
- `main/query_handler/_standalone_steps.py`: rendered prompt + macro expansion before
  `invoke_agent`; its ordinary invocation currently preprocesses the string again.
- `workflows/mentor.py`: prompt construction and the later invocation boundary.
- `llm_provider/_invoke.py`, `_invoke_setup.py`, and `main/macro_handler.py` where
  necessary to carry the internal policy without skipping unrelated preprocessing.
  Re-check direct early consumers in `sdd/_expand.py` and
  `axe/run_agent_exec_plan_accept.py` for inherited agent authorship.

Audit the final call graph with `rg` for every `render_prompt_template`,
`process_command_substitution`, macro argument resolver, and early/late preprocessing
call. The policy must survive deep copies, synthetic steps, and any serialization that
occurs on these paths. Use host-owned metadata or explicit caller state; data in a
workflow context or prompt body must not be able to claim human authorship.

Exercise real prompt-step structured-output storage, non-dict `_data`, loop
`join: text`, and bash/python producer results, not only manually constructed context
dicts. The branch-name case should use a valid no-space git-style value such as
`topic/$(printf${IFS}MARKER)`. Mock providers and ordinary workflow side effects so
tests do not launch real agents or touch live repositories.

Use a compact parameterized route matrix for prepare, parallel, embedded, tagged,
simple-macro runner, standalone, and mentor. Each route must prove its malicious payload
remains literal all the way to the provider boundary and that the expected macros/Jinja
output remains intact. Include `with_q_and_a` and `with_feedback` combined-output
behavior and the mixed authored-command/data template. Check the entire `invoke_agent`
preparation path at least once, since stopping at the first render misses the
second-pass defect.

Read `docs_writing.md` before updating `docs/reference/macros/templating.md`. Its
current command-substitution section says remaining expressions run after expansion;
replace that description with the source boundary, literal data examples, Jinja
behavior, and human/agent distinction. No memory edits or new CLI options are needed.

## Verification and coordination

Before edits, refresh `sase agent list` and in-progress plan beads. At planning time,
the other active sase workers covered video preview/startup (`sase-1tb.1/.2`) and a TUI
heap census (`sase-1t3.9`); no active scope overlapped these prompt files. Re-check at
implementation time. Follow the parent epic's rule for a newly overlapping live owner:
record a `PROPOSED FOLLOW-UP:` on the phase and a coordination note on that owner's epic
instead of editing its files.

Read `lint_and_test.md` and run focused regressions, followed by `sase tool run check`
in every changed repository. Use `/sase_monitor` for long verification and `/sase_final`
for host completion. Never run `just check-full`, relax safety assertions, add
skips/xfails, or increase test budgets to obtain green. For an identical clean-base
failure, record the independent reproduction as a `PROPOSED FOLLOW-UP:` and close the
completed assigned phase as the user instructed.

Core-only additions in the first phase and complete per-route fixes in subsequent phases
avoid exposing a selectable unfinished feature. Do not retain an executable legacy
branch or add a feature flag to re-enable model-derived cmdsub.

Phase workers close only their assigned child phases, after
`sase bead epic-symbols <their-phase>` is clear or remaining allowances are re-keyed to
a still-open owner. Record base/after evidence and verification run IDs. Record
discovered work as `PROPOSED FOLLOW-UP:` notes; do not create task beads.

## Landing responsibility

The child epic's land agent reviews the complete route and channel matrix, checks the
containing core pin and final verification, and confirms no unsafe global rescan or
key-name-only protection remains. It records
`TASK RESOLVED: sase-1sn — <base/after and verification evidence>` on `sase-1tc.1`.
After all child work is closed, it resolves any `sase-1tc.1` epic-symbol entries and
closes `sase-1tc.1` with the verification evidence. This closure is land-agent work; no
child phase worker closes an ancestor. Settle the child epic through its normal host
landing flow. Never close `sase-1tc`, any higher ancestor, or task `sase-1sn`; the outer
epic's land agent owns those decisions.
