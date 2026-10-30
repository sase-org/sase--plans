---
tier: tale
title: "Make the exempt #PLAN type mark visible with a dedicated muted ink"
goal:
  "Exempt task types such as PLAN render a readable, distinct #TYPE mark in the #task
  slot on every lane tint."
size: small
decisions:
  ink:
    ask: "Which ink should exempt types like PLAN use for their #TYPE mark?"
    choices:
      slate:
        "Blue-gray: in the exempt gray family, yet apart from unconfigured names and
        body text"
      graphite:
        "Neutral gray: matches exempt pills, but only a dotted underline sets
        unconfigured names apart"
    default: slate
    why: Stays in the exempt gray family and still differs from every other type state
    answer: slate
proposed_by: bbugyi200.apollo.bj
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 18:57:05
status: wip
---

# Make the exempt `#PLAN` type mark visible with a dedicated muted ink

## Problem

On a task line such as `- [*] #task Spike S0: … [type:: PLAN] …` in the Bob vault, the
`bob-ledger-tools` Obsidian plugin replaces the `#task` tag with a `#TYPE` identity
mark. For `#PLAN` the mark takes up space, but its text is invisible, leaving a blank
gap before the line text (most obvious on the green Next-lane tint).

## Root cause

PLAN is an exempt type. `bob types list -q plan -f json` reports
`"hue": null, "hue_source": "exempt"`, so `taskTagIdentityKind` gives the mark
`data-bob-type-state="muted"` and no `data-bob-type-hue`. In
`plugins/bob-ledger-tools/styles.css` (bob-plugins repo), the muted rule is:

```css
body.bob-task-tag-marks .bob-task-tag-mark[data-bob-type-state="muted"] {
  --bob-task-tag-ink: color-mix(
    in srgb,
    var(--bob-task-tag-ink) 55%,
    var(--text-faint)
  );
}
```

A custom property that references itself on the same element is a dependency cycle.
Under the CSS Custom Properties spec, that makes `--bob-task-tag-ink` invalid at
computed-value time. The `::after` that draws `"#" attr(data-bob-type)` then uses
`color: var(--bob-task-tag-ink)`, which becomes `unset`. It inherits the host's
`color: transparent` (the host hides its own `#task` text), so the text draws
transparent. Every muted mark is affected, on every background and in both themes. The
green tint in the screenshot only made the gap easier to see.

There is no hue to "wash" for exempt types: they carry `hue: null`, and bob never emits
the `auto` hue source the plugin also maps to muted. The fix is a dedicated muted ink
token, not a different mixing formula.

## Repositories

No files change in the sase repo. All work is in two external repos. Open each with the
`/sase_repo` skill and use only the printed paths:

- `sase repo open gh:bobs-org/bob-plugins -r "Fix invisible exempt #TYPE task-tag ink"`
  (read its `AGENTS.md` first: `styles.css` is hand-edited, `main.js` is generated, and
  `bob plugins sync` is required after changes).
- `sase repo open bob-cli -r "Update the task-tag-marks display contract for the exempt muted ink"`

## Changes

### 1. bob-plugins `plugins/bob-ledger-tools/styles.css`

1. In the "Fixed R8 theme hues (PT6)" section, add a fixed muted ink to both theme
   blocks, directly after `--bob-type-ink-red`. Use the values for the chosen `ink`
   decision:

   | `ink` choice | `.theme-light` | `.theme-dark` |
   | ------------ | -------------- | ------------- |
   | `slate`      | `#52627a`      | `#9aa8bd`     |
   | `graphite`   | `#5a5a5a`      | `#a6a6a6`     |

   ```css
   --bob-type-ink-muted: <hex>;
   ```

   Extend that section's header comment by one sentence: exempt types have no palette
   slot, so they share this fixed muted ink, tuned to at least 4.5:1 on plain and on
   every lane tint. Measured minimums at the vault's 14% task-status tint: slate 5.07
   (light) and 4.86 (dark), graphite 5.64 (light) and 4.82 (dark).

2. Replace the cyclic muted rule body with:

   ```css
   --bob-task-tag-ink: var(--bob-task-tag-color, var(--bob-type-ink-muted));
   ```

   This keeps the documented `--bob-task-tag-color` theme hook winning, as in the eight
   hue rules above it.

3. Update the "Explicit `#TYPE` identity (D11)" comment above the `[data-bob-type]`
   rules to say that muted (exempt) marks use `--bob-type-ink-muted`. Change nothing
   else. Do not hand-edit `main.js`. No JS fragment changes, so `npm run build` is not
   needed, but `npm run build:check` must still pass.

