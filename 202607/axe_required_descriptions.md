---
tier: epic
title: Require descriptions for every AXE lumberjack and chop
goal: 'Every configured lumberjack and every chop carries a non-blank `description`,
  enforced fail-closed by the shared config authority, supplied by all first-party
  and user-owned configs, and shown prominently in a dedicated always-visible banner
  on the ACE AXE tab whenever a lumberjack or chop row is selected.

  '
phases:
- id: core_description_support
  title: Rust core accepts lumberjack descriptions and can require them
  depends_on: []
  size: medium
  description: '''Phase 1 — Rust core description support'' section: teach sase_core''s
    AXE validator about `lumberjacks.<name>.description`, add non-blank validation
    for both lumberjack and chop descriptions, and implement an opt-in `require_descriptions`
    flag on the validation, compose, and entry-mutation wires that defaults to false
    so the release stays backwards compatible.

    '
- id: sase_optional_description
  title: Plumb optional descriptions through sase and describe the builtin lumberjacks
  depends_on:
  - core_description_support
  size: medium
  description: '''Phase 2 — Optional descriptions in sase'' section: bump the sase-core-rs
    window, add `description` to LumberjackConfig and the lumberjack JSON schema as
    an optional field, parse it, expose it on the AXE display snapshots, and give
    all five builtin lumberjacks in default_config.yml a description. No enforcement
    yet.

    '
- id: external_configs
  title: Describe every lumberjack and chop in chezmoi and bugyi-chops
  depends_on:
  - sase_optional_description
  size: small
  description: '''Phase 3 — External and user-owned configs'' section: add descriptions
    to every lumberjack and chop configured in the chezmoi repo''s sase.yml and sase_athena.yml,
    and to the YAML examples in the bugyi-chops README, so the user''s machines stay
    valid once enforcement is switched on.

    '
- id: axe_tab_banner
  title: AXE tab description banner
  depends_on:
  - sase_optional_description
  size: medium
  description: '''Phase 4 — AXE tab description banner'' section: add an always-visible
    AxeDescriptionBanner widget between the AXE status bar and the scrolling output
    pane that shows the selected lumberjack''s or chop''s description, wire it through
    the dashboard update paths, style it, and cover it with unit and PNG snapshot
    tests.

    '
- id: cli_surfaces
  title: Surface descriptions in the AXE CLI listings
  depends_on:
  - sase_optional_description
  size: small
  description: '''Phase 5 — CLI surfaces'' section: print the lumberjack description
    in `sase axe lumberjack list` and promote the chop Description column in `sase
    axe chop list` from verbose-only to always shown.

    '
- id: enforce_required
  title: Enforce required descriptions and document the contract
  depends_on:
  - external_configs
  - cli_surfaces
  size: medium
  description: '''Phase 6 — Enforce required descriptions'' section: flip `require_descriptions`
    on in the sase compose and mutation requests, mark description required in the
    JSON schema, drop the LumberjackConfig default, update every in-repo config fixture
    and test, and document the new contract.

    '
create_time: 2026-07-26 08:53:13
status: done
bead_id: sase-9t
---

