---
tier: tale
title: Models panel default-effort controls
goal: 'The Models panel lets users inspect, persistently edit, temporarily override,
  and clear the default reasoning-effort level through fast, polished, single-key
  workflows, including safe chezmoi-backed writes.

  '
create_time: 2026-07-21 11:46:35
status: wip
prompt: '[202607/prompts/models_panel_default_effort_controls.md](prompts/models_panel_default_effort_controls.md)'
---

# Plan: Models panel default-effort controls

## Background

The Models panel already displays the validated `llm_provider.default_effort` above the alias inventory and explains
effort suffixes carried by aliases and selector members. Persistent model-alias edits use the Rust-backed config edit
planner, preserve YAML formatting, remap home-managed files into the chezmoi source tree when `use_chezmoi` is enabled,
apply the written source, and offer a tracked commit/push. Temporary model-alias overrides use the shared duration and
absolute-time panels and are stored machine-wide.

Default effort is still read-only from this surface. Users must leave ACE to change it, cannot apply a short-lived
default-effort experiment, and have no panel workflow for seeing the configured value underneath such an experiment.
This change should make default effort a first-class global control without weakening the existing precedence rules or
adding work to row-navigation paths.

This is a **tale**: the backend contract, config edit, and TUI form one cohesive feature that one coding agent can
implement and verify end to end. It does cross the Rust-core boundary, but it does not benefit from independently landed
phases.

## Product and interaction design

### Entry point and panel status

- Add `Ctrl+E` to `ModelsPanel.BINDINGS` as a panel-local `manage_default_effort` action. It is unused in the Models
  modal today and does not conflict with `e`, which remains the selected alias's persistent model edit. This is a direct
  modal binding, not a new leader-mode setting, so the configurable `,m` entry point remains unchanged.
- Add `Ctrl+E Effort` to both footer variants, including when a bucket row is selected, because default effort is global
  rather than alias-specific. Keep `o/x/e/r` scoped to concrete alias rows exactly as they are now.
- Make the title's second line authoritative for the launch default:
  - no configured or temporary value: `default effort: provider default`;
  - configured only: `default effort: @ xhigh`;
  - active temporary value: `default effort: @ medium  override · 42m left`, followed by a restrained
    `configured @ xhigh` annotation (or `configured: provider default`).
- Derive the title, action chooser, alias effort-provenance text, and write notifications from one snapshot containing
  the configured default and active temporary override. An alias/member effort remains higher precedence, so its
  description compares against the current effective default, not stale config. Locally treat a captured override as
  inactive as soon as its expiry is reached; a small snapshot-only clock refresh may update the countdown without disk
  access or list reconstruction.

### Edit-or-override chooser

`Ctrl+E` opens a compact, centered **Default Effort** action panel. It is deliberately a choice card rather than another
navigable list, so every action is one keypress:

- A prominent status block says `Current for new launches` and shows either the active temporary `@level` plus remaining
  time, the configured `@level`, or `provider default`. When an override is active, show the underlying configured value
  separately so editing behavior is unsurprising.
- `e  Edit permanently` explains that it writes the user `sase.yml`; when chezmoi is enabled, label the destination as
  the chezmoi source.
- `o  Override temporarily` explains that it leaves configuration unchanged and will ask for a duration.
- When and only when a temporary effort override is active, show `x  Clear temporary override` as a third one-key
  action.
- `Esc`/`q` cancels back to the Models panel. A concise note states that explicit prompt effort and alias/member effort
  still win and that already-running agents are unaffected.

Use the visual language of the existing duration cards: double accent border, calm surface, aligned shortcut column, a
violet effort chip, and text labels in addition to color. Keep it legible and unclipped at the repository's supported
narrow terminal size.

### Effort-level picker

Both Edit and Override open the same new **Default Effort Level** card, with a mode-specific title/subtitle. Generate
its rows from the canonical `EFFORT_LEVELS_ORDERED` vocabulary so parser, schema, backend, and TUI spelling cannot
drift. Present the ordered ladder as single-key choices:

```text
1 none   2 minimal   3 low   4 medium   5 high   6 xhigh   7 max
```

