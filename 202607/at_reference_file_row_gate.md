---
tier: epic
title: Gate `@` reference file rows behind kind misses and explicit Ctrl+T
goal: 'The grouped `@` reference menu lists local file rows only when no artifact
  kind prefix-matches the typed text, or when the user explicitly asks for them (`Ctrl+T`
  in the ACE prompt, a manually invoked completion request over LSP). The rule is
  decided once in the shared Rust core so the TUI and every LSP client agree.

  '
phases:
- id: core-gate
  title: Shared kind-stage file-row gate in sase-core
  depends_on: []
  size: medium
  description: 'core-gate: add an at-reference menu options wire plus a `files_suppressed`
    menu flag to `sase_core`, suppress Kind-stage path rows when an artifact kind
    prefix-matches the query unless the caller opts in, expose the option through
    the `sase_core_rs` binding, and map an `Invoked` LSP completion request to the
    opt-in.

    '
- id: tui-gate
  title: ACE prompt gating and the Ctrl+T reveal
  depends_on:
  - core-gate
  size: medium
  description: 'tui-gate: thread the new menu option through the Python artifact-ref
    facade and prompt completion mixins, add a per-menu "files revealed" state that
    a first `Ctrl+T` press sets instead of force-completing, surface a `[^T] files`
    panel hint, and refresh the affected tests and docs.

    '
- id: core-floor
  title: Raise the sase-core-rs floor and verify end to end
  depends_on:
  - core-gate
  - tui-gate
  size: xsmall
  description: 'core-floor: raise the published `sase-core-rs` dependency window in
    `pyproject.toml` to the release that carries the gate, then run the full repo
    check and confirm the behavior against the published wheel.

    '
create_time: 2026-07-30 07:14:59
status: wip
bead_id: sase-b4
---

- **BEAD:** [sase-b4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-b4/README.md)
- **PROMPT:** [202607/prompts/at_reference_file_row_gate.md](prompts/at_reference_file_row_gate.md)

# Gate `@` reference file rows behind kind misses and explicit Ctrl+T

## Problem

The grouped `@` reference menu in the ACE prompt input (and the equivalent LSP completion response for external editors)
always renders two groups at the Kind stage: artifact kinds first, then local file rows under a `── files · <dir>` rule.
Typing `@f` therefore lists the `file` artifact kind followed by every path in the base directory that fuzzy-matches `f`
(`Justfile`, `mkdocs-pdf.yml`, `release-please-config.json`, …). The file rows are noise whenever the typed text is
clearly naming an artifact kind, and they push the kind rows the user was aiming for off the visible window.

Desired behavior: show file rows only when

1. no known artifact kind matches the text the user has typed, **or**
2. the user explicitly asks for them (`Ctrl+T` in the prompt input; a manually invoked completion request over LSP).

## Current behavior and where it lives

Row selection is already owned by the shared Rust core, so both frontends go through one function:

- `crates/sase_core/src/editor/at_reference.rs` — `build_kind_menu()` fuzzy matches `inventory.kinds` against the query
  and `inventory.paths` against the trailing path partial, then unconditionally concatenates `artifact_rows` +
  `file_rows` into `AtReferenceMenuWire.rows`. It also reports `artifact_count` / `file_count` and picks
  `shared_extension` from the leading non-empty group.
- `crates/sase_core_py/src/lib.rs` — `at_reference_menu(context, inventory, payload_index=None)` binding.
- `crates/sase_xprompt_lsp/src/server.rs` — `at_reference_completion()` builds the inventory
  (`at_reference_kind_inventory`, `at_reference_path_inventory`, `at_reference_payload_inventory`) and renders it
  through `at_reference_completion_response()` with `is_incomplete: true`.

In the sase repo:

- `src/sase/artifact_ref_operations.py` — `at_reference_menu()` facade over the binding (re-exported from
  `src/sase/artifact_refs.py`).
- `src/sase/ace/tui/widgets/artifact_ref_completion.py` — `build_artifact_ref_completion_result()` maps menu rows onto
  `CompletionCandidate`s, tagging kind rows with `ArtifactRefKindCompletionMetadata` and path rows with
  `AtReferenceFileCompletionMetadata`.
- `src/sase/ace/tui/widgets/_file_completion_base.py` — `_artifact_ref_completion_result()` assembles the warm catalog,
  commit/bug snapshots, and the warm prompt path snapshot.
