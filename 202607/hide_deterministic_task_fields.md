---
tier: tale
title: Hide deterministic dependsOn/id task fields behind a muted glyph
goal: "Task lines in the Bob vault read as prose again: the machine-derived [id::] pill
  disappears entirely, and the [dependsOn::] pill collapses to a quiet, theme-aware link
  glyph that reveals its value on hover without reflowing the line.

  "
create_time: 2026-09-09 19:53:15
status: wip
---

# Plan: Hide deterministic `dependsOn` / `id` task fields

## Problem

Every task line in the vault carries two Dataview inline fields whose values are derived
mechanically and never read by a human:

- `[id:: <file>__<blockid>]` — duplicates the `^blockid` caret already visible at the
  end of the same line.
- `[dependsOn:: <id>, <id>]` — duplicates the transcluded block links rendered directly
  beneath the task.

`.obsidian/snippets/dataview-properties.css` renders both as full pills (a small-caps
key chip plus a value chip). On a task line they are the loudest thing present — a
`DEPENDSON sase_cmd__sase-gate` pill visually outweighs the task text it belongs to, and
lines wrap early because the pills eat horizontal space.

The goal is presentation only. Nothing about parsing, indexing, or storage changes.

## Decisions (settled with the user)

| Question              | Decision                                                                                           |
| --------------------- | -------------------------------------------------------------------------------------------------- |
| `id` rendering        | Hide completely. The `^caret` is the icon.                                                         |
| `dependsOn` rendering | Muted, theme-aware glyph; hover reveals the value in an absolutely-positioned tooltip (no reflow). |
| CSS location          | Extend the vault snippet `.obsidian/snippets/dataview-properties.css`.                             |
| Drift checking        | Out of scope. Not worried; `!` reconciles.                                                         |

An emoji (`⛔`) was explicitly rejected: it cannot be recoloured and ignores the
light/dark theme, and every colour in this stylesheet is a `color-mix(var(--...))`. The
glyph must inherit `--text-muted` and recede.

## Research findings that constrain the implementation

These were verified against the live vault and the bundled plugin source. They are the
difference between a working snippet and a broken one — do not skip them.

1. **The normalized key is lowercase.** Dataview writes `data-dv-norm-key` via
   `canonicalizeVarName()`, which runs `VAR_NAME_CANONICALIZER` →
   `.toLocaleLowerCase()`. So `dependsOn` must be matched as **`dependson`**. Matching
   `"dependsOn"` silently matches nothing.

2. **Do NOT hide the frontmatter `id` property.** The existing `h` rule at the bottom of
   the file hides both the inline field _and_
   `.metadata-property[data-property-key="h"]`. Copying that shape for `id` looks
   natural and is a real bug: **59 notes carry a meaningful frontmatter `id:`**
   (`id: obsidian`, `id: needs_attn_tasks`, …) that shows in the Properties panel. The
   `h` rule got away with the metadata selector only because zero notes have a
   frontmatter `h:`. The new `id` rule must target inline fields **only**.

3. **Do not scope the rules to task list items.** `[id::]` and `[dependsOn::]` also
   appear on plain non-checkbox bullets (e.g. `sase.md:72`, `sase_model.md:21`). Scoping
   to `li.task-list-item` would leave those pills rendered. Use the same unscoped
   `.dataview.inline-field` selectors the `h` rule uses.

4. **The base rule clips the tooltip.** `.dataview.inline-field` sets `overflow: hidden`
   (line ~49). An absolutely-positioned tooltip child will be invisible until the
   `dependsOn` container overrides `overflow: visible`.

5. **`:has()` is already proven here.** The file uses `:has()` in both the
   standalone-value rule and the `h` hide rule, so Obsidian's Chromium supports it.

