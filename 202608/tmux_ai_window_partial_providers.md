---
tier: tale
title: Make the tmux_ai_window menu adapt to installed providers
goal:
  The tmux AI-agent menu contains only installed providers in shortcut-key order,
  selects Claude whenever it is available, otherwise selects the first available
  provider, and handles the zero-provider case without opening an unusable menu.
size: small
proposed_by: bbugyi200.athena.00z.f0.f0
create_time: 2026-08-14 10:04:39
status: wip
---

# Plan: Make the tmux_ai_window menu adapt to installed providers

## Context

`tmux_ai_window` lives in the **chezmoi** linked repo at
`home/bin/executable_tmux_ai_window` and is deployed as `~/bin/tmux_ai_window`. With no
arguments, it discovers the current tmux pane directory and builds a `display-menu` for
the known AI-agent providers. The Bash regression tests are in
`tests/bash/tmux_ai_window_test.sh` and replace tmux and provider executables with
controlled stubs.

The current implementation sorts all known providers by their single-key shortcut. It
then emits one row for every provider: installed providers are selectable, while
uninstalled providers are dimmed and disabled. Its `-C` default-choice index is
therefore calculated against the full catalog, including disabled rows.

The immediately preceding menu-order change deliberately preserved those disabled rows
to keep positions stable. This plan supersedes that decision for the new requirement: on
a machine with only a subset of the CLIs, the menu should be built from that installed
subset. Unavailable providers should not consume menu space or affect the selected-row
index.

### Opening the repo

The implementing agent MUST use `/sase_repo` before reading or modifying the linked
checkout:

```bash
sase repo open chezmoi -r "Make tmux_ai_window adapt to the installed provider subset"
```

All paths below are relative to the path printed by that command.

## Decisions

### Filter once, then use the same list for selection and rendering

Build a single `menu_providers` array containing only providers for which
`provider_is_installed` succeeds, ordered by the existing `providers_sorted_by_menu_key`
helper. Use this exact array for both:

1. determining the `-C` starting choice, and
2. emitting `display-menu` row triples.

This preserves alphabetical shortcut-key order for every installation subset and
prevents index drift between selection and rendering. Availability should be resolved
once while assembling the array; the direct `--launch` path should retain its existing
second check so removing a binary after the menu opens still fails cleanly.

### Prefer Claude; otherwise default to the first visible row

Keep `DEFAULT_PROVIDER="claude"`. If Claude is in `menu_providers`, pass its zero-based
index to `display-menu -C`. If Claude is absent, pass `0`, which selects the first
installed provider in shortcut-key order. Examples:

| Installed providers     | Visible key order | `-C` | Selected provider |
| ----------------------- | ----------------- | ---: | ----------------- |
| `agy`, `claude`, `muse` | `a`, `c`, `m`     |    1 | `claude`          |
| `opencode`, `qwen`      | `o`, `q`          |    0 | `opencode`        |
| `grok`                  | `g`               |    0 | `grok`            |

Do not derive the fallback from positions in the full `PROVIDERS` catalog. Once rows are
filtered, those positions no longer describe the tmux menu.

### Preserve the zero-provider behavior

If `menu_providers` is empty, keep the existing
`No AI agent CLI found (claude/codex/agy/qwen/opencode/grok/muse).` tmux message and
return successfully without issuing `display-menu`. Continue deriving the provider list
in that message from `PROVIDERS` so newly supported providers cannot be omitted from the
diagnostic.

### Remove obsolete disabled-row code

Once only installed providers are rendered, the disabled-row branch in
`show_provider_menu` and `provider_disabled_menu_label` are dead code. Remove them
rather than retaining a second presentation path that can drift from the installed-row
behavior.

### Leave provider definitions and launch commands unchanged

Do not change provider keys, labels, descriptions, accent colors, binary names, model
flags, `--launch` validation, tmux window creation, or the ordering of `PROVIDERS`.
`providers_sorted_by_menu_key` remains the source of visible ordering.

## Implementation

### 1. Update `home/bin/executable_tmux_ai_window`

Refactor `show_provider_menu` as follows:

1. Obtain the known providers in shortcut-key order from `providers_sorted_by_menu_key`.
2. Filter them through `provider_is_installed` into one local `menu_providers` array.
3. If that array is empty, emit the existing no-provider message and return without
   constructing a menu.
4. Initialize `default_choice` to `0`, then scan the installed array indices for
   `DEFAULT_PROVIDER`; replace the default with Claude's index when found.
5. Render every row from `menu_providers` as an enabled row using the existing label,
   key, quoting, and `--launch` callback logic. Eliminate the disabled-row `else`
   branch.
6. Remove `provider_disabled_menu_label`, which has no remaining caller.

The implementation may keep the sorting and filtering as separate local arrays or add a
narrowly named installed-provider helper. Whichever form is used, it must avoid
calculating `-C` against a different sequence from the one rendered.

### 2. Expand `tests/bash/tmux_ai_window_test.sh`

Adjust helpers and assertions so they model a filtered menu rather than the full known
provider catalog:

- Replace or generalize `menu_index`, which currently returns positions in the full
  `MENU_ORDER`; subset-default tests must assert positions in the visible subset.
- Keep the independent full `MENU_ORDER` expectation for the all-installed sort test.
- Add a partial-installation test with Claude and providers on both sides of it in key
  order. Assert only installed names/keys are emitted, unavailable-provider labels and
  callbacks are absent, rows remain key-sorted, and `-C` points to Claude's visible
  index.
- Update the Claude-missing test (`opencode` plus `qwen`) to assert the visible rows are
  `opencode`, `qwen` and `-C` is `0`.
- Update the single-provider Grok test to assert Grok is the only row and `-C` is `0`.
- Add a no-provider test that asserts the exact diagnostic is sent through tmux and no
  `display-menu` call occurs.

Retain the all-installed row-order, key-uniqueness, Claude-preference, Grok/Muse menu
wiring, launch-command, and invalid-provider coverage. The tests must continue to use
the controlled `FAKE_BIN` path so provider executables installed on the test host do not
affect results.

## Verification

From the opened chezmoi checkout:

1. Run the focused Bash suite:

   ```bash
   just test-bash
   ```

2. Run the repository-wide gate:

   ```bash
   just check
   ```

3. Review the diff and confirm it is limited to the launcher and its Bash regression
   test.

If the implementation is later committed, the chezmoi repository instruction requires
running `chezmoi update -a --force`. After applying it, manually open the menu in tmux
and confirm that the local installed subset is sorted by shortcut key and Claude is
highlighted when present. The controlled Bash tests are the authoritative verification
for the Claude-missing and zero-provider cases, which should not require uninstalling
local tools.

## Acceptance Criteria

- Only installed providers appear in the menu.
- Visible rows remain alphabetically ordered by their assigned single-key shortcuts.
- Claude is the selected row whenever Claude is installed, regardless of its visible
  position.
- Without Claude, the first visible provider is selected (`-C 0`).
- With exactly one provider, that provider is the only row and is selected.
- With no providers, the existing diagnostic appears and no menu is opened.
- Direct launch behavior and all provider-specific commands remain unchanged.
- `just test-bash` and `just check` pass.