- `src/sase/ace/tui/widgets/_file_completion_open.py` — `_try_artifact_ref_completion(force=…)`; `force=True` is the
  `Ctrl+T` path and additionally accepts a lone leading-group row or inserts `shared_extension` and recurses.
- `src/sase/ace/tui/widgets/_file_completion_tab.py` — `_try_file_completion_tab()` calls
  `_try_artifact_ref_completion(force=True)`.
- `src/sase/ace/tui/widgets/_file_completion_refresh.py` — keeps the menu in sync per keystroke and re-applies the
  sticky `_artifact_ref_completion_force` behavior.
- `src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel.py` — adaptive title (`@ reference` / `@ artifact kinds`
  / `@ <dir>`), the group rule, and `_artifact_ref_completion_subtitle()`.

## Decisions

**D1 — "matches" means a case-insensitive prefix match.** The gate fires when at least one inventory kind fuzzy-matches
the Kind-stage query at **tier 0** (`FuzzyText` tier 0 is exactly "text starts with query"). Tiers 2 and 3 (substring,
subsequence) are deliberately excluded: `@sr` subsequence-matches the `research` kind, and gating on that would hide
`src/` from a user who is plainly typing a path. Prefix matching is also what "matches the text the user has typed"
means colloquially.

Consequences worth knowing:

- Any query containing `/` can never prefix-match a kind name, so directory drill-down (`@src/`, `@~/dev/`, `@../x`) is
  completely unaffected.
- The gate self-clears as the user keeps typing: `@f` … `@file` are gated by the `file` kind, `@file_` is not (no kind
  starts with `file_`).

**D1a — a bare `@` is gated (flagged for review).** An empty query matches every kind at tier 0, so `@` alone lists
artifact kinds only and the file listing moves behind `Ctrl+T`. This is the literal reading of the request and produces
the tidiest menu, but it is the single most noticeable change, so call it out during plan review. The alternative —
treat an empty query as "no kind match" so bare `@` keeps today's kinds-then-files menu — is a one-line predicate change
(`!context.query.is_empty() && kind_prefix_hit`) if that is preferred.

**D2 — the policy lives in `sase_core`.** Per the repo's Rust-core boundary rule, any behavior another frontend must
match belongs in the core. The TUI and the LSP both render `build_kind_menu()` output, so the gate is implemented there
once and both frontends only decide _whether the request is explicit_.

**D3 — `Ctrl+T` is two-stage in the prompt input.** `Ctrl+T` today force-completes (`force=True`): it accepts a lone
leading-group row and otherwise inserts the shared extension. If it merely also passed "include files", `@f` + `Ctrl+T`
would accept `@file:` outright and the file rows would still be unreachable. So: while an `@` Kind-stage menu is gated,
the next `Ctrl+T` press **reveals the file rows and does nothing else**; once revealed (or when nothing was gated),
`Ctrl+T` behaves exactly as it does today. The reveal is sticky for the life of that menu so narrowing the query keeps
the files visible.

**D4 — the LSP analog of `Ctrl+T` is `CompletionTriggerKind::INVOKED`.** The server advertises `@` as a trigger
character and returns `is_incomplete: true`, so a client sends `TRIGGER_CHARACTER` for the `@` keystroke and
`TRIGGER_FOR_INCOMPLETE_COMPLETIONS` while narrowing — both gated. A manual invocation (`<C-Space>` in nvim-cmp,
`editor.action.triggerSuggest` in VS Code) sends `INVOKED`, which opts in. A client that over-reports `INVOKED` degrades
to today's behavior, which is safe.

**D5 — every Rust API change is additive.** `pyproject.toml` pins `sase-core-rs>=…,<0.13.0`, and `sase_xprompt_lsp` /
`sase_core` are published crates, so no existing public signature may change (that would force a semver-major bump
release-plz cannot satisfy inside the pin). New behavior arrives through new functions and `#[serde(default)]` wire
fields; the existing `build_at_reference_menu*` entry points keep working and simply pass the default (gated) options.
Do not hand-edit `[workspace.package].version` in sase-core — release-plz owns it.

## Out of scope

- No new config key. A `ace.prompt_completion.at_reference_files: auto | always` escape hatch is a plausible follow-up
  but is not part of this request.
