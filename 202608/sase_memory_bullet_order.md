---
tier: tale
title: Reorder the SASE Memory bullets to match the new section order
goal:
  The three memory-kind bullets in the `## SASE Memory` section of the generated
  `sase/memory/sase.md` note read Core memory, then Reference memory, then Memory webs —
  matching the order the generated agent instruction files now render their sections —
  with the generator template as the only edited source, no bullet text changed, and no
  committed drift.
size: small
proposed_by: bbugyi200.athena.sase-vk.land.w2.f0
create_time: 2026-08-30 11:04:40
status: wip
---

# Reorder The `SASE Memory` Bullets To Core → Reference → Memory Webs

## Goal

In the `## SASE Memory` section of the generated core note `sase/memory/sase.md`, the
three memory-kind bullets currently read **Core memory → Memory webs → Reference
memory**. Reorder them to **Core memory → Reference memory → Memory webs** so the
bullets enumerate the memory kinds in the same order the generated agent instruction
files now render their sections, and republish every generated file that inlines them.

This is a pure reordering. Not one word of any bullet changes.

## Background

- A recent change made `## Memory Webs` the third and final H2 section of generated
  agent instructions. `AGENTS.md` now renders its H2 sections as `1. Core Memory`, then
  `2. Reference Memory`, then `3. Memory Webs`, but the bullets inside the `SASE Memory`
  core note still introduce the kinds in the old order, listing webs before reference
  memory. This plan closes that mismatch; it does **not** revisit the section order
  itself.

- `sase/memory/sase.md` is a _generated_ memory note and refuses direct edits. Its only
  source of truth is the packaged Jinja template
  `src/sase/main/init_memory/templates/memory-sase.template.md`, rendered by
  `_render_sase_memory()` in `src/sase/main/init_memory/root_rendering_notes.py`.

- **No template override exists anywhere**, so the packaged template is the single
  source for both the project root and the home root. This was verified, not assumed:
  the project's `sase/sase.yml` sets neither `memory.sase_template` nor the legacy
  `memory_sase_template`, and no `memory-sase.template.md` exists in the chezmoi
  user-config directory (`home/dot_config/sase/`). Note also that
  `_user_config_dir_for_home_root()` in `src/sase/amd/_config.py` only consults a user
  config directory for the home root, never for a project root.

- In the template, each of the three bullets is a **single unwrapped line** — lines 7, 8
  and 9 of the file. The edit is therefore a swap of two whole lines, nothing more.

- Rendered output passes through `format_generated_memory_markdown()`
  (`src/sase/main/init_memory/formatting.py`), which re-wraps prose to
  `markdown_print_width()` (88). Because each bullet is re-wrapped independently of its
  neighbours, reordering them leaves every bullet's own wrapped form byte-for-byte
  identical to what is committed today — only the order of the three blocks changes.

- `src/sase/main/init_memory/templates/` is listed in `.prettierignore`, so the template
  file is not prose-wrapped by `just fmt`. Long single-line bullets are correct there.

