---
tier: tale
title: Preserve batch predecessors when launching prompt stacks
goal:
  Make no-argument waits target the preceding submitted agent consistently across TUI
  prompt stacks and named xprompt swarms, including deferred expansion and naming.
size: medium
proposed_by: bbugyi200.athena.0i1
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0i0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.0i0/README.md)
  - [bbugyi200.athena.0i1](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0i1.md)
  - [bbugyi200.athena.sase-z7.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-z7.1/README.md)
- **COMMITS:**
  - [2afe3d7](https://github.com/sase-org/sase-core/commit/2afe3d7d3cef42b88c2c6b916debd53b11c4ee17)
    — feat(agent-launch): add predecessor wait binding
  - [904f2d2](https://github.com/sase-org/sase-core/commit/904f2d2c602453755189d7d23e69842461544d3f)
    — fix(release): allow release-plz major version bumps
  - [7c949b4](https://github.com/sase-org/sase-core/commit/7c949b46c3ae656a84b5a94759ddcb926fa6f3c1)
    — feat: add usage indicator policy projection

# Preserve batch predecessors when launching prompt stacks

## Goal

Make submitting all prompt input panes in ACE obey the same ordered dependency semantics
as invoking a file-defined xprompt swarm with `#name`. A no-argument `%wait` in a later
pane must target the preceding agent in this submission, even when xprompt expansion
introduces the directive or the predecessor is slow to publish its name. Keep agents
without waits eligible to run concurrently.

This is one medium tale: the defect is in a shared launch/runner boundary, with bounded
Rust policy, Python integration, and regression coverage. No source files were modified
during diagnosis. Only this scratch plan was authored. The exact prompts from the user's
reported occurrence were unavailable; the failures below were independently reproduced
with isolated, mocked subprocess launches.

## Findings and evidence

### The stack is not reversed

- `src/sase/ace/tui/widgets/_prompt_input_bar_stack_rendering.py`,
  `_build_pane_widgets()`, renders `self._stack.items` in list order.
- `src/sase/ace/tui/widgets/_prompt_stack_state.py`, `join()`, joins non-empty agent
  panes in the same order with `\n---\n`; auxiliary panes are excluded.
- `_prompt_input_bar_submission_actions.py` submits that joined text once.
  `_prompt_bar_submit.py`, `_launch_submission.py`, and `agent_durable.py` route the
  batch through one durable `sase run` request.
- `src/sase/agent/launch_cwd_agents.py` passes the ordered expanded segments to
  `launch_multi_prompt_agents()`. `multi_prompt_launch_execution.py` iterates them with
  `enumerate(segments)`; it does not reverse them.
- The bottom pane is selected by default, which is documented in `docs/ace.md`.
  Selection does not change whole-stack order. Separate selected-pane submissions are
  separate launches and can intentionally submit the bottom pane first.

The existing focused tests passed: the two `test_submit_choice_all_submits_whole_stack`
variants and the named/automatic predecessor bare-wait tests in
`tests/test_multi_prompt_launcher_wait_vcs.py` (4 passed).

### The defect is incomplete propagation of the batch predecessor

`src/sase/agent/multi_prompt_launch_execution.py` scans and rewrites bare waits on each
raw segment before ordinary xprompt expansion. Its lookahead for whether the next
segment needs the predecessor's name also examines raw text.

`plan_segment_fanout()` in `multi_prompt_launch_plan.py` subsequently expands xprompts
to detect fan-out, but falls back to a plan containing the original segment when
expansion produces no fan-out. Thus an ordinary xprompt can carry an undiscovered bare
wait into the runner. File-swarm expansion supplies segment and provenance metadata, but
does not guarantee every ordinary nested xprompt has already expanded; file-defined
swarms containing such helpers share the bug.

There is a second route to the same loss of context. `PlannedNameAllocator` can return
no name for an unresolved xprompt-bearing predecessor. If the following raw segment has
a bare wait, the launcher polls its metadata for a name. When `wait_for_agent_naming()`
returns `None`, the launcher prints `naming timed out, continuing` and submits the
successor with the bare directive unchanged. The poll defaults to 30 seconds.

The runner expands ordinary xprompts in `src/sase/axe/run_agent_directives.py` and calls
`extract_prompt_directives()`. `src/sase/xprompt/_directive_values.py`,
`resolve_wait_agent_args()`, resolves any remaining bare wait using the global
`get_most_recent_agent_name()`. Another launch can therefore become the target; absence
of any name can also cause an error. Reversing the widgets would not fix this.

Diagnostic probes used a fake global latest name `unrelated`, patched workspace
allocation and subprocess spawning, and these xprompts:

```text
probe_build: Build
probe_review: %wait\nReview
probe_team: %id:builder\nBuild\n---\n%wait\nReview
```

| Submitted segments                                          | Observed successor dependency |
| ----------------------------------------------------------- | ----------------------------- |
| `%id:builder\nBuild`, `%wait\nReview`                       | `builder` (correct)           |
| `#probe_team`                                               | `builder` (correct)           |
| `%id:builder\nBuild`, `#probe_review`                       | `unrelated` (wrong)           |
| `#probe_build`, `%wait\nReview`, naming poll returns `None` | `unrelated` (wrong)           |
| `%id:builder\nBuild`, `%wait( )\nReview`                    | no dependency (wrong)         |

The whitespace-only parenthesized form exposes a parser disagreement:
`has_bare_wait_directive()` rejects its non-empty raw whitespace, while final directive
collection supplies no positional wait. `%wait()`, `%wait`, and `%w` do bind correctly
in the literal named-predecessor case. Normalize all supported zero-argument spellings
consistently as part of this fix.

## Behavioral contract

1. Whole-stack order is the visible top-to-bottom order after excluding empty and
   auxiliary panes. Reordering panes changes that order; selection does not.
2. Later batch segments with bare `%wait`/`%w`, including empty or whitespace-only
   parentheses and directives introduced by ordinary/local/nested xprompts, bind to this
   batch's predecessor. They never consult the global latest name.
3. Preserve established fan-out semantics: a following segment targets the last launched
   child of the preceding segment; sibling variants in the current segment inherit its
   preceding-segment dependency, rather than becoming an accidental serial chain.
   Preserve repeat's existing explicit wait chain.
4. An unresolved predecessor name must not erase the dependency. Resolve using its
   captured launch identity and retain the ordinary agent/family completion semantics.
   Missing metadata, naming delay, and predecessor failure cannot release the successor
   onto an unrelated agent. Do not silently change a wait on a whole agent/family into a
   wait only for its initial planner shell.
5. Preserve explicit agent, bead, time, and other wait targets; mixed explicit and bare
   waits compose. Fenced and xprompt-disabled literal regions stay literal. `%queue`
   does not acquire previous-agent meaning.
6. With no batch predecessor, preserve the existing standalone/first-segment behavior.
   Typed `%if`/`%proc` plans retain their existing Rust logical-unit dependency
   semantics, including their first-unit validation.
7. Preserve original submitted prompt history, local frontmatter, per-segment
   VCS/environment data, template groups, keyed-name namespaces, provenance, and
   partial-launch cleanup. A raw stack need not pretend to have a named file source to
   share launch semantics.

## Implementation

### 1. Add regression cases at the shared launch boundary

Extend the existing launcher test fixtures with the cases above before changing
behavior. Mock an unrelated global latest name and make global lookup fail if it is
consulted for a segment with an actual predecessor. Include both inline stack-style
segments and `#name` file-swarm expansion, with literal waits and ordinary xprompts that
introduce waits. Include a temporary Markdown xprompt file loaded through normal catalog
discovery, not only mocked `XPrompt` objects, to exercise the real file-backed entry
point. Keep subprocesses and workspace claims mocked; no live agents are needed.

### 2. Make predecessor binding shared backend policy

Open `sase-core` with `/sase_repo` and use the returned checkout. Follow its
`AGENTS.md`; do not assume a fixed sibling or numbered workspace path.

Implement the binding policy in `crates/sase_core/src/agent_launch/`, exposed through
`crates/sase_core_py/src/lib.rs`, with a thin typed adapter in
`src/sase/core/agent_launch_facade.py` and corresponding wire records as needed. Use the
existing Rust directive scanner and protected-region rules to recognize and
consume/rewrite only genuine no-argument waits, including whitespace-only parentheses.
Reuse the typed planner's previous-target interpretation where appropriate; do not route
every ordinary launch through the typed coordinator.

Introduce an explicit per-child predecessor context containing a schema version, the
predecessor's project, timestamp and canonical artifact directory, plus its canonical
name when already known. Validate it at the binding boundary. It must identify the
actual preceding launch independently of global name registration. Keep metadata I/O and
transport in Python adapters, and target-selection and validation decisions in Rust. Do
not add a Python backend fallback or a TUI-only wait implementation.

### 3. Carry that context from the batch launcher through directive extraction

In `multi_prompt_launch_execution.py`, derive predecessor context from the last
successful `LaunchExecutionRecord` of the preceding segment. Supply it to every child of
the next segment, including segments whose raw text contains no wait. Use the existing
slot environment transport, with a dedicated `SASE_AGENT_...` key so the established
agent environment scrub prevents leakage into unrelated child launches. The first
segment must receive no inherited predecessor context.

Keep the current explicit-name rewrite fast path and derived-name behavior where valid,
but retain enough context for waits revealed later by expansion. A parent-side naming
timeout must not leave a context-free bare wait. For `%wait`, use the captured identity
instead of depending on successful parent-side name polling; do not increase the timeout
as the fix. Preserve the distinct existing `#fork` naming requirements rather than
removing its polling indiscriminately.

At the runner's actual directive-extraction boundary, after ordinary/local xprompts
expand, apply the Rust binding policy before the global bare-wait resolver. If the
predecessor name is unavailable, preserve an identity-bound dependency through the
existing wait metadata/barrier machinery. Resolve its canonical agent/family target when
metadata becomes available, without changing existing completion rules. Reuse
`wait_identity_deps`/`wait_for_artifacts` support where compatible; add only the minimal
core policy needed to retain whole-agent semantics. Persist the resolved dependency for
resumed/repeated preprocessing so the later pass in
`src/sase/llm_provider/preprocessing.py` cannot rebind it globally.

Check deferred-workspace classification for waits revealed by expansion as well: the
successor must stay behind the dependency barrier before entering provider execution.
Preserve existing deferred VCS workflows and avoid introducing additional eager
expansion or evaluating dynamic xprompts repeatedly just to discover waits.

### 4. Cover the TUI integration and document the contract

Extend `tests/ace/tui/widgets/test_prompt_stack_submit_cancel.py` or a focused adjacent
test to inspect actual pane positions and capture submit-all with a wait in a later
pane. Exercise reordered panes, a bottom-selected pane, an empty pane, and
frontmatter-local xprompts. Connect the captured batch to the mocked launch boundary so
the test checks the resulting predecessor, not only joined text.

Preserve the durable `sase run` submission and keep all disk/name resolution work off
the Textual event loop and serial message pump. No widget reversal, new keymap, or new
submit mode is needed.

Update `docs/ace.md` at Prompt stacks and `docs/xprompt.md` at bare-wait semantics to
distinguish a batch predecessor from standalone most-recent-name lookup. Give a short
top-to-bottom two-pane example and mention xprompt-introduced waits.

## Verification and acceptance

- Rust unit tests and PyO3 binding tests cover context validation, no-argument forms and
  aliases, protected literal regions, mixed waits, absent context, and deterministic
  selection from supplied predecessor facts.
- Python launch/runner integration tests cover explicit and automatic names,
  delayed/missing names, unrelated simultaneous launches, nested/local helper waits,
  predecessor failure, family/plan promotion, and final-child fan-out selection. Assert
  the successor stays waiting until its intended dependency satisfies the existing
  completion rules. An unrelated completion cannot release it. Tests must inspect runner
  metadata/barrier behavior, not only launch order.
- Preserve the named/automatic literal-wait tests, template-group and per-segment
  environment tests, repeat/fork tests, and wait-family tests affected by the
  implementation. Add an environment-hygiene regression preventing predecessor context
  from leaking into a separate launch.
- Run the focused Textual submit-all regression and relevant existing stack
  submission/handler tests. Test selected-pane submission remains independent.
- Rebuild/install the modified local Rust binding using the repository's supported
  `just rust-install` workflow, then run the focused Python tests against it. Run
  `just check` in the opened `sase-core` root: it includes binding verification;
  `cargo test -p sase_core` alone is insufficient.
- Read `lint_and_test.md` through `/sase_memory_read` during implementation and run
  `just check` in the SASE checkout. Use the required monitor handoff if verification
  becomes long-running; broaden to `just check-full` only when the documented
  broadening/landing rules require it.
- Report the order hypothesis as denied, the confirmed context-loss cases and
  no-argument parsing case as fixed, the validation performed, and any remaining
  limitation. Do not claim the exact user's occurrence was reproduced without its
  concrete prompt stack.

## Scope boundaries

This work fixes batch dependency binding for both entry paths. It does not change the
global agent-list sorting, introduce automatic serial execution for every pane, alter
standalone wait targeting, change swarm template namespace policy, redesign the typed
coordinator, or migrate unrelated backend logic. No new CLI options, memory edits, or
keymap changes are planned.
