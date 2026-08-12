---
tier: tale
title: Tier-aware plan approval toasts with epic phase/wave/size counts
goal:
  A proposed plan's ACE toast names its tier (Tale or Epic) instead of the generic
  "Plan", and an epic toast adds a styled second line with its phase count, wave count,
  and per-size phase counts.
size: medium
proposed_by: bbugyi200.athena.ya
create_time: 2026-08-12 07:50:41
status: wip
---

# Plan: Tier-aware plan approval toasts with epic phase/wave/size counts

## Goal

When a SASE agent proposes a plan, the ACE toast currently reads:

```
Plan ready for @y4: bead_wait_store_diagnostics.md
```

"Plan" is generic: it hides whether the proposal is a **tale** (one coder agent
implements it directly) or an **epic** (phases fan out to multiple agents), which is the
single most decision-relevant fact about an arriving proposal. After this plan:

```
Tale ready for @y4: bead_wait_store_diagnostics.md
```

```
Epic ready for @y4: agent_group_clan_collapse.md
7 phases · 3 waves · 1 XS · 2 S · 3 M · 1 L
```

The tier word is colored with SASE's existing tale/epic accents, and the epic detail
line reuses the existing phase-size accent palette so the shape of the epic (how many
phases, how parallel, how heavy) is readable at a glance without opening the gate.

## Current behavior

- `src/sase/ace/tui/actions/agents/_toasts.py:66-77` handles both `PlanApproval` and
  `EpicApproval` in one branch and hardcodes `"Plan ready for @{agent}: {name}"`,
  `notes[0]`, then `"Plan ready for review"`.
- `src/sase/ace/tui/actions/agents/_toasts.py:172-187` (`_ACTION_LABELS`) labels grouped
  batches `plan`/`plans` and `epic`/`epics`.
- Toasts are emitted from
  `src/sase/ace/tui/actions/agents/_notification_polling.py:176-182` via
  `self.notify(message, severity=..., timeout=8)`.
- `AceApp.notify` (`src/sase/ace/tui/app.py:249-265`) passes `markup=True` by default
  and mirrors the message into toast history through `record_toast`.
- Toast CSS is `width: auto; min-width: 44; max-width: 72`
  (`src/sase/ace/tui/styles.tcss:5561`), so two lines of up to ~70 cells render without
  truncation.

Tier and structure are already known upstream: `create_plan_approval_gate`
(`src/sase/plan_gate.py:35-80`) reads the authored tier and calls
`require_plan_approval_validation`, which returns a `PlanValidationResult` whose
`.plan.phases` carry each phase's `id`, `depends_on`, and normalized `size`. That result
is currently discarded.

## Design

### 1. Tier and counts travel on the notification, not the render path

Compute the summary **once, at gate creation**, and store it as string values in
`presentation.action_data` (which `_build_notification` copies verbatim onto the
`Notification`, `src/sase/notification_gates/service.py:332-334`). The toast formatter
then does zero I/O and cannot fail: it reads already-decided integers.

Rationale, and the alternative rejected: deriving the counts in the TUI would mean
reading and re-validating the plan file (a Rust-binding call) inside the notification
poll loop for every arriving proposal, with failure modes (file moved, validation error)
on the render path. The stored summary also describes exactly the revision that was
proposed.

Known, accepted limitation: if the reviewer edits the plan inside the gate bundle before
approving, the stored counts are not recomputed. This cannot produce a wrong toast — the
toast is emitted when the notification arrives, before any edit is possible — but a
resurfaced (snoozed-then-returned) notification can re-toast pre-edit counts. Note it in
the module docstring; do not build recomputation for it.

### 2. Tier resolution is layered and total

`_toasts.py` resolves the tier with the first hit of:

1. `action_data["plan_tier"]` (new, written by the gate),
2. `action_data["request_kind"]` — `"epic_plan"` → epic, `"plan"` → tale
   (`src/sase/notification_gates/service.py:334-341`, `src/sase/plan_gate.py:320`),
3. `n.action` — `"EpicApproval"` → epic, `"PlanApproval"` → tale
   (`src/sase/notification_gates/adapters.py:262-282`),
4. otherwise `None` → keep the generic word `Plan`.

Steps 2-4 keep legacy in-flight gates and any hand-built notification working with no
counts line and, in almost every case, still the right tier word.

### 3. Palette: reuse, do not invent

