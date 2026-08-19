---
tier: tale
title: Define the Current Project glossary term
goal:
  The sase glossary defines Current Project, and every generated agent instruction file
  lists it.
size: small
proposed_by: bbugyi200.athena.079
create_time: 2026-08-18 20:28:11
status: wip
---

# Add a "Current Project" glossary term

## Goal

Add one new term, **Current Project**, to the sase project glossary (`memory.glossary`
in `sase/sase.yml`), then regenerate the derived agent instruction files. No other
behavior changes.

## Authorization

The user explicitly requested this glossary entry in the conversation that produced this
plan, and approving this plan re-confirms it. That request carries the permission
required by the `gotchas` memory rule for touching `AGENTS.md` / `CLAUDE.md` /
`GEMINI.md` / `OPENCODE.md` / `QWEN.md` and for running `sase memory init`. Do **not**
open a separate gate to ask again, and do **not** hand-edit any generated file:
`sase glossary add` performs the regeneration itself.

## Background (already researched; do not re-derive)

SASE gained a "current project" concept in epic `sase-pw` (closed 2026-08-18). The facts
the definition encodes, with their sources:

- `src/sase/current_project.py` — `resolve_current_project()` walks
  `~/.sase/vcs_xprompt_mru.json` head-first and returns the first entry that maps to an
  **enabled** project. Structural refs (containing `/`, starting with `~`, or equal to
  `home`) and disabled projects are skipped. An entry naming a Patch resolves to that
  Patch's owning project (`origin: "patch"`). It can return `None`. There is no pin
  file; the value is re-derived on every read.
- `src/sase/main/query_handler/_launch.py:_record_launched_vcs_xprompt_usage` —
  launching an agent on a VCS xprompt is what writes the MRU head.
- `sase project set-current` (`src/sase/main/parser_project.py`) and the ACE Projects
  tab (`set_current_project`, bound to `c`) perform the same MRU promotion without a
  launch.
- The store lives at `~/.sase/vcs_xprompt_mru.json`
  (`sase/history/vcs_xprompt_mru.py:vcs_xprompt_mru_path`), so it is one value per
  machine, shared by every workspace, agent, and ACE instance.
- The current project is **not** the cwd-inferred project. CLI `-p/--project` still
  infers from the current directory, and ACE's Artifacts scope uses cwd only as a
  fallback when the MRU resolves to nothing
  (`src/sase/ace/tui/actions/artifacts.py:_artifacts_current_project_key`).
- Consumers (`docs/ace.md` "Current project"): the top-bar `+<project>` chip, and — when
  `ace.current_project.seed_filters` is on — the first-open value of the Artifacts
  project scope, the Statistics filter, the Repos/Workspaces inventory filters, the
  Glossary project ring, and the `+` picker highlight. Seeds never override an explicit
  `project:` / `+name` term, a pick already made this session, or an already-open
  surface. Nothing outside display and those defaults reads it: it never changes project
  lifecycle state and never changes which project a command or launch targets.

Deliberately **excluded** from the definition to keep it short: the accent color scheme,
the `ace.current_project.{indicator,seed_filters,seed_agents_query}` config keys, the 5s
peek/`os.stat` change token, and the per-surface seed list. `docs/ace.md` and
`docs/configuration.md` already cover those; the glossary entry should not duplicate
them.

Two naming constraints that the wording below already satisfies:

- Do not describe it as the "active" project. `sase project` still carries suppressed
  `activate` / `deactivate` aliases for enable/disable, so "active" is exactly the
  collision this term must avoid.
- Follow the repo Patch/stitch terminology audit: `Patch` is capitalized.

## Term to add

Term: `Current Project`

Aliases: **none.** Every alias is reprinted in the tier-1 glossary list of every
generated instruction file, and "current project" already matches the term itself under
case-normalized phrase scanning, so a redundant alias would only cost tokens.

Definition (use verbatim):

```
The current project is the one sase project SASE treats as your working context: in practice, the one you most recently launched an agent on. It is derived, not stored — the first entry in the shared VCS xprompt MRU store (`~/.sase/vcs_xprompt_mru.json`) that maps to an enabled project, where a Patch entry yields its owning project. `sase project set-current` and the ACE Projects tab promote a project without launching; the working directory never does, and there may be none. It supplies only display and defaults, namely the ACE top-bar `+<project>` chip and the first-open value of project filters, so it never overrides an explicit choice, project lifecycle state, or what a command targets. `sase project current` prints it.
```

The text contains no apostrophes and no `$`, so it is safe to pass inside shell single
quotes exactly as written.

## Steps

1. `just install` (workspaces are ephemeral; dependencies may be stale).

2. Add the term with the CLI — do not hand-edit `sase/sase.yml`, because
   `sase glossary add` runs Rust validation and inserts in sorted order:

   ```bash
   sase glossary add 'Current Project' 'The current project is the one sase project SASE treats as your working context: in practice, the one you most recently launched an agent on. It is derived, not stored — the first entry in the shared VCS xprompt MRU store (`~/.sase/vcs_xprompt_mru.json`) that maps to an enabled project, where a Patch entry yields its owning project. `sase project set-current` and the ACE Projects tab promote a project without launching; the working directory never does, and there may be none. It supplies only display and defaults, namely the ACE top-bar `+<project>` chip and the first-open value of project filters, so it never overrides an explicit choice, project lifecycle state, or what a command targets. `sase project current` prints it.'
   ```

   Let the command regenerate the instruction files (no `-I/--no-init`). If it exits
   non-zero, read the printed diagnostic, fix the input, and retry; do not work around a
   validation failure by editing YAML directly.

3. Confirm the result:
   - `git diff --stat` should show `sase/sase.yml` plus the regenerated `AGENTS.md`,
     `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`, and possibly
     `sase/memory/README.md` — matching the file set of prior glossary commits such as
     `26c9f9a92`. Nothing else should change.
   - `Current Project` lands between `Artifact Reference` and `Feature Flag` in
     `sase/sase.yml` and in each instruction file's "Glossary Terms" list, with no alias
     parenthetical.
   - `sase glossary show 'Current Project'` renders the definition and a reference
     closure of `Sase Project` and `Patch` plus their existing transitive terms. No new
     term outside that closure should appear.
   - `sase glossary list` reports 30 terms (29 before this change).

4. Do **not** edit `CHANGELOG.md`; `tools/validate_changelog` requires it to stay
   release-please generated. Describe the change in the commit subject instead. Do not
   add or change docs — `docs/ace.md` already documents the feature.

5. Run `just check` and make sure it is green before reporting completion. If it is
   slow, hand it to `/sase_monitor` with a `--next` action.

## Non-goals

- Editing any `sase/memory/*.md` note.
- Changing the current-project implementation, config, docs, or tests.
- Adding aliases, or defining any additional glossary term.
