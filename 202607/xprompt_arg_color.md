---
create_time: 2026-07-14 07:35:43
status: wip
prompt: 202607/prompts/xprompt_arg_color.md
tier: tale
---
# Plan: Distinct color for xprompt arguments in the prompt input

## Goal

In the `sase ace` prompt input widget, an xprompt like `#gh:sase` currently renders the **name** (`#gh`) and its
**argument** (`:sase`) in the _identical_ color — the theme's green — differing only by font weight (name bold, arg
regular). The argument therefore does not read as a distinct token. The same problem applies to directives: `%m:opus`
renders the name (`%m`) and arg (`:opus`) in the identical amber.

Give xprompt **arguments** their own color that is clearly **distinct but not too different** from the xprompt **name**
color — a lighter, softer sibling of the same hue — so the name/argument relationship is instantly legible while
remaining harmonious and beautiful.

**Scope is the live editable prompt input** (the `PromptTextArea` the user types into — the surface in the reference
screenshot). The name colors are **not** changed; only the argument colors change.

## Background / current behavior

Two code paths highlight xprompt syntax, and they already disagree about argument color:

- **Live editable input** (`PromptTextArea`) — the user's target. Argument = **same** color as name. Defined in
  `src/sase/ace/tui/widgets/_xprompt_syntax_highlight.py`, method `_register_xprompt_text_area_theme`:

  ```python
  "xprompt.invocation":     Style(color=app_theme.success, bold=True),   # e.g. #gh
  "xprompt.invocation_arg": Style(color=app_theme.success),              # e.g. :sase  (SAME green)
  "xprompt.directive":      Style(color=app_theme.warning, bold=True),   # e.g. %m
  "xprompt.directive_arg":  Style(color=app_theme.warning),              # e.g. :opus  (SAME amber)
  "xprompt.separator":      Style(color=app_theme.secondary, dim=True, bold=True),
  ```

  These colors are pulled from `self.app.current_theme`, i.e. they are **theme-adaptive** (the app runs the `flexoki`
  dark theme, where `success` = `#66800B` olive-green and `warning` = `#AD8301` amber).

- **Read-only preview** (stash/history/preview panes) — `src/sase/ace/tui/util/xprompt_syntax.py`,
  `XPROMPT_TOKEN_STYLES`. This path **already** distinguishes the argument with a hand-tuned muted sibling (`invocation`
  `bold #87D787` → `invocation_arg` `#5FAF87`; `directive` `bold #FFD75F` → `directive_arg` `#D7AF5F`). **This path is
  left unchanged** by this plan.

The xprompt **tokenizer** (`src/sase/xprompt/xprompt_inspect.py`) already emits the argument as its own span kind
(`invocation_arg` / `directive_arg`), with the argument span including the leading colon (`:sase`). **No tokenizer
change is needed** — this is a pure recolor of existing spans. Per the repo's Rust Core Backend Boundary, this is
presentation-only styling and stays in Python; the Rust core is not touched.

## Design decisions (the design I'm leading)

### 1. Keep the argument color theme-derived (do not hardcode)

The live highlighter is deliberately theme-adaptive: there is a light-theme visual golden
(`prompt_xprompt_highlight_solo_light_120x40.png`, rendered under `textual-light`) and a unit test
(`test_xprompt_overlay_reregisters_after_app_theme_switch`) that re-derives the name color from `current_theme.success`
after a theme switch. Hardcoding a fixed hex for the argument would silently regress this adaptivity and look wrong
under a light theme. So the argument color is **derived from the name's base theme color at registration time**, keeping
it theme-adaptive.

No existing test pins the argument's exact value (the theme-switch test only asserts the _name_ color), so we are free
to choose the derivation.

### 2. Derivation recipe: blend the base ~40% toward the theme foreground

Argument color = **the name's base color blended ≈40% toward the theme's foreground color** (with a
background-brightness fallback when `foreground` is undefined).

Why this recipe:

- **Theme-correct on both polarities.** Blending toward the foreground raises contrast in the right direction
  automatically: on the dark flexoki theme the foreground is near-white, so the argument lightens (gaining contrast on
  the dark surface); on a light theme the foreground is dark, so it darkens. No per-theme branching on lightness.
