---
tier: tale
title: Make the ACE top-bar override pills terse, typographic, and self-explaining
goal: The two ACE top-bar override indicators drop the redundant "Override" word,
  disambiguate the `@alias@effort` suffix through typography instead of extra characters,
  encode "until cleared" as a single glyph, name one alias in the multi-override state,
  and move every long-form detail into new hover tooltips plus a click-to-open-Models-panel
  affordance - so the pills are far narrower while a user can still recover exactly
  the same information (and more).
create_time: 2026-07-25 07:26:27
status: done
---

- **PROMPT:** [202607/prompts/top_bar_override_pill_concision.md](prompts/top_bar_override_pill_concision.md)
- **AGENTS:**
  - [bbugyi200.athena.ka](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.ka/README.md)
  - [bbugyi200.athena.ka--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ka.md#member-code)
- **COMMITS:**
  - [41e44d1](https://github.com/sase-org/sase/commit/41e44d1c4205f7049723e109c53350460cc36913) — feat(ace): refine top-bar override pills

# Plan: Make the ACE top-bar override pills terse, typographic, and self-explaining

## Context

`sase ace` renders two temporary-model-override indicators at the right end of `#top-bar`:

| Widget                                                  | Lane                                                                    | Accent    |
| ------------------------------------------------------- | ----------------------------------------------------------------------- | --------- |
| `src/sase/ace/tui/widgets/llm_override_indicator.py`    | the `default` alias (the no-`%model` launch default)                    | `#D7AF5F` |
| `src/sase/ace/tui/widgets/alias_overrides_indicator.py` | every other alias (`coder`, `smartest`, `*_phase_worker`, user aliases) | `#AF87FF` |

Today they render (verbatim from the current source):

| State                                   | Current string                       | Cells |
| --------------------------------------- | ------------------------------------ | ----- |
| default lane, expiring, with effort     | `Override CLAUDE(opus)@xhigh 12h51m` | 38    |
| default lane, until cleared             | `Override CODEX(o3) until cleared`   | 33    |
| alias lane, one override, with effort   | `Override @smartest@max 12h51m`      | 31    |
| alias lane, one override, until cleared | `Override @coder until cleared`      | 30    |
| alias lane, several overrides           | `Overrides ×3`                       | 13    |

Three problems:

1. **`Override ` is dead weight.** The pill already inverts to a solid accent background (`bold #1a1a1a on <accent>`)
   while the calm no-override state is unstyled `dim cyan` text with no background. Color _and_ luminance already say
   "temporarily overridden", and the trailing countdown says "temporarily".
2. **It breaks the established top-bar grammar.** Every _other_ top-bar badge is `<glyph> <value>` with no prose: `⚙ N`
   (tasks), `⇅ N` (agents sync), `❄ N` (stash), `✉ N` (notifications), `↑ N` (updates). The two override pills are the
   only badges that spend nine cells on an English word.
3. **`@smartest@max` reads as one mangled token.** It is _not_ a bug and `@max` is _not_ redundant: `@max` is the
   snapshotted canonical reasoning effort, and `@<alias>@<effort>` is exactly the reference syntax a user types
   (`docs/llms.md`: "`%model:@default@medium`", "`@coder@medium`", "An explicit outer reference such as `@coder@xhigh`
   still wins"). Dropping `@max` would make an effort-pinned override indistinguishable from an unpinned one - real
   information loss. The defect is purely visual: both tokens are painted in one flat style, so the two `@`s collide.

Neither pill has a tooltip or a click action, even though every neighboring top-bar badge has both
(`StashedPromptsIndicator`, `AgentsSyncIndicator`, `UpdatesAvailableIndicator`). The alias pill never shows the
overridden _model_ at all, and its multi state shows nothing but a count.

## Design

### The pill grammar

One grammar, two lanes:

```
 <subject>[@<effort>] <time>
 ^^^^^^^^  ^^^^^^^^^  ^^^^^^
 primary   secondary  secondary
```

- **Lane identity stays in the background color** - gold `#D7AF5F` for the `default` launch lane, violet `#AF87FF` for
  every other alias. Unchanged, so existing muscle memory survives.
- **Subject** - `PROVIDER(model)` for the default lane (the SASE-wide canonical badge from
  `format_provider_model_label`), `@alias` for the alias lane. The `@` sigil _is_ that pill's glyph, so the badge
  grammar above is honored without spending a cell on a new symbol.
- **Effort** - the canonical compact `@<level>` suffix. Same characters as today, but painted in the secondary tone so
  `@smartest` reads as the subject and `@max` reads as a modifier hanging off it.
- **Time** - `12h51m` countdown, or `∞` when the override has no expiry.
- **No `Override` / `Overrides` word anywhere.**

Why `∞` and not `until cleared`: it is the standard "no expiry" symbol, it collapses 13 cells to 1, it sits in the slot
the eye already reads as the time slot, and the tooltip spells out "Until cleared" in words. It is also verified present
in the pinned visual-suite font (see _Glyph safety_ below).

### Resulting states

**`LLMOverrideIndicator` (gold, `default` lane)**

| State                         | Before                               | After                       | Cells    |
| ----------------------------- | ------------------------------------ | --------------------------- | -------- |
| no override                   | `CLAUDE(opus)` (dim cyan)            | _unchanged_                 | -        |
| resolving / unavailable       | `...` / `unavailable`                | _unchanged_                 | -        |
| override, expiring, effort    | `Override CLAUDE(opus)@xhigh 12h51m` | `CLAUDE(opus)@xhigh 12h51m` | 38 -> 28 |
| override, expiring, no effort | `Override CODEX(o3) 1h2m`            | `CODEX(o3) 1h2m`            | 26 -> 17 |
| override, until cleared       | `Override CODEX(o3) until cleared`   | `CODEX(o3) ∞`               | 33 -> 13 |
| override already expired      | falls through to the default content | _unchanged_                 | -        |

**`AliasOverridesIndicator` (violet, every other alias)**

| State              | Before                          | After                  | Cells    |
| ------------------ | ------------------------------- | ---------------------- | -------- |
| none active        | `` (zero width)                 | _unchanged_            | -        |
| one, expiring      | `Override @coder 1h2m`          | `@coder 1h2m`          | 23 -> 14 |
| one, with effort   | `Override @smartest@max 12h51m` | `@smartest@max 12h51m` | 31 -> 22 |
| one, until cleared | `Override @coder until cleared` | `@coder ∞`             | 30 -> 10 |
| several (N >= 2)   | `Overrides ×3`                  | `@coder +2`            | 13 -> 11 |

The multi state now _names_ the alphabetically-first overridden alias and counts the rest as `+N` (the familiar "+2
more" idiom). It is shorter than the bare count **and** strictly more informative. Alphabetical selection is chosen over
"soonest expiry" deliberately: the named alias must not silently swap as clocks tick, because a jumping label is exactly
the kind of unreliability that makes an indicator untrustworthy.

### Typography inside the pill

Each lane gets two foregrounds over its accent background:

| Lane   | Accent (bg) | Primary fg (subject) | Secondary fg (effort, time, `+N`) |
| ------ | ----------- | -------------------- | --------------------------------- |
| gold   | `#D7AF5F`   | `bold #1a1a1a`       | `not bold #4F3D18`                |
| violet | `#AF87FF`   | `bold #1a1a1a`       | `not bold #3A2A5F`                |

Contrast (WCAG relative-luminance ratio against the pill background) for the proposed secondary tones: `#4F3D18` on
`#D7AF5F` ~= **5.1:1**; `#3A2A5F` on `#AF87FF` ~= **4.6:1**. Both clear the 4.5:1 normal-text bar while still receding
clearly behind the ~8.4:1 / ~6.4:1 primary. The implementer should re-verify these two ratios with a short throwaway
computation (sRGB -> linear -> `0.2126 R + 0.7152 G + 0.0722 B`, `(L_light + 0.05) / (L_dark + 0.05)`) and darken the
secondary tone if either lands under 4.5:1; do not ship a secondary tone that fails.

This is the entire fix for `@smartest@max`: identical, copy-pasteable, canonical tokens, zero extra cells, but the
`@max` visibly recedes so the alias reads as one unit.

### Tooltips carry the long form (new)

This is what makes "shorter" cost nothing. Both pills gain a `self.tooltip`, matching the convention every other top-bar
badge already follows. Tooltips use SASE's canonical _spaced_ effort connective `@` (the form the Models panel and the
Agent info panel render) because there is no width pressure there.

Gold pill, override active:

```
Temporary override on @default
CLAUDE(opus) @ xhigh
12h51m left
Press ,m for the Models panel.
```

with `Until cleared` replacing the `12h51m left` line when `expires_at is None`.

Gold pill, no override active:

```
Launch default: CLAUDE(opus)
No temporary override active.
Press ,m for the Models panel.
```

Use `Launch default: resolving...` while `_cached_default is None and not _cached_default_failed`, and
`Launch default: unavailable` when `_cached_default_failed`.

Violet pill (one line per alias, sorted by alias name; covers both the single and multi states):

```
Temporary alias overrides:
@coder -> CLAUDE(opus) @ xhigh - 1h2m left
@fast -> CLAUDE(haiku) - until cleared
Press ,m for the Models panel.
```

Set the violet tooltip to `None` when nothing is overridden (the widget is zero-width and unreachable then).

Note the violet tooltip surfaces the overridden **provider/model**, which that pill has never shown in any state - so
this change is a net information _gain_, including for the multi case that previously showed only a number.

**Cost discipline:** every tooltip is pure string formatting over state the widget has already read
(`get_active_temporary_override()` / `get_active_alias_overrides()` return fully-populated `TemporaryLLMOverride`
records including `provider`, `model`, and `effort`; the gold pill's default label comes from the existing
`_cached_default` tuple). No new file, config, subprocess, or provider-registry access on any refresh path - the pills
poll on a 30s interval and must stay free.

### Click-to-open (new)

Both pills get `async def on_click(self) -> None: await self.app.run_action("open_models_panel")`, mirroring
`UpdatesAvailableIndicator.on_click` -> `open_updates_panel` and `AgentsSyncIndicator.on_click`. The Models panel is
already the authoritative detail-and-edit view for overrides; making the pill the doorway to it is the natural
affordance and the reason the tooltip can stay short.

There is no app-level action for it yet - the panel is opened by the private `LeaderModeMixin._open_models_panel()`
(`src/sase/ace/tui/actions/agent_workflow/_leader_mode.py`), reached via leader `,m`. Add a thin public
`action_open_models_panel()` to `LeaderModeMixin`, immediately next to `_open_models_panel`, that just delegates to it.
`AceApp` inherits `LeaderModeMixin` through `AgentWorkflowMixin`, so `run_action("open_models_panel")` resolves. Do not
duplicate the panel-opening or indicator-refresh logic - `_open_models_panel` already refreshes both pills on dismiss.

### Glyph safety

The visual snapshot suite rasterizes through resvg with `skip_system_fonts=True` and only the bundled
`tests/ace/tui/visual/fonts/FiraCode-{Regular,Bold}.ttf`. Confirmed against that exact font's cmap: `∞` (U+221E) **is**
present; `⧗`, `⌛`, `⏳`, `⏱`, `⇄`, `↯`, `※` are **not** (and neither, incidentally, are the `⚙`/`❄`/`✉`/`⇅` glyphs of
the neighboring badges, which simply never appear in a golden because those badges are hidden at count 0). Use `∞` and
no other new glyph. If a future variant needs another symbol, restrict the choice to the verified-present set:
`∞ ≠ ◆ ▲ ↑ · • → ‹ › ⟨ ⟩ ◷ ◇`.

## Implementation

### 1. New shared module - `src/sase/ace/tui/widgets/_override_pill.py`

One source of truth for the grammar, so the two pills cannot drift apart. Module docstring should state the grammar and
the lane/color contract.

```python
UNTIL_CLEARED_GLYPH = "∞"


@dataclass(frozen=True)
class PillPalette:
    """Two-tone foreground palette over one lane accent background."""
    accent: str
    secondary: str

    @property
    def base_style(self) -> str:
        return f"bold #1a1a1a on {self.accent}"

    @property
    def secondary_style(self) -> str:
        return f"not bold {self.secondary} on {self.accent}"


DEFAULT_LANE_PALETTE = PillPalette(accent="#D7AF5F", secondary="#4F3D18")
ALIAS_LANE_PALETTE = PillPalette(accent="#AF87FF", secondary="#3A2A5F")
```

Functions:

- `format_remaining_until(expires_at, now=None) -> str` - **move verbatim** from `llm_override_indicator.py` (same
  `math.ceil` minute rounding, same `""` for expired, same `"until cleared"` for `None`). It becomes the tooltip-facing
  formatter.
- `format_pill_remaining(expires_at, now=None) -> str | None` - the pill-facing formatter: `None` when the override has
  already expired (so callers can collapse the pill), `UNTIL_CLEARED_GLYPH` when `expires_at is None`, otherwise the
  `format_remaining_until` countdown.
- `format_tooltip_remaining(expires_at, now=None) -> str` - `"Until cleared"` when `expires_at is None`, else
  `f"{format_remaining_until(...)} left"`.
- `format_tooltip_target(override) -> str` - `"CLAUDE(opus) @ xhigh"` / `"CODEX(o3)"`, via `format_provider_model_label`
  plus the spaced connective when `override.effort` is set.
- `build_override_pill(*, subject, effort, trailing, palette) -> Text` - assembles `<subject>[@<effort>] <trailing>` as
  a `Text(style=palette.base_style)` with the effort and trailing segments appended under `palette.secondary_style`.
  Keeping the base style on the `Text` (rather than only on spans) preserves the existing `text.style` assertions and
  guarantees the leading/trailing pad cells keep the accent background.

Re-export nothing from `llm_override_indicator`; update its one internal caller and the one test import instead
(`format_remaining_until` has exactly three call sites plus one test import repo-wide).

### 2. `src/sase/ace/tui/widgets/llm_override_indicator.py`

- Delete the local `format_remaining_until` (moved) and import from `._override_pill`.
- Keep `_ACTIVE_STYLE = DEFAULT_LANE_PALETTE.base_style` and `_DEFAULT_STYLE = "dim cyan"` so nothing else needs to
  change.
- `_build_override_content`: use `format_pill_remaining`; return `None` when it returns `None` (preserving today's
  "expired -> fall through to the default content" behavior); otherwise return
  `build_override_pill(subject=format_provider_model_label(override.provider, override.model), effort=override.effort, trailing=remaining, palette=DEFAULT_LANE_PALETTE)`.
  The word `Override ` is gone.
- Add `_build_tooltip(override, *, now=None) -> str` per the spec above, plus the no-override branch that reads
  `_cached_default` / `_cached_default_failed`.
- Add a single `_apply_content()` helper that computes content and tooltip together and performs `self.update(...)` +
  `self.tooltip = ...`; call it from `on_mount`, the bare-`refresh()` path, and both `WorkerState.SUCCESS` /
  `WorkerState.ERROR` branches of `on_worker_state_changed`, replacing the current three separate
  `self.update(self._build_initial_content())` calls. In `__init__`, set `self.tooltip` only _after_
  `super().__init__(...)` (same ordering `StashedPromptsIndicator` uses).
- Add `on_click`.
- Leave the static `_build_content` synchronous-snapshot path and its "active override skips default resolution"
  behavior intact - `tests/test_llm_override_indicator.py` depends on both.

### 3. `src/sase/ace/tui/widgets/alias_overrides_indicator.py`

- Import from `._override_pill` instead of `.llm_override_indicator`; keep
  `_ACTIVE_STYLE = ALIAS_LANE_PALETTE.base_style`.
- Rewrite `_build_content` around an explicit, ordered rule (this also fixes a small existing inconsistency - today only
  the single-override branch guards against an expired entry, so a direct call with several expired entries would render
  a wrong count):
  1. Compute `format_pill_remaining` for every entry; drop entries where it is `None` (expired).
  2. Zero survivors -> `Text("")`.
  3. One survivor ->
     `build_override_pill(subject=f"@{alias}", effort=override.effort, trailing=remaining, palette=ALIAS_LANE_PALETTE)`.
  4. Two or more ->
     `build_override_pill(subject=f"@{first}", effort=None, trailing=f"+{len(survivors) - 1}", palette=ALIAS_LANE_PALETTE)`
     where `first` is `min(sorted alias names)`. Sort explicitly; do not rely on dict insertion order.
- Add `_build_tooltip(overrides, *, now=None) -> str | None` per the spec (sorted by alias name, `None` when empty), and
  set it wherever content is set (`__init__` after `super().__init__`, `on_mount`, and the bare-`refresh()` path) via
  the same `_apply_content()` pattern.
- Add `on_click`.
- Rewrite the module docstring's state list to match the new grammar.

### 4. `src/sase/ace/tui/actions/agent_workflow/_leader_mode.py`

Add, directly above or below `_open_models_panel`:

```python
def action_open_models_panel(self) -> None:
    """Open the Models panel (top-bar override pills click here)."""
    self._open_models_panel()
```

Nothing else in that file changes; the existing dismiss callback already refreshes both pills.

### 5. Docs

`docs/ace.md` around lines 1596-1603 currently reads "a single active override renders as
`Override @<alias>[@<effort>] <time-left>`, and several render as an `Overrides ×N` count". Rewrite that bullet (and the
neighboring gold-pill bullet, which says its rendering is "unchanged") to describe the new grammar: lane color carries
"override", subject is `PROVIDER(model)` or `@alias`, the `@<effort>` suffix and the countdown render in a recessive
tone, `∞` means until cleared, several overrides render as `@<alias> +N`, and both pills expose the full detail on hover
and open the Models panel on click.

Also check the `### Examples` block near line 1650 ("the violet non-default pill appears in the top bar") - it stays
true, but confirm no example quotes a literal old pill string.

No help-modal change is required (verified: `src/sase/ace/tui/modals/help_modal/` never mentions the override pills),
and no keymap/`src/sase/default_config.yml` change is required (no new binding - the click target reuses leader `,m`'s
existing action).

## Tests

### Updated

- `tests/test_llm_override_indicator.py`
  - Retarget the expected strings: `CODEX(o3) 1h2m`, `CODEX(o3)@medium 1h2m`, `CODEX(o3) ∞`,
    `VERYLONGPROVIDER(extremely-long-model-name) 15m`. The unchanged-state tests (`CODEX(gpt-5.6-sol)`, `...`,
    `unavailable`, `CLAUDE(sonnet)` after expiry, the state-file cleanup test,
    `test_init_skips_cold_default_resolution`, `test_active_override_skips_default_resolution`) must keep passing
    verbatim except for the override strings themselves.
  - Move the `format_remaining_until` import to `sase.ace.tui.widgets._override_pill`.
- `tests/test_alias_overrides_indicator.py`
  - `@coder 1h2m`, `@coder@medium ∞`, `@phase_worker ∞`, `@coder +2` for the three-override case.
  - Keep the `_ACTIVE_STYLE` / `"#AF87FF" in str(text.style)` assertions - they must still hold because the base style
    is unchanged.
- `tests/ace/tui/visual/test_ace_png_snapshots_alias_overrides_indicator.py`
  - Give the _single_-override frame an `effort` (e.g. `effort="max"`) so the golden pins the two-tone `@coder@max ∞`
    rendering, which is the whole point of the typographic fix. Both frames already use until-cleared overrides, so they
    now pin `∞` with no clock nondeterminism.
  - Update the module docstring and the two `title=` strings to the new grammar.
  - Regenerate the two goldens under `tests/ace/tui/visual/snapshots/png/` with
    `just test-visual -- --sase-update-visual-snapshots tests/ace/tui/visual/test_ace_png_snapshots_alias_overrides_indicator.py`
    and eyeball the PNGs: confirm `∞` renders as a real glyph (not tofu), and confirm the secondary tone is legible but
    visibly recessive in both lanes.

### New

- Two-tone rendering, per lane: build a pill with an effort and assert the produced `Text` carries a span whose style
  contains the lane's secondary hex over the effort/trailing segments while the subject stays `bold #1a1a1a` - i.e.
  assert the _hierarchy_, not just `text.plain`.
- Expired-entry pruning in the alias pill's multi path: three entries where two are already expired must render the
  single form for the survivor, not `@x +2`.
- Deterministic multi naming: `{"zeta": ..., "alpha": ..., "mid": ...}` renders `@alpha +2` regardless of dict order.
- Tooltip content for both widgets, across: no override / one override with effort / one until cleared / several. Assert
  the alias tooltip names the provider/model (the information the pill itself never shows).
- `on_click` opens the Models panel: assert `run_action("open_models_panel")` reaches `_open_models_panel` (patch/spy
  it) for both widgets.
- Narrow-terminal regression in `tests/ace/tui/test_top_bar_order.py` (alongside the existing
  `test_mixed_updates_indicator_keeps_narrow_top_bar_in_bounds`): at `size=(80, 30)`, with a `default` override _and_ a
  non-`default` override both active, assert the `#top-bar` children stay ordered and within the bar's bounds. This is
  the test that encodes "the bar got narrower" as a durable invariant. If it exposes clipping at 80 columns, the
  documented remedy is a TCSS `max-width` on `#alias-overrides-indicator` plus `no_wrap=True, overflow="ellipsis"` on
  the pill `Text` - apply it only if the test actually demands it, and note it in the summary.

## Verification

Run from the workspace checkout root, in order:

```bash
just install
just check
just test-visual -- --sase-update-visual-snapshots tests/ace/tui/visual/test_ace_png_snapshots_alias_overrides_indicator.py
just test-visual
```

`just install` first is mandatory (ephemeral workspace clones may hold a stale virtualenv). `just check` is mandatory
for this change because it touches source files. Never pass `--sase-update-visual-snapshots` to `just check` or
`just test`.

Then confirm visually in a real terminal: set a `default` override with an effort and a non-`default` override with an
effort from the Models panel (leader `,m`, highlight an alias, `o`), and check the top bar reads as e.g.
`CLAUDE(opus)@xhigh 12h51m` `@smartest@max 12h51m` with the effort and countdown clearly recessive, that hovering each
pill shows its long form, and that clicking either opens the Models panel.

## Scope notes

- **No Rust core work.** Per the repo's core-backend boundary rule, this is presentation-only Textual rendering of state
  the Python layer already owns (`~/.sase/llm_override.json` via `sase.llm_provider.temporary_override`). The override
  _semantics_, the state schema, the effort vocabulary, and `format_provider_model_label` are all untouched - only how
  two widgets paint them changes. Do not open `../sase-core`.
- **No behavior change to overrides themselves**: precedence, persistence, expiry, self-cleaning reads, the Models
  panel, its duration/time pickers, and its own `override · 15m left` state chips are all out of scope and must not be
  edited.
- **The `@<effort>` suffix stays in the pill.** It answers the "why are we showing `@smartest@max`" question rather than
  deleting the answer: `@max` is the pinned reasoning effort, and removing it would make an effort-pinned override look
  identical to an unpinned one. It is disambiguated typographically instead.
- The gold pill's calm no-override state (`CLAUDE(opus)`, dim cyan) is deliberately untouched; it is not an override
  indicator and it is the baseline that every other top-bar PNG golden already pins.
