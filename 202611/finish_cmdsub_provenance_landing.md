---
tier: tale
size: medium
title: Finish command-substitution provenance and land sase-1tc.1.1
goal: Close the remaining provenance and Jinja regressions, remove this epic's eager
  startup growth, verify the integrated routes, and close sase-1tc.1.1 and its parent
  phase sase-1tc.1 normally.
proposed_by: bbugyi200.athena.sase-1tc.1.1.land
proposed_with: codex/gpt-6-astra
bead: sase-1tc.1.1
status: done
---

- **PARENT:**
  [202611/cmdsub_source_provenance.md](https://github.com/sase-org/sase--plans/blob/main/202611/cmdsub_source_provenance.md)
- **BEAD:**
  [sase-1tc.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1tc/sase-1tc.1.1.md)

# Finish command-substitution provenance and land sase-1tc.1.1

This is the remaining work from the landing audit of epic `sase-1tc.1.1`. One coding
agent owns this tale and its final closeout. There is no later land agent for the tale.
Do not wait for this work's own commit SHA, push, or CI result before closing the epic:
close in the same turn as the final verified code, then use `/sase_final` for host
completion.

## Already verified

The lander read the epic, all three child scopes and every note, parent phase
`sase-1tc.1`, and the two design artifacts:

- `plan:202611/cmdsub_source_provenance.md` (this epic's plan).
- `plan:202611/backlog_epic_cmdsub_retries_launches.md` (the outer epic).

All three phases are closed. Rust commit `1191f9a6` implements typed eligibility,
balanced command spans, validated UTF-8 protected ranges, and the registered PyO3
binding. Those files have not drifted since that commit. The current sase pin
`c5e60d78575b50b8eefd1b2881e6fac2b8768f55` contains it. Python commits `efc66a420b` and
`81656950c6` add the source rendering and route integration. The existing
`test_cmdsub_source_provenance.py` and `test_workflow_routes_provenance.py` suites pass
47/47 at `81656950c6`; they do not cover the failures below.

The outer decision `mirror_fix = retire_mirror` remains final but is outside this phase.
Keep the chosen source-before-interpolation mechanism: only original human source and
admitted definition literals in a human launch may execute. All context values and
command stdout remain data; agent launches execute none. Jinja must see ordinary typed
values. Macros in generated `render.combined` values must still expand. Do not restore
key-name-only protection, blanket disabled regions, a global trust toggle, a Python
core-policy fallback, or an unsafe compatibility switch.

## Integration and ownership

Before editing, refresh the checkout through the normal VCS workflow, run
`sase agent list`, and read the current epic notes. Use `/sase_repo` for other repos and
`sase artifact read` for all plan bodies.

The live `1do` session is implementing `plan:202611/macro_guard_marker_leaks.md`. It
owns marker flattening, global typed argument reveal, clean trace evidence, and
fail-loud outer-boundary checks. It was testing during this audit. Read that plan and
incorporate its landed implementation and tests; do not overwrite or independently
duplicate the live owner's edits. If it is still active, record coordination on
`sase-1tc.1.1` and use SASE's monitor handoff if waiting is necessary. Its plan
explicitly leaves local-helper guard isolation and Jinja captured-value semantics to
this epic. Preserve its safety regressions when replacing its temporary `_rehide`
workaround with a complete guard lifetime.

Fetched `origin/master` equaled `81656950c6` at audit time. Nine other commits since
this epic's creation were reviewed: viewer changes `487825956a`, `bfa60324a6`,
`898eb8ffa5`, `5b4e61bfa7`, `f2fb7fb6b5`; artifact queries `8a6451068e`; stranded
filtering `71f0060927`; memory queries `1c626f39e5`; compact rows `be6adc20d0`. They add
no prompt-rendering route or competing cmdsub mechanism. The latter two introduce
unrelated eager imports, already routed to their active owners below. Recheck subsequent
commits and integrate any new rendering callers before closeout.

## 1. Keep one authority lifetime across recursive macro expansion

Relevant files: `src/sase/macro/_processor_engine.py`, `_processor_render.py`,
`processor.py`, `cmdsub_source.py`, and `_jinja.py`.

Current concrete failures, reproduced with a mocked shell executor and explicit
`CmdsubProvenance(HUMAN_SOURCE, "human")`:

- Catalog `main` has content `#_h`, with local helper `_h` whose content is
  `$(printf PAYLOAD)`. The executor returns `#echo($(printf DATA_EXECUTED))` for that
  source command; global `echo` has content `{{ _1 }}`. Expanding `#main` calls the
  executor for both commands. Only `printf PAYLOAD` is authorized.
- `main` contains `#_h(suffix={{ v }})`; `_h` declares `suffix: word` and outputs it.
  With `v = "ok"`, validation receives a foreign `SASEDATA` token and fails.
- Early preprocessing of `#echo(\$(printf ESCAPED))`, with `context={}`, executes
  `printf ESCAPED`. The early pass removes the escape, then the argument pass scans the
  now-unescaped text again.

Make nested local and global expansion share guard custody without letting a nested call
unwrap data into an enclosing source pass. Separate an internal expansion that returns
protected output from the outermost completion that reveals data and performs
generated-only macro expansion. A shared guard alone is insufficient while
`_catalog_expand_with_provenance` still unwraps midway through each recursive call. Keep
data hidden through every ancestor execution boundary. Command stdout, inherited scope
values, converted arguments, legacy placeholders, and nested helpers all follow the same
rule. Reveal values only for input validation and Jinja computation; those operations
must see real typed values and cannot confer command authority. On output, their text is
data again.

Carry escaped literals as protected data until the last possible execution boundary; do
not globally unescape them and then re-scan. Preserve literal output, existing
fence/disabled/directive-owned-code rules, trust refusals, conditional segments, macro
scope resolution, and clean trace records. Generated macro references still expand
without running their embedded argument commands.

Add regressions for both reproductions, at least two helper depths, local-to-global and
global-to-local nesting, typed word/int/bool and untyped comparisons, inherited scope,
legacy arguments, escaped commands, command stdout, and human/agent launches. Assert
executor calls as well as literal output; caching must not hide a second scan.

## 2. Preserve Jinja value semantics and source evaluation order

Relevant files: `source_render.py`, `_jinja.py`, `cmdsub_source.py`, and
`src/sase/llm_provider/preprocessing.py`.

Additional reproduced failures:

- `preprocess_prompt_early("{% if false %}$(printf FALSE){% endif %}")` with no context
  calls the executor. The `context is None` branch executes before Jinja.
- `{% set word %}{{ v }}{% endset %}{% if word == "ok" %}YES{% else %}NO{% endif %}`
  with `v = "ok"` renders `NO`. Finalizers insert internal markers into captures before
  Jinja compares them.
- A Jinja macro that prints fences through `{{ "```" }}` and is invoked through
  `{{ tree() }}` leaks nested `SJSMARK` markers in both renderers. The active `1do` fix
  covers that round-trip, but explicitly does not fix comparison, length, or slicing of
  captured values.

Use a shared Jinja-aware output mechanism for macro and workflow rendering. Prefer a
small AST/code-generation adapter that distinguishes emitted output from intermediate
capture buffers: execute eligible original literal text only when its branch is
evaluated; protect emitted data without modifying strings used in Jinja computation. Do
not solve capture comparisons with another string-equality heuristic or by removing all
protection. Macro returns, block captures, caller blocks, filters, indexing, length, and
slicing must operate on real values. Preserve provenance separately from those values.
Plan only commands complete in original source literals; commands assembled across
interpolation boundaries remain literal.

Render authored top-level Jinja at the source boundary even when the caller supplies no
explicit context, while preserving global template variables and macro expansion order.
Both a false Jinja branch and a discarded conditional segment must execute nothing.
Avoid compiling or eagerly executing every literal branch.

Retain the active marker fix's outer-boundary leak checks and swarm coverage. Verify
nested macro/capture/caller/filter shapes, equality and length checks before output,
case conversion, slicing, ordinary Jinja errors, and the mixed authored-command/data
template. A synthetic research-style swarm must retain its lead and two conditional
segments without any internal NUL marker. Use the real research plugin check only if
available through sanctioned discovery; do not reinstall anything globally.

The Rust core remains the source of command eligibility/span planning. Jinja compiler
integration and subprocess I/O remain Python glue. Any necessary shared policy change
belongs in the linked sase-core repo with bindings/tests and a containing CI pin.

## 3. Carry protection through late processing and pre-rendered workflow routes

Phase `sase-1tc.1.1.2` proposed these as follow-ups, but the lander verified that both
are regressions from removing the former context-value facade:

- Render `{{ review._raw }}` with `_raw = "{{ missing_data_variable }}"`; late
  preprocessing exits 1 instead of preserving the data.
- Render it with `_raw = "%model:injected_model\nresponse"`; early preprocessing sets
  `directives.model` and removes the data line.

Keep generated spans distinguishable until directive extraction and late Jinja have
finished. Do not re-render data as template source or parse literal data lines as
directives. Preserve authored directives, including arguments interpolated into an
authored directive and directives supplied by admitted macro definitions. Preserve macro
expansion of combined model output. File/artifact reference handling, continuation
capture, formatting, and HTML-comment behavior must still run at their proper
boundaries. Do not merely use `skip_preprocessing` to omit required work.

Thread the result through all existing routes: shared prompt preparation, parallel
pre-expansion, embedded/tagged prompt parts, simple-macro synthetic steps, standalone
steps, mentor, CRS, and the invocation preparation path. Centralize repeated boundary
handling where useful rather than copying another slightly different extraction pass.
Explicit metadata must survive copies; do not depend on a `str` subclass or context keys
that user data can forge.

Fix the unchanged parallel case in `workflow_executor_parallel.py`: it currently marks
only when `expanded != ns.agent`. With source `$(printf STABLE)` and mocked stdout equal
to that same string, the returned step has no pre-rendered mark and subsequent
preparation calls the executor again. Every rendered step must carry the mark even when
its bytes are unchanged. Its directives must be extracted from the appropriate
authored/expanded source and removed from provider text; extracting only the original
unexpanded template loses macro-produced directives and leaves rendered directives in
the prompt. Preserve trace and per-segment VCS metadata alongside the mark.

Add late-Jinja/directive regression cases for `_raw`, `_response`, ordinary structured
fields, and nested helper output. Include an authored `%model:{{ choice }}` case, an
admitted macro directive, a literal `%model` line in data, and unchanged parallel
output. Verify the final provider prompt and directive object, not just first render.

Complete the original acceptance evidence where existing tests only supplied context
dicts: drive structured prompt output and non-dict `_data` through the real storage path
in `workflow_executor_steps_prompt.py`; drive `join: text` through
`workflow_executor_loops.py`; exercise bash and Python producers, a no-space branch
value `topic/$(printf${IFS}DATA)`, and the `with_q_and_a`/`with_feedback`
combined-output path. Mock providers and unrelated workflow side effects. Recheck
`sdd/_expand.py` and `axe/run_agent_exec_plan_accept.py` inherit agent authorship.

## 4. Remove this epic's eager startup growth and verify

The import-budget proposal from `sase-1tc.1.1.3` used `efc66a420b` as a clean base, but
that already contains this epic's second phase. Fresh measurements at `81656950c6`:

- Isolated import-budget test fails: 11,865,876 bytes against 11,828,610.
- `tools/tui_import_closure --diff 71f0060927` measures 11,828,809 before phase two and
  +37,067 after all subsequent commits.
- This epic contributes +11,178: `cmdsub_source` 4,919; `source_render` 1,158;
  `_processor_engine` 2,336; `_processor_render` 1,493; `_jinja` 682; `processor` 298;
  `preprocessing` 292.
- Compact-row adapters contribute +19,043; memory queries contribute +6,846.

Read `ace.md`/`tui.md` before any TUI edit. Keep command rendering dependencies off the
first-paint import path with use-site imports, type-only imports, or the existing lazy
facade pattern. Remove the epic-attributable growth, and measure the resulting closure
with the same interpreter. Do not increase budgets or weaken deferred-module guards.
Keep this fix in the macro/preprocessing imports when possible; other active epics own
the compact-row and memory-query files.

The lander completed `/sase_new_task` triage before proposing this tale:

1. Late Jinja and directive proposals were declined as separate tasks: both stay here.
2. The own +11,178-byte part of the import proposal stays here.
3. The compact-row portion was attached as a `DISCOVERED ISSUE` to `sase-1t3.10.1`,
   naming proposing bead `sase-1tc.1.1.3` and commit `be6adc20d0`.
4. The memory-query portion was attached to `sase-1t3.11.1`, naming the same proposer
   and commit `1c626f39e5`.
5. No duplicate task was created. Reviewed task `sase-1ic` concerns the old
   split-sensitive module-count failure fixed by the footprint metric; this new
   attributed eager-code growth is causally owned by active epics. The pre-phase-two
   residual 199 bytes is unrelated and must not be used to dismiss this epic's growth.

These outcomes and the complete audit are already on `sase-1tc.1.1`; retain them in the
final close note. If unrelated residuals remain, report independent base/after evidence
and route them through `/sase_new_task`. Do not fix another live owner's files or
disguise a nonzero check result as a pass.

Read `lint_and_test.md`. Run focused provenance, new regressions, guard nesting,
macro/swarm, authorship, preprocessing, and route suites; run the import-budget probe
and closure comparison. Run `just fix`, then file-change verification `just check`
through `sase tool run check` in each changed repository. Use `/sase_monitor` for long
verification and carry these exact closeout obligations into its follow-up. Never run
`just check-full`; do not add skips, xfails, safety suppressions, or budget increases.
Update `docs/reference/macros/templating.md` only if needed to describe the final
boundary accurately, after reading `docs_writing.md`. No memory edits are planned.

## 5. Finish the nested landing in this tale

After the repairs and verification, perform these operations yourself, without waiting
for this work's own commit or CI:

1. Re-read `sase-1tc.1.1` and every descendant with audited `sase bead read`. Review
   newly added notes and confirm the completed source policy, preprocessing, and
   workflow-route scope plus all findings above. Recheck drift since this audit and
   incorporate the finished `macro_guard_marker_leaks` work. Do not close while any
   epic-caused failure remains.
2. Run `sase bead epic-symbols sase-1tc.1.1`. For every entry resolve the symbol (wire
   it up, privatize, add a valid non-test pragma for a real external consumer, or delete
   it), or re-key only to an actually open later bead that needs it. Follow the
   Symvision hierarchy. The audit found no entries, but query again.
3. Close normally with `sase bead close sase-1tc.1.1 --note "<verification>"`. The note
   must identify fixed regressions, source/core-pin and post-epic integration checks,
   actual test/check results and limits, and every follow-up disposition above. If stale
   symbols reject the close, finish their cleanup and retry. If unfinished descendants
   reject it, finish or reopen the required work. Never use `--force` to advance a
   successful nested landing or merely to bypass rejection.
4. Run `just symvision`. Open the plans sidecar with `/sase_repo`, read the artifact
   through `sase artifact read`, and set `status: done` in the frontmatter of
   `202611/cmdsub_source_provenance.md` under the printed plans repo path. Include that
   sidecar edit in host finalization.
5. Run `sase bead read sase-1tc.1.1 -r "Need the parent link"` and re-read parent phase
   `sase-1tc.1`. Confirm this repair satisfies the phase's four unsafe channels, real
   Jinja semantics, authored-command preservation, and verification. Record
   `TASK RESOLVED: sase-1sn — <base/after and verification evidence>` on that phase.
   Resolve or legitimately re-key `sase bead epic-symbols sase-1tc.1` entries, then
   close only the phase with `sase bead close sase-1tc.1 --note "<what was verified>"`.
   Confirm with `just symvision` after retiring those entries. If a parent requirement
   is incomplete or ambiguous, leave it open and add a concrete blocker note.
6. Leave outer epic `sase-1tc`, its plan, and task `sase-1sn` to the already-waiting
   outer land agent. Finish with `/sase_final` as the last action, committing every
   changed repository through the host declaration. Report actual closure status.
