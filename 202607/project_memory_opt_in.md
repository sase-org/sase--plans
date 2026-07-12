---
create_time: 2026-07-10 18:54:57
status: done
prompt: 202607/prompts/project_memory_opt_in.md
tier: tale
---
# Project-local opt-in for SASE-managed memory and agent instructions

## Goal

Prevent `sase memory init` (and bare `sase init`) from creating a project `memory/` tree or creating/regenerating the
project root `AGENTS.md` merely because the current directory is a version-controlled repository. Project-side memory
management must require an explicit opt-in in that project's own `./sase.yml`. Independently of that opt-in, every
existing `AGENTS.md` discovered in the project tree must continue to be copied byte-for-byte to the provider instruction
files beside it (`CLAUDE.md`, `GEMINI.md`, `QWEN.md`, and `OPENCODE.md`).

## Proposed configuration and behavior

- Add a project-local `memory.enabled` boolean, defaulting to `false`. Only a literal `memory.enabled: true` in the
  current project's own `./sase.yml` enables project memory ownership; merged user/global configuration must not opt all
  repositories in implicitly.
- Treat `amd_h1_title` as an optional title customization rather than the project-management switch. For an enabled
  project without a title, derive the existing stable `<project> - Agent Instructions` title so the new boolean is a
  complete opt-in by itself. Invalid explicitly configured values should still produce actionable blockers.
- Preserve home and chezmoi-home initialization semantics. The new switch is project-local and must not disable the
  existing home memory/configuration flow.
- Use this project behavior matrix:

  | Project state          | Project memory files                | Root `AGENTS.md`                 | Provider instruction files                                               |
  | ---------------------- | ----------------------------------- | -------------------------------- | ------------------------------------------------------------------------ |
  | `memory.enabled: true` | Generate/refresh and validate       | Generate/refresh managed content | Copy every discovered `AGENTS.md`                                        |
  | Missing or false       | Do not create, refresh, or validate | Do not create or alter           | Copy every existing discovered `AGENTS.md`; do nothing where none exists |

This intentionally removes the current onboarding fallback that infers permission to overwrite `AGENTS.md` from the
presence of memory notes. Existing projects that want managed memory must add the new explicit local field.

## Implementation plan

1. Add a focused project-local configuration reader for `memory.enabled` alongside the memory-init configuration
   helpers. Validate that the local `sase.yml` is a mapping, that `memory` is a mapping when present, and that `enabled`
   is boolean. Keep this read separate from merged SASE config so default/global values cannot authorize project writes.
   Feed the resolved opt-in and any validation error into the shared plan/apply inputs so both paths make identical
   decisions.

2. Separate project memory ownership from agent-document propagation in the root planner/application layer. In managed
   mode, retain generation of `memory/sase.md`, the memory README/map asset, long-note metadata updates, managed
   `AGENTS.md`, reachability checks, and normal provider copies. In unmanaged mode, omit all expected memory/root-agent
   files and skip project-memory reachability validation, but scan the project with the existing pruning rules and plan
   provider copies for every readable `AGENTS.md`, including the project root and nested directories. If no `AGENTS.md`
   exists in a directory, do not create one and do not touch standalone provider files there.

3. Make the explicit opt-in the only project path into AMD-managed rendering. Remove the memory-presence/onboarding
   fallback and simplify the now-obsolete onboarding plumbing and comments so neither `sase memory init` nor bare
   `sase init` can infer write permission. When enabled, continue using `amd_h1_title` when supplied and the stable
   project-name fallback otherwise. Leave user-config title resolution for home/chezmoi roots intact.

4. Update top-level configuration surfaces: add `memory.enabled: false` to the default config and its boolean schema,
   revise `amd_h1_title` schema wording, and set `memory.enabled: true` in this repository's project-local `sase.yml` so
   SASE itself remains an explicitly managed project. Document the local-only authorization rule, the title fallback,
   the unmanaged behavior, the provider-copy exception, and the migration required for projects that previously relied
   on `amd_h1_title` or inferred onboarding.

5. Expand and adjust regression coverage across config, planning, apply, and onboarding:
   - absent/false/global-only opt-ins do not create project memory or `AGENTS.md`, do not rewrite a custom root file,
     and do not report unmanaged project memory as unreachable;
   - malformed local values block without writes;
   - `memory.enabled: true` performs the full managed flow with both explicit and derived titles and remains idempotent;
   - an unmanaged existing root or nested `AGENTS.md` is preserved and copied exactly to every provider file;
   - provider files are untouched when their directory has no `AGENTS.md`;
   - home and chezmoi-home behavior remains unchanged; and
   - plan/check/apply behavior stays consistent, including bare `sase init`.

6. Run `just install` followed by focused memory/config/onboarding tests during development, then run the required
   `just check` before completion. Review the resulting diff for stale references to `amd_h1_title` as an opt-in or to
   inferred onboarding management.

## Non-goals

- Do not change provider file names, byte-for-byte copy semantics, project-tree pruning, home/chezmoi deployment, or the
  project commit/push workflow.
- Do not make project lifecycle/registry state an implicit authorization source; the checked-in project-local field is
  the explicit source of truth.