- No change to the Payload stage (`@kind:…`), which has no file rows.
- No change to path-inventory scheduling or warming. Skipping the LSP's `read_dir` when the gate is active is a valid
  follow-up optimization; it is not required for correctness and is left alone here.
- No change to the `Ctrl+R` recursive finder or to standalone (non-`@`) file completion.

---

## Phase `core-gate`: Shared kind-stage file-row gate in sase-core

Work happens in the linked `sase-core` repository. Open it with the `/sase_repo` skill
(`sase repo open sase-core -r "<reason>"`) and use the printed path as the only path for reads and writes. All paths
below are relative to that checkout.

### 1. Menu options and the suppression signal

In `crates/sase_core/src/editor/at_reference.rs`:

```rust
/// Caller policy for one at-reference menu build.
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq, Serialize, Deserialize)]
#[serde(default)]
pub struct AtReferenceMenuOptionsWire {
    /// Include Kind-stage local path rows even when an artifact kind
    /// prefix-matches the query. Set by an explicit user request.
    pub include_files: bool,
}
```

Add one field to `AtReferenceMenuWire`:

```rust
    /// Kind-stage path rows existed but were withheld by the gate.
    #[serde(default)]
    pub files_suppressed: bool,
```

`#[serde(default)]` on both keeps every existing serialized dict deserializing unchanged, which the Python binding
relies on.

### 2. Gate `build_kind_menu`

Thread `options: AtReferenceMenuOptionsWire` through `build_at_reference_menu_inner` into `build_kind_menu`, and:

- After the matched `kinds` vector is deduplicated, compute
  `let kind_prefix_hit = kinds.iter().any(|(_, m)| m.tier == 0);` (tier 0 is the prefix tier, and an empty query yields
  tier 0 for every kind — see decision D1a).
- `let gate_files = kind_prefix_hit && !options.include_files;`
- Keep computing the matched `paths` vector exactly as today. When `gate_files` is true, emit no file rows, report
  `file_count: 0`, and set `files_suppressed: !paths.is_empty()` — the flag must be false when there was nothing to
  withhold, so a frontend never advertises an empty reveal.
- Leave `shared_extension` selection as-is (`artifact_count == 0` implies the gate is off, because the gate requires at
  least one tier-0 kind match).
- `build_payload_menu` sets `files_suppressed: false`.

Keep `build_at_reference_menu` and `build_at_reference_menu_with_payload_index` as thin wrappers that pass
`AtReferenceMenuOptionsWire::default()`, and add one new entry point:

```rust
pub fn build_at_reference_menu_with_options(
    context: &AtReferenceContextWire,
    inventory: &AtReferenceInventoryWire,
    payload_index: Option<&AtReferencePayloadIndex>,
    options: AtReferenceMenuOptionsWire,
) -> AtReferenceMenuWire
```

Re-export `AtReferenceMenuOptionsWire` and the new builder from `crates/sase_core/src/editor/mod.rs` and
`crates/sase_core/src/lib.rs` (following the existing `editor_build_at_reference_menu*` aliasing convention).

`crates/sase_core/src/editor/completion.rs` builds kind-only and payload-only inventories (`paths` is always empty
there), so its two `build_at_reference_menu` calls need no change.

### 3. Rust tests

Existing tests in the `at_reference.rs` test module assert the old mixed ordering and must be updated to reflect the
gate:

- `bare_menu_groups_builtins_then_sorted_kinds_and_visible_paths` — keep the mixed-order assertion by building with
  `include_files: true`, and add a companion assertion that the default build returns kind rows only with
  `files_suppressed == true` and `file_count == 0`.
- `kind_and_file_groups_filter_independently` — the `@pl` case now gates (`plan` prefix-matches); assert the gated rows
  by default and the mixed rows with `include_files: true`. The `@src/` case is unaffected.
- `caps_each_group_but_records_pre_cap_counts` — pass `include_files: true` to keep exercising both group caps.
- `dotfile_visibility_tracks_the_trailing_partial` (empty `kinds`) and
  `shared_extension_uses_only_the_leading_non_empty_group` need no behavior change; re-verify them.

New tests:

- A tier-0 kind match gates file rows; the same query with `include_files: true` returns both groups.
- A query that only fuzzy/substring-matches a kind (e.g. `@rsch` or `@sr` against a `research` kind) is **not** gated,
  so file rows still appear.