- The note is inlined into the managed `AGENTS.md` and its byte-identical provider shims
  (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`) at the repo root. Those files are
  generated — never hand-edit them.

- `tests/main/test_init_memory_committed_drift.py::test_repo_project_memory_notes_match_generator_output`
  fails if the committed project tree does not match generator output, so regeneration
  is mandatory, not optional.

- **No test pins the order or the text of these three bullets.** The tree was searched
  for the bullet labels and for distinctive phrases from each bullet
  (`keyed collections`, `one-line description`, `is not inlined`); the only assertions
  that reach this section of the note check the heading `#### 1.1.1 SASE Memory` and the
  opening prose (`tests/main/test_init_memory_agent_docs.py`,
  `tests/main/test_init_memory_managed_agents_generation.py`), neither of which this
  change touches. Expect to update **no** test files. If `just check` nevertheless
  surfaces a failing assertion that pins the old bullet order, repoint it at the new
  order rather than reverting the template.

## Target Text

Lines 7–9 of `src/sase/main/init_memory/templates/memory-sase.template.md` currently
hold the three bullets in this order, one unwrapped line each:

1. the **Core memory** bullet
2. the **Memory webs** bullet
3. the **Reference memory** bullet

Swap bullets 2 and 3 so the final order is:

1. the **Core memory** bullet (unchanged, stays first)
2. the **Reference memory** bullet
3. the **Memory webs** bullet

Move each bullet line **verbatim**. Do not re-wrap, re-punctuate, or otherwise touch the
bullet bodies; the diff for this file must be exactly two lines moved.

After regeneration, the corresponding block in `sase/memory/sase.md`, `AGENTS.md`, and
every provider shim must read:

```markdown
- **Core memory** (`type: core`) is inlined here and into every provider instruction
  shim, so it is always in your context and is paid for on every turn.
- **Reference memory** (`type: reference`) is not inlined. Only its one-line description
  is listed here; read the body on demand with your `/sase_memory_read` skill, never by
  opening the file directly.
- **Memory webs** are keyed collections: a flat descriptor note (`sase/memory/<web>.md`)
  plus a sibling directory of strand files (`sase/memory/<web>/<slug>.md`). A web's
  descriptor is always inlined here; a strand body never is — read strands on demand
  with your `/sase_memory_read` skill (`sase memory read <web>:<keyword>`, for example
  `glossary:stitch`).
```

## Steps

1. **Edit the template.** In
   `src/sase/main/init_memory/templates/memory-sase.template.md`, swap the
   `- **Memory webs**` line and the `- **Reference memory**` line so the three bullets
   run Core → Reference → Webs. Change nothing else in the file — the heading, the
   opening paragraph, the closing `/sase_memory_write` paragraph, the Jinja `{% if %}`
   blocks, and every blank line all stay exactly as they are.

2. **Make sure the workspace's editable install is current**, because step 3 must run
   _this checkout's_ code:

   ```bash
   just install
   ```

3. **Regenerate every generated file** using the workspace virtualenv's entry point:

   ```bash
   .venv/bin/sase memory init --no-commit
   ```

   - Use `.venv/bin/sase`. A globally installed `sase` on `PATH` still carries the old
     packaged template and would regenerate the _old_ order, silently reverting step 1.
   - `--no-commit` is required: agents never create commits (see
     `sase memory read decisions:host-owned-completion`). The host's finalizers own
     landing this change.
   - Expect this to rewrite `sase/memory/sase.md`, `AGENTS.md`, `CLAUDE.md`,
     `GEMINI.md`, `OPENCODE.md`, and `QWEN.md`.
   - **Side effect to expect and report, not to revert:** on a machine configured with
     `use_chezmoi: true`, `sase memory init` also regenerates the _home_-root copy of
     this note into the chezmoi source repo, auto-commits it there, and runs
     `chezmoi apply`. `--no-commit` does not suppress that — it only skips the project
     git sequence. This is expected and in scope here, because the home root renders
     from the same packaged template and its copy of these bullets is currently in the
     old order too. Do not try to work around it and do not `cd` into the chezmoi
     checkout; if you need to inspect it, open it with your `/sase_repo` skill first and
     use only the path that prints. Name the chezmoi outcome explicitly in your final
     summary, and if `sase memory init` leaves _unrelated_ drift in that repo (a stale
     line in some other note, say), revert only that unrelated part and flag it.

4. **Confirm zero drift**, which is the direct check that steps 1 and 3 agree with the
   committed tree:

   ```bash
   .venv/bin/sase memory init --check
   ```

   It must exit 0 and report no pending memory changes. Running it a second time must
   also be clean — the reorder is a fixpoint, so there is no oscillation to absorb.

5. **Confirm the shims did not diverge.** `AGENTS.md` and the four provider shims are
   byte-identical copies apart from `CLAUDE.md`'s SASE-managed frontmatter; diff them
   against each other and confirm the only change in each is the moved bullet.

6. **Verify.** Run `just check` (whole-repo lint gates plus the diff-scoped test lane).
   If its scoped selection does not pick up
   `tests/main/test_init_memory_committed_drift.py`, run that file explicitly as well,
   since it is the one gate that fails on committed drift. Hand the run to your
   `/sase_monitor` skill if it starts to outrun the turn.

## Non-Goals

- Do not hand-edit `sase/memory/sase.md`, `AGENTS.md`, or any provider shim. They are
  generated; step 3 is the only way they change.
- Do not reword, re-wrap, or re-punctuate any of the three bullets. This is a reorder
  and nothing else.
- Do not touch the opening sentence of the section ("A note's kind — flat note or memory
  web — and a flat note's `type:` frontmatter decide how it reaches you."). It orders
  _kinds_, not the bullets, and the reorder actually improves its alignment: both flat
  kinds are now enumerated before webs.
- Do not change the H2 section order in `src/sase/amd/templates/AGENTS.template.md`.
  That ordering is already correct and is what this plan is aligning the bullets _to_.
- Do not touch `docs/memory.md`. Its own bullet list already runs Core memory →
  Reference memory and never listed webs, so it needs no update.
- Do not touch any other memory note, and do not add a config knob, a template override,
  or a compatibility shim for the old order.

## Acceptance Criteria

- In `src/sase/main/init_memory/templates/memory-sase.template.md`, the
  `- **Reference memory**` line precedes the `- **Memory webs**` line, and the file's
  diff is exactly two moved lines with no textual edits.
- `sase/memory/sase.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, and
  `QWEN.md` each contain the target block from "Target Text" verbatim, and `AGENTS.md`
  remains byte-identical to each provider shim modulo `CLAUDE.md`'s managed frontmatter.
- The bullet order in each of those files matches the rendered section order
  (`## 1. Core Memory` → `## 2. Reference Memory` → `## 3. Memory Webs`).
- `.venv/bin/sase memory init --check` exits 0 with no pending changes, twice in a row.
- `just check` passes, including `tests/main/test_init_memory_committed_drift.py`.
