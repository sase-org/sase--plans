---
tier: tale
title: Remove Linux-only paths from the shared zsh startup configuration
goal:
  The chezmoi-managed zsh configuration no longer triggers macOS automount lookups,
  while pyenv and broot initialization remain portable and conditional.
size: small
proposed_by: bbugyi200.kellys_mbp.0q.f0
create_time: 2026-09-15 12:05:09
status: wip
---

# Remove Linux-only paths from the shared zsh startup configuration

## Objective

Fix the persistent macOS automount/OpenDirectory CPU drain caused by the shared,
chezmoi-managed `.zshrc` probing `/home/bryan` during command lookup and broot startup.
Keep the same pyenv and broot behavior on machines where those tools are installed,
apply the corrected file to Kelly's MacBook Pro, and refresh the long-lived processes
that inherited the stale `PATH`.

## Confirmed context

- The linked `chezmoi` repository owns the live file as `home/dot_zshrc`;
  `chezmoi source-path ~/.zshrc` reports the corresponding managed source and
  `chezmoi diff ~/.zshrc` is currently empty.
- `home/dot_zshrc` unconditionally prepends `/home/bryan/.pyenv/bin` to `PATH` and
  conditionally sources `/home/bryan/.config/broot/launcher/bash/br`.
- On Kelly's MacBook Pro, neither `pyenv`, `broot`, `~/.pyenv/bin`, nor the generated
  broot launcher currently exists. `/home` is a macOS automount point, so the first
  entry causes expensive failed lookups for every command resolution.
- The repo already defines `insert_path` and `source_if_exists` in `home/dot_profile`,
  which `home/dot_zprofile` sources before `.zshrc` runs. Use those helpers instead of
  introducing another path-management mechanism.
- `sase/sase.yml` already declares `commit_hooks.after: "chezmoi update -a --force"`.
  Preserve that hook; it is the post-commit application path required by the repo.

## Implementation

1. Update only the two cross-platform startup entries in `home/dot_zshrc`:
   - Replace the unconditional, hard-coded pyenv prepend with a readable zsh block that
     first checks whether `${HOME}/.pyenv/bin` is a directory, then prepends that
     directory through the existing `insert_path` helper. This keeps pyenv available on
     Linux installations under the user's actual home, avoids adding a nonexistent
     directory on this Mac, and avoids duplicate entries.
   - Replace the hard-coded broot launcher source with a portable path rooted at
     `${XDG_CONFIG_HOME:-${HOME}/.config}`. Preserve the `cmd_exists broot` condition
     and use `source_if_exists` so an installed broot binary without a generated
     launcher cannot make shell startup fail.
   - Do not change the other `/home/bryan` strings in the repository: they belong to
     Linux-only services, host-specific application state, examples, or unrelated
     commands and were not implicated in this machine's command-search path.

2. Add a focused bashunit regression test under `tests/bash/` for the shared zsh
   configuration. It must assert that `home/dot_zshrc` contains no `/home/bryan`
   reference, that the pyenv path is home-relative and guarded by directory existence,
   and that the broot launcher is home/XDG-relative and sourced only if it exists. Keep
   the test static and side-effect-free rather than sourcing the user's complete
   interactive zsh configuration.

3. Validate the source change before touching the live home directory:
   - Run `zsh -n home/dot_zshrc`.
   - Run the new regression test through the repo's `just test-bash` target.
   - Run the full `just check` repository gate. If a pre-existing, unrelated failure
     occurs, record it precisely and still demonstrate that the focused syntax and
     bashunit checks pass.
   - Inspect the rendered candidate with chezmoi and confirm the only intended live
     `.zshrc` diff is the pyenv/broot portability change.

4. Apply and activate the fix after validation:
   - Apply the candidate `home/dot_zshrc` from the opened chezmoi checkout to
     `~/.zshrc`; the existing post-commit hook will reapply the committed source with
     `chezmoi update -a --force` when the change lands.
   - Obtain a clean `PATH` from a fresh login zsh using the newly applied file. Confirm
     it has no `/home/bryan/.pyenv/bin` component, then update tmux's global `PATH`
     environment in place so existing sessions are preserved and future panes/status
     commands stop inheriting the stale entry.
   - Restart axe with that clean `PATH` via `sase axe restart`, then require a healthy
     `sase axe status`. Do not restart or destroy the tmux server.

## Acceptance checks

- `home/dot_zshrc` and the applied `~/.zshrc` contain no `/home/bryan` reference.
- A clean login zsh on Kelly's MacBook Pro has no `/home/bryan/.pyenv/bin` component and
  still resolves `sase` and `tmux`.
- tmux's global `PATH` and the restarted axe orchestrator's environment contain no
  `/home/bryan/.pyenv/bin` component.
- `zsh -n home/dot_zshrc`, the focused bashunit suite, and `just check` pass, subject
  only to a clearly documented unrelated pre-existing failure in the full gate.
- On a machine where `${HOME}/.pyenv/bin` exists, that directory remains prepended
  exactly once; broot initialization remains conditional on both the command and its
  generated launcher being present.