The TUI already has tale/epic accents at
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py:53-57`
(`_AUTO_APPROVE_KIND_STYLES`): tale `#FFD75F`, epic `#AF87FF`, generic plan `#5FD7FF`.
Promote exactly those values into a new shared presentation module (below) so the toast
and the auto-approve chips cannot drift. Phase sizes reuse
`PHASE_SIZE_ACCENTS`/`PHASE_SIZE_ABBREVIATIONS` from
`src/sase/phase_size_presentation.py`; counts reuse the plan-display idiom of a
plan-primary number with a dim unit and `·` separators
(`src/sase/sdd/_plan_display_rendering.py:483-500`).

### 4. Markup safety is a prerequisite, not a nicety

`AceApp.notify` already renders toasts with `markup=True`, and Textual 8.0.1 **silently
deletes** any bracketed run it reads as a tag. Verified in this workspace:

```
Content.from_markup('[URGENT] fix now').plain  -> ' fix now'
Content.from_markup('plan[v2].md').plain       -> 'plan.md'
```

`textual.markup.escape` does not fix this: its regex only escapes tags starting with
`[a-z#/@]`, so `escape('[URGENT] fix')` is a no-op and the text is still swallowed. A
plain `str.replace("[", "\\[")` round-trips correctly for every case checked, including
uppercase tags, embedded backslashes, and a trailing backslash:

| input          | rendered plain |
| -------------- | -------------- |
| `[URGENT] fix` | `[URGENT] fix` |
| `plan[v2].md`  | `plan[v2].md`  |
| `a\b [X]`      | `a\b [X]`      |
| `back\[X] end` | `back\[X] end` |
| `trail\`       | `trail\`       |

So: every dynamic segment this module interpolates — plan basenames, agent names, notes
in **all** branches, not just the plan branch — goes through one local
`_markup_safe(text)` helper. This is a real fix (an agent question containing `[URGENT]`
currently loses that text in its toast), and it is what makes it safe to add styling.

Ordering rule: humanize → truncate → escape. `_truncate` must never run on escaped text,
or it can cut a `\[` in half.

### 5. Toast anatomy

Line 1 (both tiers), with `{Tier}` = `Tale` / `Epic` / `Plan`:

- `{Tier} ready for @{agent}: {basename}` when both are known,
- `{Tier} ready for review: {basename}` when the agent name is missing,
- `notes[0]` when there is no plan basename (preserves today's fallback),
- `{Tier} ready for review` when there is nothing else.

Styles: tier word `bold {tier accent}`; `@{agent}` `#87D7AF` (the plan-display agent
accent, `PLAN_PROVENANCE_AGENT_STYLE`); basename `bold #87AFFF`
(`COLOR_PLAN_PATH_BASENAME`); connecting words unstyled.

Line 2 (epic only, appended after `\n`, omitted entirely when no counts were stored):

```
{P} phases · {W} waves · {n} XS · {n} S · {n} M · {n} L · {n} XL
```

- Counts pluralize (`1 phase`, `1 wave`).
- Numbers `bold #D7AFFF` (`COLOR_PLAN_PRIMARY`); units and `·` separators `dim`.
- Size groups appear in canonical `PHASE_SIZE_VALUES` order, zero counts omitted; the
  count is unstyled and the abbreviation is `bold {PHASE_SIZE_ACCENTS[size]}`.
- The wave group (and its separator) is omitted when the wave count is unavailable.

Worst case width is ~52 cells (`12 phases · 4 waves · 1 XS · 2 S · 3 M · 4 L · 2 XL`),
inside the 72-cell toast max; long basenames wrap as they do today.

Severity stays `warning` for both tiers.

### 6. Deliberate non-goals in the visible design

- **No tale size line.** A tale declares a single `size`, and a one-word second line is
  weaker than a clean single line; the size is already visible in the gate modal. The
  encoder still records the tier for tales, so adding it later is a rendering-only
  change.
- **No Textual toast `title=`.** It would be the idiomatic place for the tier word, but
  `.toast--title` is styled by CSS per severity, so the tier could not be color-coded —
  which is the point. It would also change the `(message, severity)` contract of
  `format_batch_toasts` and every caller.

### 7. Boundary note (Rust core)

This derivation stays in Python. It is a fold over data `sase-core` already returns
through `plan_validate`, and its sibling — dependency-wave layering in
`src/sase/sdd/plan_waves.py` — already lives in Python for the same reason. If core ever
grows a first-class plan-summary API, this module becomes a thin adapter over it.

## Implementation

### Step 1 — `src/sase/plan_tier_presentation.py` (new)

