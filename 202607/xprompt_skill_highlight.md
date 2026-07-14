---
create_time: 2026-07-14 07:59:20
status: wip
prompt: 202607/prompts/xprompt_skill_highlight.md
tier: tale
---
# Plan: Syntax highlighting for known xprompt skills (`/sase_plan`) in the prompt input

## Goal

In the `sase ace` live prompt input (`PromptTextArea`), a reference to a known agent skill — e.g. `/sase_plan`,
`/sase_repo`, `/sase_git_commit` — currently renders as plain, unstyled text. Give **known** skill references their own
distinct, beautiful, theme-adaptive color so the user can instantly see, while typing, that a `/name` token is a real,
invocable skill (and, just as usefully, that a mistyped `/sase_pln` is **not**).

The feature must be:

- **Intuitive** — a `/name` lights up exactly when the completion menu would offer it, because both are driven by the
  same catalog. If `/` autocomplete knows the skill, the highlighter colors it. No surprises.
- **Reliable** — only _known_ skills highlight. Unknown slashes (absolute paths `/usr/bin`, URLs, typos) stay plain, so
  a highlight is always a truthful "this skill exists" signal.
- **Beautiful** — a bold, harmonious hue that completes the existing prompt palette (green xprompts, amber directives,
  blue/cyan separators) without clashing.
- **Fast** — zero new per-keystroke disk I/O. The known-skill set is read from an in-memory cache the widget already
  maintains for `/` completion.

**Scope is the live editable prompt input only** (the `PromptTextArea` the user types into). The read-only preview path
is intentionally left unchanged (see Out of scope).

## Background / current behavior

Three facts from the codebase shape this design:

1. **A skill _is_ an xprompt.** The skills in `src/sase/xprompts/skills/*.md` (`sase_plan.md`, `sase_repo.md`, …) are
   ordinary xprompts whose frontmatter sets `skill: true` (`src/sase/xprompt/models.py:169`). The canonical name is the
   frontmatter `name:` (e.g. `sase_plan`). The authoritative "is this name a skill?" test everywhere is
   `bool(xprompt.skill)`, surfaced as `XPromptAssistEntry.is_skill`
   (`src/sase/ace/tui/widgets/_xprompt_arg_assist_models.py:33`).

2. **`#name` and `/name` are the same object, consumed differently.**
   - `#sase_plan` is an xprompt reference that SASE **expands/inlines at launch** (`src/sase/xprompt/processor.py`).
   - `/sase_plan` is a **skill reference passed through as literal text** to the agent runtime, which loads the
     installed `SKILL.md` and runs it. Launch processing never rewrites `/name`. This is why skill highlighting is a
     pure editor affordance — it colors true references without changing any launch semantics.

3. **A `/`-skill token shape and a disk-free known-skill source already exist — for completion.**
   - Token shape: `_SLASH_SKILL_TOKEN_RE = re.compile(r"^/[A-Za-z0-9_]*$")`
     (`src/sase/ace/tui/widgets/xprompt_completion.py:13`); `build_xprompt_completion_candidates` already filters `/`
     tokens to `entry.is_skill` and inserts `/{entry.name}` (`xprompt_completion.py:42,50-56`).
   - Known-skill source: `PromptTextArea` already keeps a warm, per-project catalog of `XPromptAssistEntry` in memory
     (`self._xprompt_arg_assist_entries_by_project`), read disk-free via `_get_warm_xprompt_arg_assist_entries()`
     (`src/sase/ace/tui/widgets/_xprompt_arg_hints.py:166`). Cold catalogs **defer** rather than loading synchronously
     (the completion path is deliberately "warm-only"). Disk loads happen only off-thread when prompt-source files
     change — never per keystroke.

   **The highlighter will consume this exact same warm cache and `is_skill` filter**, guaranteeing highlight and
   autocomplete never disagree.