### 2. bob-plugins `scripts/test-ledger-tools-task-type-identity.cjs`

1. **Cycle guard (regression test):** add a test that reads `styles.css` and fails on
   any custom property whose own declaration references it, for example with
   `/(--[\w-]+)\s*:[^;{}]*var\(\s*\1\s*[,)]/g`. Report every offender. First confirm
   that this test fails against the unmodified stylesheet (it catches the current muted
   rule), then that it passes after the fix.
2. **Muted ink contrast:** add a test that parses `--bob-type-ink-muted: #rrggbb` from
   the `.theme-light` and `.theme-dark` blocks (use the existing `parseThemeBlock`;
   assert that the token is present) and checks it is ≥ 4.5:1 against plain, Blocked, In
   Progress, and Next, in light and dark. Check both the existing 7% wash and the vault
   task-status snippet's 14% tint. Lift the `palettes` object from the existing D11
   contrast test into a shared helper so both tests use the same lane colors.
3. **Wiring:** extend the "styles.css defines type ink, identity ::after, and ledger
   marks" test to assert that the stylesheet contains `var(--bob-type-ink-muted)`.
4. **Exempt identity:** add a test that builds
   `taskTagIdentityFromField({ fieldState: "set", type: "PLAN", key: "plan" }, { hue: null, hue_source: "exempt", configured: true })`.
   Assert `state === "muted"` and `hue === null`. Then run `applyTaskTagTypeAttrs` on a
   `fakeElementDoc()` element and assert: `data-bob-type="PLAN"`,
   `data-bob-type-state="muted"`, and no `data-bob-type-hue`.

### 3. bob-plugins docs and version

- `docs/bob-ledger-tools.md`, "## Task type identity": add one sentence. Exempt types
  such as PLAN have no hue, so their `#TYPE` uses the muted ink (`--bob-type-ink-muted`)
  instead of a palette color.
- Bump `plugins/bob-ledger-tools/manifest.json` from `1.80.0` to `1.80.1`. Run
  `npm run docs:gen` to refresh the generated Version row, then
  `npm run docs:inventory:check`.

### 4. bob-cli `docs/task-tag-marks.md` (the display contract)

- In the type-ink paragraph, replace "Exempt and `auto`-muted hues use muted ink;" with
  a definition of the fixed `--bob-type-ink-muted` token (one hex per theme, defined on
  `.theme-light` / `.theme-dark` beside the slot inks; `--bob-task-tag-color` still
  wins). State that a node test holds it to ≥ 4.5:1 on the same lane matrix. Keep the
  unconfigured and repair clauses unchanged.
- Add a row to the type-identity table:
  `TI9 | - [*] #task Spike [type:: PLAN] (exempt, no hue) | #PLAN in the muted ink, readable on the Next tint`.
- Extend the "Contrast:" line to read "8 hue slots plus the muted ink × {plain, Blocked,
  In Progress, Next} × {light, dark} all ≥ 4.5:1".

## Verification

In the bob-plugins checkout:

```bash
npm run build:check
node --test scripts/test-ledger-tools-task-type-identity.cjs scripts/test-ledger-tools-task-tag-marks.cjs
npm test
npm run validate
BOB_CLI_DIR=<bob-cli checkout path printed by sase repo open> npm run docs:check
```

All must pass. `docs:check` runs bob-cli's documentation gate against both repos' edited
docs.

## Deploy

As the bob-plugins `AGENTS.md` requires, deploy to this host's vault from the opened
checkout. Do not pull and never use `--force`:

```bash
bob plugins sync --repo <bob-plugins checkout> --no-pull --dry-run
bob plugins sync --repo <bob-plugins checkout> --no-pull
```

Confirm that the dry run's diff only touches `bob-ledger-tools` `styles.css` and
`manifest.json`. If sync refuses (for example on provenance), stop and report its
message verbatim.

In the final response, tell Bryan that the screenshot came from the MacBook. `bob-*`
plugin files are gitignored in the vault, so the Mac picks up the fix only after this
change lands and `bob plugins sync` runs there, followed by reloading Obsidian or
toggling the plugin.

## Out of scope

- Ledger disagreement prefixes (`.bob-ledger-type-mark`) for exempt types. They carry no
  state attribute and fall back to the visible teal base color, so they are not broken.
- The vestigial `auto` hue source in `taskTagIdentityKind`.
- Pomodoro type pills, which already render exempt types as a visible gray pill.
