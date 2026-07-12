---
create_time: 2026-07-11 12:55:27
status: done
prompt: 202607/prompts/models_panel_alias_references.md
tier: tale
---
# Models Panel: Surface Alias-to-Alias References

## Problem

The `sase ace` Models panel (leader `,m`) does not make it clear which model aliases are defined as a reference to
_another_ alias (i.e. whose configured value is `@<other_alias>`, such as `coder: "@default"`).

Today each alias row renders as one line:

```
<kind badge>   <alias name>      <PROVIDER(model) badge>        <state tag>
```

The rightmost **state tag** column shows provenance:

- `override · 15m left` / `override · until cleared` — a temporary override is active
- `configured` — the alias is an explicit config entry
- `implicit` — the implicit `default` alias
- `implicit → @coder` — an implicit `<provider>_coder` alias falling back to `@coder`
- `implicit → @default` — an implicit role alias falling back to `@default`

The gap: a **configured** alias that points at another alias (e.g. `coder: "@default"`, or a user alias
`blogger: "@default"`) renders exactly the same as a configured alias pinned to a concrete model (e.g.
`phase_worker: "codex/o3"`) — both just say `configured`, and the badge shows only the _resolved terminal_ model. From
the panel you cannot tell that `coder` _follows_ `default` rather than independently naming a model. If you later change
`default`, you cannot see from the panel which aliases move with it.

Notably, the panel **already** carries the raw reference in its data layer (`AliasView.configured_value`, e.g.
`"@default"`) — it is simply never rendered — and the state-tag column **already** speaks the `→ @alias` arrow idiom for
implicit fallbacks. This feature closes the gap by extending that same idiom to configured references, so alias-to-alias
links become visible and scannable.

## Goals

- Every alias configured as `@<other_alias>` visibly shows the alias it follows, on its existing single row (hard
  constraint: **one line per alias**).
- The reference reads intuitively and reuses the panel's existing visual language (the `→ @alias` arrow already used for
  implicit fallbacks).
- Every `@alias` reference token — configured _and_ implicit — is styled identically, so the eye can scan the state
  column and instantly catch every alias-to-alias link ("beautiful": one consistent, glowing reference accent).
- Reliable and pure: derived from data already resolved by the Python config layer; no guessing, no new failure modes,
  no Rust core change.

## Non-Goals

- **No multi-hop chain expansion on the row.** A row shows only its _immediate_ target (`coder → @default`), not the
  full transitive chain (`codex_coder → @coder → @default → opus`). Each alias is one row; the reader follows the chain
  by reading neighboring rows. Showing full chains per row would be noisy and could not stay on one line.
- **No panel resize.** The change fits comfortably in the current 110-cell container (worst-case row ≈ 92 of ~104 usable
  cells; the existing whole-row ellipsis remains the safety net for narrow viewports). We keep the panel its current
  size.
- **No override-row change.** When a temporary override is active it still wins the state column (`override · …`),
  because that is the _currently effective_ state the rest of the panel already reflects. The underlying configured
  reference is intentionally not shown while overridden.
- **No `sase-core` change.** Model-alias resolution (including `@<other_alias>` indirection) is entirely
  Python-resident; the Rust core does not model alias-to-alias references. This is a presentation + thin Python
  data-layer change. (Litmus test: the reusable `references` accessor lives in the Textual-agnostic `llm_provider` layer
  so a future CLI/web surface could reuse it, but it is derived from config the Python layer already owns — it does not
  belong behind the Rust binding.)

## Design

### Row layout (unchanged skeleton, richer state tag)

The four-column skeleton stays exactly as-is. Only the **state tag** column gains reference information for configured
references, mirroring the implicit form already in use:

Before (configured reference is indistinguishable from a concrete pin):

```
role     coder            CLAUDE(opus)           configured
role     phase_worker     CODEX(o3)              configured
```

After:

```
role     coder            CLAUDE(opus)           configured → @default
role     phase_worker     CODEX(o3)              configured
```

The resolved `PROVIDER(model)` badge is unchanged — it still shows the terminal model the alias actually runs
(`CLAUDE(opus)`). The new `→ @default` tells you _why_: `coder` follows `default`. Reading both, the user sees the
immediate pointer **and** the effective model on one line.

Chosen wording is **`configured → @target`** (keep the provenance word), not a bare `→ @target`, because:

- It is perfectly parallel to the existing `implicit → @target`, so the column reads as one learnable system.
- It preserves a genuinely useful distinction. A role alias like `coder` resolves to `@default` whether it is
  _configured_ `@default` or _implicitly_ falls back to it — same effective model, different provenance.
  `configured → @default` vs `implicit → @default` tells you whether the config line is pinning the reference or merely
  reinforcing the built-in fallback.

