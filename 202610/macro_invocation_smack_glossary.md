---
tier: tale
title: Name macro use sites "Macro Invocation" (smack) in the glossary
goal:
  The glossary names the `#foo` use site Macro Invocation, with the aliases smack, macro
  reference, and macro ref. A new Raw Prompt strand defines the submitted, raw, and
  expanded prompt stages. The roster, provider shims, and terminology guard are
  regenerated or updated, and no code, CLI, or TUI identifiers change.
size: small
proposed_by: bbugyi200.athena.sase-1eq.11.w0
create_time: 2026-10-05 09:16:45
status: wip
---

# Name the `#foo` Use Site: "Macro Invocation" (smack) and "Raw Prompt" Glossary Strands

## Goal

SASE has a glossary term for the macro **definition** (`Macro`) but none for the `#foo`
token a user types in a prompt. Code, docs, and agent prose call that token several
different things ("macro reference", "macro ref", "reference", "invocation"). Give the
use site a name:

- **Macro Invocation** is the canonical glossary keyword.
- **smack** is a short alias. It is a blend of **S**ase + **MAC**ro + invo**C**ation,
  and the "kay" sound is spelled _k_.
- Add aliases `macro reference` and `macro ref` too, so existing prose links to the new
  term.

Also add a **Raw Prompt** strand. It defines the submitted, raw, and expanded prompt
stages, which is the confusion behind the idea of a "smack prompt".

This plan follows the consolidated research report
`research:202610/macro_invocation_glossary_term_smack/macro_invocation_glossary_term_smack.md`.
The user read the report and agreed with all of its recommendations. In summary:

