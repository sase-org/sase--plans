---
tier: tale
title: Fix the macOS bash 3.2 crash in tmux_ai_window
goal:
  The tmux AI-agent menu builds its installed-provider list without any bash 4-only
  builtin, so `tmux_ai_window` renders correctly on macOS's bash 3.2 as well as on
  Linux's bash 5, and a static bashunit guard fails loudly if bash 4-only syntax is
  reintroduced.
size: small
proposed_by: bbugyi200.athena.00z.f0.f0.f0
create_time: 2026-08-14 10:28:03
status: wip
---

# Plan: Fix the macOS bash 3.2 crash in tmux_ai_window

## Context

`tmux_ai_window` is a bash script in the **chezmoi** linked repo at
`home/bin/executable_tmux_ai_window` (deployed to `~/bin/tmux_ai_window`), bound in
`home/dot_config/tmux/tmux.conf` as `bind A run "tmux_ai_window"`. Its bashunit
regression tests live in `tests/bash/tmux_ai_window_test.sh`.

chezmoi deploys `home/bin/` to every host. `home/.chezmoiignore` only excludes
host-specific `sase_*.yml` config files, so the launcher runs unchanged on the Linux
hosts and on the MacBook.

On the MacBook it fails:

```
❯ tmux_ai_window
/Users/bbugyi/bin/tmux_ai_window: line 48: mapfile: command not found
```

### Opening the repo

The implementing agent MUST open the chezmoi repo through the `/sase_repo` skill before
reading or writing any of these files:

```bash
sase repo open chezmoi -r "Fix the macOS bash 3.2 mapfile crash in tmux_ai_window"
```

Every path below is relative to the path that command prints.

## Root Cause

Line 48 is `mapfile -t sorted_providers < <(providers_sorted_by_menu_key)` inside
`show_provider_menu`.

`mapfile` (and its synonym `readarray`) is a **bash 4.0 builtin**. macOS ships **bash
3.2.57** as `/bin/bash` and always will — bash 4.0+ is GPLv3, which Apple does not ship.
The script's shebang is `#!/bin/bash`, an absolute path, so a Homebrew bash 5 in `$PATH`
is irrelevant: the kernel runs `/bin/bash` regardless.

This was verified directly rather than assumed. Running the current script under the
`bash:3.2` container (bash 3.2.57, the exact macOS version) reproduces the user's error
verbatim:

```
/repo/home/bin/executable_tmux_ai_window: line 48: mapfile: command not found
```

### The failure is worse than the error line suggests

The script sets no `set -e`, so execution continues after the failed builtin.
`sorted_providers` stays empty, so the filter loop that follows produces an empty
`menu_providers`, so the zero-provider branch fires and the user gets a **second, false
message**:

```
No AI agent CLI found (claude/codex/agy/qwen/opencode/grok/muse).
```

That message is a lie on a machine where Claude is installed. Expect the user to have
seen it too, and confirm it disappears after the fix.

### Regression origin

`git log -S mapfile` pins the introduction to `7a709bda`
(`feat(tmux_ai_window): sort AI-agent menu rows by shortcut key`). The two commits since
then kept the call. This is a recent regression in a previously portable script, not a
long-standing gap.

### What is NOT the problem — do not chase these

Each of these was checked; re-investigating them wastes the implementation turn.

1. **`getopt` is fine on that Mac.** `parse_cli_args` (line 159) calls
   `getopt -o "hv" -l "help,verbose,launch:,dir:"`. BSD `getopt` has no `-l`, so on a
   stock macOS this would `die -x 2 "Invalid command-line arguments."` long before
   line 48. The error message proves execution reached line 48, so GNU `getopt` is
   already on that machine's `PATH`. Do **not** add a getopt shim or a BSD fallback.

2. **`~/lib/bugyi.sh` is fine.** It is installed by `basher`, not by chezmoi, and greps
   clean of bash 4-only builtins (`mapfile`, `readarray`, `declare -A`, `declare -n`,
   `${var^^}`, `${var@Q}`, `;;&`, `wait -n`). Independently, everything before line 48 —
   sourcing the library, `usage()`, and the `read -r -d '' DOC` heredoc — demonstrably
   ran on the Mac. It is out of scope and out of this repo.

