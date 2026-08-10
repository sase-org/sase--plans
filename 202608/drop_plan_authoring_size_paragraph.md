---
tier: tale
title: Drop the plan-authoring-size paragraph from the generated SASE sizes memory
goal:
  The "Authoring a plan with `/sase_plan` is itself `large` or `xlarge` work" paragraph
  is deleted from the packaged sizes template and from the regenerated
  `sase/memory/sase_sizes.md`, with no other memory content changed.
size: small
proposed_by: bbugyi200.athena.xd
create_time: 2026-08-10 12:57:31
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.xd](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.xd.md)
- **COMMITS:**
  - [24dde37](https://github.com/sase-org/sase/commit/24dde377599fa0bdcec8207663fdd11517e87204)
    — docs: drop plan authoring size paragraph

# Drop the plan-authoring-size paragraph from the generated SASE sizes memory

## Problem

`sase/memory/sase_sizes.md` contains a paragraph that the project owner judged to be not
useful, not clear, and not accurate:

```
Authoring a plan with `/sase_plan` is itself `large` or `xlarge` work: `large` means the
agent authors a tale, while `xlarge` means the agent authors an epic. The task or phase
size names the handoff; the tale plan's own `size` then names the follow-up
implementation scope.
```

That memory note is **generated** — `sase memory init` renders it from a packaged
template, so editing `sase/memory/sase_sizes.md` by hand would be reverted on the next
init (which also runs as a sase post-commit hook). The real source of truth is the
packaged template:

- `src/sase/main/init_memory/templates/memory-sase-sizes.template.md`

The template is rendered by `_render_generated_long_memory_content` in
`src/sase/main/init_memory/root_rendering.py` (spec `_GENERATED_SIZES_MEMORY_SPEC`,
constant `MEMORY_SASE_SIZES_TEMPLATE_FILENAME`). Unlike `memory-sase.template.md` and
`memory-README.template.md`, this template has **no** user-override hook
(`resolve_markdown_template_override` is not consulted for it), so the packaged file is
the only place the text can come from.

The paragraph exists verbatim in exactly two files in this repo — the template and the
generated note. A repo-wide search for its distinctive phrases (`authors a tale`,
`authors an epic`, `names the handoff`, `follow-up implementation scope`) returns no
other hits.

## Goal

Remove the paragraph from the packaged template and regenerate the derived memory
artifacts so the checked-in `sase/memory/sase_sizes.md` no longer contains it.

## Non-goals

- Do **not** rewrite, condense, or relocate the paragraph's content elsewhere. The owner
  asked for a complete deletion, not a rewording.
- Do **not** touch the neighbouring paragraphs in the template. In particular, keep the
  paragraph directly above it ("Tale plans MUST declare `size: ...`") and the one
  directly below it ("When creating a new task bead, default to `large`. ...") exactly
  as they are.
- Do **not** change the `/sase_plan` skill source
  (`src/sase/xprompts/skills/sase_plan.md`). Step 2 of that skill contains a related but
  separate line — "Authoring a tale plan is `large` work; authoring an epic plan is
  `xlarge` work." — which was **not** part of the deletion request. See "Follow-up worth
  reporting" below.
- Do not change any other memory note, template, or `sase memory init` logic.

## Implementation

### Step 1 — Delete the paragraph from the packaged template

Edit `src/sase/main/init_memory/templates/memory-sase-sizes.template.md`.

Note that the template is stored with a wider line-wrap than the generated note (the
generated note is re-wrapped by `format_generated_memory_markdown`), so the paragraph
occupies three lines in the template rather than four. Delete this paragraph and the
blank line that separates it from the following paragraph:

```
Authoring a plan with `/sase_plan` is itself `large` or `xlarge` work: `large` means the agent authors a tale, while
`xlarge` means the agent authors an epic. The task or phase size names the handoff; the tale plan's own `size` then
names the follow-up implementation scope.
```

After the edit, the paragraph beginning `Tale plans MUST declare` must be followed by a
single blank line and then the paragraph beginning
`When creating a new task bead, default to `large`.` — no double blank line, no trailing
whitespace.

Leave the template's YAML frontmatter (`type`, `parent`, `description`) untouched: the
`description` still accurately describes the note, and `sase memory init` propagates it
into `AGENTS.md` and `sase/memory/README.md`.

### Step 2 — Regenerate the derived memory artifacts

The workspace may be stale, so install first, then regenerate:

```bash
just install
sase memory init
```

This is the mandatory completion step for any memory change: it re-renders
`sase/memory/sase_sizes.md` from the edited template, refreshes `sase/memory/README.md`
(which records per-note `Lines:` and `Approx. tokens:` statistics — the sizes entry
currently reads `Lines: 46` / `Approx. tokens: 546` and must drop), and rewrites
`AGENTS.md` plus the provider instruction shims (`CLAUDE.md`, `GEMINI.md`,
`OPENCODE.md`, `QWEN.md`).

Do not hand-edit `sase/memory/sase_sizes.md`, `sase/memory/README.md`, `AGENTS.md`, or
the shims — let `sase memory init` write them.

### Step 3 — Verify the regenerated output

Confirm all of the following:

1. `sase/memory/sase_sizes.md` no longer contains the paragraph, and the surrounding
   paragraphs survived intact:

   ```bash
   grep -rn "authors a tale\|authors an epic\|names the handoff\|follow-up implementation scope" . \
     --exclude-dir=.git
   ```

   The only remaining acceptable hits are none — the phrases should be gone from the
   repo entirely. (`src/sase/xprompts/skills/sase_plan.md` uses different wording and
   will not match these phrases.)

2. `git diff --stat` shows changes limited to: the template,
   `sase/memory/sase_sizes.md`, `sase/memory/README.md`, and — only if their content
   actually shifted — `AGENTS.md` and the provider shims. `AGENTS.md` is expected to be
   unchanged, because `sase_sizes.md` is a `long` note and `AGENTS.md` carries only its
   unchanged `description`. If `AGENTS.md` or the shims _do_ change, inspect the diff
   before accepting it; an unexpected change there means something beyond this edit
   moved.

3. `sase memory init` is idempotent — running it a second time produces no further diff.

### Step 4 — Gate

```bash
just check
```

Run `just install` first if it was not already run in Step 2. `just check` runs the
whole-repo lint gates plus the diff-scoped test lane.

The relevant existing coverage lives in
`tests/main/test_init_memory_markdown_templates.py` — notably
`test_default_sizes_template_renders_canonical_child_long_note`, which asserts the
rendered note's `type`, `parent` (`sase/memory/sase_beads.md`), and non-empty
`description`. No test asserts the deleted paragraph's text, so no test edits are
expected. If `just check`'s scoped selection escalates or reports anything unusual, run
`just check-full` instead.

## Acceptance criteria

- The paragraph is gone from
  `src/sase/main/init_memory/templates/memory-sase-sizes.template.md`.
- The paragraph is gone from the regenerated `sase/memory/sase_sizes.md`, and that file
  was produced by `sase memory init` rather than hand-edited.
- Every other paragraph of the sizes memory is byte-for-byte unchanged apart from the
  removal.
- `sase/memory/README.md` statistics for `sase/memory/sase_sizes.md` reflect the shorter
  note.
- A second `sase memory init` produces no diff.
- `just check` passes.

## Follow-up worth reporting

`src/sase/xprompts/skills/sase_plan.md` step 2 still says "Authoring a tale plan is
`large` work; authoring an epic plan is `xlarge` work." That sentence expresses roughly
the claim the owner called inaccurate, but it lives in the plan skill rather than in the
sizes memory and was **not** in scope for this deletion. Do not change it. Instead,
mention it in the completion report so the owner can decide whether it deserves its own
task bead.