- **PROMPT:** [prompts/202607/axe_required_descriptions.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/axe_required_descriptions.md)
- **BEAD:** [sase-9t](https://github.com/sase-org/sase--beads/blob/main/pages/sase-9t/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-9t.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-9t.land/README.md)
  - [bbugyi200.athena.sase-9t.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-9t.land.md#member-code)
- **COMMITS:**
  - [20c131b](https://github.com/sase-org/sase/commit/20c131b55788ae5d07ea32fcae66328c48e748ab) — build(deps): require sase-core-rs 0.10 (sase-9t)

# Plan: Require descriptions for every AXE lumberjack and chop

## Goal

Make `description` a required, non-blank field on every configured AXE lumberjack and every configured chop, enforced by
the single shared config authority (the Rust core), supplied everywhere descriptions are configured today, and surfaced
prominently in the ACE AXE tab whenever a lumberjack or chop row is selected.

## Current state

- `ChopConfig.description` already exists (`src/sase/axe/_config_types.py`) and is a required dataclass field, but the
  config layer treats it as optional: `crates/sase_core/src/axe_chop/config.rs` only type-checks it, and
  `src/sase/config/sase.schema.json` lists it without a `required` entry.
- Lumberjacks have **no** `description` concept at all. `LUMBERJACK_KEYS` in `axe_chop/config.rs` is
  `["interval", "chop_timeout", "chops", "env"]`, so adding the key to a config today produces an `unknown_key`
  diagnostic and fails the config closed.
- All builtin chops in `src/sase/default_config.yml` already have descriptions; none of the five builtin lumberjacks
  (`hooks`, `waits`, `checks`, `comments`, `housekeeping`) do.
- The chezmoi repo configures four lumberjacks with no descriptions (`telegram` in `sase.yml`; `run_every`,
  `code_quality`, `refresh_docs` in `sase_athena.yml`). Every chop under them already has a description.
- The `bugyi-chops` README documents two example lumberjacks (`maintenance`, `audits`) and two chops
  (`recent_bug_audit`, `recent_improvement_audit`) without descriptions.
- `sase-telegram`, `sase-github`, and `sase-nvim` ship **no** AXE config; the telegram chops are configured from
  chezmoi. No plugin repo work is needed beyond that.
- On the AXE tab, a chop description is currently rendered only in one place: the "no runs recorded for this chop yet"
  empty state in `AxeDashboard.update_chop_run_display`. It is invisible for every chop that has run at least once, and
  a lumberjack has nothing to show at all.

## Design decisions

These are settled; implementing agents should not re-litigate them.

### 1. The config authority owns enforcement

The Rust `validate_axe_config` in `crates/sase_core/src/axe_chop/config.rs` is the single fail-closed authority for AXE
config. Requiredness is enforced there, producing a `required_missing` diagnostic that flows through `AxeConfigError` to
`sase axe`, `sase doctor`, and the ACE AXE tab's degraded-status bar (which already renders the first diagnostic via
`_invalid_config_status` in `src/sase/ace/tui/actions/axe_display/_data.py`). The bundled JSON schema
(`sase.schema.json`) gets a matching `required` entry so the Config Center and the AXE entry editor agree.

### 2. Requiredness is checked on the merged config only, never per layer

`compose_values` in `crates/sase_core/src/config/axe.rs` validates **each raw layer** and then the **merged** result. A
sparse overlay that only sets `axe.lumberjacks.hooks.interval` must stay legal, so the required-field check must run
only on the merged pass. This is why the check is gated behind an explicit request flag rather than being unconditional.
The same reasoning applies to `_validate_target_config` in `src/sase/axe/_config_targets.py`, which validates a
synthesized partial config for `for_each` target overrides — it keeps the default (not required).

The bundled JSON schema is only ever applied to merged/effective configs (`config/plan.rs`, `config/provenance.rs`,
`config/axe.rs:291`), so adding `required` there is safe for sparse layers as well.

### 3. The flip lives in Python, so the core release stays backwards compatible

`require_descriptions` defaults to `false` on the wire. The sase repo turns it on by adding the key to the request dicts
built in `src/sase/axe/config_backend.py` (`compose_axe_config` and `plan_axe_entry_edit`). This matters because
`pyproject.toml` pins `sase-core-rs>=0.9.1,<0.10.0`: a core release inside that window is picked up by an unchanged sase
install. If the flip lived in Rust, publishing the core would immediately break every existing user config. With the
flag defaulting off, phases can land in a safe order: core → sase plumbing → external configs → enforcement.

### 4. Bare-string list-form chops become invalid

`chops: [hook_checks, mentor_checks]` cannot carry a description, so it stops being valid once enforcement lands. The
diagnostic must say so explicitly (see the message text in Phase 1). Map form is already the documented composable form;
this change makes it the only form for new configuration.

### 5. Prominent display = a dedicated, non-scrolling banner

The AXE tab detail pane is `AxeDashboard` → `AxeStatusSection` (one line, `max-height: 1`) + `VerticalScroll` →
`AxeOutputSection`. A new `AxeDescriptionBanner` sits **between** them, so the description is always visible and never
scrolls away, and it repaints on every selection change through the existing dashboard update methods.

Alternatives considered and rejected:

- **Sidebar (`BgCmdList`) rows.** The sidebar auto-sizes its container to the widest row
  (`WidthChanged`/`_WIDTH_PADDING`), so descriptions there would balloon the panel width. Rejected.
- **`AxeInfoPanel` top bar.** Fixed at `height: 2` and already carrying the position counter, refresh countdown, and
  help-guide hint. No room. Rejected.
- **A description column or sub-line in the lumberjack overview CHOPS table.** The wide table is a fixed 68-column
  layout; a fifth column does not fit, and a wrapped sub-line per chop doubles the table height and hurts scanability.
  The user's requirement is about the **selected** entity, which the banner answers directly. Rejected.

### 6. No `{target.*}` templating in descriptions

Generated `for_each` instances inherit the base chop's description verbatim (they already do, via `_deep_patch` in
`_config_targets.py`). Templating descriptions is deliberately out of scope; the banner instead appends a dim target
chip for generated instances, which conveys the same information without a new failure mode.

### 7. Cross-phase rule

From Phase 2 onward, **every new or edited AXE config fixture, YAML example, or `LumberjackConfig` / `ChopConfig`
construction must include a meaningful description.** Phase 6 turns enforcement on; any fixture added without one after
that point fails the suite.

### Description style guide

Use these rules everywhere descriptions are authored in this epic:

- One sentence, no trailing period, sentence case.
- Present tense, active voice, describing what the entity does — not what it is. Good:
  `Complete finished hooks and start stale ones, with zombie detection`.
- Lumberjack descriptions describe the **lane**: its cadence and the class of work it owns. Example for `hooks`:
  `Fast lane that advances hook, mentor, and workflow lifecycle state every few seconds`.
- Aim for under 100 characters so the banner stays on one line at typical terminal widths.

---

## Phase 1 — Rust core description support

Repo: `sase-core` (open with `/sase_repo`; do not edit it any other way).

### Changes

1. `crates/sase_core/src/axe_chop/config.rs`
   - Add `"description"` to `LUMBERJACK_KEYS`.
   - In `validate_lumberjacks`, validate `description` when present: it must be a string and must not be blank
     (reuse/extend the existing `validate_nonblank_string` helper used for chop `name`/`script`).
   - In `validate_chop_config`, upgrade the existing `description` check from "optional string" to "optional non-blank
     string".
   - Add a `require_descriptions: bool` field to `AxeConfigValidationRequestWire`
     (`crates/sase_core/src/axe_chop/wire.rs`) with `#[serde(default)]` so it deserializes to `false` for existing
     callers. Thread it through `validate_axe_config` into `validate_lumberjacks` / `validate_chop_config`.
   - When `require_descriptions` is true and a lumberjack or chop has no `description` key, emit:
     - code: `required_missing`
     - path: `axe.lumberjacks.<name>.description` or `axe.lumberjacks.<name>.chops.<chop>.description`
     - lumberjack message: ``lumberjack `<name>`requires a non-empty`description```
     - chop message:
       ``chop `<name>` requires a non-empty `description`; list-form string entries cannot carry one, so use the map form``
   - Blank-but-present descriptions keep the existing `blank_value` code; only _absent_ ones are `required_missing`.

2. `crates/sase_core/src/config/axe.rs`
   - Add `require_descriptions: bool` (with `#[serde(default)]`) to `AxeConfigComposeRequestWire` and
     `AxeEntryMutationRequestWire`.
   - In `compose_values`, pass `require_descriptions: false` for the **per-layer** validation request and the
     request-supplied value for the **final merged** request. `compose_values` will need the flag threaded in from
     `compose_axe_config`.
   - In `plan_axe_entry_mutation`, propagate the request's flag into both internal `compose_axe_config` calls (the
     `original` composition and the `candidate` composition) so the AXE entry editor's preview reports the same
     diagnostics the runtime will.