- A query containing `/` is never gated.
- `files_suppressed` is false when the gate is active but no path row matched.
- `shared_extension` still comes from the kind group while gated.

### 4. Python binding

In `crates/sase_core_py/src/lib.rs`, extend `py_at_reference_menu` to
`#[pyo3(signature = (context, inventory, payload_index = None, options = None))]` with
`options: Option<Bound<'_, PyDict>>` deserialized into `AtReferenceMenuOptionsWire` (`None` → `Default::default()`), and
dispatch to `editor_build_at_reference_menu_with_options`. Update the binding inventory comment block at the top of that
file (the `at_reference_menu(...)` line).

### 5. LSP wiring

In `crates/sase_xprompt_lsp/src/server.rs`:

- Give `at_reference_completion()` an `options: AtReferenceMenuOptionsWire` parameter and build through
  `editor_build_at_reference_menu_with_options`.
- Add
  `pub async fn completion_for_text_with_trigger(&self, text: String, position: Position, trigger: Option<CompletionTriggerKind>)`
  carrying the current body, and keep `completion_for_text` as a delegating wrapper that passes `None` (D5: no existing
  public signature changes).
- `include_files` is `trigger == Some(CompletionTriggerKind::INVOKED)`.
- In the `completion()` LSP handler, pass `params.context.map(|context| context.trigger_kind)`.

Tests in that crate: a `@`-triggered request (`TRIGGER_CHARACTER`) returns artifact-kind items only when a kind
prefix-matches; the same position with `INVOKED` also returns `CompletionItemKind::FILE` / `FOLDER` items; a
non-matching query returns file items under either trigger kind.

### 6. Verification

Run the sase-core repo's own checks (`cargo fmt`, `cargo clippy`, `cargo test` per that repo's `AGENTS.md` / CI
configuration). Land the change as a non-breaking `feat:` commit so release-plz cuts a `0.12.x` release; record the
released version for phase `core-floor`.

---

## Phase `tui-gate`: ACE prompt gating and the Ctrl+T reveal

Work happens in the sase repo. Run `just install` first (workspace virtualenvs are ephemeral, and this build compiles
`sase_core_rs` from the linked `sase-core` checkout, so it picks up phase `core-gate` without waiting for a published
wheel).

### 1. Facade and result plumbing

- `src/sase/artifact_ref_operations.py` — `at_reference_menu()` gains a keyword
  `options: Mapping[str, Any] | None = None` forwarded to the binding.
- `src/sase/ace/tui/widgets/artifact_ref_completion.py`:
  - `build_artifact_ref_completion_result()` gains `include_files: bool = False` and passes
    `options={"include_files": include_files}`.
  - `ArtifactRefCompletionResult` gains `files_suppressed: bool = False`, read from
    `menu.get("files_suppressed", False)`.
  - The `paths_loading` placeholder branch stays as-is; it only fires when there are no candidates at all, which cannot
    happen while the gate is active.

### 2. Per-menu reveal state

- `src/sase/ace/tui/widgets/prompt_text_area.py` — initialize `self._artifact_ref_files_revealed: bool = False` and
  `self._artifact_ref_files_suppressed: bool = False` alongside `_artifact_ref_completion_force`.
- `src/sase/ace/tui/widgets/_file_completion_base.py`:
  - `_artifact_ref_completion_result()` passes `include_files=self._artifact_ref_files_revealed`, so the open, refresh,
    and accept paths all agree without extra call-site changes.
  - `_clear_file_completion()` resets both new flags.
  - `_update_file_completion_panel()` forwards `artifact_ref_files_suppressed=self._artifact_ref_files_suppressed` to
    `bar.show_file_completions(...)`.
- `src/sase/ace/tui/widgets/_file_completion_open.py` and `_file_completion_refresh.py` — wherever
  `_artifact_ref_completion_stats` is assigned from a result, also store `result.files_suppressed`.

### 3. The two-stage `Ctrl+T`

In `src/sase/ace/tui/widgets/_file_completion_tab.py`, replace the
`if self._try_artifact_ref_completion(force=True): return True` call site with a reveal-aware helper (implemented next
to the other artifact helpers in `_file_completion_open.py` to keep the tab dispatcher readable):