Two existing highlight paths handle xprompt syntax; both call the shared tokenizer `xprompt_inspect.tokenize`
(`src/sase/xprompt/xprompt_inspect.py`), which emits kinds
`invocation | invocation_arg | directive | directive_arg | separator`. It has **no** `/name` handling today. The live
overlay (`src/sase/ace/tui/widgets/_xprompt_syntax_highlight.py`) registers theme styles and appends spans; the
read-only preview (`src/sase/ace/tui/util/xprompt_syntax.py`) styles the same spans for stash/history/preview panes. Per
the Rust Core Backend Boundary, this is presentation-only styling that stays in Python; the known-skill _data_ already
comes from existing in-process catalog machinery, so **no Rust core change is needed** (consistent with the existing
xprompt overlay being Python).

## Design decisions (the design I'm leading)

### 1. Highlight only _known_ skills — this is what makes `/`-highlighting safe and truthful

`/` is extremely common in prose (paths `src/foo`, URLs `https://…`, dates `07/14`, "and/or"). Highlighting every
`/word` would be noise and would mis-fire constantly. Two guards make the feature reliable:

- **Leading-context guard** (reused from the xprompt grammar, `_parsing_references.XPROMPT_REFERENCE_LEADING_CONTEXT`):
  a skill reference may only begin at start-of-line, after whitespace, or after an opening `(` `[` `{` `"` `'`. This
  alone rejects mid-path/mid-word slashes (`research/sase_plan`, `https://x/sase_plan`).
- **Known-set membership**: the matched name must be in the warm known-skill set. Absolute paths (`/usr/bin`), unknown
  commands, and typos (`/sase_pln`, `/sase_planner` when only `sase_plan` exists) stay plain.

Because a highlight can _only_ appear for a real skill, the color carries real meaning and the feature never "cries
wolf." This is the core of "reliable" and "intuitive."

### 2. Put the grammar in the shared tokenizer, gated by a `known_skills` parameter (keeps it pure and fast)

Add an optional `known_skills: frozenset[str] = frozenset()` parameter to `xprompt_inspect.tokenize` and a new `"skill"`
span kind. When `/` is present **and** `known_skills` is non-empty, scan for `/name` references (leading-context
grammar), keep those whose name is in `known_skills`, and emit a `skill` span covering the whole `/name` (slash
included, mirroring how `#gh` includes its `#`). The scan **reuses the fenced-block / disabled-region protection already
computed once** in `tokenize`, so a `/sase_plan` inside a ```fence or a`%xprompts_enabled:false` region is correctly
skipped — same behavior as xprompt tokens.

Why the tokenizer (not a bespoke widget scan):

- **Purity & testability** — `tokenize` stays a pure function with no disk/widget dependency (the caller supplies the
  set), so the grammar is unit-testable without a running TUI.
- **One pass, shared protection** — no second computation of fenced/disabled ranges on the hot path.
- **Single grammar home** — one definition of "where a reference may start" and "how a `/name` is delimited."

The default empty `known_skills` means the read-only preview and every other caller are **behavior-unchanged**: no
`skill` spans are emitted unless a caller opts in by passing a non-empty set.

### 3. Source the known-skill set from the widget's warm cache — no new disk I/O

The live overlay (`XPromptSyntaxHighlightMixin`, which is mixed into `PromptTextArea`) computes `known_skills` from
`self._get_warm_xprompt_arg_assist_entries()`, filtering `entry.is_skill` to a `frozenset` of names, and passes it into
`tokenize`. Properties this gives us for free:

- **Disk-free** — pure in-memory read of the same warm cache `/` completion uses. If the catalog is cold, the accessor
  returns `None`/empty and skill highlighting simply defers (no sync load), exactly matching the completion path's
  warm-only contract. In practice, typing `/` already triggers the completion warm, so the skill lights up within a
  keystroke or two — graceful, never blocking.
- **Consistent** — highlight and autocomplete are driven by identical data, so they can never disagree.
- **Cheap** — the frozenset is memoized against the warm entries list identity so it is rebuilt only when the catalog
  actually changes, not on every keystroke.

### 4. Performance: pay the `/` cost only when it can matter

- The overlay/tokenizer fast-exit guard gains `/`, but **gated on a non-empty `known_skills`**: proceed if `#`/`%`/`---`
  is present, _or_ (`/` present _and_ `known_skills` non-empty). Callers with no skill set (preview, pure-grammar tests)
  keep their current early-return and pay nothing.
