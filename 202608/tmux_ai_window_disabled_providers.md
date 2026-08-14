---
tier: tale
title: Show unavailable tmux AI providers as polished disabled rows
goal:
  The tmux AI-agent menu always shows the complete provider catalog in shortcut-key
  order when at least one provider is installed, presents unavailable providers as
  clearly muted and non-selectable rows, and still defaults to Claude or the first
  installed provider as appropriate.
size: small
proposed_by: bbugyi200.athena.00z.f0.f0.f2
create_time: 2026-08-14 10:39:40
status: wip
---

# Plan: Show unavailable tmux AI providers as polished disabled rows

## Context

`tmux_ai_window` lives in the **chezmoi** linked repo at
`home/bin/executable_tmux_ai_window` and is deployed as `~/bin/tmux_ai_window`. With no
arguments, it discovers the current tmux pane directory and builds a centered
`display-menu` for the known AI-agent providers. Its Bash regression tests live in
`tests/bash/tmux_ai_window_test.sh` and replace tmux and provider executables with
controlled stubs.

The menu currently sorts providers by their assigned single-key shortcuts, but filters
the sorted list down to installed binaries before both rendering and calculating tmux's
`-C` starting-choice index. That makes partial installations function correctly, at the
cost of hiding the full provider catalog and causing rows to move when a CLI is added or
removed.

This change restores a stable, complete catalog without restoring the earlier default
index bug. Installed providers remain vivid and actionable. Missing providers occupy
their normal key-sorted positions, use a restrained gray treatment with an explicit
`not installed` status, and are structurally disabled so neither Enter nor a shortcut
can choose them.

### Opening the repo

The implementing agent MUST use `/sase_repo` before reading or modifying the linked
checkout:

```bash
sase repo open chezmoi -r "Show unavailable tmux AI providers as disabled menu rows"
```

All paths below are relative to the path printed by that command.

## Design Decisions

### Keep every provider in one stable, key-sorted catalog

Continue deriving the menu order from `providers_sorted_by_menu_key`; do not reorder the
`PROVIDERS` declaration or any provider metadata `case` statements. The rendered order
remains:

| row | key | provider   |
| --: | :-: | ---------- |
|   0 | `a` | `agy`      |
|   1 | `c` | `claude`   |
|   2 | `g` | `grok`     |
|   3 | `m` | `muse`     |
|   4 | `o` | `opencode` |
|   5 | `q` | `qwen`     |
|   6 | `x` | `codex`    |

On a partial installation, all seven rows stay in these positions. Installing or
removing a CLI changes only that provider's presentation and actionability; it does not
reflow the menu or move the other providers.

### Use tmux's native disabled-item contract

Tmux documents a menu item whose name begins with `-` as disabled, dimmed, and
unselectable. Use that native contract rather than attaching a no-op command. For an
unavailable provider, emit this item triple:

1. a `-`-prefixed muted label,
2. an empty key, and
3. an empty command.

The leading `-` is the authoritative disabled state. Empty key and command fields make
the generated argument structure self-evidently inert as well: the unavailable row
advertises neither a working shortcut nor a callback. The direct `--launch` installation
check remains unchanged as defense against a binary disappearing after the menu opens or
an internal launch invocation bypassing the menu.

### Make availability legible without visual noise

Keep installed rows exactly as they are today: provider name in its individual accent
color and bold weight, followed by the vendor in the existing muted color. Add a
dedicated `provider_disabled_menu_label` helper that renders the provider name, vendor,
and an explicit `(not installed)` status in the same muted gray family, with bold
disabled and fixed-width columns so status text aligns cleanly across disabled rows.

The exact output should follow this restrained shape:

```text
provider    vendor       (not installed)
```

Use tmux formatting and ASCII punctuation already supported by the script; do not add
Unicode status glyphs whose width can vary by terminal or font. Do not add a legend,
dynamic count, or other chrome: the inline status is the clearest explanation and keeps
the compact menu title unchanged.

### Calculate the default against the rows that tmux actually receives

`display-menu -C` is a zero-based index into the complete item list, including disabled
rows. Scan the full sorted catalog, not a filtered subset, and track:

- the full-catalog index of the first installed provider,
- the full-catalog index of `DEFAULT_PROVIDER="claude"` when Claude is installed, and
- whether at least one provider is installed.

Prefer Claude's full-catalog index; otherwise use the first installed provider's
full-catalog index. Examples:

| Installed providers     | `-C` | Selected provider |
| ----------------------- | ---: | ----------------- |
| `agy`, `claude`, `muse` |    1 | `claude`          |
| `opencode`, `qwen`      |    4 | `opencode`        |
| `grok`                  |    2 | `grok`            |

Capture each provider's availability while constructing the menu model and reuse that
snapshot for rendering. A pair of parallel Bash 3.2-compatible indexed arrays is
acceptable. This avoids both repeated executable lookups and a mismatch between the
state used to calculate `-C` and the state used to decide whether a row is actionable.
The launch callback must still recheck installation at execution time.

