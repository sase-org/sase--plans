---
tier: tale
title: Add grok and muse providers to tmux_ai_window
goal:
  The tmux AI-agent launcher offers Grok Build and Muse Code alongside the five existing
  providers, launching each with sane max-effort, approval-bypassing flags, and a
  bashunit regression test pins the launch commands and menu wiring.
size: small
proposed_by: bbugyi200.athena.00z
create_time: 2026-08-14 09:30:06
status: wip
---

# Plan: Add grok and muse providers to tmux_ai_window

## Context

`tmux_ai_window` is a bash script in the **chezmoi** linked repo at
`home/bin/executable_tmux_ai_window` (deployed to `~/bin/tmux_ai_window`). It is bound
in `home/dot_config/tmux/tmux.conf` as `bind A run "tmux_ai_window"`.

With no arguments it renders a tmux `display-menu` of AI agent CLIs; choosing one
re-invokes the script as `tmux_ai_window --launch <provider> --dir <pane_dir>`, which
opens a new tmux window named `ai`/`ai2`/`ai3`/... running that CLI in the originating
pane's directory, plus a `wait-for` cleanup job that calls `tm-fix-layout` and
`tm-renumber-ai-windows` when the CLI exits.

Today it supports five providers: `claude`, `codex`, `agy`, `qwen`, `opencode`. Both
`grok` (xAI Grok Build) and `muse` (Meta Muse Code) are already installed on this
machine and already have SASE skills deployed (`home/dot_grok/skills/` and
`home/dot_config/muse/skills/`), so they are the two obvious gaps.

### Opening the repo

The implementing agent MUST open the chezmoi repo through the `/sase_repo` skill before
reading or writing any of these files:

```bash
sase repo open chezmoi -r "Add grok and muse providers to tmux_ai_window"
```

Every path below is relative to the path that command prints.

### Script structure

The script keeps one flat `PROVIDERS` array plus six parallel `case` statements
(`provider_is_known`, `provider_binary`, `provider_command`, `provider_menu_key`,
`provider_display_name`, `provider_description`, `provider_accent`). Adding a provider
means adding one arm to each. This is verbose but is the file's established idiom; **do
not refactor it to a table-driven design as part of this change** — keep the diff
mechanical and reviewable.

## Decisions

These were researched against the installed CLIs (`grok --help`, `grok models`,
`muse --help`) and against SASE's own provider definitions
(`src/sase/llm_provider/grok.py`, `src/sase/llm_provider/muse.py` in the sase repo).
Implement them as written; if a flag is rejected at runtime, fix it and say so rather
than silently dropping the flag.

### Launch commands

| provider | command                                                       |
| -------- | ------------------------------------------------------------- |
| `grok`   | `grok --effort xhigh --always-approve`                        |
| `muse`   | `muse --model muse-spark-1.2 --reasoning-effort ultra --yolo` |

Rationale:

- **Max effort, matching the existing entries.** Every current provider launches at its
  top reasoning tier (`claude --effort max`, `codex` with
  `model_reasoning_effort="xhigh"`). Grok's ceiling is `xhigh` (it has no `max`);
  `--effort` is the documented alias for `--reasoning-effort`. Muse's ceiling is `ultra`
  — this is the value SASE's canonical `max` maps onto for Muse.
- **Approval bypass, matching the existing entries.** `grok --always-approve` is the
  native "auto-approve all tool executions" flag; it is preferred over the Claude-compat
  `--permission-mode bypassPermissions` spelling. `muse --yolo` disables approval _and_
  the sandbox and trusts the workspace for the run — which matters because Muse's
  sandbox makes `.git` read-only and would break in-session commits.
- **Grok's model is deliberately not pinned.** `grok models` currently reports exactly
  one model (`grok-4.6`, the default), so pinning adds nothing and would rot the day xAI
  retires it. This matches `claude`, `codex`, and `opencode`, which also do not pin.
