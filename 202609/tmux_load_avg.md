---
tier: tale
title: Show the 1m load average next to the hostname in the tmux status line
goal:
  The tmux status line ends with the hostname followed by the 1-minute load average in
  parentheses (`athena (26.46)`), parsed from the `uptime` command, on both athena and
  the mac.
size: small
proposed_by: bbugyi200.athena.0id
create_time: 2026-09-10 12:54:00
status: wip
---

# Plan: Show the 1m load average next to the hostname in the tmux status line

The tmux status line currently ends with the bare hostname (`athena`). It should end
with the hostname followed by the 1-minute load average in parentheses
(`athena (26.46)`), parsed from the `uptime` command.

## Where the work happens

All changes land in the **`chezmoi`** repo, which is a linked repo of this project. Open
it first and use the printed path as the only path for every read and write in this
plan:

```bash
sase repo open chezmoi -r "Add a tmux status-line helper that shows the 1m load average"
```

Every file path below is relative to that printed chezmoi repo root.

## Current state (verified)

`home/dot_config/tmux/tmux.conf` line 42:

```tmux
set -g status-right '#(bob tmux-pomodoro)#(tmux_ping)#(hostname)'
```

Facts that shape the design, all confirmed on the live machines:

- **Two `uptime` wordings must be handled.** These dotfiles are shared between `athena`
  (Linux) and `mac` (macOS), and the two platforms disagree on both the label and the
  separator:
  - athena: `11:01:57 up 4 days, 18:56,  3 users,  load average: 20.71, 24.19, 24.33` —
    singular `average:`, comma-separated.
  - mac: `11:03  up 5 days, 23:50, 4 users, load averages: 3.36 3.26 2.94` — plural
    `averages:`, space-separated.
- **Status helpers own their own decoration.** `tmux_ping` prints `✓ 5/20 | ` (trailing
  separator included) and prints nothing at all when it has nothing to say;
  `tmux_zo_pomodoro` does the same. The new helper follows that convention.
- **`status-interval` is 2**, so this script runs every two seconds. Keep it cheap.
- **`~/bin` is already on `PATH`** (set in `home/dot_profile`), and chezmoi's
  `executable_` filename prefix sets the exec bit on apply. This is exactly how
  `#(tmux_ping)` resolves today, so a new `home/bin/executable_tmux_load_avg` resolves
  the same way with no tmux `PATH` work needed.
- **`status-right-length` is 75 and there is ample room.** Worst case today is
  `[<2m] 1040-1105 — GTD | ` (~24) + `✓ 20/20 | ` (~10, macOS only) + `athena` (6) ≈ 40.
  Adding ` (20.71)` (8) lands near 48. Do not change `status-right-length`.
- **`home/.chezmoiscripts/run_onchange_after_reload_tmux_config.tmpl` hashes
  `tmux.conf`**, so editing `tmux.conf` makes `chezmoi apply` reload tmux automatically.

## Design

Add a `tmux_load_avg` helper that prints the decorated fragment ` (26.46)` — leading
space and parentheses included — and appends `#(tmux_load_avg)` to `status-right` right
after the existing `#(hostname)`.

Why this shape rather than the alternatives:

- **Do not inline the shell pipeline in `tmux.conf`.** Nesting the quoting that `sed`
  and `awk` need inside a single-quoted tmux option is unreadable and untestable.
- **Do not fold `hostname` into the new script.** Leaving `#(hostname)` untouched keeps
  the diff to one appended format directive and keeps `tmux.conf` self-describing.
- **The script emits the parentheses, not `tmux.conf`.** If `tmux.conf` carried them
  (`#(hostname) (#(tmux_load_avg))`), a failed or unparseable `uptime` would render
  `athena ()`. With the script owning them, the failure mode is a clean `athena`.

## Files to change

### 1. NEW `home/bin/executable_tmux_load_avg`

Follow the self-contained pattern of `home/bin/executable_tmp_trash_empty` — `set -e`, a
`usage()` function, `run "$@"` at the bottom — rather than the `source ~/lib/bugyi.sh` +
`SCRIPTNAME` guard pattern used by `tmux_ping`. Sourcing `bugyi.sh` (8.3K, plus its own
setup work) every two seconds is not worth it for a script whose entire job is one
regex.

This reference implementation has had its parsing verified against the real athena and
mac `uptime` output plus the edge cases; use it as written unless you find a defect:

```bash
#!/bin/bash

##############################################################################
#  Print the 1-minute load average, formatted for the tmux status line.      #
#                                                                            #
#  The output carries its own leading space and parentheses -- " (20.71)" -- #
#  so tmux.conf can append `#(tmux_load_avg)` directly after `#(hostname)`   #
#  and render `athena (20.71)`. On ANY failure (uptime missing, uptime       #
#  erroring, output that does not parse) this prints nothing, so the status  #
#  line degrades to a bare `athena` rather than an empty `athena ()`.        #
#                                                                            #
#  tmux runs this every `status-interval` (2s), so it deliberately does not  #
#  source ~/lib/bugyi.sh and forks `uptime` exactly once.                    #
#                                                                            #
#  The `uptime` wording differs by platform and both are matched here:       #
#    Linux: `... 3 users,  load average: 20.71, 24.19, 24.33`                #
#    macOS: `... 4 users, load averages: 3.36 3.26 2.94`                     #
##############################################################################

set -e

