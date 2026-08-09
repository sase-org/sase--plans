---
tier: tale
title: Render ALIASES instead of Aliases in generated glossary memory
goal:
  Glossary entries with configured aliases render an uppercase `ALIASES:` label in
  `sase/memory/glossary.md` and in every generated agent instruction file.
size: medium
proposed_by: bbugyi200.athena.wa
create_time: 2026-08-09 07:39:19
status: wip
---

# Render `ALIASES:` instead of `Aliases:` in generated glossary memory

## Problem

`sase memory init` renders a project's `glossary` config (from `sase/sase.yml`) into the
managed `sase/memory/glossary.md` short note, and that note body is inlined verbatim
into `AGENTS.md` and every provider instruction shim (`CLAUDE.md`, `GEMINI.md`,
`OPENCODE.md`, `QWEN.md`). Each entry that declares `aliases:` currently gets a
paragraph labeled `Aliases: <a, b, c>`.

The user wants that label rendered in all caps — `ALIASES: <a, b, c>` — so it reads as a
structured field marker in agent instruction files (consistent with the `IMPORTANT:`
style emphasis those files already use) rather than as ordinary prose.

## Current behavior

The label is produced in exactly one place:

- `src/sase/main/init_memory/glossary.py:150` inside `_render_glossary_memory()`:

  ```python
  if entry.configured_aliases:
      aliases = ", ".join(md_escape(alias) for alias in entry.configured_aliases)
      lines.extend(["", f"Aliases: {aliases}"])
  ```

Everything downstream is derived from that string:

1. `_render_glossary_memory()` passes the joined lines through
   `format_generated_memory_markdown()` (`src/sase/main/init_memory/formatting.py`),
   which wraps paragraphs to the prettier print width so the output is a prettier
   fixpoint.
2. `apply_memory_frontmatter()` stamps `sase_generated: glossary` frontmatter, producing
   `GeneratedGlossaryMemory.content` (the note file) and `.body` (what gets inlined).
3. `src/sase/main/init_memory/root_rendering.py` writes `sase/memory/glossary.md` from
   `.content` and composes `.body` into `AGENTS.md` Tier 1 (section
   `### N. Glossary (glossary)`), then copies `AGENTS.md` to the provider shims.

So a one-line change to the label, plus regenerating derived files, is the whole change.

## Explicitly out of scope

These also print `Aliases:` but are **different surfaces** and MUST NOT be changed by
this plan:

- `src/sase/ace/tui/widgets/_prompt_glossary.py:373` — the ACE TUI hover/`K` preview
  Markdown for a glossary term. Human-facing TUI presentation, not an agent instruction
  file. Leaving it alone also keeps `tests/ace/tui/widgets/test_prompt_glossary.py:268`
  and the ACE PNG visual snapshots untouched.
- `src/sase/main/project_handler.py:142`,
  `src/sase/ace/tui/modals/project_management_rendering.py:237`,
  `src/sase/ace/tui/modals/project_alias_editor_modal.py:33` — these render _project_
  aliases, an unrelated concept from glossary aliases.

## Implementation steps

### 1. Change the rendered label

In `src/sase/main/init_memory/glossary.py`, `_render_glossary_memory()`, change the
f-string at line 150 from `f"Aliases: {aliases}"` to `f"ALIASES: {aliases}"`.

Prefer introducing a module-level constant next to the other module constants so the
literal is named once and tests can reference it, e.g.:

```python
GLOSSARY_ALIASES_LABEL = "ALIASES"
```

and render `f"{GLOSSARY_ALIASES_LABEL}: {aliases}"`. Keep the change to the label only —
do not touch `md_escape`, the join separator, the blank-line placement, or the
`entry.configured_aliases` guard.

Note on wrapping: `ALIASES: ` is two characters longer than `Aliases: `. The longest
current alias paragraph in this repo is
`Aliases: agent instruction file, agents.md files, agents.md file` (64 chars), so it
stays well under the wrap width and no reflow is expected. Do not hand-adjust any
wrapping — step 3 regenerates everything.