3. Tests
   - `crates/sase_core/src/axe_chop/tests.rs`: lumberjack `description` accepted; blank lumberjack/chop description
     rejected; `require_descriptions: false` (the default) accepts a description-less config;
     `require_descriptions: true` produces exactly the expected `required_missing` diagnostics for a missing lumberjack
     description, a missing map-form chop description, and a bare-string list-form chop.
   - `crates/sase_core/tests/config_parity.rs`: a compose case proving a sparse overlay that only overrides `interval`
     produces no `required_missing` diagnostic when the base layer supplies the description, plus an entry-mutation case
     proving the flag reaches `plan_axe_entry_mutation`'s `axe_diagnostics`.

4. `crates/sase_core_py/src/lib.rs`: no signature change is needed (the bindings are dict-based and serde fills the
   default), but add a binding-level test asserting the new key round-trips, and refresh the module doc comment if it
   enumerates the request shape.

5. Bump the workspace version in `Cargo.toml` and release so the sase repo can depend on it.

### Acceptance

- `cargo test` passes in `sase-core`.
- A config with no descriptions anywhere still validates clean when `require_descriptions` is absent/false.
- The same config produces precise `required_missing` diagnostics when the flag is true.

---

## Phase 2 — Optional descriptions in sase

Repo: `sase`. Run `just install` before `just check` (ephemeral workspaces drift).