- The added hot-path work for a slash-bearing prompt is one linear regex scan plus O(1) set lookups — the same order as
  the existing xprompt/directive scans — and only when the widget has a warm skill set. Fenced/disabled ranges are
  computed once and shared. The existing `_MAX_OVERLAY_BYTES` (80 000) / `_MAX_OVERLAY_LINES` (1 200) guards still cap
  total work, and the whole overlay remains fail-open.
- **Verification gate:** confirm no regression with the key-to-paint bench (`SASE_TUI_PERF=1`, target p95 < 16 ms) and
  the j/k bench, on a representative slash-heavy prompt. (Consistent with the TUI perf rule "measure, don't guess.")

### 5. Beauty: a bold, theme-adaptive accent that completes the palette

The prompt palette currently reads green (xprompt name) / sage (xprompt arg) / amber (directive name) / gold (directive
arg) / dim blue-cyan (separator). Skills get their **own** hue so the families stay legible at a glance. The recommended
choice is the theme's **`accent`** role, rendered **bold** (skills are first-class, actionable commands — bold makes
them pop and reads as "invoke me"). On the default flexoki dark theme `accent` is a violet, which:

- is clearly distinct from green, amber, and blue-cyan (completes the hue wheel);
- echoes the existing purple used for "special" tokens in the read-only preview (`alt_delimiter`), so it feels native;
- stays theme-adaptive: registered in `_register_xprompt_text_area_theme` alongside the other xprompt styles, it
  re-derives on theme switch (preserving the existing theme-switch adaptivity contract).

To guarantee legibility on very dark backgrounds, the exact rendered color may be lightened slightly toward the theme
foreground (reusing the established `_derive_argument_color` lightening approach) if the raw `accent` proves too dim in
the visual gate. As with the just-shipped argument-color change, the **exact hex, blend factor, and bold/weight are
locked during implementation via rendered swatches + the visual "beauty gate,"** pinning the real computed value in a
unit test. The `/` is kept inside the single `skill` span (uniform styling); splitting the slash into a dim delimiter is
a possible future refinement, out of scope.

### 6. Explicitly out of scope

- **The read-only preview path** (`xprompt_syntax.py`, stash/history/preview) — left unchanged. It calls `tokenize` with
  the default empty `known_skills`, so it emits no skill spans and cannot regress. (Extending skill highlighting to the
  preview — add a `skill` style to `XPROMPT_TOKEN_STYLES` and pass a known set — is a clean follow-up.)
- **Skill arguments** (e.g. `/sase_repo open foo`) — only the `/name` token is highlighted; trailing text is freeform
  prose. Arg tokenization for skills is out of scope.
- **Name/directive colors, separators, jinja, alt, search** — unchanged.
- **Launch semantics** — untouched; `/name` remains literal pass-through text.

## Implementation outline

### Source

1. **`src/sase/xprompt/xprompt_inspect.py`** — add `"skill"` to `XPromptSpanKind`; add a
   `known_skills: frozenset[str] = frozenset()` parameter to `tokenize`; add a `/name` scan (leading-context grammar
   reusing `XPROMPT_REFERENCE_LEADING_CONTEXT`) that emits `skill` spans for names in `known_skills`, respecting the
   existing protected (fenced/disabled) ranges; extend the fast-exit guard to include the `/`-with-known-skills case.
   Update the module docstring to note the opt-in editor-affordance behavior.

