---
create_time: 2026-07-11 12:45:13
status: done
prompt: 202607/prompts/artifact_type_icons.md
tier: tale
---
# Plan: Artifact‑type icons in the Agents‑tab "Artifacts:" panel

## Goal

Add a small, per‑type icon in front of every entry in the **Artifacts:** field of the agent metadata panel (the
prompt‑panel header on the `sase ace` "Agents" tab) so a user can tell at a glance whether each artifact is an
**image**, a **video**, a **document** (PDF — and markdown, which the artifacts panel renders _as_ a PDF), or a generic
file.

Design pillars the user asked for, and how this plan meets each:

- **Intuitive** — a distinct geometric glyph _and_ a distinct accent color per type, with shapes chosen to evoke the
  type (a play triangle for video, a lined square for a document, a hatched square for an image).
- **Reliable** — the icon is derived from the **exact same** classifier the artifacts panel already uses to decide how
  to render an artifact, so the icon can never disagree with what actually opens. Every glyph is single terminal‑cell
  width and is present in the font the visual snapshot suite pins, so alignment and rendering are deterministic.
- **Beautiful** — monochrome geometric glyphs tinted with hex styles, matching the panel's established visual language
  (it already uses `◇ ◆ ▣ ↳ •`), rather than multi‑color emoji that would clash with that language, render as tofu under
  the pinned snapshot font, and break column alignment.

## Background / current behavior

- The **Artifacts:** section is rendered by `src/sase/ace/tui/widgets/prompt_panel/_agent_artifacts.py`. Today every
  entry is prefixed with a single fixed bullet `•` (`_ARTIFACT_ENTRY_PREFIX`), styled gold, with no indication of type.
  The rendered line is: `  • [N] <path>` (the `[N]` is the optional keyboard‑hint number for the "open this file"
  action).
- The list of entries is built by `agent_artifact_paths(agent)` in the same file, which reads each artifact's `kind`
  from `list_agent_artifacts(...)` and then throws the type information away — `AgentArtifactPath` only carries
  `display_path`, `actual_path`, and `exists`. **The type is already known at build time; we simply need to keep it.**
- The **single source of truth** for "how is this artifact rendered" is `artifact_view_mode(path, *, kind=None)` in
  `src/sase/ace/tui/graphics/_viewer_render.py`, which returns one of `image | video | pdf | markdown | text` (or
  `None`). This is the exact function the artifacts viewer/opener uses. Keying our icons off it guarantees the icon
  matches how the artifact actually opens, and it transparently handles two important nuances:
  - **video** is not a stored `kind` (videos are persisted as `kind="file"`); the viewer recognizes them by extension.
    `artifact_view_mode` already does this, so we get correct video icons "for free."
  - **markdown** and **pdf** are separate view modes but are _both displayed as a PDF_ (markdown is converted MD → PDF →
    page images). Per the user's requirement, both map to the **same document icon**.
- **Scope boundary:** this is terminal‑presentation only. The classifier already lives in the Python TUI layer, and
  choosing a glyph/color for a terminal is inherently frontend‑specific. So there is **no change to the Rust core** and
  no change to the stored artifact data model or `AgentArtifactKind` enum. (Adding a first‑class `video` artifact kind
  is a real but separate gap and is explicitly out of scope here.)

## Icon design

Map the render view‑mode → (glyph, accent color). `pdf` and `markdown` deliberately collapse to one **document** icon —
this is the crux of "markdown treated as PDF."

| View mode                  | Applies to                                           | Glyph | Codepoint | Accent color (hex)       | Why this glyph                         |
| -------------------------- | ---------------------------------------------------- | ----- | --------- | ------------------------ | -------------------------------------- |
| `image`                    | png/jpg/jpeg/webp/gif (or stored `kind=image`)       | `▨`   | U+25A8    | mint/green `#5FD7AF`     | hatched square = filled picture area   |
| `video`                    | mp4/m4v/mov/webm (stored as `kind=file`)             | `▶`   | U+25B6    | coral/orange `#FF875F`   | play triangle = video                  |
| `pdf`                      | `.pdf`                                               | `▤`   | U+25A4    | parchment/tan `#D7D7AF`  | horizontal lines = lines of a document |
| `markdown`                 | md/markdown/mdown/mkd, and plan/chat/file md sources | `▤`   | U+25A4    | parchment/tan `#D7D7AF`  | **same as PDF** (rendered as a PDF)    |
| `text` / `None` (fallback) | everything else / generic files                      | `•`   | U+2022    | gold `#FFD787` (current) | unchanged generic bullet               |

Why this satisfies the pillars, concretely:

- **Reliability of glyph rendering** was verified against the bundled snapshot font
  (`tests/ace/tui/visual/fonts/FiraCode-Regular.ttf`, the _only_ font the PNG suite consults, with system fonts
  disabled): `▶ ▤ ▨ •` are all present in Fira Code 6.2 and all measure exactly **one** terminal cell. Because the new
  glyph occupies the same single cell the old bullet did, the `[N]` hint and path columns stay perfectly aligned. Emoji
  candidates (`🖼 🎬 📄`) are **absent** from that font and are double‑width, so they were rejected.