### Unified reference accent (the "beautiful" part)

Introduce one **reference accent** style applied to every `@alias` token in the state column — both configured
(`configured → @default`) and implicit (`implicit → @coder`, `implicit → @default`). The provenance word keeps its own
style (configured = green, implicit = dim), and the `→` is a dim connector, but the `@alias` target itself always
renders in the single reference accent (a link-like cyan in the panel's existing palette family). Result: scanning the
state column, every alias-to-alias link visually pops in the same consistent color, regardless of whether it was
configured or implicit.

To style the target token differently from the provenance word within one tag, `state_tag` returns a fully-styled Rich
`Text` (composed of multiple spans) rather than the current `(text, style)` tuple. This is a small, contained refactor;
the plain text of every existing case is unchanged, so only the call/return shape moves, not the visible wording of
non-reference rows.

### Data layer: a pure `references` accessor

Add a pure, reusable accessor to `AliasView` (in the Textual-agnostic `src/sase/llm_provider/alias_view.py`) that
reports the alias this one is _configured_ to follow:

- Returns the bare target alias name when `configured_value` is an `@<alias>` reference (using the documented `@`
  convention, consistent with the config resolver, the doctor check, and `%model` directive formatting).
- Returns `None` for a concrete model target (`claude/opus`) or an implicit alias (`configured_value is None`).

This keeps all reference detection derived from data the config layer already resolved (no re-parsing, no alias-set
lookup, no new I/O), and makes the concept unit-testable without any TUI. The renderer consumes it for configured rows;
implicit references continue to be derived by the renderer exactly as today (via alias kind/name), now routed through
the same accent-styling helper so the two sources render identically.

### Dangling references (deliberately lightweight)

A configured `@<typo>` that names no real alias resolves (per the existing resolver) to the bare name as a model, so
such a row already renders `configured → @typo` with a badge showing the literal `typo` model — the arrow plus the
odd-looking badge together signal the problem. The authoritative validation of unknown/retired references stays where it
already lives (the `doctor` config-alias check). We intentionally do **not** add alias-set validation into the render
path, keeping the renderer pure and free of new inputs.

## Files to change

- `src/sase/llm_provider/alias_view.py`
  - Add the pure `references` accessor to `AliasView` (immediate configured `@<alias>` target, else `None`).
- `src/sase/ace/tui/modals/models_panel_rendering.py`
  - Add a reference-accent style constant and a small helper that appends a styled ` → @<target>` reference to a Rich
    `Text`.
  - Change `state_tag` to return a fully-styled `Text`: configured-with-reference → `configured → @target`;
    configured-concrete → `configured`; implicit cases route their existing `→ @coder` / `→ @default` targets through
    the shared accent helper; override/`implicit`/`default` wording unchanged.
  - Update `render_alias_row` to append the styled state-tag `Text`.
- `src/sase/ace/tui/modals/models_panel.py`
  - No behavior change; keep the `state_tag as _state_tag` re-export intact (only its return type changes).

## Testing

- `tests/llm_provider/test_alias_view.py`
  - Add cases for the `references` accessor: `@default` → `"default"`; concrete `claude/opus` → `None`; implicit
    (`configured_value is None`) → `None`.
- `tests/test_models_panel.py`
  - Update the existing `_state_tag(...)` call sites to read `.plain` (return type is now a `Text`); existing plain-text
    expectations for `configured`, `implicit`, `implicit → @default`, `implicit → @coder`, and override tags stay the
    same.
  - Add a configured-reference case: configured alias with `configured_value="@default"` → plain
    `configured → @default`, and assert the `@default` span carries the reference accent style (and that an implicit
    `→ @coder`/`→ @default` target carries the _same_ accent, proving the unified styling).
  - The alignment/ellipsis row tests continue to pass (state tag is the final, ellipsis-tolerant column; the badge
    column that governs alignment is unchanged).
- `tests/ace/tui/visual/test_ace_png_snapshots_models_panel.py`
  - The calm fixture already includes `coder` configured as `@default`, so the default snapshot will now show
    `configured → @default` — exercising the new rendering with no fixture change required. Regenerate the affected PNG
    goldens with `--sase-update-visual-snapshots` and visually review that the reference reads cleanly, the accent is
    legible, and the state column still aligns.

## Verification

- `just check` (with `just install` first, per the ephemeral-workspace rule).
- Run the Models panel (`sase ace`, leader `,m`) and confirm at a glance that configured references (e.g. `coder`) show
  `configured → @default`, concrete pins (e.g. `phase_worker`) still show `configured`, implicit rows still show
  `implicit → @…`, and every `@alias` token — configured or implicit — shares the same reference accent.