Render these as aligned vertical rows rather than one dense line, with a subtle low-to-high intensity treatment and
explicit `current`, `configured`, or `override` annotations where applicable. Color is supplementary, never the only
state signal. Include one short best-effort note: providers that cannot honor a config-derived level keep their provider
behavior.

Edit mode also offers `0  Provider default`, which writes the schema's empty sentinel so it reliably masks any
lower-precedence configured value. Override mode does not offer that pseudo-level; cancelling or clearing is the honest
way to use the configured/provider default. `Esc` returns without changing state.

### Persistent Edit flow

After an Edit level is chosen, open a preview/confirm panel consistent with the existing model-alias preview. It must
show:

- `llm_provider.default_effort` and the chosen value (`provider default` for the empty sentinel);
- the actual target path, with a clear chezmoi-source annotation when remapped;
- configured before/after values and the source-preserving YAML diff;
- validation errors, no-op handling, and—if a temporary override is active—a note that the override will remain
  launch-effective until it expires or is cleared.

Target the writable **user base** config layer, not a project-local layer. Plan the edit with the existing Rust config
API, write only after `y`, Enter, or `Ctrl+S`, clear the config cache, and use the established `apply_chezmoi` behavior.
The actual written path must be the chezmoi source when `use_chezmoi: true`; failures to plan, validate, write, or apply
stay visible and never report success.

Extract or generalize the worker-backed scalar preview/write scaffold from the alias editor rather than creating a
second subtly different implementation. Keep alias wording and existing alias visual snapshots unchanged. After a
successful effort write, refresh the Models snapshot and offer the existing tracked commit/pull/push flow with an
effort-specific subject such as `chore: update default model effort`. Skip the offer gracefully when the written file is
not dirty inside a Git repository.

### Temporary Override flow

After an Override level is chosen, push the exact existing `DurationPickerModal`; do not copy its presets or styling.
Preserve all current model-override duration behavior: `15m`, `30m`, `1h`, `2h`, `4h`, until cleared, custom combined
durations, and `t` for the shared absolute local-time panel. Back/cancel behavior should match model aliases, including
returning from the absolute-time panel to the duration picker.

Set, replace, and clear effort overrides in worker threads. While a state write is active, suppress duplicate
submissions and prevent the parent panel from closing as if the write had finished. On success, refresh the effort
snapshot without moving the selected alias and notify with the level and compact expiry; on failure, preserve the
previous snapshot and show the actionable error.

## Backend state and launch semantics

Temporary effort override behavior is shared domain state, so implement it in the linked `sase-core` repository and
expose it through `sase_core_rs`. Python should contain only typed facade/adapter code and Textual orchestration.

Use a separate versioned state file, `~/.sase/llm_effort_override.json`, rather than teaching Rust and the legacy Python
model-override implementation to concurrently rewrite the same JSON envelope. A record contains the canonical effort,
`created_at`, optional `expires_at`, and `source`. The Rust store must:

- validate effort with the core canonical vocabulary and reject invalid, non-finite, or non-future duration/timestamp
  inputs;
- serialize writers with a bounded advisory lock, write atomically with a same-directory temporary file and rename, and
  avoid exposing partial JSON;
- ignore and self-clean malformed or expired state, with expiry occurring at `now >= expires_at` and clear remaining
  idempotent;
- accept/inject the SASE home and clock at its domain seam so Rust tests are deterministic and `$SASE_HOME` continues to
  work through the Python facade;
- publish get, set-relative, set-until, and clear bindings with a stable wire record, registering them in the extension
  and testing the binding surface.

Extend the Rust effort domain with an effective-effort resolution result/source so precedence is specified once.
Preserve the public Python `resolve_effective_effort(...)->(level, explicit)` contract through a thin adapter, but
resolve new launches in this order:

1. explicit `%effort` / model `@effort` from the prompt (`explicit=True`);
2. effort on the resolved alias or selected selector member;
3. active temporary default-effort override;
4. configured `llm_provider.default_effort`;
5. no value, allowing the provider's own default.

The temporary value is config-derived (`explicit=False`) and therefore keeps the existing best-effort provider behavior.
It affects every new launch path that already calls `resolve_effective_effort`, including delegated/workflow steps and
stored launch metadata, but never overrides prompt/alias effort and never mutates already-running agents. Keep the
existing `default_reasoning_effort()` configured-value accessor compatible; introduce a separate typed effective-default
snapshot rather than silently changing its meaning.