Mirror the structure of the sibling modules `src/sase/phase_size_presentation.py` and
`src/sase/bead_type_presentation.py` (frozen dataclass, dict registry, exact
normalization, `__all__`):

- `PlanTierValue = Literal["tale", "epic"]`.
- `PLAN_TIER_PRESENTATIONS` with `label` (`"Tale"`/`"Epic"`) and `accent_color`
  (`#FFD75F` / `#AF87FF`), plus a `rich_style` property returning `f"bold {accent}"`.
- `GENERIC_PLAN_LABEL = "Plan"` and `GENERIC_PLAN_ACCENT = "#5FD7FF"` for the
  tier-unknown fallback.
- `normalize_plan_tier_value(value) -> PlanTierValue | None`, delegating to
  `sase.sdd.plan_tiers.normalize_plan_tier` so the accepted spellings stay in one place.
- `plan_tier_presentation(value)` returning the registry entry.

Then rewrite `_AUTO_APPROVE_KIND_STYLES` in
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py` to build its
`"tale"`/`"epic"` entries from this module (`("⚡ TALE", presentation.rich_style)`). The
hex values are identical, so rendering is unchanged and the existing
`agents_auto_approve_metadata_*` PNG goldens must stay green — if any of them changes,
the values were mistyped.

### Step 2 — `src/sase/sdd/plan_summary.py` (new)

Derivation plus the action_data codec, so producer and consumer share one definition of
the keys.

```python
@dataclass(frozen=True, slots=True)
class PlanCountsSummary:
    tier: PlanTierValue
    phase_count: int
    wave_count: int | None
    size_counts: tuple[tuple[PhaseSizeValue, int], ...]
