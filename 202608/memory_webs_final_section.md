---
tier: tale
title: Render Memory Webs as the final agent-instruction section
goal:
  Make SASE-managed agent instruction files render Core Memory first, Reference Memory
  second, and Memory Webs third and last without losing compatibility with existing
  documents or webless roots.
size: medium
proposed_by: bbugyi200.athena.sase-vk.land.w2
create_time: 2026-08-30 09:07:59
status: wip
---

# Plan: Render Memory Webs as the final agent-instruction section

## Outcome and boundaries

Change the canonical managed-agent template used by `sase init`/`sase memory init` so
the three generated memory H2 sections appear in this order:

1. `Core Memory`
2. `Reference Memory`
3. `Memory Webs`

`Memory Webs` remains an optional, self-contained section: when a root has no memory
webs, its empty `web_sections` expansion must disappear and the document must continue
to contain only Core Memory followed by Reference Memory. The content and ordering of
notes _within_ each section do not change.

Keep the AMD parser order-agnostic. Existing agent documents and user-owned template
overrides may still have the former Core → Memory Webs → Reference order when SASE reads
them for migration or description recovery; changing the default output must not make
those documents unreadable. Do not silently rewrite the authored structure of a custom
`memory.agents_template`; instead, update SASE's documented/example template shape to
put `{{ web_sections }}` last, while retaining structural validation of the rendered
sections and entries.

The accepted Memory Webs decision contains one historical placement sentence saying the
section sits between core and reference memory. That durable memory cannot be changed or
superseded in this turn because the required `/sase_memory_write` skill is not
installed. Do not bypass the memory-write workflow during implementation; record the
inconsistency for a separately authorized memory update if the skill is still
unavailable.

## Implementation

### 1. Establish the canonical template order

- In `src/sase/amd/templates/AGENTS.template.md`, move the complete `{{ web_sections }}`
  expansion below the Reference Memory heading and `{{ reference_entries }}` body.
  Preserve the fact that `_render_web_sections()` owns the Memory Webs H2 heading and
  instruction paragraph, so an empty expansion leaves no blank/empty Memory Webs
  section.
- Leave `src/sase/amd/_memory.py`, `src/sase/amd/_template.py`, and
  `src/sase/amd/_agents_doc.py` behavior unchanged unless a focused regression exposes a
  real ordering assumption. Their current split is desirable: the template owns output
  order, numbering happens after expansion, and the parser discovers each section
  independently of order.
- Update `docs/configuration.md` to state the canonical managed-template placement:
  `{{ web_sections }}` follows the Reference Memory block and renders the final H2 when
  webs exist. Clarify that custom templates control their authored layout rather than
  implying SASE will reorder arbitrary custom H2 sections.

### 2. Migrate section-number and ordering coverage

- Update the direct section-numbering example in `tests/main/test_section_numbers.py` to
  expect Reference Memory and its H3 children under section 2 and Memory Webs and its H3
  children under section 3.
- Update the managed-template fixture and rendered-document expectations in
  `tests/main/test_init_memory_agents_templates.py` to use the new canonical order.
  Preserve explicit parser coverage for the former order (by retaining a dedicated
  old-order example or parameterizing both layouts) so a first init after upgrade can
  still recover reference descriptions from an existing document.
- Adjust the Memory Webs and glossary integration helpers/assertions so they slice the
  final web section correctly, assert Core < Reference < Memory Webs, and verify that
  web descriptors are absent from both earlier sections. Make at least one integration
  assertion prove Memory Webs is the last H2 rather than only checking its number.
- Renumber affected generated-output assertions throughout the init-memory suite:
  top-level reference-note entries move from `3.x` to `2.x`, while web descriptors move
  from `2.x` to `3.x`. Keep webless-root assertions at `## 2. Reference Memory` and
  retain checks that no Memory Webs heading is emitted in that case.
- Prefer semantic section-order assertions in the main end-to-end tests and reserve
  exact numeric assertions for the numbering-focused tests, reducing future fixture
  churn without weakening the requested contract.

### 3. Regenerate tracked instruction outputs and prove convergence

- After installing the edited checkout, run `sase memory init --no-commit` so the
  repository's managed `AGENTS.md` is rendered from the changed template and every
  tracked provider shim (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, and `QWEN.md`) remains
  a byte-for-byte copy with Memory Webs as section 3 and the final H2.
- Review the generated diff to ensure it is an ordering/renumbering change only: Core
  content is unchanged, Reference Memory precedes Memory Webs, reference descriptions
  survive regeneration, web descriptor bodies/rosters remain intact, and strand bodies
  remain absent.
- Run `sase memory init --check` after regeneration to prove a second init is clean and
  no output oscillates between layouts.

## Verification

1. Run `just install` before invoking the checkout's CLI or test environment.
2. Run focused pytest coverage for section numbering, managed templates, Memory Webs,
   glossary rendering, managed generation/frontmatter/descriptions, validation, plan
   rendering, and init onboarding. Fix every failure as a semantic expectation update;
   do not bulk-replace section numbers without checking whether the fixture has webs.
3. Run `sase memory init --check` and confirm zero drift after the generated files have
   been refreshed.
4. Run the required repository gate with `just check`. If the change reaches the
   full-suite landing gate, run `just check-full` only through `/sase_monitor`, per the
   project's two-speed verification policy.

## Acceptance criteria

- A managed root with memory webs renders exactly three generated memory H2 sections in
  the order Core Memory, Reference Memory, Memory Webs; numbering is `1`, `2`, `3`, and
  Memory Webs is the final H2.
- Reference-note entries are numbered under section 2 and web-descriptor entries under
  section 3, with their prior content, ordering, and inline/strand visibility rules
  unchanged.
- A webless managed root still renders Core Memory followed by Reference Memory, with no
  empty Memory Webs heading and Reference Memory numbered as section 2.
- Existing documents in the former Core → Memory Webs → Reference order remain parseable
  for migration and reference-description recovery.
- The root `AGENTS.md` and all tracked provider instruction shims converge to identical
  generated content in the new order; `sase memory init --check` and `just check` pass.
