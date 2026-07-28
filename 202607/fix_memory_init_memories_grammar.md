---
tier: tale
title: Fix the grammar in generated memory instructions
goal: Default managed agent documents use correct plural subject-verb agreement and
  remain synchronized and drift-free.
create_time: 2026-07-28 06:55:36
status: done
---

- **PROMPT:** [202607/prompts/fix_memory_init_memories_grammar.md](prompts/fix_memory_init_memories_grammar.md)
- **AGENTS:**
  - [bbugyi200.athena.ms](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.ms/README.md)
  - [bbugyi200.athena.ms--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ms.md#member-code)
- **COMMITS:**
  - [3bd59cd](https://github.com/sase-org/sase/commit/3bd59cdda2a0317ee4b6e60a6ed72b9f1bcec83b) — fix(memory): correct generated instruction grammar

# Plan: Fix the Grammar in Generated Memory Instructions

## Goal

Make the default managed agent documents rendered by `sase memory init` say:

```text
The following memories contain core (always loaded) context:
```

instead of the ungrammatical `memories contains` form, then refresh this repository's checked-in generated instruction
files and prove the initializer is drift-free.

## Current Behavior and Root Cause

- `src/sase/amd/templates/AGENTS.template.md` is the packaged source of the sentence. `render_agents_template()` loads
  that template, and the managed-memory renderer supplies only the title and Tier 1/Tier 2 content; the grammar does not
  come from Python control flow.
- The typo was introduced when an earlier change pluralized `memory` to `memories` without changing `contains` to
  `contain`, and it was later moved into the packaged template.
- `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, and `QWEN.md` currently contain identical generated content, so
  the correction must reach all five through the normal initialization workflow.
- Three tests contain the typo: two assert default generated output, while one builds a representative managed
  `AGENTS.md` fixture for agent-document inventory parsing.
- The minimal template does not contain this prose. User-provided managed-template overrides own their wording and
  should remain untouched.

## Scope

### In Scope

- Correct the verb in the packaged default managed-agent template.
- Update focused generated-output assertions and representative current-output fixtures to use the corrected sentence.
- Regenerate the repository's managed agent document and provider instruction shims with `sase memory init`.
- Verify exact generated wording, provider-shim synchronization, initializer idempotence, focused behavior, and the
  complete repository check suite.

### Out of Scope

- Changes to memory-note content under `sase/memory/`, the memory README, headings, inline-memory layout, parsing rules,
  CLI options, or initialization behavior.
- Changes to `AGENTS.minimal.template.md` or to user/project template overrides.
- Compatibility logic for recognizing the old prose: structural parsing depends on section headings and memory-entry
  shapes, not this explanatory sentence.
- A commit or push; the implementation should use the initializer's `--no-commit` mode.

## Implementation

1. In `src/sase/amd/templates/AGENTS.template.md`, change only `contains` to `contain` in the Tier 1 explanatory
   sentence. Do not move the prose into Python or alter template variables and structural anchors.
2. Update the exact generated-output expectations in:
   - `tests/main/test_init_memory_agent_docs.py`, where `_assert_derived_managed_agents()` checks the beginning of a
     default managed document.
   - `tests/main/test_init_memory_managed_agents.py`, where the managed initialization integration test checks the Tier
     1 sentence.
   - `tests/main/test_memory_agent_docs_list.py`, where the inventory fixture represents current managed output.
3. Run `just install` so the workspace CLI and tests use the edited packaged template, then run the three focused test
   modules before regenerating checked-in output.
4. Preview initialization drift with `sase memory init --check` (and `--diff` if needed to inspect all planned paths),
   then run `sase memory init --no-commit` from the repository root. This must refresh `AGENTS.md` and its provider
   shims without invoking the project commit/pull/push path. Inspect the resulting status and diff; retain only the
   expected one-word generated changes alongside the source and test edits. Because `--no-commit` can still follow the
   configured home/chezmoi deployment path, record any such external generated-file refresh in the handoff.
5. Confirm `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, and `QWEN.md` remain byte-for-byte equal and all carry
   the corrected sentence.

## Validation

Run these checks after the edits and regeneration:

```bash
just install
just test tests/main/test_init_memory_agent_docs.py \
  tests/main/test_init_memory_managed_agents.py \
  tests/main/test_memory_agent_docs_list.py
sase memory init --check
rg -n "memories contains" \
  src/sase/amd/templates/AGENTS.template.md \
  tests/main/test_init_memory_agent_docs.py \
  tests/main/test_init_memory_managed_agents.py \
  tests/main/test_memory_agent_docs_list.py \
  AGENTS.md CLAUDE.md GEMINI.md OPENCODE.md QWEN.md
just check
```

The negative `rg` check must return no matches (exit status 1). Also inspect the positive corrected phrase and compare
each provider shim with `AGENTS.md`; do not treat the absence check alone as proof that the intended sentence exists.

## Acceptance Criteria

- Default managed output from `sase memory init` contains exactly
  `The following memories contain core (always loaded) context:`.
- The incorrect `memories contains` wording is absent from the packaged template, focused tests/fixtures, and all five
  checked-in managed instruction files.
- The source change is confined to the default managed template; minimal and override-template behavior is unchanged.
- Provider shims are byte-for-byte synchronized with `AGENTS.md`.
- `sase memory init --check` reports no remaining initialization drift after regeneration.
- The focused tests and mandatory `just check` complete successfully.

## Risks and Mitigations

- **Generated/source drift:** Editing only checked-in instruction files would allow the typo to return. Correct the
  packaged template first and regenerate through the command.
- **Over-broad generation:** Memory initialization may expose unrelated pre-existing drift or configured home/chezmoi
  writes. Preview, inspect every changed path, preserve unrelated user changes, and report external refreshes.
- **False confidence from parser coverage:** The inventory parser ignores this prose. Keep a direct default-render
  assertion in addition to updating the representative inventory fixture.
- **Implicit repository mutation:** Plain `sase memory init` can commit, rebase, and push. Use `--no-commit`, inspect
  status, and leave committing to an explicitly authorized workflow.
