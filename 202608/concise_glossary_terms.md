---
tier: tale
title: Render Tier 2 glossary terms as one concise GLOSSARY TERMS line
goal:
  Generated agent instruction files list glossary terms as a single wrapped `**GLOSSARY
  TERMS:**` paragraph at the end of the `Glossary Terms` section instead of one bullet
  per term, cutting ~23 always-loaded lines from every managed project's `AGENTS.md` and
  provider shims.
size: small
proposed_by: bbugyi200.athena.07g
create_time: 2026-08-18 21:18:23
status: wip
---

# Plan: Render Tier 2 glossary terms as one concise `**GLOSSARY TERMS:**` line

## Motivation

Every managed project's `AGENTS.md` (and its `CLAUDE.md` / `GEMINI.md` / `OPENCODE.md` /
`QWEN.md` copies) is always-loaded context. The Tier 2 `Glossary Terms` section
currently spends one line per term. In the `sase` repo that is 30 bullet lines today and
grows by one line for every term anyone adds with `sase glossary add`. The terms are a
lookup index, not prose to read top to bottom: an agent only needs to know which names
are glossary-backed so it can run `sase glossary read`. A single semicolon-separated,
prose-wrapped paragraph carries exactly the same information in ~7 lines.

This restores the pre-`445afde7c` `**GLOSSARY TERMS:**` paragraph shape, but keeps the
`### Glossary Terms` H3 section and its instruction paragraph that `445afde7c`
introduced. Only the term list itself becomes a paragraph.

## Current behavior

- `src/sase/amd/_memory.py`
  - `_GLOSSARY_TERMS_TITLE` / `_GLOSSARY_TERMS_HEADING` — the `### Glossary Terms` H3.
  - `_GLOSSARY_TERMS_INTRO` — the instruction paragraph ending
    `Aliases follow in parentheses.`
  - `_render_glossary_term_entry(term, display_aliases)` — renders `Term` or
    `Term (alias, alias)` with `md_escape`. Unchanged by this plan.
  - `_render_glossary_terms_section(glossary_terms)` — returns
    `heading + "\n\n" + intro + "\n\n" + "\n".join(f"- {entry}" ...)`, or `""` when the
    project configures no terms.
  - `_render_managed_agents` splices that section after the `Long-Term Memory Files`
    section into `tier2_entries`, then validates the rendered document still contains a
    heading titled `Glossary Terms`.
- The renderer emits unwrapped text; `sase.main.init_memory.root_rendering` runs the
  result through `sase.main.init_memory.formatting.format_generated_memory_markdown`,
  which wraps prose paragraphs to the Markdown print width before the file is written.

## Target rendering

The H3 heading and the instruction paragraph stay exactly where they are. The bullet
list is replaced by one trailing paragraph that begins with the bold label:

```markdown
### 2.2 Glossary Terms

Run `sase glossary read <term> [<term> ...] -r "<why>"` before relying on any of these
SASE terms; it prints each term's definition plus every term those definitions depend
on. Pass every term you need in one command — one batched read costs far fewer tokens
than one read per term, because terms shared between definitions are printed once. Terms
are separated by semicolons; aliases follow in parentheses.

**GLOSSARY TERMS:** Agent Clan; Agent Family; Agent Hood (hood, agent neighborhood);
Agent Instruction File (agents.md file); Agent Neighbor; Agent Node; Agent Shell; Agent
Tribe; Artifact Reference (ref); Current Project; Feature Flag; Flag Bead (flag bead);
Patch; Proc (background task); Proc Shell; Required Plugin (required plugin); Sase Agent
(agent); Sase Monitor (monitor); Sase Node (node); Sase Project (project); Sase Repo
(repo); Sase Shell (shell); Sase Workspace (workspace); Stitch; Task Type (task type);
Xprompt; Xprompt Memory (memory file); Xprompt Part; Xprompt Swarm; Xprompt Workflow
```

Design decisions, already validated against the real term list:

- **Semicolon separators, not commas.** Aliases are comma-separated inside parentheses,
  so commas between terms would be ambiguous (`Agent Hood (hood, agent neighborhood)`).
  Semicolons keep term boundaries unambiguous and match the pre-`445afde7c` separator.
- **Keep the bold `**GLOSSARY TERMS:**` label** even though the H3 already names the
  section: it makes the terms line visually distinct from the instruction paragraph
  above it, and it is a cheap grep anchor.
- **Term order and alias rendering are unchanged** — same `_render_glossary_term_entry`
  output, same catalog order, derivable plurals still omitted upstream by
  `ProjectGlossaryTerms`.