function usage() {
  cat <<EOM
usage: $(basename "$0") [-h]

Print the 1-minute load average, parsed from \`uptime\`, decorated for the tmux
status line as " (<load>)". Prints nothing when the load average cannot be read,
so the status line falls back to showing just the hostname.

Optional Arguments
------------------
-h | --help
    View this help message.

Examples
--------
tmux_load_avg     # ' (20.71)'
EOM
}

function run() {
  while [[ -n "$1" ]]; do
    case $1 in
    -h | --help)
      usage
      exit 0
      ;;
    *)
      printf 'error: unrecognized argument: %s\n\n' "$1" >&2
      usage >&2
      exit 2
      ;;
    esac
    shift
  done

  local load
  load="$(one_minute_load)" || exit 0

  printf ' (%s)' "${load}"
}

# Echo the 1-minute load average, or return non-zero if it cannot be determined.
#
# LC_ALL=C stops `uptime` from rendering the number with a locale-specific
# decimal separator; the comma substitution below is a second guard for any
# platform that ignores it.
function one_minute_load() {
  local uptime_out
  uptime_out="$(LC_ALL=C uptime 2> /dev/null)" || return 1

  [[ "${uptime_out}" =~ load[[:space:]]+averages?:[[:space:]]*([0-9]+([.,][0-9]+)?) ]] \
    || return 1

  printf '%s' "${BASH_REMATCH[1]/,/.}"
}

run "$@"
```

Verified parse results for this regex:

| Input                                   | Output      |
| --------------------------------------- | ----------- |
| `... load average: 20.71, 24.19, 24.33` | ` (20.71)`  |
| `... load averages: 3.36 3.26 2.94`     | ` (3.36)`   |
| `... load average: 20,71, 24,19, 24,33` | ` (20.71)`  |
| `... load average: 0, 0, 0`             | ` (0)`      |
| `uptime: command not found`             | (no output) |
| (empty)                                 | (no output) |

### 2. NEW `tests/bash/tmux_load_avg_test.sh`

bashunit tests, modeled directly on `tests/bash/tmp_trash_empty_test.sh`: stub the
dependency (`uptime`) into a temp `FAKE_BIN` directory, put it first on `PATH`, and run
the script with `bash "${SCRIPT}"`. Follow that file's structure — the boxed header
comment, `SCRIPT="${PWD}/home/bin/executable_tmux_load_avg"` at the top (tests run from
the repo root), `set_up`/`tear_down` with `mktemp -d`, and a small helper that invokes
the script with the stubbed `PATH`.

Make the `uptime` stub's output configurable per test (e.g. write the desired line into
a file the stub `cat`s, or bake it into the stub inside each test's setup helper), and
cover:

1. Linux `uptime` wording yields ` (20.71)`.
2. macOS `uptime` wording yields ` (3.36)`.
3. Comma-decimal-separator output yields ` (20.71)` — the separator is normalized.
4. Output with no recognizable load average yields **empty** stdout, and specifically
   contains no `(` or `)`.
5. A stub `uptime` that exits non-zero yields empty stdout and exit code 0 — a broken
   `uptime` must not make tmux render an error.
6. `uptime` absent from `PATH` entirely yields empty stdout and exit code 0. (Set `PATH`
   to just `FAKE_BIN` with no `uptime` stub in it; note the script itself needs no other
   external binaries.)
7. `--help` prints the usage text and exits 0.
8. An unrecognized argument exits 2.

These tests stub `uptime`, so the macOS wording is covered on Linux and CI (`just test`
on `ubuntu-latest`) exercises every case.

### 3. EDIT `home/dot_config/tmux/tmux.conf`

Append `#(tmux_load_avg)` to the end of the `status-right` value on line 42:

```tmux
set -g status-right '#(bob tmux-pomodoro)#(tmux_ping)#(hostname)#(tmux_load_avg)'
```

That is the only line to change in this file.

**Do not touch `home/dot_config/tmux/theme.conf`.** It also sets `status-right`, but it
is vendored verbatim from the upstream tokyonight tmux theme (see its header comment)
and `tmux.conf` sources it on line 11 and then overrides `status-right` on line 42, so
the theme's value never takes effect.

## Verification

Run from the chezmoi repo root:

1. **Unit tests** — the focused loop, then the whole bash suite:

   ```bash
   bashunit ./tests/bash/tmux_load_avg_test.sh
   just test-bash
   ```

2. **Script output matches `uptime`** — the printed number must equal the first load
   figure in `uptime`'s own output:

   ```bash
   uptime
   bash home/bin/executable_tmux_load_avg; echo
   ```

3. **Deploy.** Commit the change (per this repo's instructions, a commit must be
   followed by an apply), then:

   ```bash
   chezmoi update -a --force
   ```

4. **Confirm the deploy landed:**

   ```bash
   test -x ~/bin/tmux_load_avg && echo "helper installed"
   tmux show -gv status-right   # must now end with #(tmux_load_avg)
   ```

   `chezmoi apply` triggers `run_onchange_after_reload_tmux_config.tmpl`, which reloads
   tmux; `tmux show -gv status-right` reading back the new value confirms the reload
   took effect.

5. **Eyeball the live status line.** The bottom-right of the tmux status bar should read
   `athena (26.46)` (with the current load), where it previously read `athena`. Verify
   this visually rather than through `tmux display-message -p "#{E:status-right}"` —
   that one-shot expansion was tried on tmux 3.5a here and returns an empty string,
   because `#()` job output is only populated for the client that draws the status line.

## Out of scope

Do not add any of the following; the request was specifically the 1-minute load average
next to the hostname:

- Color or threshold highlighting when the load is high.
- The 5-minute or 15-minute averages, or a flag to select which average to print.
- Normalizing the load by CPU count.
- Replacing `#(hostname)` with tmux's built-in `#h`.