- **Muse's model IS deliberately pinned**, unlike Grok's. Muse ships
  `muse-spark-1.2-contributor`, which is ~95% cheaper _because Meta trains on its inputs
  and outputs_. Pinning `muse-spark-1.2` guarantees an interactive session over real
  source code can never land on the contributor model, even if Muse's default ever
  flips. Keep this pin and keep the comment explaining it.

### Menu keys, ordering, and colors

- Keys: `grok` → `g`, `muse` → `m`. Both are unused (`c`, `x`, `a`, `q`, `o` are taken).
- Append both to the end of `PROVIDERS`, after `opencode`, in the order `grok muse`.
  Order matters twice: it is the menu row order, and the first _installed_ provider
  becomes the menu's default `-C` choice. Appending preserves `claude` as the default.
- Accent colors must come from the tokyonight-storm palette the rest of the menu and
  `home/dot_config/tmux/theme.conf` use. Both unused-so-far picks:
  - `grok` → `#2ac3de` (tokyonight blue5 — the near-exact analog of SASE's own Grok
    accent `#00C8D7`)
  - `muse` → `#4a9dff` (Meta blue; the same value SASE's `provider_styles.py` uses for
    Muse's model style, and brighter/more legible on `#1f2335` than Meta's `#0064E0`)
  - If either reads too close to a neighbour in the live menu (`opencode` is `#7dcfff`,
    `agy` is `#7aa2f7`), say so in the completion notes rather than silently
    substituting a different palette.
- Descriptions (the dim right-hand column): `grok` → `xAI`, `muse` → `Meta`.
- The `%-10s` label padding already accommodates both four-character names; no format
  change is needed.

## Implementation Steps

### 1. `home/bin/executable_tmux_ai_window`

Nine edit sites, all in that one file:

1. **`DOC` heredoc, `--launch PROVIDER` section** — extend the supported-values list to
   `claude, codex, agy, qwen, opencode, grok, muse`. Note: this heredoc is expanded at
   script-load time, _before_ the `PROVIDERS` array is declared further down the file,
   so it cannot be generated from `PROVIDERS`. Update the literal text; do not try to
   templatize it and do not move the array declaration to make that possible.
2. **`PROVIDERS` array** —
   `readonly -a PROVIDERS=(claude codex agy qwen opencode grok muse)`.
3. **`show_provider_menu`'s "no CLI found" message** — currently the hardcoded
   `"No AI agent CLI found (claude/codex/agy/qwen/opencode)."`. Derive it from
   `PROVIDERS` instead so it can never drift again:

   ```bash
   local provider_list
   provider_list="$(IFS=/; echo "${PROVIDERS[*]}")"
   tmux_message "No AI agent CLI found (${provider_list})."
   ```

   Use a subshell (as written) so the `IFS` change does not leak into the rest of the
   function.

4. **`provider_is_known`** — add `grok` and `muse` to the matching pattern.
5. **`provider_binary`** — `grok` → `grok`, `muse` → `muse`.
6. **`provider_command`** — the two commands from the Decisions table above. Put a short
   comment above the `muse` arm recording _why_ the model is pinned (the
   contributor-model training hazard); that is the one non-obvious choice in the file.
7. **`provider_menu_key`** — `grok` → `g`, `muse` → `m`.
8. **`provider_display_name`** — `grok` → `grok`, `muse` → `muse`.
9. **`provider_description`** and **`provider_accent`** — `xAI`/`#2ac3de` and
   `Meta`/`#4a9dff`.

Keep each `case` arm in the same order as the `PROVIDERS` array so the file stays
scannable.

`home/dot_config/tmux/tmux.conf` needs **no** change — `bind A run "tmux_ai_window"` is
a single entry point with no per-provider bindings.

### 2. New test: `tests/bash/tmux_ai_window_test.sh`

The chezmoi repo runs bashunit over `tests/bash/` via `just test-bash`, and there is
currently no coverage for this script. Add one. Follow the stubbing pattern already
established in `tests/bash/bas_test.sh`: a `set_up` that builds a `mktemp -d` fake-bin
directory prepended to `PATH`, and a `tear_down` that removes it.

Stubs needed:

