---
tier: tale
title: Tone the top-bar launch-default pill with its LLM provider's color
goal:
  The ACE top-bar launch-default pill paints its `<model>[@<effort>]` label in the
  provider palette SASE already uses for that provider's model text everywhere else, so
  the provider behind the current launch default is readable at a glance.
size: medium
proposed_by: bbugyi200.apollo.0s.f0.f0
create_time: 2026-09-20 15:50:22
status: wip
---

# Plan: Tone the top-bar launch-default pill with its LLM provider's color

## Goal

The calm launch-default pill in the ACE top bar currently renders its whole label —
`grok-4.6@high`, the shortest `%model` spelling plus the launch-effective effort — in
one flat `dim cyan`, identical for every provider. Paint it instead in the provider's
own hue, reusing the palette in `src/sase/ace/tui/provider_styles.py` that already
colors model text in the model picker, provider group headers, and the `PROVIDER(model)`
badge.

After this change the pill reads as `grok-4.6@high` in Grok cyan, `opus-5@high` in
Claude amber, `gpt-5.6-sol@high` in Codex mint — same glyphs, same width, provider
identity restored to the one surface that deliberately dropped the provider name.

## Design

### Why color belongs on this pill

The pill's subject was shortened from `GROK(grok-4.6)` to `grok-4.6` precisely because
the top bar is width-constrained. That shortening bought width by deleting the only
provider signal on the surface; `docs/ace.md` currently concedes as much ("hovering is
how you see which provider the launch default resolves to"). Color buys the signal back
for free: it costs zero columns, and the mapping is already learnable because every
denser model surface in ACE paints model names in exactly these hues.

### The two-tone grammar

`src/sase/ace/tui/widgets/_override_pill.py` already defines the top bar's pill grammar:
`<subject>[@<effort>] <trailing>`, subject in a primary foreground, effort and trailing
state in a recessive secondary foreground. The calm lane adopts the same two-tone split
rather than inventing a second grammar:

- **subject** (the shortest `%model` spelling) → the provider's `model_style`, the
  un-bolded, light, dark-background-tuned hue that `model_option_text()` and the model
  segment of `provider_model_badge_markup()` already use for model names. Not bold: the
  calm lane must stay calmer than the gold override pill, and hue alone is enough of a
  lift from the current `dim`.
- **`@<effort>` suffix** → the provider's `dim_style`, a recessive tone in the same hue
  family. The suffix keeps its subordinate weight, and the pill still reads as one
  object rather than two chips.

Concretely, a Grok default with `high` effort renders ` grok-4.6` in `#5FE3EF` followed
by `@high` in `dim #7FD4DD`, padded exactly as today.

### What deliberately does not change

- **The gold and violet override pills.** `_override_pill.py`'s lane accents encode
  _state_ — gold means "a temporary override is running the launch default", violet
  means "some other alias or launch setting is overridden". Provider hue encodes
  _identity_. Those are different axes and the background accent is the only carrier the
  state axis has, so it keeps it. The override lanes also still spell the provider out
  in full (`CODEX(o3) 1h2m`), so they never needed the color; and their foreground is
  near-black on a bright accent, where a provider hue would destroy contrast. The two
  lanes stay complementary: the calm lane is short text plus color, the override lane is
  long text plus accent.
- **Unresolved states.** `...` (resolving) and `unavailable` keep neutral `dim cyan`.
  The pill must never guess a provider hue before resolution lands — a hue that flickers
  from one provider to another as a worker finishes would be worse than no hue at all.
- **The tooltip.** It already states `Launch default: GROK(grok-4.6) @ high` in words.
  Color is therefore decorative and never the sole carrier of information, which is also
  the answer for color-vision-deficient users: the authoritative provider name is one
  hover (or `,m`) away, exactly as it is today.
- **Width, cadence, and resolution path.** No glyph is added or removed, the 5-second
  tick stays a pure peek, and the launch default is still resolved only off-thread.

### Theme handling, stated as a deliberate limit

`provider_styles.py` hues are fixed truecolor values, applied identically under light
and dark themes, across every ACE surface that uses them today. This plan matches that
convention rather than adding a light-theme lane for one pill: a light lane for the
provider palette is a whole-palette change, not a pill change, and inventing it here
would leave the pill disagreeing with the model picker two keystrokes away. Note for the
record that this does move the pill from the ANSI-adaptive `cyan` to fixed truecolor.
Reopen this if ACE gains real light-theme usage; the fix then belongs in
`provider_styles.py` for all callers at once, following the `dark=`-threading precedent
already in `src/sase/ace/tui/widgets/_provider_usage_indicator.py`.

### Unregistered providers get a derived hue, not a shared violet

`_provider_style_for()` looks a provider up in `_PROVIDER_FALLBACK_STYLES` and, when
plugin metadata supplies a primary color, overrides _only_ `name_style` with it. A
provider absent from that hardcoded table therefore keeps the neutral violet
`model_style` even when it declares a brand color. Today that is invisible, because all
eight registered providers (`agy`, `claude`, `codex`, `fakey`, `grok`, `muse`,
`opencode`, `qwen`) have table entries. It stops being invisible the moment this pill
ships, because the pill has no provider-name segment to carry the primary: two
third-party provider plugins would render identically.

So derive the whole palette from the declared primary when the table has no entry:
subject blended toward white for dark-background readability, detail as a dim primary.
This is a few lines in one function, has zero effect on any currently registered
provider, and keeps the invariant the feature depends on — one provider, one hue family,
everywhere.

### Not a feature flag

Per `sase/memory/sase_flags.md`, a flag covers an unproven beta behind an opt-in, an
unfinished epic phase reaching users, or a deprecation whose old branch must stay
reachable. This is a single-tale visual change with no old branch to preserve and
nothing for a user to choose forever, so it gets neither a feature flag nor a config
field.

## Implementation

### 1. Provider text palette (`src/sase/ace/tui/provider_styles.py`)

- Add a frozen, slotted public dataclass and accessor:

  ```python
  @dataclass(frozen=True, slots=True)
  class ProviderTextPalette:
      """Two-tone foreground palette for provider-colored plain text."""

      subject_style: str
      detail_style: str


  def provider_text_palette(provider: str | None) -> ProviderTextPalette:
      """Return the subject/detail hues for provider-colored plain text."""
  ```

  It returns `_ProviderStyle.model_style` as `subject_style` and
  `_ProviderStyle.dim_style` as `detail_style`. Returning the pair as one object (rather
  than two independent accessors) is deliberate: the two hues must always come from the
  same provider resolution, so they cannot drift apart at a call site.

- Harden `_provider_style_for()`:
  - Wrap the `provider_cli_status_color_map()` probe in `try/except Exception` and fall
    back to the built-in palette. This function now runs for a top-bar widget on every
    content build; a registry failure must degrade to neutral, never raise into a render
    path.
  - When `_PROVIDER_FALLBACK_STYLES` has no entry for the provider but metadata does
    supply a primary, build the palette from that primary instead of returning
    `_NEUTRAL_PROVIDER_STYLE`. Split the current `_with_primary()` into the existing
    "table entry + plugin primary name" path and a new `_derived_style(primary)` that
    produces all five roles from one brand color — use `rich.color.Color.parse(...)` and
    `.blend(...)`, following `_bullet_dash_color()` in
    `src/sase/ace/tui/widgets/_bullet_highlight.py`. Pick blend ratios that keep the
    derived `model_style` clearly lighter than the primary and the derived `dim_style`
    recessive; state the chosen ratios as named module constants rather than inline
    literals.
  - Providers that are unknown _and_ colorless keep `_NEUTRAL_PROVIDER_STYLE`.

### 2. Calm-lane pill builder (`src/sase/ace/tui/widgets/_override_pill.py`)

- Broaden the module docstring: it owns the rendering grammar for _all_ ACE top-bar
  model pills, of which the override lanes are one kind. Keep it explicit that the
  override lanes carry a lane accent background and the calm lane does not.
- Add a builder beside `build_override_pill()`:

  ```python
  def build_calm_default_pill(
      *,
      subject: str,
      effort: str | None,
      palette: ProviderTextPalette,
  ) -> Text:
      """Build the backgroundless ``<subject>[@<effort>]`` launch-default pill."""
  ```

  It mirrors `build_override_pill()`'s shape exactly — ` {subject}` in
  `palette.subject_style`, then `@{effort}` in `palette.detail_style` when effort is
  set, then the trailing pad — so the two lanes provably share one grammar. Put the
  leading and trailing padding spaces in the subject style; with no background they are
  invisible either way, and it keeps `Text.style` meaningful for tests.

### 3. Widget wiring (`src/sase/ace/tui/widgets/llm_override_indicator.py`)

- Give `_LaunchDefaultSnapshot` a `palette: ProviderTextPalette | None = None` field. A
  default of `None` is required: existing tests construct this dataclass positionally
  and by keyword without the new field.
- Resolve the palette **inside the off-thread worker** in
  `_schedule_default_resolution_if_needed()`, next to where `directive_label` is already
  precomputed for the same reason. `provider_cli_status_color_map()` is
  `functools.cache`d, but the first call walks plugin metadata; per
  `sase/memory/tui_perf.md` rule 1 and rule 8 that walk must not be able to land on the
  UI thread. After this, `_build_cached_default_content()` does pure string and `Text`
  work.
- `_build_cached_default_content()` calls `build_calm_default_pill()` with the cached
  palette. When `_cached_snapshot` is `None` (a `_cached_default` tuple with no
  snapshot), fall back to a neutral `ProviderTextPalette` built from `_DEFAULT_STYLE`,
  preserving today's rendering for that path. Placeholder and unavailable text keep
  `_DEFAULT_STYLE` untouched.
- `_build_default_content()` (the synchronous path kept for tests) resolves the palette
  inline inside its existing `try` block, so a palette failure degrades to `unavailable`
  exactly like any other resolution failure.
- Leave `_format_default_tooltip_label()`, `_build_tooltip()`,
  `_build_override_content()`, and every cache/token/re-arm path untouched.

### 4. Tests

Do **not** grow `tests/test_llm_override_indicator.py`: it is at 832 lines against a
`toobig` warning tier of 850. Add new cases in new sibling modules.

- Update the four existing `cyan` assertions in `tests/test_llm_override_indicator.py`:
  - line ~357 (`...` placeholder) **keeps** its `cyan` assertion — that is the
    regression guard for "no hue before resolution".
  - lines ~140, ~598, ~704 assert resolved content for a `codex` default and must now
    assert the Codex subject hue instead. Assert against
    `provider_text_palette("codex").subject_style` rather than a copy-pasted hex
    literal, so the palette stays the single source of truth.
- New `tests/test_provider_text_palette.py`:
  - a registered provider returns its table `model_style` / `dim_style`;
  - two different registered providers return different `subject_style` values (the
    property the whole feature rests on);
  - a provider absent from the fallback table but carrying a metadata color returns a
    palette derived from that color, distinct from the neutral palette and from another
    such provider's;
  - an unknown, colorless provider and `None` return the neutral palette;
  - a `provider_cli_status_color_map()` that raises yields the neutral palette and does
    not propagate.
- New `tests/test_llm_override_indicator_provider_color.py`:
  - the calm pill paints its subject in the provider hue and `@<effort>` in the detail
    hue, asserted through `text.spans` the way `test_active_override_renders_effort`
    already asserts the gold pill;
  - two providers produce two different pill subject styles for the same model name;
  - `_build_cached_default_content()` renders from the cached palette without touching
    the registry — monkeypatch `provider_text_palette` in the widget module to raise and
    assert the cached pill still renders;
  - a `_cached_snapshot` of `None` still renders the neutral pill;
  - a regression assertion that the gold override pill's styles are unchanged.
- New `tests/ace/tui/visual/test_ace_png_snapshots_launch_default_pill.py`: two 80x24
  goldens of the top bar with the launch default pinned to two different providers
  (Claude and Grok read most distinctly), using `quiet_top_bar()` from
  `tests/ace/tui/visual/_provider_usage_indicator_fixtures.py` to silence the
  neighboring indicators. These two goldens are the visual proof that the hues are
  actually distinct on screen.

### 5. Golden churn (expect it; it is the largest mechanical step)

`tests/ace/tui/visual/_ace_png_snapshot_startup.py` pins every snapshot's launch default
to `codex/visual-snapshot-model`, so the pill turns Codex mint in **every** existing
golden whose top bar is visible — a large fraction of the 680 files under
`tests/ace/tui/visual/snapshots/png/`.

- Run `just fix-tui-screenshots` through `/sase_monitor` with a generously raised
  timeout; a full mutating run also does a bounded verification pass.
- Then inspect the retained report per `sase/memory/tui_screenshot.md`: generation is
  not approval. The expected signature is uniform — in every update group the only
  differing region is the top-right pill's hue, with identical glyphs and identical
  width. Any golden that differs anywhere else, or whose pill text changed, is a real
  regression: stop and fix the code rather than accepting the golden.
- Inspect every creation and removal individually; the only expected creations are the
  two new pill goldens from step 4.

### 6. Docs

- `docs/ace.md` (the launch-default paragraph around line 4080): replace "in a calmer
  dim-cyan tone" with the provider-toned description, and correct the closing sentence
  that says hovering is how you see which provider the default resolves to — hover is
  now the authoritative confirmation, not the only signal. Note explicitly that a
  cross-provider `|` pool makes the pill's hue flip as the round-robin cursor advances,
  so the color tells you which provider the _next_ launch will actually hit. Keep the
  existing statement that the override lanes' accents carry the override meaning, and
  add that provider hue is the calm lane's identity axis.
- `docs/llms.md` (the launch-default pill paragraph around line 1694): extend the
  `<shortest %model value>[@<effort>]` description with the provider-toned rendering and
  the unchanged `PROVIDER(model)` tooltip.
- Do **not** hand-edit `CHANGELOG.md`; it is generated from the conventional commit.

## Verification

1. `just install` first if the workspace virtualenv is stale, then `just fix` inline.
2. `sase tool run check` (the agent default recipe). Do **not** run `just check-full`:
   `sase/memory/lint_and_test.md` and `decisions:check-full-is-explicit` make it
   explicit-instruction-only, and nothing here is a CI repair.
3. `just fix-tui-screenshots` through `/sase_monitor` with the `TESTING` / `TESTED`
   status pair and a raised timeout, followed by the report and golden inspection
   described in step 5.
4. Confirm the real surface with a live capture, since this change is entirely about
   pixels: `sase screenshot -o /tmp/pill.png` and read the top-right pill's hue out of
   the PNG. Repeat with `llm_provider.default_model` pointed at a second provider and
   confirm the hue actually changes.

## Done when

- The calm launch-default pill renders its subject in the provider's model hue and its
  `@<effort>` suffix in the provider's dim hue, and two different providers are visibly
  different in the top bar.
- `...`, `unavailable`, the gold default-override pill, and the violet alias pill are
  byte-for-byte unchanged.
- The pill still never resolves anything on the UI thread and still never advances a
  pool cursor.
- A provider that declares a metadata color but has no built-in palette entry gets its
  own hue family rather than the shared neutral violet.
- `sase tool run check` passes, PNG goldens are regenerated and individually inspected,
  and `docs/ace.md` plus `docs/llms.md` describe the pill as it now renders.
