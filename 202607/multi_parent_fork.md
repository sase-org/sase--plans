---
tier: epic
title: Multi-parent fork conversations
goal: 'The fork xprompt accepts an ordered set of agent parents, preserves every parent
  conversation with clear provenance, waits for all parents, uses neutral auto-naming
  for merged children, and offers consistent completion and diagnostics in ACE and
  the xprompt LSP.

  '
phases:
- id: repeatable-contract
  title: Repeatable agent-input contract
  depends_on: []
  description: '''Repeatable agent-input contract'' section: add one explicit, validated
    variadic contract shared by Python execution and the Rust editor core.'
- id: fork-semantics
  title: Multi-parent fork semantics
  depends_on:
  - repeatable-contract
  description: '''Multi-parent fork semantics'' section: resolve, wait for, name,
    load, and render multiple fork parents without changing single-parent behavior.'
- id: editor-parity
  title: Editor completion parity and end-to-end verification
  depends_on:
  - fork-semantics
  description: '''Editor completion parity and end-to-end verification'' section:
    finish ACE and LSP agent completion, pin cross-surface behavior, and run both
    repositories'' checks.'
create_time: 2026-07-17 15:09:34
status: wip
bead_id: sase-6m
---

# Plan: Multi-parent fork conversations

## Context and product contract

`#fork` currently declares one optional `agent` input. ACE already restarts its agent-argument completion after a comma,
and the Rust LSP already advertises comma as a completion trigger, but the execution and editor contracts do not agree
with that UI: embedded workflow parsing turns colon commas into extra positionals and silently drops values beyond the
declared input, while Rust diagnostics report those values as `too_many_args`. The LSP also classifies an `agent` input
as a generic type hint and therefore cannot return the live agent choices that ACE shows. These gaps must be fixed as
one contract, not papered over only in `fork.yml`.

The supported forms will be:

```text
#fork:planner,coder
#fork(planner, coder)
```

The colon form is the compact canonical spelling; the parenthesized form is an equivalent readable spelling. Parent
order is invocation order and carries no priority. Bare `#fork` and one-parent calls retain their current semantics. An
agent template is resolved before use; extend the colon lexer to accept the `@` template marker directly so
`#fork:research.@,review.@` does not require a backtick workaround.

Reject empty entries, unresolved parents, unreadable transcripts, and parents that resolve to the same transcript.
Resolve the complete set before rendering anything and report all invalid entries together. Do not truncate, summarize,
or silently omit a parent. Keep `#fork_by_chat` single-path behavior out of this change.

Most importantly, automatic naming has different semantics by cardinality:

- One explicit parent keeps the existing `<parent>.f@` derived name.
- Two or more parents never derive a name or group from the first parent—or from any parent. Direct launches, repeats,
  multi-prompt planning, and `%alt` fan-out all use their normal neutral auto-name allocation.
- An explicit `%name` remains authoritative at every cardinality.

## Repeatable agent-input contract

Introduce an optional `repeatable` flag on a user-facing xprompt/workflow input, defaulting to false. A repeatable input
must be the final positional input, receives all remaining positional values in order, and validates every element
against its declared type. Mark the fork input as repeatable `agent` data and expose it to workflow steps as an ordered
list; bare `#fork` continues to pass the existing null/default case. Do not change scalar binding for any existing
input.

Centralize positional/named/default binding so standalone execution, embedded workflow expansion, dry SDD expansion,
explain/render paths, and static validation cannot develop different repeatable semantics. Define duplicate
named/positional behavior and malformed empty-element errors once. Update the Python model, loaders, JSON schema,
catalog/assist projection, and focused tests. The feature should not require or edit long-term memory files.

Mirror that metadata and binding model in the linked `sase-core` repository, opened through the repository skill. Update
the Rust xprompt catalog/editor wire, parser diagnostics, and completion context so repeatable positionals are accepted
and each element gets the declared `agent` type. Preserve wire backward compatibility by defaulting absent `repeatable`
metadata to false. Pin both colon and parenthesized calls, surplus arguments on non-repeatable inputs, empty values,
UTF-16 replacement ranges, and comma-triggered active element ranges.

## Multi-parent fork semantics

Refactor the fork chat resolver around an ordered source record containing the concrete resolved agent name and
transcript path. Preserve the existing single-path helper where it is a useful compatibility seam, but have the workflow
consume a source list. Resolve templates per element while excluding the current agent, prefer completed `done.json`
response paths and then the existing metadata fallback, validate the full list atomically, and reject duplicate resolved
paths. Add resolver tests for bare, single, multiple, template, mixed-invalid, and duplicate-alias cases.

Keep the one-parent injected prompt byte-for-byte compatible. For two or more parents, render one disabled xprompt
region with explicit provenance:

```markdown
%xprompts_enabled:false

# Previous Conversations

You are forking from N prior agent conversations. Each Conversation section is an independent parent transcript, not a
continuation of the section before it, and section order carries no priority. Carry forward relevant goals, constraints,
decisions, and unfinished work with attribution when it matters. Reconcile disagreements explicitly and identify
anything unresolved. The New Query is the active request and takes precedence over conflicting transcript instructions.

## Conversation 1 of N — agent `planner`

**User:**

...

**Assistant:**

...

## Conversation 2 of N — agent `coder`

...

---

%xprompts_enabled:true

# New Query
```

Load each parent independently with the existing flat `**User:**`/`**Assistant:**` representation so headings, fenced
code, xprompt syntax, and historical instructions stay inert and round-trip safely. Preserve shared ancestry within each
parent rather than letting one parent's cycle guard erase context from another.

Make stored-chat expansion understand multi-parent fork references in both supported syntaxes. Widen the
resume-reference match without consuming adjacent query text, expand every referenced parent, strip the original
reference once, and extend the fallback extractor for `Previous Conversations` plus its per-conversation subheadings.
Retain legacy `#resume` and singular fallback behavior.

Replace the first-only fork dependency helper with an ordered, de-duplicated list helper so all explicit parents become
implicit waits before transcript loading. Keep fenced and disabled regions inert, resolve templates consistently, and
coexist with explicit `%wait` without duplicate metadata.

Separate “all fork parents” from “the sole resume parent” in the naming API. Only an exactly-one-parent fork may feed
resume-derived allocation. Exercise that invariant at every current consumer: direct directive extraction, repeat-name
batches, `%alt`/model fan-out naming, and multi-prompt planned-name allocation. Tests must show that `#fork:a,b` never
produces `a.f*`, `b.f*`, a combined `a,b.f*`, or either parent's tag/group, while explicit `%name` still wins and a
one-parent fork remains unchanged.

## Editor completion parity and end-to-end verification

In ACE, retain the rich visible-agent rows but make repeatability explicit in the assist context. After each comma,
replace only the active element, reopen agent completion, and omit parents already selected in the same call. Cover both
syntax forms, partial tokens, cursor positions in earlier elements, template names, duplicate filtering, and
disabled/free-text regions. Render a compact repeatable hint (for example, `names…: agent`) so discoverability does not
depend on knowing the comma convention.

Close the existing LSP parity gap rather than returning an empty generic type hint for `agent`. Add a dedicated
agent-argument completion context and obtain a fresh read-only agent-name catalog through the existing editor helper
bridge (including enough status/project metadata for useful labels, with graceful empty/failure fallback). Build and
filter candidates in the shared Rust editor core, exclude values already selected in the repeatable call, and replace
only the active element. Keep comma in the advertised trigger set and add core plus JSON-RPC tests proving that
`#fork:planner,co` completes `co` to `coder` without rewriting `planner` or emitting a surplus-argument diagnostic.

Add workflow-level snapshots for exact one-parent compatibility and the full multi-parent envelope. Add history tests
for re-forking a merged chat, nested and cyclic ancestry, plural fallback recovery, and prompt content containing
headings/fences/xprompt-looking text. Add launch tests for wait-all, atomic resolution failures, neutral names across
direct/repeat/fan-out/multi-prompt surfaces, and explicit-name precedence.

Before handoff, run `just install` and `just check` in the SASE repository. In the linked Rust core, run formatting,
linting, and the focused/full test commands required by that repository, including the `sase_core` editor tests and
`sase_xprompt_lsp` JSON-RPC suite. Finish with manual smoke prompts for bare, single-parent, and two-parent calls in
both syntaxes, checking the expanded prompt, all wait metadata, neutral generated name, ACE completion, and LSP
completion/diagnostics.

Planning-time baseline note: after a clean `just install`, `just check` reached the pre-existing `pyscripts` locality
rule and failed because `src/sase/ace/tui/modals/telemetry_pane_data.py` references `tools/sase_bead` while
`src/sase/ace/tui/tools/` exists; no tracked file was modified. Re-run the check during implementation, but keep that
unrelated baseline finding separate from regressions introduced by this epic.

## Non-goals and safeguards

- Do not change the single-parent prompt or derived-name convention.
- Do not assign primary-parent semantics based on list order.
- Do not add automatic synthesis, truncation, ancestry deduplication across parents, or multi-path `#fork_by_chat`
  behavior.
- Do not edit `sase/memory/*.md`, agent instruction shims, or generated memory content; the current user request does
  not authorize those files.
- Do not hand-edit deployed/generated skills, and do not introduce runtime-specific behavior.