### Changes

1. `pyproject.toml`: bump the `sase-core-rs` window to require the Phase 1 release.

2. `src/sase/core/axe_chop_facade.py`: add a `require_descriptions: bool = False` keyword to `validate_axe_config` and
   include it in the request payload.

3. `src/sase/axe/_config_types.py`: add `description: str = ""` to `LumberjackConfig`, immediately after `name`. (Phase
   6 removes the default.)

4. `src/sase/axe/_config_targets.py`: in `parse_lumberjacks`, read `description` from the raw lumberjack mapping and
   pass it to `LumberjackConfig`. Leave `_validate_target_config` on the default (non-requiring) validation scope — it
   validates a synthesized partial config and must not demand descriptions.

5. `src/sase/config/sase.schema.json`: add an optional `description` property to the lumberjack object
   (`axe.lumberjacks.additionalProperties.properties`) with a helpful `description` string of its own. Do **not** add
   `required` yet.

6. `src/sase/default_config.yml`: add a `description` to each of the five builtin lumberjacks, following the style
   guide. Suggested text (adjust for accuracy while implementing):
   - `hooks`: `Fast lane that advances hook, mentor, and workflow lifecycle state every few seconds`
   - `waits`: `Resolve agent wait dependencies and keep bead claims and stores in sync`
   - `checks`: `Poll slower PR-submission and workspace-claim checks on a five-minute cadence`
   - `comments`: `Start background critique-comment checks for mailed PRs every minute`
   - `housekeeping`: `Hourly error digest and managed-temp cleanup`

7. `src/sase/ace/tui/actions/axe_display/_data.py`: add `description: str = ""` to `LumberjackSnapshot` and populate it
   from `config.lumberjacks[name].description` in `collect_axe_status_data`. This keeps the AXE navigation path
   cache-only (no new disk I/O — see the TUI performance rules).

8. `src/sase/ace/tui/actions/axe_display/_render.py`: include `description=""` in the `LumberjackSnapshot` cache-miss
   fallback construction.

9. `src/sase/ace/tui/modals/axe_entry_editor_types.py`: put `description` first in `_BASICS_BY_KIND` for both kinds:
   - `"lumberjack": ("description", "interval", "chop_timeout")`
   - `"chop": ("description", "script", "enabled", "run_every", "timeout")` The AXE entry sheet is schema-driven, so it
     picks up requiredness automatically in Phase 6; ordering it first now makes the editor read correctly in both
     phases.