## TUI structure and performance

- Put effort-specific result types, render helpers, and the action/level cards in focused Models-panel modules; keep
  `models_panel.py` as the stable facade and monkeypatch surface, following the current display/override/edit split.
- Read configured and temporary effort once per Models snapshot and pass the values into pure render helpers. Do not
  read/stat/lock state from title, footer, row rendering, highlight handlers, or countdown callbacks.
- Reuse the current OptionList highlight guard and preserve the highlighted row across effort refreshes. The global
  `Ctrl+E` action must work from top-level aliases, collapsed buckets, and drilled-in buckets without changing focus.
- Perform config planning, YAML/disk writes, chezmoi application, Git discovery, and temporary-state mutations off the
  Textual event loop. Reuse the tracked task queue for commit/push and the existing pump-safe patterns for any refresh
  launched by a timer or callback.
- Cancel owned workers on unmount and re-check mount state before applying results. Bound lock waits in Rust and surface
  contention as a warning/error rather than freezing key handling.

No new always-visible top-bar pill is required in this tale; the Models header and chooser are the detailed,
discoverable authority, without consuming more horizontal space in ACE's global chrome.

## Testing and visual acceptance

Add focused coverage at every seam:

1. **Rust core and binding** — every canonical level; invalid values; relative, exact, no-expiry, boundary-expiry,
   replacement, clear, malformed-state cleanup, atomic/locked access, deterministic SASE-home paths; precedence and
   source for explicit/alias/temporary/configured/provider-default cases; PyO3 round-trip and stale/missing-field
   rejection.
2. **Python launch integration** — facade rehydration and errors; temporary effort reaches normal, delegated, and
   workflow launch metadata/CLI options; prompt and alias effort win; expiry falls back to configured/provider default;
   the override remains non-explicit so unsupported providers retain best-effort behavior.
3. **Persistent edit** — user-base targeting, all levels and empty sentinel, effective preview/diff, comments and
   key-order preservation, no-op and validation errors, config-cache refresh, `use_chezmoi` source remapping and scoped
   apply, apply failure, non-Git skip, and effort-specific commit offer.
4. **Mounted TUI flows** — `Ctrl+E` binding/footer in every bucket state; accurate chooser status with/without an
   override; conditional clear; all picker shortcuts sourced from the canonical vocabulary; Edit preview and
   active-override warning; Override → shared duration/exact-time flow; duplicate/busy guards, worker success/error,
   expiry rendering, cursor preservation, cancel/back, and clean shutdown.
5. **PNG snapshots** — inspect and accept intentional 120×40 goldens for the Models title with an active effort
   override, the action chooser showing current plus configured values, both level-picker modes, and the chezmoi-backed
   edit preview. Retain the existing duration golden to prove the shared panel did not visually fork, and add
   narrow-layout assertions so the new cards do not clip.

Run focused Rust and Python suites while iterating. Before handoff, format and test the linked Rust workspace
(`cargo fmt --check`, strict clippy, and `cargo test --workspace`), rebuild/install the editable Rust binding used by
the Python checkout, run the Models-panel unit and complete visual snapshot suites, visually inspect the new PNGs, and
finish with the repository-required `just install` followed by `just check` in the SASE checkout.

## Documentation and acceptance criteria

Update `docs/ace.md` and `docs/llms.md` with the `Ctrl+E` workflow, the two single-key cards, provider-default option,
exact shared duration behavior, clear path, precedence, best-effort semantics, state-file location, chezmoi write/commit
behavior, and the fact that persistent edits do not displace an active temporary override. Update
configuration/keybinding documentation only to identify `Ctrl+E` as a fixed Models-panel binding; do not add it to the
leader keymap schema.

The feature is complete when a user can open `,m`, press `Ctrl+E`, understand the exact current and configured default
at a glance, select Edit or Override with one key, select every canonical effort with one key, safely write the user or
chezmoi source after a truthful preview, reuse the full model-alias duration experience for a machine-wide temporary
override, clear that override from the same chooser, and observe launch behavior matching the documented precedence
without UI-thread stalls or visual regressions.
