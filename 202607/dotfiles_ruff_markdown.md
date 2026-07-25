---
tier: tale
title: Restore dotfiles CI after Ruff 0.16 Markdown discovery
goal: Keep Ruff scoped to Python sources, preserve Prettier as the Markdown formatter,
  and return the dotfiles CI checks to green.
create_time: 2026-07-25 06:46:13
status: wip
---

- **PROMPT:** [202607/prompts/dotfiles_ruff_markdown.md](prompts/dotfiles_ruff_markdown.md)

# Plan: Restore dotfiles CI after Ruff 0.16 Markdown discovery

## Evidence and root cause

- `actstat --repo bbugyi200/dotfiles -n 8 --format json` identifies six consecutive failing commits. Each failure is the
  `CI` workflow's `lint (ubuntu-latest)` job at step 15, `Run linters`; the test job is not the source of the reported
  failure.
- The latest failed job, run `30143922162` / job `89642158482`, reaches `just lint` and fails specifically at
  `.venv/bin/ruff format --check home/lib`. Ruff reports that `home/lib/CLAUDE.md` would be reformatted; the preceding
  Ruff lint check passes.
- The last green run (`30036813818`, commit `7dd805e`) installed Ruff `0.15.22`. The first red run (`30037880416`,
  commit `eb31d13`) installed Ruff `0.16.0` from the unpinned `ruff` entry in `requirements-dev.txt`. Those commits are
  only 15 minutes apart, and the intervening repository change touches only `home/dot_config/sase/sase_athena.yml`, not
  the formatter inputs or configuration.
- Ruff 0.16 discovers and formats Python fenced code blocks in Markdown when a directory is passed to `ruff format`. The
  broad `home/lib` argument therefore began including `home/lib/CLAUDE.md`. The Justfile separately defines Prettier as
  the Markdown formatter, while its Ruff recipes are explicitly described as Python formatting.

## Implementation

1. Add Ruff's documented Markdown opt-out, `extend-exclude = ["*.md"]`, to the existing `[tool.ruff]` configuration in
   `pyproject.toml`.
2. Do not pin or downgrade Ruff and do not rewrite `home/lib/CLAUDE.md`. The configuration should preserve current and
   future Ruff upgrades while making the intended formatter ownership explicit: Ruff handles Python files and Prettier
   handles Markdown files.
3. Keep the patch limited to the Ruff configuration unless validation reveals a directly related defect. Preserve any
   unrelated user changes if the linked checkout is no longer clean.

## Validation

1. Install or refresh the repository development dependencies so validation uses Ruff 0.16 or newer rather than the
   host's older Ruff.
2. Re-run the exact failing path with `just lint-py`; confirm `ruff format --check home/lib` no longer discovers
   `home/lib/CLAUDE.md`.
3. Run `just lint` to exercise the complete GitHub Actions lint command, including keep-sorted, Python, Lua, and
   Markdown checks.
4. Run `just check` for the repository's complete local format, lint, and test suite. Resolve and re-run any
   change-related failures until it passes; distinguish unavailable host-only tooling from source failures if the local
   environment cannot reproduce a CI dependency.
5. Inspect `git diff --check`, the focused diff, and `git status` to verify the final patch is minimal and contains no
   generated or environment artifacts.
6. Re-run `actstat --repo bbugyi200/dotfiles` after the fix has reached GitHub. If this implementation run does not
   include authorization to commit/push, report that remote CI revalidation is pending rather than mutating Git or
   GitHub implicitly.