10. `src/sase/axe/cli.py`: the `_oneshot` fallback constructs `ChopConfig(name=chop_name, description="")`. Give it a
    real synthetic description, e.g. `f"Manual one-shot run of {chop_name}"`, so no surface ever renders a blank one.

11. Tests: update any `LumberjackConfig(...)` construction that needs it (none should break, since the field has a
    default in this phase), and add coverage that `parse_lumberjacks` round-trips a lumberjack description and that
    `LumberjackSnapshot.description` is populated by the collector.

### Acceptance

- `just check` passes.
- `sase axe lumberjack list` still works with the new builtin descriptions present.
- Adding `description:` to a lumberjack in any config layer no longer produces an `unknown_key` diagnostic.

---

## Phase 3 — External and user-owned configs

Open each repo with `/sase_repo` and use only the printed path.

### chezmoi

`sase repo open chezmoi -r "..."`, then:

1. `home/dot_config/sase/sase.yml` — add a `description` to the `telegram` lumberjack. Its two chops (`tg_inbound`,
   `tg_outbound`) already have descriptions; verify and leave them alone.
2. `home/dot_config/sase/sase_athena.yml` — add a `description` to the `run_every`, `code_quality`, and `refresh_docs`
   lumberjacks. All chops under them (`toobig_split`, `recent_bug_audit`, `recent_improvement_audit`, `refresh_docs`)
   already have descriptions; verify and leave them alone.

Re-read both files after editing and confirm **every** lumberjack key and **every** chop key under `axe.lumberjacks` has
a non-blank `description`. This is the file set that decides whether the user's daemon survives Phase 6.

### bugyi-chops

`sase repo open gh:bbugyi200/bugyi-chops -r "..."`, then update `README.md`: the two YAML examples define lumberjacks
`maintenance` and `audits` (no descriptions) and chops `recent_bug_audit` / `recent_improvement_audit` (no
descriptions). Add descriptions to all four so the documented examples are valid under the new contract. The
`toobig_split` example already has a chop description; it still needs one on its `maintenance` lumberjack.

### Acceptance

- `sase axe lumberjack list` on the user's machine shows every lumberjack with a description.
- `grep`ing the chezmoi AXE config for lumberjack and chop keys finds a `description` for each.

---

## Phase 4 — AXE tab description banner

Repo: `sase`. Depends only on Phase 2's `LumberjackSnapshot.description`.

### The widget

New file `src/sase/ace/tui/widgets/axe_description_banner.py` defining `AxeDescriptionBanner(Static)` and exported from
`src/sase/ace/tui/widgets/__init__.py`.

API (keep it declarative so it is unit-testable without mounting a full app):

- `show_lumberjack(name: str, description: str) -> None`
- `show_chop(chop_name: str, description: str, *, generated: bool = False, target_key: str | None = None) -> None`
- `hide() -> None` — sets `self.display = False` for bgcmd views and the zero-lumberjack empty state.

### Rendering

A single `rich.text.Text` with `overflow="ellipsis"`, wrapping allowed, capped at two lines by CSS:

- Leading accent glyph `▌ ` in the entity's sidebar hue — gold `bold #FFD700` for a lumberjack, copper `#D7AF87` for a
  chop — so the banner echoes the sidebar row taxonomy already established in
  `src/sase/ace/tui/widgets/_axe_dashboard_render.py` (`LJ_NAME_STYLE`, `CHOP_NAME_STYLE`) and `bgcmd_list.py`.
- The description itself in `italic #D7D7AF` — readable, not `dim`, because this is the prominent element.
- For a generated `for_each` instance, append a dim chip: `  · <target_key>` in `dim #B87333`, matching the `instance`
  badge colour already used in the sidebar.
- When the description is empty (only possible before Phase 6, or for the synthetic `_oneshot` lumberjack), render
  `No description configured` in `dim italic` rather than hiding the banner, so the row's vertical rhythm never shifts
  while navigating.

### Wiring

`src/sase/ace/tui/widgets/axe_dashboard.py`:

- `compose()` yields `AxeDescriptionBanner(id="axe-description-banner")` between `#axe-status-section` and the
  `VerticalScroll`.