- **Formatting is a verified fixpoint.** Both `prettier` (`proseWrap: always`,
  `printWidth: 88`) and `format_generated_memory_markdown` were run against the exact
  paragraph above and both are idempotent and agree with each other, so no change to
  `formatting.py` is expected. The label line is not a `_STANDALONE_STRONG_LABEL_RE`
  match because content follows it on the same logical line, so it wraps as ordinary
  prose.

## Implementation

### 1. `src/sase/amd/_memory.py`

- Add a module constant next to the other glossary constants:

  ```python
  _GLOSSARY_TERMS_LABEL = "**GLOSSARY TERMS:**"
  ```

- Change the last sentence of `_GLOSSARY_TERMS_INTRO` from
  `"... printed once. Aliases follow in parentheses."` to
  `"... printed once. Terms are separated by semicolons; aliases follow in parentheses."`
  so the separator convention is stated where an agent reads it.
- Rewrite `_render_glossary_terms_section` to join the entries with `"; "` and emit them
  as a single trailing paragraph rather than a bullet list:

  ```python
  def _render_glossary_terms_section(glossary_terms: ProjectGlossaryTerms) -> str:
      """Render the Tier 2 ``Glossary Terms`` H3 section."""
      if not glossary_terms.terms:
          return ""
      entries = "; ".join(
          _render_glossary_term_entry(term, display_aliases)
          for term, display_aliases in glossary_terms.terms
      )
      return (
          f"{_GLOSSARY_TERMS_HEADING}\n\n{_GLOSSARY_TERMS_INTRO}\n\n"
          f"{_GLOSSARY_TERMS_LABEL} {entries}"
      )
  ```

- Do not emit a trailing period after the last term (matches the prior shape and avoids
  a period being read as part of a term).
- No other function in this module needs to change: the empty-glossary early return, the
  section ordering in `_render_managed_agents`, and the `_rendered_has_heading` guard
  for `Glossary Terms` all still hold.

### 2. Tests

`tests/main/test_init_memory_glossary.py`

- `test_memory_plan_renders_glossary_terms_block_in_tier2`: replace the bullet
  assertions (`"- Agent Clan (clan)"`, `"- Workspace"`) with the paragraph form, and
  flip the stale negative assertion:
  - assert `"**GLOSSARY TERMS:**" in agents`;
  - assert the single-spaced Tier 2 text (`" ".join(tier2.split())`) contains
    `"**GLOSSARY TERMS:** Agent Clan (clan); Workspace"` — normalizing whitespace is
    required because the paragraph is prose-wrapped;
  - keep `"agent clans" not in tier2` (derivable plural still suppressed) and keep the
    heading/order assertions;
  - update the `"Aliases follow in parentheses."` assertion to the new sentence
    (`"Terms are separated by semicolons; aliases follow in parentheses."`).
  - assert the section contains no `- ` list item any more, e.g. that no line of the
    text after `### 2.2 Glossary Terms` starts with `- `.
- `test_memory_plan_omits_parens_when_only_alias_is_term_plural`: replace the
  `"- Patch\n" in tier2 or ...endswith("- Patch")` assertion with
  `"**GLOSSARY TERMS:** Patch" in " ".join(tier2.split())`; keep
  `"Patch (patches)" not in tier2` and `"patches" not in tier2`.
- `test_memory_plan_glossary_block_terms_are_semicolon_separated_and_format_stable`:
  this test's name already describes the target behavior; make the body match it. Assert
  on whitespace-normalized Tier 2 text that it contains
  `"**GLOSSARY TERMS:** Agent Clan (hood, agent neighborhood); Artifact Reference (ref)"`,
  keep `"artifact references" not in tier2`, and keep the
  `format_generated_memory_markdown(agents) == agents` fixpoint assertion.
- `test_memory_apply_generates_glossary_block_idempotently_and_copies_provider_shims`:
  replace `"- Workspace"` with the `**GLOSSARY TERMS:** Workspace` paragraph form
  (whitespace-normalized). Leave the provider-shim equality, `--check`, and
  no-further-actions assertions as they are — they are the idempotency guard that
  matters most here.
- The two no-glossary tests
  (`test_memory_init_deletes_stale_generated_glossary_note_without_configured_terms`,
  `test_memory_init_ignores_home_glossary_config`) already assert
  `"GLOSSARY TERMS" not in ...`; they stay correct and unchanged.

`tests/main/test_init_memory_markdown_templates.py`

- `test_glossary_terms_block_is_sole_tier2_content_without_other_notes`: replace
  `"- Agent Hood (hood, agent neighborhood)"` and `"- Stitch"` with the
  whitespace-normalized `**GLOSSARY TERMS:** ...` paragraph assertion; keep
  `"### 2.1 Glossary Terms" in plan.agents_content` and
  `parsed.long_memory_entries == ()`.