- If there is an `@` context, its stage is `kind`, `_artifact_ref_files_revealed` is `False`, and the freshly built
  result reports `files_suppressed`, then set `_artifact_ref_files_revealed = True`, re-open the menu **without**
  `force` (so no lone-match acceptance and no shared-extension insertion happens), and return `True`.
- Otherwise fall through to today's `_try_artifact_ref_completion(force=True)`.

Because `_refresh_file_completion_from_cursor()` rebuilds through `_artifact_ref_completion_result()`, the revealed
state survives further typing until the menu closes. Accepting a kind row or a directory row already calls
`_clear_file_completion()`, which resets the flag — correct in both cases (the payload stage has no file rows, and a
drilled-down `@dir/` query can never be gated).

### 4. Panel affordance

In `src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel.py`, thread the new `artifact_ref_files_suppressed`
keyword into `show_file_completions()` and have `_artifact_ref_completion_subtitle()` append a dim `[^T] files` segment
when it is true, using the existing `·` separator convention. The adaptive title already degrades correctly
(`@ artifact kinds` when only kind rows are present) and the `── files · <dir>` rule already only draws when both groups
exist, so neither needs changes.

### 5. Tests

`tests/ace/tui/widgets/test_artifact_ref_completion.py`:

- `test_kind_menu_filters_artifacts_and_files_through_shared_policy` — `@pl` now yields `["@plans:"]`; add an
  `include_files=True` case asserting the old three-row result.
- `test_path_query_returns_only_rows_from_the_requested_directory` — unchanged (no kind prefix-matches `src/`); keep as
  a regression guard.
- `test_bare_at_opens_artifacts_then_files` — becomes "bare `@` opens artifact kinds only", asserting no
  `AtReferenceFileCompletionMetadata` row and `files_suppressed`.
- `test_directory_accept_drills_down_and_file_accept_closes` — unchanged (`@sr` prefix-matches no kind in the fixture);
  keep as a regression guard.
- New: `Ctrl+T` on a gated menu reveals file rows without accepting the lone `file` kind; a second `Ctrl+T` then behaves
  as before (accepts the lone leading-group row).
- New: the reveal survives an additional keystroke and is dropped when the menu closes.
- New: a query no kind prefix-matches (e.g. `@zz`) lists file rows with no `Ctrl+T`.

`tests/ace/tui/widgets/test_at_reference_completion_rendering.py`: cover the `[^T] files` subtitle in the suppressed
state and its absence otherwise.

`tests/ace/tui/visual/test_ace_png_snapshots_at_reference_completion.py` builds `CompletionCandidate`s directly, so it
should be unaffected; re-run `just test-visual` to confirm, and only accept golden changes with
`--sase-update-visual-snapshots` if a deliberate change appears.

### 6. Docs

- `docs/ace.md` (grouped `@` menu paragraph, around the `auto_artifact_menu` / `── files · <base-dir>` discussion):
  describe the gate and the `Ctrl+T` reveal.
- `docs/configuration.md` (the `@` reference completion paragraph): replace "artifact kinds are shown first and local
  file rows second" with the new rule.
- `docs/editor.md` (artifact-reference rows in the LSP feature table and the "Artifact assistance is local-only"
  paragraph): document that file rows are gated and that a manually invoked completion request includes them.
- Per `src/sase/ace/CLAUDE.md`, re-check the `?` help popup: the prompt entry is
  `("Ctrl+T / Ctrl+L", "Manual completion / accept")` in `src/sase/ace/tui/modals/help_modal/binding_common.py`; update
  it only if the new two-stage behavior warrants clearer wording.

### 7. Verification

`just install && just check`, plus `just test-visual`. Manually confirm in `sase ace` that `@f` lists only artifact
kinds, `Ctrl+T` reveals the file rows, and `@src/` still drills down unchanged.

---

## Phase `core-floor`: Raise the sase-core-rs floor and verify end to end

Once the phase `core-gate` release is published:

- Raise the floor in the sase repo's `pyproject.toml` from `sase-core-rs>=0.12.18,<0.13.0` to the released version that
  carries the gate, keeping the `<0.13.0` upper bound.
- Run `just install && just check` and confirm the dev-install path no longer prints the "bump the published
  sase-core-rs window" warning.
- Sanity-check the behavior once more against the published wheel (temporarily unset any local `SASE_CORE_DIR` override
  if needed) so the shipped combination, not just the local build, is verified.
