---
tier: tale
goal: 'memory/glossary.md accurately documents that ProjectSpec/ChangeSpec paths use
  the project directory key (which differs from the user-facing project name for #gh
  projects), with no growth in always-loaded context size.

  '
create_time: 2026-07-16 07:50:41
status: done
prompt: 202607/prompts/glossary_project_key.md
---

# Plan: Clarify project directory key vs project name in the glossary memory

## Context

`memory/glossary.md` (Tier 1, always loaded) says a project's ProjectSpec is `~/.sase/projects/<name>/<name>.sase`. The
path segment is actually the project **directory key**, which equals the user-facing name only for `#git:<name>`
projects. `#gh:<org>/<repo>` projects use `gh_<org>__<repo>` — ex: the `sase` project's spec is
`~/.sase/projects/gh_sase-org__sase/gh_sase-org__sase.sase`, with `PROJECT_NAME: sase` inside. The ChangeSpec entry has
the same `<project>` placeholder problem and additionally says specs are stored in `.gp` files: the canonical extension
today is `.sase` (`.gp` is a legacy fallback only — see `src/sase/workflows/utils.py:14`), and active ChangeSpecs live
inside the ProjectSpec file itself.

Verified facts to rely on:

- "Directory key" is the established codebase term (`src/sase/project_display_names.py:41`,
  `src/sase/workflows/utils.py:44`).
- On disk under `~/.sase/projects/`: `chezmoi/chezmoi.sase` has no `PROJECT_NAME:` (the user-facing name falls back to
  the key), while `gh_sase-org__sase/…` and `gh_bbugyi200__actstat/…` set `PROJECT_NAME:` (`sase`, `actstat`).
- Active ChangeSpecs live in `<key>.sase`; terminal ones in `<key>-archive.sase`. No `.gp` files remain on disk.

## Authorization

The user explicitly requested this glossary clarification in the conversation that produced this plan, and their
approval of this proposal is the permission grant for editing `memory/glossary.md`. Touch no other memory file,
`AGENTS.md`, or generated provider shims (`CLAUDE.md`, `GEMINI.md`, etc.) — the post-commit `sase init` hook regenerates
shims.

## Changes (memory/glossary.md only)

1. **Projects, Repos, and Workspaces** entry — replace the sentence
   `Its ProjectSpec is ~/.sase/projects/<name>/<name>.sase.` with:

   > Its ProjectSpec is `~/.sase/projects/<key>/<key>.sase`, where the directory key `<key>` is `<name>` for `#git`
   > projects but `gh_<org>__<repo>` for `#gh` projects (ex: `gh_sase-org__sase`); the user-facing name is the spec's
   > `PROJECT_NAME:` (ex: `sase`) or, if unset, the key.

2. **ChangeSpec** entry — replace the two storage sentences (`Stored in .gp files at ~/.sase/projects/<project>/.` and
   `Active specs in <project>.gp; terminal ones (Submitted, Archived, Reverted) in <project>-archive.gp.`) with one:

   > Active specs live in the owning project's ProjectSpec file (`<key>.sase`; directory key `<key>` — see Projects,
   > Repos, and Workspaces); terminal ones (Submitted, Archived, Reverted) in `<key>-archive.sase`.

   Keep the Sections list and Status lifecycle sentences unchanged.

Net size must stay roughly flat (±2 lines): this glossary is loaded into every agent's context, so wording must remain
as tight as the original.

## Verification

- Re-read both edited entries against the verified facts above; confirm no other glossary entry still references `.gp`
  or `~/.sase/projects/<name>/`.
- Run `just install` then `just check` before finishing (memory edits are not in the check-exemption list).