- **Hue-preserving.** Blending toward the near-neutral foreground softens/desaturates but keeps the hue, so a green name
  keeps a green argument and an amber name keeps a gold argument. (A cool hue-rotation was prototyped and rejected: it
  turned the directive's gold argument sickly yellow-green.)
- **"Distinct but not too different."** The argument becomes a visibly lighter, softer tint of the same hue — clearly a
  different shade, unmistakably the same family. Combined with the existing bold-name / regular-arg weight contrast, the
  name stays primary and the argument reads as the value.

**Concrete flexoki-dark result** (blend factor 0.40 toward foreground `#FFFCF0`):

- invocation arg: `#66800B` (olive, name) → **≈`#A3B166`** (soft sage) for `:sase`
- directive arg: `#AD8301` (amber, name) → **≈`#CEB361`** (soft gold) for `:opus`

(Exact hex depends on the color library's rounding — pin the real computed value in the unit test during implementation.
Blend factor `0.40` is the recommended starting point; the visual verification step below may nudge it within
`0.35–0.45` if the real render calls for it.)

This choice was validated by rendering the name+argument pairs as swatches against the flexoki surface before writing
this plan; the sage/gold arguments are clearly distinct from and harmonious with the bold name colors, and markedly
better than the current identical-color state.

### 3. Keep the leading colon inside the argument color

The tokenizer's argument span includes the leading `:` (so the whole `:sase` recolors). This is kept as-is — the
recolored colon acts as a clean "argument begins here" delimiter cue. No tokenizer change, and it keeps the diff
minimal. (Splitting the colon into its own dim token is a possible future refinement but is intentionally out of scope.)

### 4. Explicitly out of scope

- Name colors (unchanged, per the request).
- The read-only preview path (`xprompt_syntax.py`) — it already differentiates arguments; leaving it avoids churn to its
  exact-value tests and unrelated goldens. A minor, acceptable cross-path divergence remains (live arg = lighter
  sibling; preview arg = darker sibling), each internally beautiful. Aligning the two paths could be a follow-up if
  desired.
- The xprompt completion dropdown rows and the `%{a|b}` alt-fan-out delimiters (different token families / surfaces, not
  arguments in typed text).

## Implementation outline

### Source

1. **`src/sase/ace/tui/widgets/_xprompt_syntax_highlight.py`**
   - Add a small, pure, module-level helper, e.g.:

     ```python
     def derive_argument_color(base: str, *, foreground: str | None, background: str) -> str:
         """Return a lighter/softer, theme-adaptive sibling of an xprompt name color.

         Blends the name's base color toward the theme foreground so the argument reads
         as a distinct-but-related tint while staying legible on light and dark themes.
         """
     ```

     Implemented with Textual's `textual.color.Color` (`Color.parse(...).blend(target, 0.40).hex`). Target =
     `foreground` when present, else white/black chosen by `background` brightness. Guard: if `base` is falsy, return it
     unchanged (match current no-color fallback behavior).

   - In `_register_xprompt_text_area_theme`, derive the two argument colors from `app_theme.success` /
     `app_theme.warning` (using `app_theme.foreground` / `app_theme.background`) and apply them:
     ```python
     "xprompt.invocation_arg": Style(color=<derived from success>),
     "xprompt.directive_arg":  Style(color=<derived from warning>),
     ```
     Names (`invocation`, `directive`) and `separator` are unchanged.

### Tests

2. **`tests/ace/tui/widgets/test_prompt_xprompt_highlight.py`**
   - Extend `test_xprompt_highlight_overlay_marks_spans_and_registers_styles` (or add a focused test) to assert the
     feature invariant: the registered `xprompt.invocation_arg` color is **different** from `xprompt.invocation`, and
     likewise `directive_arg` ≠ `directive` — while the name colors still equal `current_theme.success` / `.warning`.
3. **New unit test for `derive_argument_color`** (pure function):
   - flexoki-like base + light foreground → a lighter sibling; pin the exact computed hex.
   - dark-foreground (light-theme) case → a darker sibling (verifies polarity adaptivity).
   - result ≠ input; empty/None base → returned unchanged.

### Visual snapshots (PNG goldens)

4. Run the visual suite (`just test-visual`). The argument recolor legitimately changes goldens that render the **live**
   prompt text area containing an xprompt argument. Expected to change:
   - `prompt_xprompt_highlight_solo_light_120x40.png` (light theme)
   - `prompt_xprompt_highlight_stack_120x40.png`
   - Any `prompt_stack_*` goldens whose live pane shows `#gh:sase` / `%m:opus` (e.g. `prompt_stack_two_panes`,
     `prompt_stack_active_upper`, `prompt_stack_completion_panel`, `prompt_stack_g_prefix_hints`) — confirm per the diff
     artifacts.
   - Read-only-rendered goldens (`preview_panel_xprompt`, `prompt_history_*`, `stashed_prompts_*`) should **not**
     change; if one does, investigate before regenerating.
   - For each failing golden, inspect the actual/expected/diff artifacts in `.pytest_cache/sase-visual/`, confirm the
     only change is the argument color, then accept with `--sase-update-visual-snapshots`. **Eyeball each regenerated
     golden** to confirm the argument is distinct, harmonious, and legible (this is the "beautiful" gate; nudge the
     blend factor if needed and regenerate).

## Verification

- `just install` (ephemeral workspace) then `just check` (ruff + mypy + fast test suite, including the visual PNG
  suite).
- Manually render / open the prompt input with representative prompts (`#gh:sase`, `#git:home`, `%m:opus`,
  `#gh:sase %auto #pr:my_change %m:opus fix the bug`) under the default flexoki theme and confirm: arguments are a soft
  sage/gold clearly distinct from the bold name yet obviously the same family; separators and body text are unaffected;
  empty-arg cases like `#research_swarm::` are unaffected.

## Acceptance criteria

- In the live prompt input, xprompt/directive **arguments** render in a color distinct from their **name** color, but
  recognizably a sibling of it (same hue, lighter/softer).
- Name colors, separators, body text, and the read-only preview path are unchanged.
- The argument color remains theme-adaptive (derived from the active theme).
- `just check` passes; visual goldens regenerated and visually confirmed.
