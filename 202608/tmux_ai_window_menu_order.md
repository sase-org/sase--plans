---
tier: tale
title: Order the tmux_ai_window menu alphabetically by shortcut key
goal:
  The tmux AI-agent menu renders its rows in alphabetical order of their single-key
  shortcuts, claude stays the pre-selected default wherever it lands in that order, and
  bashunit regression tests pin both the row order and the default-choice preference.
size: small
proposed_by: bbugyi200.athena.00z.f0
create_time: 2026-08-14 09:43:46
status: wip
---

# Plan: Order the tmux_ai_window menu alphabetically by shortcut key

## Context

`tmux_ai_window` is a bash script in the **chezmoi** linked repo at
`home/bin/executable_tmux_ai_window` (deployed to `~/bin/tmux_ai_window`), bound in
`home/dot_config/tmux/tmux.conf` as `bind A run "tmux_ai_window"`. With no arguments it
renders a tmux `display-menu` of AI agent CLIs.

Today `readonly -a PROVIDERS=(claude codex agy qwen opencode grok muse)` is a single
array that drives **three** separate things:

1. the on-screen menu row order,
2. the `-C` default-choice index (the first _installed_ provider wins), and
3. the provider list in the "no AI agent CLI found" message.

Providers have been appended over time, so the current on-screen order is chronological,
not alphabetical:

| row | provider   | key | description |
| --- | ---------- | --- | ----------- |
| 0   | `claude`   | `c` | Anthropic   |
| 1   | `codex`    | `x` | OpenAI      |
| 2   | `agy`      | `a` | Antigravity |
| 3   | `qwen`     | `q` | Alibaba     |
| 4   | `opencode` | `o` | SST         |
| 5   | `grok`     | `g` | xAI         |
| 6   | `muse`     | `m` | Meta        |

### Opening the repo

The implementing agent MUST open the chezmoi repo through the `/sase_repo` skill before
reading or writing any of these files:

```bash
sase repo open chezmoi -r "Order the tmux_ai_window menu alphabetically by shortcut key"
```

Every path below is relative to the path that command prints.

## Decisions

### The target order

Sorting by **shortcut key** (what the user asked for) gives:

| row | key | provider   |
| --- | --- | ---------- |
| 0   | `a` | `agy`      |
| 1   | `c` | `claude`   |
| 2   | `g` | `grok`     |
| 3   | `m` | `muse`     |
| 4   | `o` | `opencode` |
| 5   | `q` | `qwen`     |
| 6   | `x` | `codex`    |

Note the one visible surprise: **`codex` lands last**, because its key is `x` (`c` was
already taken by `claude`). Sorting by key is not the same as sorting by provider name.
This is the requested behaviour, not a bug — do not "fix" it by reassigning keys, and do
not sort by name instead.

### Sort at render time; do not reorder `PROVIDERS`

Derive the menu order from `PROVIDERS` by sorting on `provider_menu_key`, rather than
rewriting the array into key order. Reasons:

- The file's established idiom keeps the six parallel `case` statements in the same
  order as `PROVIDERS` for scannability. Reordering the array would either churn ~40
  lines across six `case` blocks or break that convention.
- A derived sort is self-maintaining. The natural way to add the eighth provider is to
  append it to `PROVIDERS`; with a derived sort that provider lands in the right menu
  row for free, instead of silently un-alphabetizing the menu.

### `claude` stays the default via an explicit named constant

Today the default is implicit: "whichever installed provider comes first in the array,"
which only produced `claude` because `claude` happened to be element 0. Once rows are
key-sorted, `agy` would inherit that by accident.

Introduce `readonly DEFAULT_PROVIDER="claude"` and select its row when it is installed.
Keep the existing "first installed provider" rule as the fallback for when `claude` is
absent, so behaviour on a machine without `claude` is unchanged.

### `-C` is a 0-based index over **all** menu items, including disabled ones

This is load-bearing and the man page does not spell it out, so it was verified against
tmux's source (`menu.c`, `menu_prepare()`, tag `3.5a`; installed tmux is 3.5a):

- `starting_choice` indexes `menu->items`, which contains every item added — including
  rows whose name begins with `-` (disabled) and separators.
- If the item at that index is disabled, tmux scans _forward_ (wrapping) to the first
  selectable item.

The script emits exactly one menu item per provider (an enabled row when installed, a
`-`-prefixed disabled row when not), so index == provider position. That invariant is
what makes the current code correct, and it survives this change **only if the
default-choice loop and the row-rendering loop iterate the same sequence**. Both must
iterate the sorted list — this is the one way to get this change subtly wrong.