- **Palette** is drawn from the panel's existing accent family (headers `#87D7FF`, paths `#87AFFF`, gold `#FFD787`,
  context glyphs `#5FD7FF/#5FD75F/#FF87D7`). The three type colors — warm coral, cool green, neutral tan — are distinct
  from each other and from the blue paths, so type is legible from color alone even in a quick glance. Exact hex values
  are presentation constants and easy to tune during review.
- **Missing artifacts:** when an entry is missing on disk it already renders dimmed with a `(missing)` suffix. The icon
  will follow suit — a dimmed variant of the type color — so the row stays visually coherent. The view‑mode
  classification itself is suffix/kind‑based and does not require the file to exist, so a missing artifact still shows
  the _correct_ type icon, just dimmed.

Ordering stays `  <icon> [N] <path>` (type first, then the actionable hint) — the icon simply takes over the slot the
bullet used.

## Implementation outline

All changes are confined to the TUI presentation layer. No Rust, no data‑model, no storage changes.

### 1. Carry the type through to the renderer — `_agent_artifacts.py`

- Add a defaulted field to `AgentArtifactPath`, e.g. `view_mode: str = "text"`, holding the resolved render mode.
  Defaulting it keeps every existing constructor (including the two test files that build `AgentArtifactPath` directly)
  backward compatible.
- In `agent_artifact_paths(...)`, where each `AgentArtifact` is still in scope, resolve the mode with
  `artifact_view_mode(artifact.path, kind=artifact.kind)` (lazy import from `sase.ace.tui.graphics`, matching the
  module's existing lazy‑import style). Thread the resolved mode through the internal display tuple into
  `_dedupe_paths`, and set it on the constructed `AgentArtifactPath`. For the aggregated follow‑up `followup_prompt*.md`
  entries, the mode is `markdown` (document icon), consistent with how they open.
- Add an icon table and a small resolver, e.g. `_ICON_BY_VIEW_MODE: dict[str, tuple[str, str]]` mapping mode → (glyph,
  style), with a `text`/unknown fallback to the current bullet, plus a helper that returns a dimmed style variant when
  the artifact is missing.
- In `append_agent_artifacts_section(...)`, replace the fixed
  `text.append(f"  {_ARTIFACT_ENTRY_PREFIX} ", style=_GLYPH_STYLE)` with: indent `"  "`, the per‑type glyph in its
  per‑type (or dimmed) style, then a trailing space. Leave the `[N]` hint bookkeeping and `_append_path` rendering
  untouched.

### 2. Keep the classifier authoritative

Do **not** re‑implement extension/kind logic in the panel. Everything routes through `artifact_view_mode`, and the
`{pdf, markdown} → document icon` collapse lives only in the presentation table. If the viewer's classification evolves,
icons follow automatically.

## Testing

Reuse the established harness in `tests/ace/tui/widgets/test_prompt_panel_persisted_artifacts.py` (the `_StubAgent` +
`store_default_agent_artifact` pattern, asserting on `agent_artifact_paths(...)` entries and on the rendered
`rich.text.Text`). Add coverage (extend that file or add a sibling `test_prompt_panel_artifact_icons.py`):

- `agent_artifact_paths` sets `view_mode` correctly for: an image, a **video stored as `kind=file`** (proves we classify
  by the viewer, not raw kind), a markdown/plan `.md` source, a `.pdf`, and a generic non‑media file.
- `append_agent_artifacts_section` emits the expected glyph **and** the expected style span for each type (assert the
  glyph char appears in `text.plain` and the corresponding `Text` span carries the type's style; assert markdown and pdf
  share the document glyph).
- Alignment/width regression guard: a mix of several types each contributes exactly one glyph cell before the `[N]`/path
  columns.
- Missing‑artifact row still shows the type icon (dimmed) alongside the existing `(missing)` suffix.

## Visual snapshots

The PNG snapshot suite pins Fira Code and renders the Agents tab, and at least one fixture seeds an artifact, so the
bullet→icon change will shift affected goldens.

- Run `just test-visual`; for any golden that legitimately changes only because of the new icon, inspect
  `.pytest_cache/sase-visual/` (actual/expected/diff) to confirm the icon renders crisply and aligned, then accept with
  `--sase-update-visual-snapshots`.
- Recommended (locks in the "beautiful" result): add one focused snapshot fixture that seeds an agent with one artifact
  of each type (image, video, pdf/markdown, generic file) so the full icon family is captured as a golden and protected
  from regressions.

## Verification / checklist

- `just install` then `just check` (ruff + mypy + unit tests) — required because source files change.
- `just test-visual` and update goldens as above.
- Manual: launch `sase ace`, select an agent that produced mixed artifacts, and eyeball the **Artifacts:** section —
  each row should lead with the correct, aligned, tinted icon.
- Confirm no Rust‑core / data‑model / stored‑index changes were introduced.

## Explicitly out of scope

- Adding a first‑class `video` member to the stored `AgentArtifactKind` enum / index (a known gap, but the viewer
  already infers video by extension, so icons are correct without it).
- Changing which artifacts appear in the list (the existing `kind in {"chat", "pdf"}` filter and the follow‑up
  aggregation are left as‑is).
- Introducing MIME detection (the codebase is entirely extension/kind based).
- Icons anywhere other than the Agents‑tab metadata panel (the artifact picker modal and file/notification panels are
  untouched by this change).