- `update_lumberjack_overview` calls `show_lumberjack(snapshot.name, snapshot.description)`.
- `update_chop_run_display` calls `show_chop(...)` using `snapshot.description`, `snapshot.generated`, and
  `snapshot.target_key`.
- `update_bgcmd_display`, `update_empty_axe_display`, `update_display`, and `update_lumberjack_display` call `hide()`.

No changes are needed in `actions/axe_display/_render.py` beyond what Phase 2 already did — it routes through these
dashboard methods.

### Styling

`src/sase/ace/tui/styles.tcss`, next to the existing `#axe-status-section` block:

```
#axe-description-banner {
    height: auto;
    max-height: 2;
    padding: 0 1;
    background: $surface;
}
```

Keep `#axe-output-scroll`'s `border-top` so the banner reads as part of the header group rather than the log.

### Follow-on cleanup

In `update_chop_run_display`, the "no runs recorded for this chop yet" empty state currently re-prints the description.
Remove that duplication — the banner now owns it — and leave the empty state as the plain
`No runs recorded for this chop yet.` line.

### Tests

- Unit tests for the widget's three render modes plus the empty-description fallback, in a new
  `tests/ace/tui/test_axe_description_banner.py` (no app mount required).
- Extend `tests/ace/tui/test_axe_chop_selection.py` / `test_axe_navigation.py` coverage to assert the banner text
  follows selection between a lumberjack row, a chop row, a generated instance row, and a bgcmd row.
- PNG snapshots in `tests/ace/tui/visual/test_ace_png_snapshots_axe.py`: add `axe_lumberjack_description_120x40` and
  `axe_chop_description_120x40`. The extra banner row shifts every existing AXE-tab snapshot, so regenerate them with
  `just test-visual --sase-update-visual-snapshots` and **review each regenerated PNG** before accepting.
- Document the banner in `docs/ace.md` near the Axe tab coverage. Do not touch `docs/axe.md` (Phases 5 and 6 own it).

### Acceptance

- Selecting a lumberjack, a chop, and a generated chop instance each show the right description without scrolling.
- Navigating with `j`/`k` repaints the banner from cache only; `SASE_TUI_PERF=1` j/k p95 on the AXE tab stays under 16
  ms.
- `just test-visual` passes with reviewed snapshots.

---

## Phase 5 — CLI surfaces

Repo: `sase`.

1. `src/sase/axe/cli.py`, `handle_axe_lumberjack_list`: print the description under the lumberjack name, before
   `interval`, as `  [dim]description:[/dim] <text>`. Omit the line when the description is empty (still possible before
   Phase 6).

2. `src/sase/axe/chop_render.py`, `_configured_chops_table`: promote `Description` from a verbose-only column to a
   column that is always present for configured chops, placed directly after `Chop`. Keep the verbose-only
   `Policy / Last Decision` column where it is.

3. Update `tests/test_axe_cli.py` and `tests/test_axe_chop_inventory.py` / chop-render tests for the new output.

4. Update the `sase axe lumberjack list` and `sase axe chop list` rows in the `docs/axe.md` CLI table only if their
   one-line descriptions become inaccurate. Do not edit the `docs/axe.md` configuration sections — Phase 6 owns those.

### Acceptance

- `sase axe lumberjack list` shows each lumberjack's description.
- `sase axe chop list` shows a Description column without `--verbose`.

---

## Phase 6 — Enforce required descriptions

Repo: `sase`. This is the flip. Depends on Phases 3 and 5.

### Changes

1. `src/sase/axe/config_backend.py`: add `"require_descriptions": True` to the request dicts built in
   `compose_axe_config` and `plan_axe_entry_edit`.

2. `src/sase/config/sase.schema.json`:
   - `axeChop`: add `"required": ["description"]` and `"minLength": 1` on the `description` property.
   - The lumberjack object under `axe.lumberjacks.additionalProperties`: add `"required": ["description"]` and
     `"minLength": 1` on its `description`.
   - The inline list-form chop object (`axe.lumberjacks.*.chops.items.oneOf[1]`): change its `required` to
     `["name", "description"]` and add `"minLength": 1`, for documentation consistency. (Merged configs normalize list
     form to map form before validation, so this branch is informational.)