- **`tmux`** — a single stub that appends its full argument list to a log file and
  dispatches on `$1`: `display-message -p` prints a fixed fake pane dir, `list-windows`
  prints nothing (so the launcher picks the name `ai`), and `run-shell` / `new-window` /
  `display-menu` / `display-message` just record and exit 0.
- **provider binaries** — empty executables named `grok`, `muse`, and the five existing
  providers, created per-test so each test controls exactly which providers
  `provider_is_installed` sees.

Invoke the script as
`PATH="${FAKE_BIN}:${PATH}" bash "${PWD}/home/bin/executable_tmux_ai_window" ...`,
matching how `bas_test.sh` invokes its target.

Assertions to cover:

1. `--launch grok --dir <dir>` produces a `new-window` whose command contains
   `grok --effort xhigh --always-approve` and the quoted target dir.
2. `--launch muse --dir <dir>` produces a `new-window` whose command contains
   `muse --model muse-spark-1.2 --reasoning-effort ultra --yolo`.
3. The muse launch command does **not** contain `contributor` — this is the regression
   guard for the data-training hazard, so give the test an explicit name and a comment.
4. With every provider stubbed installed, the no-argument menu run passes `display-menu`
   args that include both `grok`/`xAI` and `muse`/`Meta` rows, bound to keys `g` and
   `m`.
5. With only `grok` installed, `grok` becomes the default `-C` choice (this pins the
   "first installed provider wins" behaviour that the array ordering relies on).
6. `--launch bogus --dir <dir>` exits `2`.
7. Every `provider_menu_key` across `PROVIDERS` is unique — a cheap guard that makes the
   _next_ provider addition fail loudly instead of shadowing a key.

Give the file the same banner-comment header style the other `tests/bash/*.sh` files
use.

## Verification

Run from the chezmoi repo root:

```bash
just test-bash          # fast, targeted: the new bashunit test
just check              # fmt-check + lint + full test suite, before landing
```

`just check` also runs the nvim/hammerspoon/python lanes, which this change does not
touch; if a pre-existing failure appears in one of those lanes, do not try to fix it
inside this change — confirm it reproduces on a clean tree and file it with
`/sase_new_task` instead.

Manual smoke test (cannot be automated, and is the only way to confirm the flags are
actually accepted by the real CLIs):

1. Commit, then run `chezmoi update -a --force` — **required** by the chezmoi repo's own
   `CLAUDE.md` after any commit, and the only way `~/bin/tmux_ai_window` picks up the
   change.
2. In tmux, press the `A` binding. Confirm seven rows render, that `grok` and `muse` are
   colored and legible, and that no two rows share a key hint.
3. Press `g`. Confirm a new `ai*` window opens in the right directory and Grok starts.
   Note that Grok reports `Project trusted: no` for un-trusted directories — if
   `--always-approve` does not also clear the startup workspace-trust prompt, record
   that in the completion notes; it is acceptable behaviour, not a bug to fix here.
4. Press `m` in a fresh menu. Confirm Muse starts, and that its header/status shows
   `muse-spark-1.2` (**not** `muse-spark-1.2-contributor`).
5. Exit each agent and confirm the window closes and the remaining `ai*` windows
   renumber, i.e. the `wait-for` cleanup path still works.

## Out of Scope

- **Refactoring the parallel `case` statements into a single provider table.** Real
  maintenance smell at seven providers, but a much larger and riskier diff than this
  request; keep it separate.
- **Guarding against the `grok` name collision.** SASE's own doctor warns that three
  unrelated tools compete for the `grok` executable name (xAI's Grok Build, `grok-dev`,
  and Homebrew's deprecated regex tool), and `provider_is_installed` only does a
  `command -v` presence check. On this machine `grok` resolves correctly to
  `~/.grok/bin/grok`, and the failure mode of a mis-resolved binary is a visibly broken
  tmux window rather than anything destructive — not worth a version-shape probe on
  every menu render.
- Any change to `tm-fix-layout`, `tm-renumber-ai-windows`, or the `ai*` window naming
  scheme.
