---
tier: tale
title: Collapse the sase.md repository reminders into one IMPORTANT paragraph
goal:
  The generated sase/memory/sase.md note, AGENTS.md, and every provider shim state the
  sidecar-artifact and cross-repo rules as a single bolded **IMPORTANT** paragraph
  instead of an IMPORTANT REMINDERS bullet list, with the generator template as the only
  edited source and no committed drift.
size: small
proposed_by: bbugyi200.athena.0g6
---

- **AGENTS:**
  - [bbugyi200.athena.0g6](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0g6.md)
  - [bbugyi200.athena.sase-vs.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-vs.1/README.md)
  - [bbugyi200.athena.sase-vs.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-vs.2/README.md)
  - [bbugyi200.athena.sase-vs.3](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-vs.3/README.md)
  - [bbugyi200.athena.sase-vs.4](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-vs.4/README.md)
  - [bbugyi200.athena.sase-vs.5](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-vs.5/README.md)
  - [bbugyi200.athena.sase-vs.6](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-vs.6/README.md)
  - [bbugyi200.athena.toobig-4l.read_log.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-4l.read_log.0/README.md)
- **COMMITS:**
  - [0e47ef6](https://github.com/sase-org/sase/commit/0e47ef6482937cf35ae29529fcb69ba5b840765a)
    — docs(memory): collapse repository reminders into one IMPORTANT paragraph
  - [da1da7a](https://github.com/sase-org/sase/commit/da1da7aea46802e2f58e9fda6fac6c56788798e2)
    — refactor(memory): split read log module
  - [6e0e586](https://github.com/sase-org/sase/commit/6e0e5860b0bcf4e1b08a50e68a72c32c62e1c5bd)
    — feat(plan-approval): stamp approval waits onto the tale coder successor prompt
  - [9c5cbea](https://github.com/sase-org/sase/commit/9c5cbeac56ea753c88550e8095016f2c3a5a153b)
    — feat(bead): add wait-spec parser and sase bead work --wait
  - [2bf5164](https://github.com/sase-org/sase/commit/2bf51641d2aa1952c359f787b5f075e8dbe9b47e)
    — feat(bead): thread wait spec through the host-owned epic launch
  - [15be5ac](https://github.com/sase-org/sase/commit/15be5ac470cafd2f31ba03b511ae11b959c951d6)
    — feat(plan): accept wait on tale and epic approval options
  - [c507cea](https://github.com/sase-org/sase/commit/c507ceab9b2334268aeefda9a6272c838e33d677)
    — feat(plan): add approval wait CLI
  - [18fa499](https://github.com/sase-org/sase/commit/18fa499a3af9c4d941123f51aa0827c6ab0a68d6)
    — feat(ace): add approval wait editor

# Collapse The `sase.md` Repository Reminders Into One `**IMPORTANT**` Paragraph

## Goal

Replace the two-bullet `IMPORTANT REMINDERS:` list at the end of the `## Repositories`
section of the generated core memory note `sase/memory/sase.md` with a single bolded
paragraph, and republish every generated file that inlines it.

The exact replacement text the user asked for is pinned in "Target Text" below. Do not
paraphrase it, reorder its clauses, or reflow it.

## Background

- `sase/memory/sase.md` is a _generated_ memory note and refuses direct edits. Its only
  source of truth is the packaged Jinja template
  `src/sase/main/init_memory/templates/memory-sase.template.md`, rendered by
  `_render_sase_memory()` in `src/sase/main/init_memory/root_rendering_notes.py`.
- No template override is configured anywhere: `memory.sase_template` is `null` in
  `src/sase/default_config.yml`, the project's `sase/sase.yml` sets neither
  `memory.sase_template` nor the legacy `memory_sase_template`, and no
  `memory-sase.template.md` exists in the SASE user config directory. The packaged
  template is the single source.
- Rendered output passes through `format_generated_memory_markdown()`
  (`src/sase/main/init_memory/formatting.py`), which re-wraps prose to
  `markdown_print_width()` (88). Raw wrapping inside the template does not survive, so
  the template body may be written on any line lengths.
- `src/sase/main/init_memory/templates/` is listed in `.prettierignore`, so the template
  file itself is not prose-wrapped by `just fmt`.
- The note is inlined into the managed `AGENTS.md` and its byte-identical provider shims
  (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`) at the repo root. Those files are
  generated — never hand-edit them.
- `tests/main/test_init_memory_committed_drift.py::test_repo_project_memory_notes_match_generator_output`
  fails if the committed project tree does not match generator output, so regeneration
  is mandatory, not optional.

## Target Text

The block to remove from the template (currently the last thing in the `## Repositories`
section, immediately before the `## SASE Final Declaration` heading):

```markdown
IMPORTANT REMINDERS:

- Do NOT locate, clone, or web-fetch another repo's contents any other way than by using
  `/sase_repo` or `sase artifact read`!
- The `sase artifact read <ref> "<reason>"` command MUST be used to read artifacts (so
  the reads are audited) from sidecar repos. Do NOT read sidecar artifact files
  directly.
```

The block that replaces it:

```markdown
**IMPORTANT**: The `sase artifact read <ref> "<reason>"` command MUST be used to read
artifacts (so the reads are audited) from sidecar repos. Do NOT read sidecar artifact
files directly or locate, clone, or web-fetch another repo's contents any other way than
by using `/sase_repo` or `sase artifact read`!
```

That replacement is already an exact fixpoint of the 88-column memory formatter, so the
four lines above are byte-for-byte what `sase/memory/sase.md`, `AGENTS.md`, and every
provider shim must contain when you are done. It is a plain paragraph, not a standalone
strong label (`_STANDALONE_STRONG_LABEL_RE` only matches a line that is _entirely_
bold), so the formatter wraps it as prose with no trailing hard break.

## Steps

1. **Edit the template.** In
   `src/sase/main/init_memory/templates/memory-sase.template.md`, replace the
   `IMPORTANT REMINDERS:` block with the replacement paragraph from "Target Text".
   Change nothing else in the file — the preceding `## Repositories` prose, the Jinja
   `{% if %}` blocks, and the surrounding blank lines all stay as they are.

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
     packaged template and would regenerate the _old_ text, silently reverting step 1.
   - `--no-commit` is required: agents never create commits (see
     `sase memory read decisions:host-owned-completion`). The host's finalizers own
     landing this change.
   - Expect this to rewrite `sase/memory/sase.md`, `AGENTS.md`, `CLAUDE.md`,
     `GEMINI.md`, `OPENCODE.md`, and `QWEN.md`.
   - Side effect to expect and report: on a machine configured with `use_chezmoi: true`,
     `sase memory init` also regenerates the _home_-root copy of this note into the
     chezmoi source repo, auto-commits it there, and runs `chezmoi apply`. `--no-commit`
     does not suppress that — it only skips the project git sequence. This is
     `sase memory init`'s normal republish path and is expected here, since the home
     root renders from the same packaged template. Do not try to work around it, do not
     `cd` into the chezmoi checkout, and if you need to inspect it, open it with your
     `/sase_repo` skill first and use only the path that prints. Name the chezmoi
     outcome explicitly in your final summary.

4. **Update `tests/main/test_init_memory_handler_repositories.py`.** Inside
   `test_init_memory_uses_local_linked_repos_for_project_and_global_for_home`, the
   `for memory in (project_memory, home_memory):` loop currently ends with three
   assertions pinning the old block: `assert ("IMPORTANT REMINDERS:") in memory_line`
   and one assertion per old bullet. Replace all three with a single assertion on the
   new paragraph, expressed against the whitespace-collapsed `memory_line` produced by
   the `single_line` helper:

   ```python
   assert (
       '**IMPORTANT**: The `sase artifact read <ref> "<reason>"` command MUST be '
       "used to read artifacts (so the reads are audited) from sidecar repos. Do "
       "NOT read sidecar artifact files directly or locate, clone, or web-fetch "
       "another repo's contents any other way than by using `/sase_repo` or "
       "`sase artifact read`!"
   ) in memory_line
   ```

   Also add `assert "IMPORTANT REMINDERS:" not in memory` alongside the existing
   `assert 'sase repo open <linked_repo> -r "<reason>"' not in memory` negative check,
   so the old list heading cannot creep back in. Leave every other assertion in that
   test untouched.

5. **Update `tests/main/test_init_memory_chezmoi.py`.** In
   `test_init_memory_uses_chezmoi_home_and_global_config_source`, the assertion
   `assert "Do NOT locate, clone, or web-fetch another repo's contents" in _single_line(chezmoi_memory)`
   pins a substring that the new wording no longer contains verbatim (the new sentence
   reads "... or locate, clone, or web-fetch another repo's contents ..."). Repoint it
   at a substring the new paragraph actually contains, for example:

   ```python
   assert (
       "locate, clone, or web-fetch another repo's contents any other way than "
       "by using `/sase_repo` or `sase artifact read`!"
   ) in _single_line(chezmoi_memory)
   ```

   Keep this test's remaining assertions as they are.

6. **Confirm zero drift**, which is the direct check that steps 1 and 3 agree with the
   committed tree:

   ```bash
   .venv/bin/sase memory init --check
   ```

   It must exit 0 and report no pending memory changes.

7. **Verify.** Run `just check` (whole-repo lint gates plus the diff-scoped test lane).
   If its scoped selection does not pick up
   `tests/main/test_init_memory_committed_drift.py`,
   `tests/main/test_init_memory_handler_repositories.py`, and
   `tests/main/test_init_memory_chezmoi.py`, run those three files explicitly as well.
   Hand the run to your `/sase_monitor` skill if it starts to outrun the turn.

## Non-Goals

- Do not hand-edit `sase/memory/sase.md`, `AGENTS.md`, or any provider shim. They are
  generated; step 3 is the only way they change.
- Do not touch any other memory note, and do not reword any other paragraph of the
  `## Repositories` section — the `/sase_repo` trigger sentence and the "applies
  regardless of transport" paragraph both stay exactly as they are.
- Do not change `src/sase/xprompts/skills/sase_repo.md` or `docs/artifact_links.md`.
  They discuss `sase artifact read` and `/sase_repo` in their own words and do not quote
  this block; they were checked and need no update.
- Do not add a config knob, a template override, or a compatibility shim for the old
  wording. This is a straight text replacement in one generated note.

## Acceptance Criteria

- `src/sase/main/init_memory/templates/memory-sase.template.md` contains the new
  `**IMPORTANT**:` paragraph and no `IMPORTANT REMINDERS:` list.
- `sase/memory/sase.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, and
  `QWEN.md` each contain the four target lines verbatim, and `AGENTS.md` is still
  byte-identical to each provider shim.
- `grep -rn "IMPORTANT REMINDERS" .` returns nothing in tracked files.
- `.venv/bin/sase memory init --check` exits 0 with no pending changes.
- `just check` passes, including the three test files named in step 7.