Do not try to simplify by always passing `claude`'s index and letting tmux's forward
scan handle an uninstalled `claude`; that quietly changes which provider is selected (it
would wrap forward from `claude`'s row rather than picking the first installed row) and
it makes the behaviour untestable through the `-C` argument.

### Rows keep their sorted position whether or not the CLI is installed

The sort is over all of `PROVIDERS`, not just installed ones, so the menu layout is
stable: installing or removing a CLI toggles a row between enabled and dim, it never
reshuffles the other rows. This matches today's behaviour.

### Left alone deliberately

- The `DOC` heredoc's `Supported values: claude, codex, agy, qwen, opencode, grok, muse`
  — that is `--launch` help text, not the menu, and the heredoc is expanded before
  `PROVIDERS` is declared.
- The `"No AI agent CLI found (...)"` message, which stays derived from `PROVIDERS` in
  array order. It is a flat list of names; reordering it by key would look arbitrary.

## Implementation Steps

### 1. `home/bin/executable_tmux_ai_window`

**(a)** Just below the `readonly -a PROVIDERS=(...)` declaration (currently line 220),
add the default-provider constant and the sort helper:

```bash
# Provider that starts selected in the menu whenever it is installed.
readonly DEFAULT_PROVIDER="claude"

# Menu rows render in shortcut-key order so that appending to PROVIDERS never
# has to match the on-screen order. Menu keys are unique (pinned by the
# bashunit regression test), so sorting on the key alone is a total order.
function providers_sorted_by_menu_key() {
  local provider
  for provider in "${PROVIDERS[@]}"; do
    printf '%s\t%s\n' "$(provider_menu_key "${provider}")" "${provider}"
  done | LC_ALL=C sort | cut -f2
}
```

`LC_ALL=C` keeps the order locale-independent. `~/lib/bugyi.sh` sets no
`set -e`/`set -u`/`pipefail`, so the pipeline needs no extra guarding.

**(b)** Replace the top of `show_provider_menu` (currently lines 44-66) so both the
default-choice scan and the render loop walk the sorted list:

```bash
  local pane_dir
  pane_dir="$(tmux display-message -p '#{pane_current_path}')"

  local -a menu_providers=()
  mapfile -t menu_providers < <(providers_sorted_by_menu_key)

  local default_choice=""
  local first_installed_choice=""
  local installed_count=0
  local index=0
  local provider
  for provider in "${menu_providers[@]}"; do
    if provider_is_installed "${provider}"; then
      installed_count=$((installed_count + 1))
      if [[ -z "${first_installed_choice}" ]]; then
        first_installed_choice="${index}"
      fi
      if [[ "${provider}" == "${DEFAULT_PROVIDER}" ]]; then
        default_choice="${index}"
      fi
    fi
    index=$((index + 1))
  done

  if [[ "${installed_count}" -eq 0 ]]; then
    local provider_list
    provider_list="$(IFS=/; echo "${PROVIDERS[*]}")"
    tmux_message "No AI agent CLI found (${provider_list})."
    return 0
  fi

  if [[ -z "${default_choice}" ]]; then
    default_choice="${first_installed_choice}"
  fi
```

**(c)** Change the row-rendering loop's header (currently line 80) from
`for provider in "${PROVIDERS[@]}"; do` to `for provider in "${menu_providers[@]}"; do`.
Its body is unchanged.

The `-C "${default_choice}"` line in `menu_args` and everything else in the function
stay as they are. No other file changes: `launch_provider`, `parse_cli_args`, the six
`case` statements, and `home/dot_config/tmux/tmux.conf` are all untouched.

### 2. `tests/bash/tmux_ai_window_test.sh`

**(a)** Next to the existing `PROVIDERS=(...)` line at the top, add the expected menu
order as a separate, hand-written constant. Deriving it from the production sort would
defeat the purpose; writing it out independently is the point.

```bash
# Expected on-screen row order: providers sorted by their menu key
# (a=agy, c=claude, g=grok, m=muse, o=opencode, q=qwen, x=codex).
MENU_ORDER=(agy claude grok muse opencode qwen codex)
```

**(b)** Add a `menu_index` helper that maps a provider to its expected `-C` value, so
the default-choice tests stop hardcoding magic numbers:

```bash
function menu_index() {
  local target="$1"
  local index=0
  local provider
  for provider in "${MENU_ORDER[@]}"; do
    if [[ "${provider}" == "${target}" ]]; then
      printf '%s\n' "${index}"
      return 0
    fi
    index=$((index + 1))
  done
  return 1
}
```

**(c)** Add a `display_menu_provider_names` helper alongside the existing
`display_menu_provider_keys`, extracting the display name out of each enabled row's
label (`#[fg=<accent>,bold]%-10s#[fg=#565f89,nobold] <description>`) in row order:

```bash
function display_menu_provider_names() {
  awk '
    /^ARG:#\[fg=/ {
      line = $0
      sub(/^ARG:#\[fg=[^]]*\]/, "", line)
      sub(/#\[fg=#565f89,nobold\].*$/, "", line)
      sub(/ +$/, "", line)
      print line
    }
  ' < <(display_menu_args)
}
```

Any equivalent extraction is fine as long as it yields the enabled rows' display names
in menu order. Disabled rows are prefixed with `-#[fg=#565f89]` and correctly do not
match `^ARG:#\[fg=`.

**(d)** Update the one existing test whose expectation moves:
`test_only_grok_installed_makes_grok_the_default_choice` currently asserts `"5"`. Change
it to `"$(menu_index grok)"` (which is now `2`). Its meaning is unchanged — it still
pins the first-installed fallback.

**(e)** Add these tests:

1. `test_menu_rows_are_sorted_by_menu_key` — install all providers, run the menu, and
   assert the emitted key sequence is exactly `a c g m o q x`, one per line:

   ```bash
   assert_same "$(printf 'a\nc\ng\nm\no\nq\nx')" "$(display_menu_provider_keys)"
   ```

   This is the direct pin of the user-visible requirement.

2. `test_menu_rows_are_ordered_by_provider_name_matching_keys` — same setup, assert
   `"$(printf '%s\n' "${MENU_ORDER[@]}")"` equals `"$(display_menu_provider_names)"`.
   Together with the previous test this pins both the row order _and_ the label-to-key
   pairing, so a future edit cannot put `agy`'s label on `claude`'s key.

3. `test_claude_is_the_default_choice_even_when_not_the_first_row` — install all
   providers, assert the `-C` value is `"$(menu_index claude)"` (now `1`, not `0`). This
   is the regression guard for the whole request: it fails if the menu is re-sorted
   without the `DEFAULT_PROVIDER` preference, and it fails if the preference is added
   without re-sorting.

4. `test_claude_is_preferred_over_an_earlier_installed_provider` — install **only**
   `agy` and `claude`, assert `-C` is `"$(menu_index claude)"`. `agy` occupies row 0 and
   is installed, so this is what distinguishes "prefer claude" from "first installed".

5. `test_first_installed_provider_is_default_when_claude_is_missing` — install only
   `qwen` and `opencode` (no `claude`), assert `-C` is `"$(menu_index opencode)"`
   (`opencode`'s key `o` sorts before `qwen`'s `q`, so it is the earlier row). This pins
   the fallback against the _sorted_ order rather than the array order.

**(f)** Add a one-line comment on the existing `test_provider_menu_keys_are_unique`
noting that key uniqueness is now what makes the menu sort deterministic, so the test is
load-bearing for ordering and not merely a nicety.

Leave `test_menu_includes_grok_and_muse_rows_with_keys` alone — it uses
`assert_contains` against the whole argument blob and is order-independent.

## Verification

Run from the chezmoi repo root:

```bash
just test-bash          # fast, targeted: the bashunit lane
just check              # fmt-check + lint + full test suite, before landing
```

Note the repo's `lint` lane covers python/lua/markdown/keep-sorted only; there is no
bash formatter to satisfy, so just match the surrounding style by hand (two-space
indent, `function name() {`, `${var}` braces).

If `just check` surfaces a failure in the nvim/hammerspoon/python lanes, confirm it
reproduces on a clean tree and file it with `/sase_new_task` rather than folding a fix
into this change.

Manual smoke test (the only way to confirm real tmux agrees with the `-C` index
semantics read out of its source):

1. Commit, then run `chezmoi update -a --force` — required by the chezmoi repo's own
   `CLAUDE.md` after any commit, and the only way `~/bin/tmux_ai_window` picks up the
   change.
2. In tmux, press the `A` binding. Confirm the rows read top-to-bottom as `agy`,
   `claude`, `grok`, `muse`, `opencode`, `qwen`, `codex`.
3. Confirm the **`claude`** row (second from the top) is the highlighted row when the
   menu opens, and that pressing Enter immediately launches `claude`.
4. Press a key that is not the default (e.g. `x`) and confirm `codex` — the last row —
   still launches correctly, i.e. the row-to-command wiring survived the reorder.

If the highlighted row on open is not `claude`, report it rather than papering over it
with an off-by-one nudge: that would mean tmux is counting only selectable items, and
the fix is to count installed providers instead of all providers in the index loop.

## Out of Scope

- **Refactoring the six parallel `case` statements into one provider table.** A real
  maintenance smell at seven providers, but a much larger and riskier diff; keep it
  separate.
- **Reassigning shortcut keys so the menu is also alphabetical by provider name** (e.g.
  giving `codex` a key that sorts near `claude`). The user asked for key order; changing
  established muscle-memory keys is a different request.
- **Sorting the `DOC` heredoc list or the "no CLI found" message.**
- **Deriving the test's `PROVIDERS`/`MENU_ORDER` arrays from the script instead of
  hardcoding them.** They can drift from production, but sourcing the script or parsing
  its array is a separate, more invasive change to the test harness.