### 2. Update the unit test

`tests/main/test_init_memory_glossary.py:72` asserts:

```python
assert "Aliases: agent clans, clan" in glossary_text
```

Update it to `"ALIASES: agent clans, clan"`. While there, add a negative assertion in
the same test so a future regression to title case is caught, e.g.
`assert "Aliases: agent clans, clan" not in glossary_text`.

This is the only test in the repo that asserts on the generated glossary label; a full
`grep -rn "Aliases" tests/ src/` was used to confirm the other hits are the out-of-scope
surfaces listed above. Re-run that grep after editing to confirm nothing was missed.

### 3. Regenerate the derived memory artifacts

The generated glossary note and the agent instruction files are checked in, and CI runs
`sase init memory --no-commit` (`.github/workflows/ci.yml:92`), so stale copies will
fail. After step 1, run:

```bash
just install      # ephemeral workspace: dependencies may be stale
sase memory init
```

That regenerates, in this repo:

- `sase/memory/glossary.md` (17 `Aliases:` lines → `ALIASES:`)
- `AGENTS.md` and the provider shims `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`
  (17 lines each)

**Authorization note:** these are SASE memory / agent instruction files, which normally
require explicit user permission to modify. The user explicitly requested this rendering
change, which carries approval for the mandatory `sase memory init` regeneration. Do NOT
hand-edit any of those six files — every one of them must be produced by
`sase memory init` so the content is exactly what the generator emits. Verify with
`git diff --stat` that only the expected six files changed and that each shows purely
`Aliases:` → `ALIASES:` line replacements (no reflow, no other content drift).

Then confirm idempotence and check-mode cleanliness:

```bash
sase memory init --check
```

It must exit 0 with no pending actions.

### 4. Document the rendered label (small docs touch)

`docs/configuration.md` (the `### glossary` section, around lines 505-540) describes
what `sase memory init` generates from the glossary config but never states the rendered
alias label. Add one short sentence there noting that entries with `aliases:` render an
`ALIASES: <comma-separated>` line in the generated note and in the inlined agent
instructions. Keep it to a sentence in the existing paragraph that begins "Run
`sase memory init` after editing glossary entries."; do not restructure the section.

`docs/init.md` (around lines 201-210) describes the same generation at a higher level
and does not mention the label — leave it unchanged.

## Verification

```bash
just install
just check
```

`just check` runs the whole-repo lint gates plus the diff-scoped test lane. The scoped
lane should pick up `tests/main/test_init_memory_glossary.py`; if it does not, run it
explicitly. Because this change touches checked-in `AGENTS.md` / memory files and docs,
also run:

```bash
just check-full
```

Manual confirmation:

```bash
grep -n "ALIASES:" sase/memory/glossary.md | head
grep -rn "^Aliases:" AGENTS.md CLAUDE.md GEMINI.md OPENCODE.md QWEN.md sase/memory/glossary.md
```

The first must show the new label; the second must return no matches.

## Definition of done

- `src/sase/main/init_memory/glossary.py` emits `ALIASES: ` for glossary entries with
  configured aliases.
- `tests/main/test_init_memory_glossary.py` asserts the new label and rejects the old
  one.
- `sase/memory/glossary.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, and
  `QWEN.md` are regenerated by `sase memory init` (never hand-edited) and contain no
  `Aliases:` lines.
- `sase memory init --check` exits 0.
- `docs/configuration.md` documents the `ALIASES:` label.
- The ACE TUI glossary preview and all project-alias surfaces are untouched.
- `just check-full` passes.

## Notes for the implementing agent

- Other SASE projects with their own `glossary:` config (plugin repos, chezmoi) will
  pick up the new label the next time their own `sase memory init` runs. Do not attempt
  to regenerate other repos as part of this change.
- Glossary entries cannot be supplied by global/user/plugin config (only a project's
  `sase/sase.yml`), so there is no home-root `AGENTS.md` glossary section to regenerate.