6. **Never select the value element by `data-dv-norm-key` — it is not always there.**
   Dataview has two inline-field render paths and they attach attributes _differently_:
   - The Live Preview / CodeMirror widget path builds the value as
     `cls: ["dataview","inline-field-value"], attr: { "data-dv-key", "data-dv-norm-key" }`.
   - The Reading-view post-processor path builds the value as
     `cls: ["dataview","inline-field-value"], attr: { id: "dataview-inline-field-" + x }`
     — **no `data-dv-norm-key` and no `data-dv-key` at all.**

   So a rule like `.dataview.inline-field-value[data-dv-norm-key="dependson"]` works
   while the cursor is elsewhere in Live Preview and then silently fails in Reading
   view, leaving a glyph sitting next to a bare, unstyled dependency value — the worst
   possible outcome, and one that is easy to miss because Live Preview looks perfect.
   **The key element carries `data-dv-norm-key` in both paths**, so every rule below
   anchors on the parent via
   `:has(> .dataview.inline-field-key[data-dv-norm-key="dependson"])` and reaches the
   value as a plain child. This is also why the existing `h` rule defensively lists the
   key, value, and standalone-value variants.

7. **Transclusions are a different surface and stay as-is.** The Tasks plugin is
   configured `taskFormat: "dataview"` (it _parses_ `[id::]`/`[dependsOn::]`), but its
   own renderer re-serializes task lines using its emoji symbol table
   (`idSymbol: "\u{1F194}"` 🆔, `dependsOnSymbol: "⛔"` ⛔). Those embedded lines are
   **not** `.dataview.inline-field` elements, so this snippet does not and cannot touch
   them. Source lines will show the quiet glyph while transclusions keep showing ⛔/🆔.
   This divergence is a known, accepted consequence of choosing the muted glyph over the
   emoji.

## Design

Add to `.obsidian/snippets/dataview-properties.css`, following the file's existing
organization: design tokens go in the top-level `body { … }` block, rules go at the
bottom next to the `h` precedent. Follow house style — **every `var()` gets a literal
fallback**, and colours are built with `color-mix()`.

### Tokens (append inside the existing top `body` block)

```css
--dataview-dep-glyph: var(--text-muted, #777777);
--dataview-dep-glyph-hover: var(--dataview-property-accent);
--dataview-dep-glyph-size: 0.85em;
/* Lucide `link-2`, inlined so it needs no plugin and no network. */
--dataview-dep-glyph-icon: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%23000' stroke-width='2.25' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpath d='M9 17H7A5 5 0 0 1 7 7h2'/%3E%3Cpath d='M15 7h2a5 5 0 1 1 0 10h-2'/%3E%3Cpath d='M8 12h8'/%3E%3C/svg%3E");
```

The glyph is painted with `background-color: var(--dataview-dep-glyph)` masked by that
SVG — **not** rendered as a font character. This is what makes it beautiful and
theme-correct: it is a crisp vector at any zoom, it inherits `--text-muted` exactly, it
can transition to the accent on hover, and it can never be substituted by a colour emoji
font.

### Rules (append after the `h` block)