3. **Nothing else in the script is bash 4-only.** Every remaining non-trivial construct
   was executed side by side under bash 3.2.57 and bash 5.2.37 with byte-identical
   output: `tmux_quote`'s `${value//...}` substitutions, `printf "%q"`, the `%-10s`
   menu-label format, `command -v ... &>/dev/null`, the `<<<` herestring plus `grep`
   `ai`-window numbering with `((n++))`, `export USAGE_GRAMMAR=(...)`, and
   `$(IFS=/; echo "${PROVIDERS[*]}")`. Array append `+=` is bash 3.1. Process
   substitution is bash 3.x.

4. **This is not a fleet-wide port.** `mapfile`, `readarray`, `declare -A`, `local -A`
   and `;;&` appear in exactly one file across all ~101 scripts in `home/bin/`:
   `executable_tmux_ai_window`. Fixing this one call site restores macOS.

## Decisions

### Replace `mapfile` with a `while read` loop, and merge the two loops

The current shape reads the sorted list into one array and then filters it into a
second:

```bash
  local -a sorted_providers=()
  mapfile -t sorted_providers < <(providers_sorted_by_menu_key)

  local -a menu_providers=()
  local provider
  for provider in "${sorted_providers[@]}"; do
    if provider_is_installed "${provider}"; then
      menu_providers+=("${provider}")
    fi
  done
```

Collapse both into one portable loop that filters as it reads. This removes the bash
4-only builtin _and_ the now-pointless intermediate array:

```bash
  # Collect the installed providers in menu-key order. macOS ships bash 3.2,
  # which has no mapfile/readarray builtin, so read the sorted list with a
  # while-read loop instead.
  local -a menu_providers=()
  local provider
  while IFS= read -r provider; do
    if provider_is_installed "${provider}"; then
      menu_providers+=("${provider}")
    fi
  done < <(providers_sorted_by_menu_key)
```

Notes on why this is safe:

- `IFS=` and `-r` preserve each line verbatim, matching `mapfile -t` semantics for the
  single-token provider names this emits.
- Process substitution (not a pipe) keeps the loop body in the current shell, so
  `menu_providers+=` survives the loop. A pipe would silently produce an empty array.
- The loop body must never read stdin, or it will swallow the provider list.
  `provider_is_installed` runs only `command -v`, which does not. Keep it that way.

This exact patched file was run end-to-end under bash 3.2.57 with stubbed `tmux` and
provider binaries: it renders the `agy`/`claude`/`muse` rows in key order with `-C 1`,
identical to bash 5.

### Do not change the shebang to `#!/usr/bin/env bash`

Rejected deliberately:

- It only helps if Homebrew bash 5 is installed on that Mac, which is an unstated new
  dependency for a problem one loop rewrite removes.
- `bind A run "tmux_ai_window"` executes through the **tmux server's** environment. On
  macOS that server is frequently started from a launchd-derived `PATH` that lacks
  `/opt/homebrew/bin`, so `env bash` could still resolve to `/bin/bash` — a fix that
  works from an interactive shell and silently fails from the keybinding is worse than
  no fix.
- 99 of the 101 scripts in `home/bin/` use `#!/bin/bash`. Diverging here for one script
  makes the convention noise, not signal.

If the user later wants bash 5 features repo-wide, that is a separate, deliberate
decision about a Homebrew bash dependency — not something to smuggle in under a bug fix.

### Guard the regression statically, not behaviorally

The bashunit harness invokes the script as `bash "${TMUX_AI_WINDOW_SCRIPT}"` with
`SYSTEM_PATH="/usr/bin:/bin"`, so tests run under whatever bash the host has — bash 5 on
the Linux hosts and in CI. **No behavioral test can ever catch this class of bug
there**, which is exactly why the full existing suite passed while the Mac was broken.
The repo also has no bash linter at all (`just lint` covers keep-sorted, Python, Lua,
and Markdown only), so nothing else is watching.