- `test_long_memory_files_section_precedes_glossary_terms`: no change expected — it
  asserts on headings and long-memory parsing only. Re-run it to confirm the paragraph
  does not perturb `parse_amd_agents_document`'s long-memory entry collection (that
  parser stops description collection at the next heading, and the glossary paragraph
  lives under its own H3, so it must stay out of the preceding note's description).

`tests/main/test_init_memory_managed_agents.py`

- Line ~146 asserts `"Glossary Terms" not in agents` for a project with no glossary; no
  change expected. Re-run to confirm.

Add one new focused test (in `tests/main/test_init_memory_glossary.py`) that pins the
regression this plan is really about: for a project with several multi-word terms, the
rendered Tier 2 `Glossary Terms` section is at most a small, bounded number of lines and
contains zero Markdown list items — i.e. term count no longer drives line count. Keep it
assertion-precise rather than a snapshot (e.g. assert no line in the section matches
`^- `, and that every term appears exactly once in the normalized paragraph).

### 3. Documentation

Update the prose that describes the rendered shape. Do not restructure these pages;
these are one- or two-sentence edits:

- `docs/memory.md` (§ Glossary, ~line 138-142): it already says "compact"; make it say
  the section ends with a single `**GLOSSARY TERMS:**` line naming every term,
  semicolon-separated, with aliases in parentheses.
- `docs/init.md` (~line 205-212): "It lists every displayed glossary term with its
  aliases in parentheses" → state that the terms are rendered as one semicolon-separated
  `**GLOSSARY TERMS:**` paragraph at the end of the section.
- `docs/configuration.md` (§ `memory.glossary`, ~line 538-548): same wording update
  where it describes the section "naming every displayed term and alias".
- `docs/configuration.md` (§ generated templates, ~line 493-497 and §
  `sase memory init`, ~line 4248-4252): re-read these; they describe the section
  generically and probably need no change. Only touch them if they claim a bullet list.

Search the docs tree for any other place that shows the bullet-rendered section before
declaring this step done.

### 4. Regenerate the managed instruction files

This repo's own `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, and `QWEN.md` are
generated, and they will drift the moment the renderer changes. Regenerate them without
letting the command touch git:

```bash
sase memory init --no-commit
```

Permission note for the implementing agent: the standing rule is that generated
instruction files and `sase/memory/*.md` notes are never edited without explicit user
permission. The user's request for this work is specifically a request to change how
glossary terms are rendered _in the agent instruction files_, so regenerating them via
`sase memory init` is the authorized, mandatory completion of that request — no separate
approval is needed for the regeneration itself. What is still **not** authorized:
editing any canonical note under `sase/memory/` by hand, or changing any instruction
content other than the glossary-terms rendering this plan describes. The regenerated
diff should touch only the `Glossary Terms` sections of the five generated files.

Use `--no-commit` so the run does not create a git commit or push; committing goes
through the normal `/sase_git_commit` flow if and when the user asks for it.

## Verification

Run from the workspace checkout, in order:

1. `just install` — required before anything else; ephemeral workspaces can carry a
   stale virtualenv, and `sase` will fail with a missing `sase_core_rs` import until
   this is run.
2. `sase memory init --no-commit`, then `sase memory init --check` — the second run must
   exit 0, proving the new rendering is a generation fixpoint.
3. `git diff` on `AGENTS.md` and the four provider shims — confirm the only change is
   the `Glossary Terms` section, the five files are byte-identical to each other, and
   the section shrank from 30 bullet lines to a wrapped paragraph.
4. `just check` — whole-repo lint gates plus the diff-scoped test lane. This covers
   `fmt-md-check` (prettier must accept the regenerated Markdown unchanged) and the
   updated tests. If it runs long, hand it to `/sase_monitor` with a `--next` action
   rather than blocking inline.
5. If the scoped selection reports anything unusual, escalate to `just check-full`
   through `/sase_monitor`.

## Non-goals

- Do not change `_render_glossary_term_entry`, alias suppression, catalog ordering, or
  anything in `src/sase/main/init_memory/glossary.py`.
- Do not change the `sase glossary` CLI output (`list` / `show` / `read`) — this plan is
  only about the generated instruction-file rendering.
- Do not change `format_generated_memory_markdown`; the target paragraph was verified to
  be a fixpoint for both it and prettier.
- Do not remove or reword the `Glossary Terms` H3 heading, its position after
  `Long-Term Memory Files`, or the Tier 2 numbering.
- Do not shorten the instruction paragraph beyond the one separator sentence described
  above; the token win here comes from the term list, not the prose.