```

- `plan_counts_summary(validation, *, tier) -> PlanCountsSummary | None` — takes the
  `PlanValidationResult` from `require_plan_approval_validation`; returns `None` when
  `validation.plan` is `None`. Phase count is `len(validation.plan.phases)`; wave count
  is `len(plan_phase_waves(phases))` (`src/sase/sdd/plan_waves.py`), or `None` when that
  returns `None`; size counts are a histogram over `normalize_phase_size(phase.size)` in
  `PHASE_SIZE_VALUES` order, skipping zeros and any size that fails to normalize.
  `ValidatedPlanPhase` already satisfies the `_WavePhase` protocol — its docstring names
  it — so pass the phases straight through.
- `encode_plan_counts(summary) -> dict[str, str]` — always `plan_tier`; and for epics
  `plan_phase_count`, `plan_wave_count` (omitted when `None`), and `plan_phase_sizes`
  formatted `xsmall=1,small=2,medium=3` (canonical order, zeros omitted, omitted
  entirely when empty).
- `decode_plan_counts(action_data: Mapping[str, str]) -> PlanCountsSummary | None` —
  **total: it must never raise.** Unknown tier → `None`; non-integer, negative, or
  malformed values → that field degrades to absent (`wave_count=None`,
  `size_counts=()`), unparsable size names are skipped, and a missing/invalid
  `plan_phase_count` degrades to `0`. The renderer then simply omits the detail line.

Document the key names and the format in the module docstring; they are a compatibility
surface read by a different process than the one that writes them.

### Step 3 — `src/sase/plan_gate.py`

- `create_plan_approval_gate` already binds the validation result
  (`require_plan_approval_validation(plan_path, typed_tier)`); keep it and pass it into
  `_build_plan_gate_spec(..., validation=validation)`.
- In `_build_plan_gate_spec`, merge
  `encode_plan_counts(plan_counts_summary(validation, tier=tier))` into the dict
  returned by `_plan_action_data(...)`. Merge only when the summary is not `None`, and
  keep the "drop empty values" behavior of `_plan_action_data`. The new keys collide
  with no reserved key (`src/sase/notification_gates/validation.py:143-160`), and
  plan-kind spec validation (`src/sase/notification_gates/kind_validation/plan.py`) does
  not constrain `action_data`.
- Fix the tier wording in `presentation.notes` at `src/sase/plan_gate.py:132-141`:
  `"Epic ready for review: "` stays; the tale branch becomes
  `"Tale ready for review: "`. The note is display-only — it is rendered in
  `notification_modal_gate.py` / `notification_modal_options.py` and read as the toast
  fallback, and nothing parses it — but tests assert the old string, so update them.

### Step 4 — `src/sase/ace/tui/actions/agents/_toasts.py`

- Add `_markup_safe(text: str) -> str` returning `text.replace("[", "\\[")`, with a
  comment recording why `textual.markup.escape` is not used (uppercase tags are not
  escaped, and Textual silently drops them).
- Apply `_markup_safe` to every interpolated dynamic value in **every** branch — notes,
  agent names, plan basenames — and to the plain fallback returns. Static wording and
  the grouped-batch strings are already safe.
- Replace the `{"PlanApproval", "EpicApproval"}` branch with the tier-aware builder from
  the Design section: resolve tier, pick label/accent from `plan_tier_presentation`,
  build line 1, and append the epic detail line from
  `decode_plan_counts(n.action_data)`.
- Humanize the plan agent name with the existing `_humanize_text` before escaping, as
  the `UserQuestion` branch already does, so a project key never leaks into a toast (see
  the "Show Project Names, Never ProjectSpec Keys" convention in `CLAUDE.md`).
- `_ACTION_LABELS`: `"PlanApproval"` becomes `("tale", "tales")`; `"EpicApproval"` stays
  `("epic", "epics")`. Grouped batches then read `5 warnings: 2 tales, 3 epics`.
- Keep the module's existing contract: `format_batch_toasts` still returns
  `list[tuple[str, Severity]]`, and `_format_notification_toast` stays pure with no I/O.

### Step 5 — Tests

- `tests/test_plan_tier_presentation.py` (new): labels, accents, exact normalization
  (`None` for `"Tale "`-style junk only if `normalize_plan_tier` rejects it — assert the
  delegation), and that `_AUTO_APPROVE_KIND_STYLES` values are unchanged strings.
- `tests/test_plan_summary.py` (new): derivation over a validated epic (phase count,
  wave count from a multi-wave `depends_on` graph, size histogram order and
  zero-omission); `encode` → `decode` round-trip; and a hostile-input table for
  `decode_plan_counts` (missing keys, `"abc"`, `"-3"`, `"xsmall=abc"`, `"bogus=2"`,
  empty strings) asserting it never raises and degrades field-by-field.
- `tests/test_notification_toasts.py` (update): existing plan assertions become `Tale`/
  `Epic`. Compare rendered plain text via a small helper
  (`textual.content.Content.from_markup(msg).plain`) so content assertions stay
  readable, and add focused assertions on the markup itself for the tier accent, the
  size accents, and that a basename containing `[` survives round-trip.
- `tests/test_notification_toast_polling.py` (update): the note-only fallback cases keep
  their current messages (no brackets, no basename, so nothing changes) — confirm rather
  than assume; the grouped-batch assertions pick up `tales`.
- Plan-gate tests: no test currently pins the tale gate's note text —
  `rg -n "ready for review" tests/` finds only synthetic notes in the toast tests plus
  the out-of-scope desktop-notification assertion at
  `tests/test_plan_approval_responses.py:192`. Re-run that search anyway, then add a
  gate test asserting the new note wording, that an epic gate's
  `presentation.action_data` carries correct `plan_phase_count`, `plan_wave_count`, and
  `plan_phase_sizes`, and that a tale gate carries `plan_tier` alone.
- `tests/ace/tui/visual/test_ace_png_snapshots_plan_toast.py` (new): follow
  `tests/ace/tui/visual/test_ace_png_snapshots_update_toast.py`. It needs no update
  plumbing — mount `AcePage`, call `page.app.notify(message, severity="warning")` with
  the message built by `_format_notification_toast` from a synthetic epic notification,
  wait for the `Toast` widget and `wait_for_visual_idle`, then
  `ace_png_visual.assert_page_png(page, "plan_toast_epic_120x40", ...)`. Add a second
  case for the tale toast. Generate goldens with
  `just test-visual --sase-update-visual-snapshots` and **look at the produced PNGs** —
  they are the acceptance check for the visual design, not just a regression net.

## Verification

```bash
just install
just check
just test-visual
```

`just check-full` before landing. Confirm by eye in the generated PNGs: the tier word
reads as the anchor of the toast, the epic detail line is subordinate but legible, and
the five size accents remain distinguishable next to each other.

## Out of scope

- The desktop notification text `"Plan Complete" / "Plan ready for review in sase ace"`
  (`src/sase/llm_provider/_plan_utils.py:213-216`) — same generic wording, different
  surface; worth a follow-up task bead.
- Showing the counts in the notification list rows or the plan approval modal, which
  render their own layouts from the gate bundle.
- Rendering markup in the Logs pane toast history
  (`src/sase/ace/tui/modals/logs_pane_toasts.py:_append_toast_row` shows the recorded
  message as plain text, so markup tags appear literally there — already true today for
  the update toasts). File a task bead rather than widening this change.