Add a static guard that greps the production script for bash 4-only constructs. It is
cheap, deterministic, runs everywhere, and would have caught `7a709bda` on the day it
landed.

Do **not** add a Docker-based bash 3.2 lane to `just test-bash`. That would make the
repo's default test target require Docker — including on the MacBook this change exists
to support.

## Implementation

### 1. `home/bin/executable_tmux_ai_window`

Replace lines 47-56 (the `local -a sorted_providers=()` declaration through the `done`
of the filter loop) with the merged `while read` loop given in Decisions above, comment
included.

Nothing else in the file changes: the shebang, `providers_sorted_by_menu_key`, the
zero-provider branch, the `default_choice` scan, the `menu_args` construction, the
render loop, `launch_provider`, `parse_cli_args`, and all seven parallel `case`
statements stay exactly as they are. `home/dot_config/tmux/tmux.conf` is untouched.

### 2. `tests/bash/tmux_ai_window_test.sh`

Add one test. Place it near `test_provider_menu_keys_are_unique`, which is the file's
other structural guard.

```bash
# tmux_ai_window is deployed to macOS too, where the `#!/bin/bash` shebang
# resolves to Apple's bash 3.2.57 (macOS cannot ship bash 4+, which is GPLv3).
# Bash 4-only builtins therefore fail at runtime there: a `mapfile` call once
# broke the menu with `line 48: mapfile: command not found` and left the user
# with a false "No AI agent CLI found" message. Every other test in this file
# runs under the host's bash (5.x on Linux and in CI) and so cannot catch that
# class of regression -- this static check is the only thing that can.
function test_script_avoids_bash_4_only_features() {
  local bash_4_only='(mapfile|readarray)'
  bash_4_only+='|(declare|local|typeset|readonly) +-[A-Za-z]*(A|n)([^A-Za-z]|$)'
  bash_4_only+='|\$\{[A-Za-z_][A-Za-z_0-9]*(\[[^]]*\])?(\^\^?|,,?|@[A-Za-z])'
  bash_4_only+='|;;&|\|&|&>>|wait +-n|globstar'

  # Comment lines are excluded so the script can name the builtins it avoids.
  assert_same "" \
    "$(grep -nE "${bash_4_only}" "${TMUX_AI_WINDOW_SCRIPT}" |
      grep -vE '^[0-9]+:[[:space:]]*#')"
}
```

The pattern covers `mapfile`/`readarray` (4.0), associative arrays and namerefs
(`declare -A`, `local -n`, 4.0/4.3), case-modification and `@`-transform expansions
(`${v^^}`, `${v,,}`, `${v@Q}`, 4.0/4.4), `;;&` fallthrough (4.0), `|&` (4.0), `&>>`
(4.0), `wait -n` (4.3), and `globstar` (4.0).

It was validated before being written into this plan:

- Against the **current** script it reports exactly one line — line 48, the `mapfile`
  call — and nothing else across all 437 lines. No false positives on `local -a`,
  `readonly -a`, `${PROVIDERS[*]}`, `${#menu_providers[@]}`, `${value//\\/\\\\}`,
  `&>/dev/null`, or `tmux wait-for`.
- Against the **patched** script it reports nothing.
- Against a synthetic file containing all ten constructs it matches all ten.

Two constraints worth preserving if the implementer reshapes it:

- **No `\b` word boundaries.** They are a GNU extension; BSD `grep` on macOS is not
  reliably compatible, and `just test-bash` may well be run on the MacBook. The
  `([^A-Za-z]|$)` tail does the same job portably.
- **Keep the comment filter.** Without it, the explanatory comment added to the script
  in step 1 — which names `mapfile` — would trip the guard.

Leave every existing test unchanged. The behavior of `show_provider_menu` is not
changing on bash 5, so all thirteen must still pass untouched; if any of them needs
editing, the implementation drifted and should be reworked rather than the test relaxed.

## Verification

From the opened chezmoi checkout:

1. Targeted suite — expect the existing thirteen tests plus the new guard, all passing:

   ```bash
   just test-bash
   ```

2. Repository-wide gate:

   ```bash
   just check
   ```

3. Confirm the diff touches only `home/bin/executable_tmux_ai_window` and
   `tests/bash/tmux_ai_window_test.sh`.

4. **Optional but recommended** — prove it on real bash 3.2 rather than trusting the
   version tables. Docker is available on athena and the `bash:3.2` image is 3.2.57, the
   exact macOS build. This recipe was used to validate the fix and works as written:

   ```bash
   cat > /tmp/e2e32.sh <<'EOF'
   mkdir -p /fake
   printf '#!/bin/sh\ncase "$1" in\ndisplay-message) [ "$2" = "-p" ] && echo "/tmp/panedir"; exit 0 ;;\ndisplay-menu) printf "MENU:%%s\\n" "$@" ;;\nesac\nexit 0\n' >/fake/tmux
   chmod +x /fake/tmux
   for p in claude agy muse; do printf '#!/bin/sh\nexit 0\n' >"/fake/$p"; chmod +x "/fake/$p"; done
   export PATH="/fake:$PATH"
   mkdir -p /root/lib && cp /host_lib/bugyi.sh /root/lib/bugyi.sh
   echo "BASH=$BASH_VERSION"
   bash /repo/home/bin/executable_tmux_ai_window 2>&1 | head -40
   EOF
   docker run --rm -v "$PWD:/repo:ro" -v "$HOME/lib:/host_lib:ro" \
     -v /tmp/e2e32.sh:/e2e.sh:ro bash:3.2 bash /e2e.sh
   ```

   Expect `BASH=3.2.57(1)-release`, no `command not found`, `MENU:-C` followed by `1`,
   and the `agy`/`claude`/`muse` rows in that order. Two container artifacts are **not**
   script bugs: stub shebangs must be `#!/bin/sh` because that image has no `/bin/bash`,
   and any `perl: command not found` from `bugyi.sh` only appears on the `log::warn`
   fallback path when the `tmux` stub returns non-zero.

Manual smoke test on the MacBook — the only true end-to-end confirmation, and the user's
call to run:

1. Commit, then run `chezmoi update -a --force` (required by the chezmoi repo's own
   `CLAUDE.md`, and the only way `~/bin/tmux_ai_window` picks up the change).
2. On the MacBook, run `tmux_ai_window` from a shell. Confirm there is no
   `mapfile: command not found` and no false `No AI agent CLI found (...)` message.
3. In tmux, press the `A` binding. Confirm the menu lists exactly the AI CLIs installed
   on that Mac, in shortcut-key order, with `claude` pre-highlighted if installed.
4. Pick a provider and confirm the `ai` window opens in the originating pane's
   directory.

## Out of Scope

- **A repo-wide bash 3.2 guard over all of `home/bin/`.** Attractive, and it would pass
  today — the same grep finds no violations in the other ~100 scripts. But it is a
  fleet-wide portability policy affecting every script in the repo, not part of fixing
  this bug, and it does not belong in a file named `tmux_ai_window_test.sh`. If the user
  wants it, file it with `/sase_new_task`.
- **Changing the shebang, or adopting a Homebrew bash 5 dependency.** See Decisions.
- **Anything in `~/lib/bugyi.sh`.** Different repo (`bbugyi200/basher`), installed by
  `basher`, and already clean.
- **Adding a Docker-based bash 3.2 lane to `just test-bash` or CI.**
- **Refactoring the seven parallel `case` statements into one provider table.** A real
  maintenance smell at seven providers, and still a separate change.

## Acceptance Criteria

- `home/bin/executable_tmux_ai_window` contains no `mapfile`/`readarray` call, and no
  other bash 4-only builtin or syntax.
- `show_provider_menu` builds one `menu_providers` array and drives both the `-C`
  default choice and the rendered rows from it, exactly as it does today.
- Menu behavior on bash 5 is unchanged: all thirteen existing bashunit tests pass
  without modification.
- The new static guard fails if a bash 4-only construct is reintroduced outside a
  comment.
- The script renders the correct menu under bash 3.2.57.
- `just test-bash` and `just check` pass.