### Preserve the all-missing fast path

If no known CLI is installed, preserve the existing
`No AI agent CLI found (claude/codex/agy/qwen/opencode/grok/muse).` tmux message and
return successfully without opening `display-menu`. An all-disabled menu has no valid
default or useful action, so the direct diagnostic is the more intuitive zero-state.
Continue deriving its provider list from `PROVIDERS` so the message cannot drift when a
provider is added.

### Preserve portability and provider behavior

The launcher is deployed on macOS, where `/bin/bash` is Bash 3.2. Keep the current
`while read` array-loading pattern and use only indexed-array syntax available there; do
not reintroduce `mapfile`, `readarray`, or associative arrays. Do not change provider
keys, binary names, launch commands, model flags, display names, vendor descriptions,
accent colors, argument parsing, tmux window lifecycle, or tmux configuration.

## Implementation

### 1. Update `home/bin/executable_tmux_ai_window`

Refactor `show_provider_menu` around a full-catalog availability snapshot:

1. Read every provider from `providers_sorted_by_menu_key` into `menu_providers`, while
   recording an aligned installed/uninstalled flag for each row.
2. During that same scan, count installed providers and record the full-catalog indexes
   of the first installed provider and installed Claude.
3. Preserve the current diagnostic-and-return branch when the installed count is zero.
4. Set `default_choice` to Claude's recorded index when available, otherwise the first
   installed provider's recorded index.
5. Render one row per entry in `menu_providers`. Installed rows retain the current
   label, assigned key, quoting, and `--launch` callback. Missing rows receive a
   `-`-prefixed disabled label plus empty key and command arguments.
6. Add `provider_disabled_menu_label` next to `provider_menu_label`; keep the formatting
   concerns out of the menu-control flow.

Keep the menu option styles, title, border, placement, and direct-launch logic
unchanged.

### 2. Update `tests/bash/tmux_ai_window_test.sh`

Make the test helpers model full-catalog rows, including disabled item triples, rather
than treating only accent-colored enabled labels as visible providers. Keep `MENU_ORDER`
as the independent expected order and replace `visible_menu_index` with a full-catalog
`menu_index` helper where appropriate.

Cover these behaviors:

- With a partial set such as `agy`, `claude`, and `muse`, all seven provider names are
  present in key order; the three installed rows keep their accent labels, keys, and
  launch callbacks; the other rows use the exact muted `not installed` presentation,
  begin with tmux's `-` disabled marker, and have empty key and command fields.
- The partial-install default is Claude's full-catalog row (`-C 1`), not an index in an
  installed-only subset.
- With Claude missing and only `opencode` and `qwen` installed, `-C 4` selects the first
  installed row in key order even though four disabled rows precede it.
- With only Grok installed, `-C 2` selects Grok and every other provider is disabled.
- With all providers installed, the existing key-order, provider-order, uniqueness,
  Grok/Muse wiring, and Claude-default assertions still pass, and no row is marked
  `not installed`.
- With no providers installed, the existing diagnostic is emitted and no menu opens.
- A direct `--launch` for a known but missing provider still exits with status 1 and
  reports its binary as not installed, pinning the defense-in-depth behavior.

Keep provider discovery isolated through `FAKE_BIN` so real executables on the test
machine cannot influence any scenario. Prefer helpers that inspect each menu item's
three arguments together, so tests distinguish visual dimming from actual disabled and
non-actionable wiring.

## Verification

From the opened chezmoi checkout:

1. Run the focused Bash regression suite:

   ```bash
   just test-bash
   ```

2. Run the repository-wide gate:

   ```bash
   just check
   ```

3. Review the diff and confirm it is limited to `home/bin/executable_tmux_ai_window` and
   `tests/bash/tmux_ai_window_test.sh`.

If the implementation is later committed, follow the chezmoi repository instruction and
run:

```bash
chezmoi update -a --force
```

Then manually open the menu in tmux on a partial-install machine and confirm the catalog
is stable, unavailable rows are visibly subdued with aligned status text, arrow-key
navigation skips them, their shortcut keys do nothing, and Claude is highlighted when
installed. When Claude is absent, confirm the first installed provider in shortcut-key
order is highlighted. The controlled Bash suite remains authoritative for installation
matrices that are inconvenient to reproduce interactively.

## Acceptance Criteria

- When at least one known provider is installed, all known providers appear in
  alphabetical shortcut-key order.
- Installed providers retain their current colorful labels, shortcuts, and launch
  commands.
- Missing providers are clearly gray, say `(not installed)`, and cannot be selected by
  Enter, mouse, or shortcut key.
- Claude is selected by default whenever installed; otherwise the first installed
  provider in shortcut-key order is selected using its full-catalog row index.
- With no providers installed, the existing message appears and no inert menu opens.
- Direct launch of a missing known provider continues to fail cleanly.
- The script remains compatible with macOS Bash 3.2.
- No provider metadata, launch command, key binding, model flag, or tmux lifecycle
  behavior changes.
- `just test-bash` and `just check` pass.
