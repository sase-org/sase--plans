---
tier: tale
title: Refine the Models panel into Launch Control
goal:
  Launch Control presents a polished launch-configuration layout and safely manages the
  big-epic phase threshold.
size: medium
proposed_by: bbugyi200.athena.02w
create_time: 2026-08-15 18:55:40
status: done
---

- **PROMPT:**
  [prompts/202608/launch_control_panel.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/launch_control_panel.md)
- **AGENTS:**
  - [bbugyi200.athena.02w](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.02w.md)
- **COMMITS:**
  - [75c670c](https://github.com/sase-org/sase/commit/75c670c4b671c81f5d919542c41ef96f78721eee)
    — feat(ace): add launch control threshold editing

# Refine the Models panel into Launch Control

## Goal

Turn the existing ACE `,m` modal into a clearer, more polished control surface for
agent-launch behavior. The finished panel should be named **Launch Control**, remove the
redundant row-kind column, separate its visible sections with one blank row, and expose
`bead.big_epic_phase_threshold` as a safe persistent launch setting.

The change must preserve the existing model-alias, effort, runner-limit,
provider-disable, bucket, temporary-override, and persistent-edit behavior. Keep the
internal `ModelsPanel` class/module names, `models_panel` action ID, CSS/widget IDs, and
legacy action compatibility stable; this is a user-facing rename, not a risky
public/internal API migration.

## User experience and visual design

1. Rename every current user-facing reference from **Models panel** / **Models** to
   **Launch Control**. This includes the modal title (and drilled-in bucket title),
   leader command/help labels, top-bar indicator guidance/tooltips, documentation
   headings and cross-links, and relevant test expectations. Preserve an explicit legacy
   `#models-panel` documentation anchor while making `#launch-control` canonical so old
   external links do not break. Historical changelog text and internal identifiers do
   not need rewriting.
2. Remove the entire kind column from data rows. Values such as `launch`, `setting`,
   `role`, `user`, and `bucket` must no longer occupy a left-hand column. Keep the
   two-cell ownership gutter because it conveys custom ownership independently of row
   kind; keep `!` for misplaced built-ins and move the bucket disclosure marker `▸` into
   the name cell so those affordances remain visible without recreating a kind column.
3. Recompute the header and row grid around
   `ownership gutter | name | value/model | state`. Reinvest the removed 13-cell kind
   column plus its separator into the value/model budget (raise the existing 32-cell cap
   by those 14 reclaimed cells) so long alias/model expressions are more readable while
   the state column retains the existing 110-column alignment budget. Preserve Rich cell
   measurement, literal-safe rendering, ellipsis at narrow widths, aligned state tags,
   and the current semantic colors.
4. Insert exactly one non-selectable blank option between consecutive visible sections:
   Launch settings → Built-in size aliases → Your aliases at the top level, and Built-in
   → Custom inside a mixed bucket. Use deterministic decorative IDs and centralized
   section assembly so there is never a leading/trailing blank or doubled spacing.
   Homogeneous drilled-in buckets should remain headerless and spacer-free. Keyboard
   navigation must skip blank rows just as it skips headers and the empty-custom hint,
   including wraparound and highlight restoration.
5. Add the threshold row to **Launch settings**, next to the two epic-lander rows and
   before default effort / running agents:
   - label: `big epic starts at`
   - value: `<N> phase` / `<N> phases`, with correct singularization
   - row ID: `setting:big_epic_phase_threshold`
   - state: the same configured scalar-setting treatment used by the runner limit
   - description: explicitly say that epics with `N` or more authored phases use the big
     epic lander and smaller epics use the regular epic lander, followed by
     `bead.big_epic_phase_threshold: N`

   This wording deliberately expresses the inclusive boundary and avoids the common
   “allowed phases” off-by-one ambiguity. The section count should become six settings.

## Threshold data and edit workflow

1. Add a typed threshold display row to the panel row union and build it from the same
   off-thread provider/routing snapshot that already captures
   `get_big_epic_phase_threshold()`. Keep the threshold row and both lander descriptions
   on one snapshot so an edit cannot show a new boundary beside stale “below N” / “N+”
   copy. Initialize with `DEFAULT_BIG_EPIC_PHASE_THRESHOLD` until the snapshot arrives;
   do not add config reads to render, highlight, or timer paths.
2. Make the threshold persistent-only: there is no temporary override file or duration
   flow for this configuration field. On its row, `Enter` and `e` open a focused editor;
   `r` previews a reset. The context footer should advertise only Edit and Reset for the
   row (plus the existing global Effort, Limit, Providers, navigation, and close
   actions). If `o` or `x` is pressed anyway, give concise guidance that this setting
   has no temporary override instead of silently opening a model workflow.
3. Reuse/extract the existing strict positive-integer parsing behavior rather than
   accepting YAML-ish coercions: require an unsigned, whitespace-free base-10 integer of
   at least `1`, reject booleans/floats/signs/spaces, select the current effective value
   on open, validate inline, and show `minimum 1 · package default 5`. Give the editor
   threshold-specific title and copy explaining that this is the authored phase count
   where an epic becomes big.
4. Implement set and reset previews through the established Rust-backed
   `plan_config_edit` / `AliasEditPreviewModal` path, targeting the writable user-base
   `sase.yml` (or its chezmoi source) at `bead.big_epic_phase_threshold`. Set writes the
   chosen integer. Reset uses `ConfigEditOp.unset()` so lower-precedence/package
   defaults resume rather than hard-coding `5`. Planning, application, chezmoi
   propagation, and commit-offer discovery must remain in worker threads.
5. After a successful write, reload the effective threshold with
   `get_big_epic_phase_threshold()`, mark the panel changed, preserve focus on the
   threshold row, refresh the atomic provider/launch-row snapshot, and report the actual
   effective value. If a higher-precedence layer means the requested user value does not
   win, make that mismatch explicit in the notification. Offer the normal tracked
   commit/pull/push flow for a dirty Git-backed target with a threshold-specific commit
   subject, and cancel all new workers during panel teardown.

## Implementation areas

- Extend `src/sase/ace/tui/modals/models_panel_rows.py` with the threshold row and
  include it in the display union and launch-setting ordering.
- Integrate snapshot state, loading, action routing, worker teardown, and row rebuilding
  through `models_panel.py`, `models_panel_providers.py`, `models_panel_display.py`, and
  the existing edit/override dispatch mixins. Add a focused threshold workflow/edit
  helper module where that keeps scalar planning and Textual orchestration separated,
  following the established default-effort and runner-limit patterns.
- Update `models_panel_rendering.py` for the new grid, threshold
  value/state/description, bucket/name affordances, wider value budget, section headers,
  and blank-section options. Adjust `styles.tcss` only as needed for the threshold
  editor and verify the existing 110-column modal and constrained/narrow viewport
  behavior rather than increasing the modal size.
- Update user-facing labels in the leader command catalog, per-tab help modal bindings,
  and model/provider/alias override indicators while retaining the stable internal
  action/class names.
- Rewrite the ACE Launch Control documentation and relevant configuration/LLM/xprompt/
  troubleshooting references to match the new title, column anatomy, section spacing,
  six launch settings, threshold semantics, and Edit/Reset workflow.

## Tests and validation

1. Add pure tests for threshold parsing, singular/plural rendering, the exact config
   path, set versus unset planning, minimum/schema rejection, user-layer targeting,
   chezmoi behavior, effective-value reload, higher-precedence mismatch reporting, and
   threshold-specific commit offers.
2. Extend mounted panel tests for the six launch row IDs/order, atomic threshold refresh
   of both lander descriptions, `Enter`/`e`/`r` routing, unsupported `o`/`x` guidance,
   context footer, focus preservation, close-while-worker-busy protection, teardown, and
   the **Launch Control** top-level/drilled-in/provider-disabled titles.
3. Update rendering/navigation tests to prove the kind column is absent, warning and
   bucket markers live in the name cell, state columns remain aligned, reclaimed width
   prevents avoidable truncation, decorative spacers are not actionable, and `j`/`k`
   skip headers/spacers/hints with correct wraparound at both top level and in mixed
   buckets.
4. Update command/help/indicator tests for the user-facing rename. Keep compatibility
   assertions for the `models_panel` and legacy `temporary_llm_override` action IDs.
5. Update the Models-panel visual snapshot fixtures/tests and all intentionally affected
   PNG goldens. Include normal `120x40`, narrow viewport, empty-custom, collapsed
   bucket, mixed drilled-in bucket, override, and provider-disabled states. Inspect
   generated actual/expected/diff artifacts—not just test exit status—to confirm one
   blank row per section, readable values, stable title/footer/description containment,
   and no clipped state chips.
6. From an installed workspace (`just install` first), run focused non-visual panel,
   config-edit, help/catalog, and indicator tests; regenerate intentional goldens with
   `just test-visual -- --sase-update-visual-snapshots`; rerun `just test-visual`
   without the update flag; then run the required whole-repo `just check` before
   handoff.

## Acceptance criteria

- The visible title and all current user guidance call the surface **Launch Control**,
  while existing internal APIs/keymap action IDs continue to work.
- No data row renders the old kind column or its labels; custom ownership, warnings, and
  bucket discoverability remain immediately understandable.
- Each adjacent visible section is separated by exactly one blank, non-focusable line,
  with navigation and narrow-layout behavior unchanged apart from the intentional space.
- `big epic starts at` shows the effective positive threshold and accurately explains
  the inclusive lander boundary.
- Editing and resetting the threshold are previewed, schema-validated,
  source-preserving, chezmoi-aware, asynchronous, commit-offered, and reflected
  atomically in the row and both epic-lander descriptions.
- Focus, footer actions, temporary model/effort/runner overrides, provider routing,
  alias buckets, responsive rendering, and all non-targeted panel behavior remain
  correct; focused tests, reviewed PNG snapshots, and `just check` pass.