| Topic                 | Decision                                                                                                                                                                                                     |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Keyword               | `Macro Invocation`. Aliases: `smack`, `macro reference`, `macro ref`.                                                                                                                                        |
| "Sase" in the keyword | No. The macro family is Macro, Macro Part, Macro Swarm, and so on. The body gives the "sase macro invocation" etymology, and the matcher already highlights that phrase through its "macro invocation" span. |
| "smack prompt"        | **Rejected.** A prompt is a container, not one expression inside it. Add a `Raw Prompt` strand instead.                                                                                                      |
| Scope                 | Glossary strands, one sentence in the Macro strand, and one sentence in the docs. Change nothing else.                                                                                                       |
| Public surfaces       | Keep "smack" out of `docs/`, CLI help, config keys, and TUI labels until it is promoted (see the research's promotion test).                                                                                 |
| Sequencing            | The xprompt to macro rename phase `sase-1eq.11` (audit and guardrail) closed on 2026-10-05, so nothing blocks this work. Rebase onto current `master` and recheck the allowlist lines named below.           |

## Authorization and memory-edit procedure

The user's prompt asked for the new glossary strand, and the user agreed with the
research recommendations. Those recommendations include the Raw Prompt strand and the
one-sentence Macro strand edit. Approving this plan authorizes every memory edit below.

- Before editing any memory file, use the `/sase_memory_write` skill.
- Edit only canonical files under `sase/memory/`. Never hand-edit `AGENTS.md`,
  `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`, or the generated roster block in
  `sase/memory/glossary.md`.
- Regenerate those files with `sase memory init --no-commit`. Agents never commit, and
  completion is host-owned. A plain `sase memory init` would try to commit. It would
  also refuse, because the docs and test edits in this plan leave foreign dirty files.

## Steps

### 1. Add the Macro Invocation strand

Create `sase/memory/glossary/macro-invocation.md` with exactly this content. The slug
matches the `artifact-reference` and `project-tag` naming.

```markdown
---
keyword: Macro Invocation
aliases:
  - smack
  - macro reference
  - macro ref
---

A macro invocation (smack) is one `#name` or `#!name` use of a macro in prompt text,
with its arguments and modifiers: `#review(path=a)`, `#review:x`, `#review: text`,
`#flag+`, `#ns/name`, or a standalone `#!sync`. SASE expands or runs it before the agent
sees the prompt. The macro is the definition and the invocation is one use of it; a
prompt can hold zero or many. VCS workspace references such as `#gh:sase` and
`#git:home` are invocations of workspace workflows, and a project tag expands into one.
A `#` token that names no macro passes through as literal text, and one inside a literal
zone is not an invocation. Smack is short for sase macro invocation.
```

Constraints for this file:

- Do not add a `type:` key.
- Do not add plural aliases. The matcher derives plurals itself.
- Do not add a `macro call` alias. It would mislink "Jinja macro call".
- Do not add a `sase macro invocation` alias.
- The body must not contain "xprompt", because the terminology guard would trip.
- The body must not contain the bare word "ref". The matcher would mislink it to
  Artifact Reference.
- If `just fmt` rewraps the body, that is fine. Keep the wording.

### 2. Add the Raw Prompt strand

Create `sase/memory/glossary/raw-prompt.md` with exactly this content:

```markdown
---
keyword: Raw Prompt
---

The raw prompt is an agent's launch prompt after project tags and macro aliases are
canonicalized (`+sase` becomes a `#gh:` workspace reference and `#c` becomes `#commit`)
and before any macro invocation expands. It is saved as `raw_prompt.md`, shown on the
RAW PROMPT tab, and reused by restarts. It follows the submitted prompt
(`submitted_prompt.md`, the launch-boundary text before canonicalization) and precedes
the expanded prompt the model receives, from which directives are stripped. It is
unrelated to `sase prompt show -f raw` and to raw `<label>` placeholders.
```

The research verified each claim against the code:

- `src/sase/axe/run_agent_runner_setup_prompt.py` runs
  `canonicalize_project_aliases_in_prompt`, then `resolve_macro_aliases`, then writes
  `raw_prompt.md`, then calls `process_macro_references`.
- `write_submitted_prompt_artifact` in the same file persists the submitted prompt.
- `get_restartable_prompt_content` in `src/sase/ace/tui/models/artifact_files.py` reads
  `raw_prompt.md` for restarts.
- `PREVIEW_TAB_LABEL = "RAW PROMPT"` sets the tab label.
- `src/sase/default_config.yml` defines `macro_aliases` with `c: commit`.

If any of these claims has changed on current `master`, correct that sentence. Do not
drop the strand.

### 3. Point the Macro strand at the new term

In `sase/memory/glossary/macro.md`, append this sentence to the end of the body:

```text
Each use of a macro in a prompt is a macro invocation (smack).
```

The phrase creates the implicit link by itself, so do not add a `[[...]]` link.

Leave the alias line `- xprompt` and the first body line byte-identical. That first line
begins `Formerly called an xprompt. Triggered with`. Both lines are pinned in the
`tests/_macro_terminology_docs.py` allowlist. Do not reword the existing sentences. You
may append to the paragraph, or reflow only the lines after the pinned first line.

### 4. Add one docs sentence

In `docs/macros.md`, under `## Reference Syntax`, add one sentence after the opening
paragraph. That paragraph ends "...but new prompts should use `#name`."

```text
Each `#name` or `#!name` use is a *macro invocation*; the file or config entry it names is the macro.
```

Rules for this edit:

- Keep the section title and the `#reference-syntax` anchor.
- Do **not** mention "smack" in `docs/`.
- Do not sweep the existing "macro reference" prose. The `macro reference` alias already
  links that prose to the new term.

### 5. Regenerate memory outputs

Run `sase memory init --no-commit`. Expect it to regenerate the roster block in
`sase/memory/glossary.md` and the provider shims `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`,
`OPENCODE.md`, and `QWEN.md`. It may also regenerate `sase/memory/README.md`.

The roster sorts by keyword. After regeneration it should read
`... Macro (xprompt); Macro Invocation (smack, macro reference, macro ref); Macro Memory ...`
and `... Prompt Stash (stash); Raw Prompt; Receipt; ...`.

Then run `sase memory init --check`. It must report no drift.

### 6. Update the terminology-guard allowlist

`tests/_macro_terminology_docs.py` (`_MACRO_DOCS_ALLOWLIST`) pins the exact roster line
that contains `Macro (xprompt)` in **six** files:

- `sase/memory/glossary.md`
- `AGENTS.md`
- `CLAUDE.md`
- `GEMINI.md`
- `OPENCODE.md`
- `QWEN.md`

Inserting the new term rewraps that line. For each of the six entries, replace the old
string with the regenerated line, copied from the regenerated files:

- Old:
  `"Calls; Machine Tab; Macro (xprompt); Macro Memory (memory file, sase memory); Macro"`
- Predicted new:
  `"Calls; Machine Tab; Macro (xprompt); Macro Invocation (smack, macro reference, macro"`

Use the actual regenerated text if it differs from the prediction. Keep the guard just
as strict:

- Do not add wildcard or prefix matching.
- Do not drop entries.
- Leave `_MACRO_DOCS_REASONS` unchanged unless a file key changes. No file key should
  change.

If `just fmt` reflows the macro strand so that the pinned macro-strand line changes,
undo that reflow. Do not edit the allowlist for it.

### 7. Verify

1. Confirm that the new terms resolve and that their closures are right:

   ```bash
   sase memory read glossary:smack glossary:raw-prompt -r "Verify new Macro Invocation and Raw Prompt strands resolve"
   ```

   - `smack` must resolve to **Macro Invocation**.
   - The closure must include Macro, Project Tag, and Sase Workspace.
   - It must not include Artifact Reference as a direct dependency.

2. Check the Project Tag mislink fix with `sase memory show glossary:project-tag`. Today
   the Project Tag body's "VCS macro ref" links Artifact Reference through the bare word
   "ref". After the change, the level-two (`##`) dependency headings must include
   **Macro Invocation** and must not include **Artifact Reference**. Artifact Reference
   may still appear at a deeper, transitive level.

3. Run this matcher probe from the repo root with the project virtualenv's Python. It
   uses the same Rust matcher that drives TUI highlighting and LSP semantic tokens.

   ```python
   from pathlib import Path
   from sase.memory.web.catalog import find_memory_web, memory_web_glossary_entries
   from sase.core.glossary_facade import compile_glossary_catalog, scan_glossary_spans

   web = find_memory_web(Path.cwd(), "glossary")
   cat = compile_glossary_catalog(memory_web_glossary_entries(web))
   for text in [
       "expands to the project's VCS macro ref",
       "two smacks and an unresolved smack",
       "lip-smacking smacked smackdown",
       "a sase macro invocation",
       "Unknown macro reference(s)",
       "the raw prompt",
   ]:
       print(text, "->", [(s.matched_text, s.term) for s in scan_glossary_spans(cat, text)])
   ```

   Compare the output with these expectations:

   | Text                                     | Expected matches                                                   |
   | ---------------------------------------- | ------------------------------------------------------------------ |
   | "expands to the project's VCS macro ref" | `macro ref` → Macro Invocation, with no `ref` → Artifact Reference |
   | "two smacks and an unresolved smack"     | `smacks` and `smack` → Macro Invocation                            |
   | "lip-smacking smacked smackdown"         | nothing                                                            |
   | "a sase macro invocation"                | `macro invocation` → Macro Invocation                              |
   | "Unknown macro reference(s)"             | `macro reference` → Macro Invocation                               |
   | "the raw prompt"                         | `raw prompt` → Raw Prompt                                          |

   Before this change, the probe printed `ref` → Artifact Reference for the first line
   and nothing for the smack and raw-prompt lines.

4. Run `just fmt`.

5. Follow the `lint_and_test.md` reference memory, using the guarded
   `sase tool run check`. The terminology guard tests in
   `tests/test_macro_terminology.py` must pass, especially
   `test_macro_docs_and_memory_avoid_xprompt_terms` and
   `test_macro_docs_allowlist_is_classified`.

6. Expect no TUI PNG golden changes, so do not run screenshot updates. The prompt-pane
   visual tests use the deterministic glossary fixtures in
   `tests/ace/tui/visual/_ace_prompt_png_snapshot_glossary_fixtures.py`, not the live
   glossary.

### 8. Record the deferred follow-up

Use `/sase_new_task` to file one follow-up task bead for a **Directive** glossary
strand. `%` is the only prompt sigil whose instance would still lack a strand: `#` →
Macro Invocation, `@` → Artifact Reference, `+` → Project Tag. The research deliberately
left this out of the change. The skill's duplicate check decides whether a bead is
warranted. If it is, it also decides the task type and size.

## Explicitly out of scope

- Renaming code identifiers, including `MacroReference` and `MACRO_REFERENCE_PATTERN`.
- Renaming the files `raw_prompt.md` and `submitted_prompt.md`, the RAW PROMPT tab, the
  LSP legend, or the `Unknown macro reference(s)` warning text.
- A `sase smack` command, or any config key, JSON field, or CLI help text that contains
  "smack".
- A prose sweep of the docs or memory from "macro reference" to "macro invocation".
- Coining "smack prompt", "smack language", "smack catalog", or nicknames for the other
  sigils.
- Promoting `smack` to the keyword. Per the research's promotion test, that is a later
  one-line swap of `keyword:` and the alias, plus an allowlist update.
- Changes in sase-core or any linked repo. The glossary matcher is data-driven and needs
  no code change.

## Acceptance criteria

- `sase/memory/glossary/macro-invocation.md` and `sase/memory/glossary/raw-prompt.md`
  exist with the content above.
- `sase memory read glossary:smack` resolves to Macro Invocation.
- The regenerated roster in `sase/memory/glossary.md` and in every provider shim lists
  `Macro Invocation (smack, macro reference, macro ref)` and `Raw Prompt`.
- `sase memory init --check` reports no drift.
- `sase/memory/glossary/macro.md` ends with the macro invocation (smack) sentence, and
  its two allowlisted lines are unchanged.
- The `docs/macros.md` Reference Syntax section defines _macro invocation_ in one
  sentence and does not mention "smack".
- All six roster allowlist entries in `tests/_macro_terminology_docs.py` match the
  regenerated line, and the guard is no weaker than before.
- The matcher probe produces the expected matches. In particular, the Project Tag body's
  "macro ref" now links to Macro Invocation, not Artifact Reference.
- `sase tool run check` passes.