```css
/* ── Machine-derived task fields ───────────────────────────────────────────
   [id::] and [dependsOn::] are both derived deterministically (block id and
   transcluded block links), so their pills are noise on every task line.
   Dataview lowercases inline-field keys into data-dv-norm-key, so `dependsOn`
   is matched here as `dependson`.
   NOTE: unlike the `h` rule above, the `id` rule deliberately omits
   .metadata-property[data-property-key="id"] — many notes carry a real
   frontmatter `id:` that must keep rendering in the Properties panel. */

/* id: erased. The trailing ^block-id caret already marks the line. */
.dataview.inline-field:has(> .dataview.inline-field-key[data-dv-norm-key="id"]),
.dataview.inline-field:has(> .dataview.inline-field-value[data-dv-norm-key="id"]),
.dataview.inline-field:has(
    > .dataview.inline-field-standalone-value[data-dv-norm-key="id"]
  ) {
  display: none !important;
}

/* dependsOn: the pill collapses to a single muted glyph. */
.dataview.inline-field:has(> .dataview.inline-field-key[data-dv-norm-key="dependson"]) {
  position: relative;
  margin: 0 0.14em;
  border: none;
  background: none;
  box-shadow: none;
  overflow: visible; /* base .inline-field clips; the tooltip must escape */
  cursor: help;
}

.dataview.inline-field:has(
    > .dataview.inline-field-key[data-dv-norm-key="dependson"]
  )::before {
  content: "";
  display: inline-block;
  width: var(--dataview-dep-glyph-size);
  height: var(--dataview-dep-glyph-size);
  vertical-align: -0.12em;
  background-color: var(--dataview-dep-glyph);
  -webkit-mask-image: var(--dataview-dep-glyph-icon);
  mask-image: var(--dataview-dep-glyph-icon);
  -webkit-mask-size: contain;
  mask-size: contain;
  -webkit-mask-repeat: no-repeat;
  mask-repeat: no-repeat;
  -webkit-mask-position: center;
  mask-position: center;
  opacity: 0.7;
  transition:
    opacity 120ms ease,
    background-color 120ms ease;
}

.dataview.inline-field:has(
    > .dataview.inline-field-key[data-dv-norm-key="dependson"]
  ):hover::before {
  background-color: var(--dataview-dep-glyph-hover);
  opacity: 1;
}

/* The key chip is gone; the glyph speaks for it. */
.dataview.inline-field:has(> .dataview.inline-field-key[data-dv-norm-key="dependson"])
  > .dataview.inline-field-key {
  display: none;
}

/* The value is permanently out of flow, so revealing it never reflows text.
   Anchored on the parent, NOT on the value's own data-dv-norm-key: the
   Reading-view render path does not put that attribute on the value span. */
.dataview.inline-field:has(> .dataview.inline-field-key[data-dv-norm-key="dependson"])
  > .dataview.inline-field-value {
  position: absolute;
  bottom: calc(100% + 0.35em);
  left: 50%;
  z-index: var(--layer-tooltip, 50);
  transform: translateX(-50%) translateY(0.15em);
  max-width: min(40ch, 60vw);
  padding: 0.25em 0.55em;
  border: 1px solid var(--background-modifier-border, rgba(127, 127, 127, 0.18));
  border-radius: var(--radius-s, 4px);
  background: var(--background-secondary, #f5f5f5);
  box-shadow: var(--shadow-s, 0 2px 8px rgba(0, 0, 0, 0.15));
  color: var(--text-normal, #222222);
  font-family: var(--font-interface);
  font-size: var(--font-ui-smaller, 0.75em);
  white-space: normal;
  overflow-wrap: anywhere;
  opacity: 0;
  pointer-events: none;
  transition:
    opacity 120ms ease,
    transform 120ms ease;
}

.dataview.inline-field:has(
    > .dataview.inline-field-key[data-dv-norm-key="dependson"]
  ):hover
  > .dataview.inline-field-value {
  opacity: 1;
  transform: translateX(-50%) translateY(0);
}
```

### Why this shape

- **Intuitive** — a link glyph reads as "this connects to something", and it sits
  exactly where the dependency information always was. Hovering is the universal gesture
  for "tell me more", and `cursor: help` advertises it.
- **Reliable** — no plugin dependency, no font dependency, no network. Editing is
  untouched: in Live Preview, moving the cursor onto the line unrenders the Dataview
  widget and shows the raw `[dependsOn:: …]` text natively, so the values remain
  directly editable. Dataview indexing, Tasks parsing, and `bob query` are all
  unaffected because CSS cannot reach them.
- **Beautiful** — the tooltip is out of flow, so nothing shifts on hover; the glyph is a
  masked vector that inherits the theme's muted colour and warms to the stylesheet's own
  `--dataview-property-accent` on hover, matching the existing pill language rather than
  fighting it.

## Non-goals

- No drift detection between `dependsOn` and the transclusions (explicitly deferred).
- No change to frontmatter `id:` rendering.
- No change to how transclusions render (`⛔`/`🆔` come from the Tasks plugin).
- No changes to `bob-cli`, `bob-plugins`, or any plugin. This is a vault snippet only.