3. `src/sase/axe/_config_types.py`: remove the default from `LumberjackConfig.description` so it mirrors
   `ChopConfig.description`. Update the ~43 `LumberjackConfig(...)` construction sites across `src/` and `tests/` —
   `src/sase/axe/_config_targets.py` is the only production one; the rest are fixtures. Give each fixture a real, short
   description rather than `""`.

4. In-repo config fixtures: any YAML or dict fixture under `tests/` that defines `axe.lumberjacks` must gain
   descriptions, and bare-string chop lists (`"chops": ["hook_checks"]` in `tests/test_axe_lumberjack_config.py`,
   `tests/test_axe_cli.py`, `tests/test_axe_orchestrator.py`, `tests/test_axe_status_collector.py`,
   `tests/axe_chop_runner_helpers.py`, and others) must convert to map form with descriptions — except where a test
   deliberately asserts the _rejection_ of list-form string chops, which should now be added as positive coverage of the
   new diagnostic.

5. New tests:
   - `tests/test_axe_lumberjack_config.py`: a merged config missing a lumberjack description raises `AxeConfigError`
     with code `required_missing` and the exact dotted path; the same for a chop; a sparse overlay that only overrides
     `interval` does **not** raise.
   - `tests/test_config_schema_automation.py`: the schema's required entries match the runtime contract.
   - `tests/ace/tui/test_axe_entry_editor_modal.py` (or the nearest existing editor test): clearing `description` in the
     AXE entry sheet reports the schema-driven `required field must have a value` diagnostic and blocks the save.

6. Docs:
   - `docs/configuration.md`: add `description` to the **Lumberjack fields** table (Required: yes); change the **Chop
     fields** table's `description` row to Required `yes` with no default; update the `axe.lumberjacks` YAML examples
     around lines 1183 and 1306 to include lumberjack descriptions; note that bare-string chop lists are no longer
     valid.
   - `docs/axe.md`: update the Lumberjack Configuration YAML example (~line 261) and the `refresh_docs` example
     (~line 465) with lumberjack descriptions; add `description` to the Chop Fields table as required.
   - `docs/plugins.md`: update the `audits` YAML example (~line 603) with lumberjack and chop descriptions.
   - Add a CHANGELOG-worthy note that this is a breaking config change and how to fix the resulting diagnostic.

### Acceptance

- `just check` passes.
- Removing any description from `default_config.yml` fails `sase axe status`, `sase axe lumberjack list`, and the ACE
  AXE tab with a precise `[required_missing] axe.lumberjacks.<name>.description` diagnostic naming the source layer.
- The ACE AXE tab's degraded-status bar renders that diagnostic rather than crashing.
- With Phase 3 landed, the user's real machine config loads clean.

---

## Risks

- **User-machine breakage.** The phase order exists to prevent it: the core release is behavior-neutral, the user's
  chezmoi config gains descriptions before enforcement, and enforcement lands last. Do not reorder Phases 3 and 6.
- **Transient Config Center diagnostic.** Between Phases 1 and 2, a chezmoi lumberjack `description` is accepted by the
  runtime but the bundled `sase.schema.json` still has `additionalProperties: false` on the lumberjack object, so the
  Config Center may show one `additional_property` diagnostic. It does not affect `load_axe_config()` and disappears
  when Phase 2 lands. Phase 3 depends on Phase 2 specifically to keep this window closed.
- **PNG snapshot churn.** The banner shifts every AXE-tab snapshot by one row. Regenerate deliberately and review the
  images; do not blanket-accept.
- **Test-fixture volume in Phase 6.** ~43 `LumberjackConfig` sites plus several bare-string chop lists. Mechanical, but
  budget for it and prefer real descriptions over placeholders so the fixtures stay readable.