2. **`src/sase/ace/tui/widgets/_xprompt_syntax_highlight.py`** — in `_build_highlight_map`, extend the sentinel to
   include `/`, compute the warm known-skill `frozenset` (memoized) from `self._get_warm_xprompt_arg_assist_entries()`
   filtered by `is_skill`, and pass it to `tokenize`. In `_register_xprompt_text_area_theme`, register `"xprompt.skill"`
   with the chosen bold accent-derived style so it re-registers on theme switch. (The existing loop already maps span
   kind `skill` → style `xprompt.skill`.)

### Tests

3. **Tokenizer unit tests** (pure `tokenize(text, known_skills=…)`): known `/sase_plan` emits a `skill` span covering
   the full `/sase_plan`; unknown `/foo` and greedy `/sase_planner` (when only `sase_plan` known) emit nothing; mid-path
   `research/sase_plan` and URL `https://x/sase_plan` emit nothing (leading context); `/sase_plan` inside a fence /
   disabled region emits nothing; empty `known_skills` emits no skill spans (preview-path invariant); coexists with
   `#`/`%`/`---` spans and stays sorted.

4. **Widget/overlay tests** (reuse `tests/ace/tui/widgets/_completion_helpers.py::CompletionTestApp` + the existing
   `_seed_entries(..., is_skill=True)` pattern from `test_auto_xprompt_completion.py`): seeding `sase_plan` as a skill
   and loading `/sase_plan` produces an `xprompt.skill` highlight whose registered style is bold and a color distinct
   from `invocation`/`directive`; a cold (unseeded) catalog produces no skill highlight and triggers no sync build
   (mirror `test_slash_skill_cold_catalog_defers_without_sync_build`); theme switch re-registers the skill style;
   tokenizer failure remains fail-open.

### Visual snapshots (PNG goldens)

5. Extend the live-input highlight scenes (`_XPROMPT_HIGHLIGHT_SOLO` / `_XPROMPT_HIGHLIGHT_STACK` in
   `tests/ace/tui/visual/test_ace_png_snapshots_prompt_stack.py`) to include a real known skill (e.g.
   `use /sase_plan to draft the plan`), ensuring the catalog is warm/seeded so the skill renders deterministically.
   Regenerate only the affected goldens, inspect each diff to confirm the **only** change is the new skill color, and
   **eyeball each** for distinctness/harmony/legibility (the "beautiful" gate). Read-only-rendered goldens
   (`preview_panel_xprompt`, `prompt_history_*`, `stashed_prompts_*`) must **not** change; if one does, investigate
   before regenerating.

## Verification

- `just install` (ephemeral workspace) then `just check` (ruff + mypy + fast suite incl. the visual PNG suite).
- TUI perf bench (`SASE_TUI_PERF=1` key-to-paint p95 < 16 ms; j/k bench) on a slash-heavy prompt to confirm no
  regression from adding the `/` path.
- Manual render under the default flexoki theme with representative prompts (`use /sase_plan`, `/sase_repo open x`,
  `#gh:sase %m:opus then /sase_git_commit`, plus negatives `/unknown`, `src/pkg/file.py`, `https://host/sase_plan`,
  `/sase_plan` inside a ``` fence): confirm known skills glow in a bold accent clearly distinct from names/directives,
  and every negative stays plain.

## Acceptance criteria

- In the live prompt input, a `/name` referencing a **known** skill renders in a distinct, beautiful, bold accent color;
  unknown slashes, paths, URLs, typos, and fenced/disabled occurrences stay plain.
- Highlighting is driven by the same warm catalog as `/` completion (they never disagree) and adds **no** per-keystroke
  disk I/O.
- The skill color is theme-adaptive (re-derives on theme switch).
- Names, directives, separators, body text, and the read-only preview path are unchanged.
- `just check` passes; no TUI key-to-paint perf regression; visual goldens regenerated and visually confirmed.