## Implementation steps

1. Open the vault repo — it is **not** in this project's inventory, so open it as an
   external GitHub repo and use only the path it prints:

   ```bash
   sase repo open gh:bobs-org/bob -r "Hide deterministic dependsOn/id inline fields"
   ```

2. Edit `.obsidian/snippets/dataview-properties.css` in that checkout: add the tokens to
   the existing top `body` block and the rules after the existing `h` hide block.

3. Commit **only** that file, with `/sase_git_commit`.

## Verification

There is no headless render path for this: `ob` only does Obsidian Sync (`sync`,
`publish`, `login`), it cannot rasterize a note. So verification is static plus a visual
confirmation by the user.

Static checks the implementing agent must do:

- Confirm the selector uses `dependson` (lowercase), not `dependsOn`.
- Confirm `.metadata-property[data-property-key="id"]` is **absent**.
- Confirm the `dependsOn` container sets `overflow: visible`.
- Confirm **no rule selects `.dataview.inline-field-value` by `data-dv-norm-key`**
  (finding #6). Every `dependsOn` rule must anchor on
  `:has(> .dataview.inline-field-key[data-dv-norm-key="dependson"])`. A quick guard:
  `grep -n 'inline-field-value\[data-dv-norm-key="dependson"\]'` must return nothing.
- Confirm no `var()` was introduced without a literal fallback (house style).
- Sanity-check that the data-URI SVG has its `<`, `>`, `#` percent-encoded, and is
  wrapped in double quotes with single quotes inside.

Then, in GUI Obsidian (snippets hot-reload on save; `dataview-properties.css` is already
enabled, so no toggle is needed), confirm on a note like `sase_cmd.md`:

- Task lines show no `ID` pill and no `DEPENDSON` pill.
- A single muted glyph sits where the `dependsOn` pill was; hovering reveals the ids and
  **nothing on the line moves**.
- The glyph is legible and recedes in both light and dark themes.
- Clicking into the line still shows raw `[dependsOn:: …]` text for editing.
- **Check Reading view explicitly, not just Live Preview** (Ctrl/Cmd-E). This is the
  render path that omits `data-dv-norm-key` from the value span (finding #6); if a rule
  regressed to selecting the value directly, Reading view is where it shows up as a
  glyph followed by raw dependency text.
- A note with a frontmatter `id:` (e.g. `obsidian.md`) still shows `id` in its
  Properties panel — this is the regression guard for finding #2.

## Delivery note (needs a decision from the user)

`sase repo open` prepares a **separate clone**, not the live `~/bob`. Both are currently
at `master @ 8d974fd`. Committing in the clone therefore does not make the change
visible in Obsidian; the live vault has to receive it.

`~/bob` currently has ~33 unrelated modified files (plugin/sync churn — obsidian-linter,
obsidian-tasks-plugin, …). Per the vault's `AGENTS.md`, those must not be reverted,
staged, or committed. They do not touch `.obsidian/snippets/`, so pulling the snippet
commit into `~/bob` should apply cleanly without disturbing them.

The implementing agent should confirm with the user how the commit should reach the live
vault (push + `git -C ~/bob pull`, or the user's own sync routine) rather than assuming.

## Risks

- **Low — tooltip clipped near the top of the viewport.** The tooltip renders above the
  glyph. On the very first visible line it could be cut off. Accepted; the value is
  still reachable by clicking into the line.
- **Low — tooltip centring near the left margin.** `translateX(-50%)` could push a long
  tooltip past the left edge for a glyph at the very start of a line. Accepted.
- **Low — `mask-image` support.** Obsidian's Chromium supports it; both prefixed and
  unprefixed properties are declared. If the glyph ever renders as a solid block, the
  mask failed and the fix is to check the data-URI encoding first.
- **Reversibility — total.** Deleting the added block restores the pills exactly.
  Nothing on disk or in any index is modified.
