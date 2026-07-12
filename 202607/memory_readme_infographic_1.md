---
tier: epic
status: done
create_time: '2026-07-11 13:52:25'
---

# Plan: Self-Maintaining `memory/` README + Infographic

## Goal

Give the `memory/` directory a first-class, self-maintaining `README.md` — matching the polish of the SDD store README
(`src/sase/sdd/_init_files.py` → `SDD_README_CONTENT`) — that:

1. Opens with a **beautiful GPT-image infographic** explaining how the markdown files in `memory/` are actually used
   (which notes are always in context vs. read on demand, how they flow into `AGENTS.md` and the provider shims, and how
   `sase memory init` / `sase validate` keep everything fresh).
2. Explains the memory model clearly (tiers, frontmatter schema, linking) in prose that reads well on GitHub.
3. Lists **every memory note with its description and useful, deterministic statistics** (type, line count, approximate
   token count, parent), plus aggregate totals.

The README must be **regenerated and kept current by `sase memory init`** — never hand-edited — and must stay green under
the `sase validate` drift gate. The infographic ships as a packaged, drift-tracked binary asset with a regenerable prompt
sidecar, mirroring how `src/sase/sdd/assets/sdd-directory-map.png` backs the SDD README.

## Why This Design (Lead Rationale)

The SDD README is the north star. It is auto-generated from a Python source constant, embeds a packaged PNG copied into
the store on `sase sdd init`, and is drift-tracked so it can never silently rot. The `memory/` README should work the
same way, but go one step further: because memory notes carry their own `description`/`type` frontmatter and have
measurable size, the memory README can be **data-driven** — its per-file section and statistics are computed from the
notes themselves, so the README literally cannot drift out of sync with the files it documents. Running
`sase memory init` after touching any note refreshes the README, exactly as it already refreshes `AGENTS.md` descriptions.

### The Determinism Contract (Non-Negotiable)

`sase memory init` and `sase validate` (via `sase init --check` → `plan_init_memory` → `plan_memory_root`) compare the
**expected** README render against the on-disk bytes. The plan path and the write path share
`_render_expected_memory_files` (`src/sase/main/init_memory/roots.py:169`), so anything the README render depends on must
be a **pure, deterministic function of repository contents**:

- Statistics are derived only from note file contents (line counts, approximate token counts, note type, description,
  parent). Reuse the existing primitives in `src/sase/memory/inventory.py` (`_MemoryStats`, `_stats_for_text`,
  `MemoryInventory` counts) rather than inventing new counting logic.
- The per-note section iterates notes in a **stable, sorted order** (e.g. short notes then long notes, alphabetical
  within each group).
- **No timestamps, no "generated on" lines, no host/user/workspace-specific paths, no volatile ordering, no random
  IDs.** The rendered README for a given repository state must be byte-identical on every machine.
- The output must survive `format_generated_memory_markdown` (`src/sase/main/init_memory/formatting.py:56`) unchanged
  (idempotent 120-column, fence-aware reflow). Prefer a Markdown structure the formatter renders stably; if Markdown
  tables reflow unpredictably, use a definition-list / bulleted layout instead.

If this contract is violated, the `sase validate` freshness gate will go permanently red. Every code phase must prove
idempotency: `sase memory init` followed immediately by `sase memory init --check` must report zero drift.

## Scope Boundaries

- **Do not hand-edit** `memory/README.md`, `memory/*.md` canonical notes, `AGENTS.md`, or any provider shim
  (`CLAUDE.md`, `GEMINI.md`, `QWEN.md`, `OPENCODE.md`). All generated files change only as a byproduct of running
  `sase memory init`. The canonical memory notes' bodies are out of scope and must not be reworded.
- The README enrichment lives in the memory-init generator, not in a checked-in Markdown file that could drift.
- The infographic is a **static committed asset**. `sase memory init` copies and drift-checks it but never invokes image
  generation at runtime (image generation is non-deterministic and slow).

## Asset Policy

Mirror the SDD packaged-asset machinery (`src/sase/sdd/_init_files.py:117-200`: `expected_sdd_directory_map`,
`read_sdd_directory_map_bytes`, `write_sdd_directory_map`, `planned_bytes_operation`).

- Package source of truth: `src/sase/memory/assets/memory-directory-map.png`
- Copied into the repo on init: `memory/assets/memory-directory-map.png`
- Embedded in the generated README via a relative link:

  ```markdown
  ![How SASE memory files are used](assets/memory-directory-map.png)
  ```

- Regenerable prompt sidecar: `src/sase/memory/assets/memory-directory-map.prompt.md` — the final GPT image prompt,
  intended alt text, composition notes, and any deterministic post-processing steps. This sidecar is developer
  documentation living with the package source; it is not part of the copied/drift-tracked set.

Alt text must be useful on its own and must not merely repeat the title.

## Visual Direction (Infographic Brief)

Follow the established house style (`docs/images/infographic-style-brief.md`, `docs/agent_images.md`, and the epic
`epics/202605/docs_gpt_image_infographics.md`):

- Clean landscape ~16:9 architecture infographic (~1600x900), not marketing art.
- Light / neutral background with strong contrast for dark labels.
- Restrained multi-accent palette (distinct colors per concept, e.g. teal / blue / amber / green / slate); no single-hue
  theme, no dark mode, no gradients that reduce label contrast.
- Crisp flat vector-like panels, thin outlines, arrows, and swimlanes.
- Generated raster text is unreliable: ask GPT image for a **text-free structural base with generous blank label zones**,
  then add short, legible labels **deterministically** (SVG/ImageMagick with a pinned font) so terminology stays exact.
- No logos, no fake terminal screenshots, no dense paragraphs baked into the raster.

### Concept

Left-to-right story of how a memory note becomes agent context:

- **Left — the source:** a stack of `memory/*.md` note cards, each showing a small frontmatter tag
  (`type:`, `description:`, `parent:`).
- **Middle — the split by tier:**
  - **Tier 1 · short notes → always loaded.** They are *inlined* into `AGENTS.md`, which fans out **byte-identically** to
    the provider shims (`CLAUDE.md`, `GEMINI.md`, `QWEN.md`, `OPENCODE.md`). Label this lane "always in context."
  - **Tier 2 · long notes → read on demand.** Pulled by an agent via `sase memory read` (audited). Label this lane
    "reference, loaded when relevant."
- **Right — the outputs:** the agent's working context (short notes present, long notes fetched as needed) and the
  generated, self-updating `memory/README.md`.
- **Bottom band — the freshness loop:** `sase memory init` regenerates README + `AGENTS.md` + shims; `sase validate`
  checks for drift (a small ⟳ / gate motif).
- Include a compact frontmatter legend: `type: short | long`, `parent:`, `description:`, `keywords:`.

Keep labels short and unambiguous; the final committed image must be readable at GitHub Markdown width without relying on
nearby prose.

## Phase 1: Foundation, Contract & Asset Placeholder

Owner: distinct agent instance.

Objective: lock the README information architecture, the deterministic statistics spec, and the shared visual brief, and
establish the asset paths with a committed placeholder so the code phase and the image phase can proceed against a stable
contract.

Scope:

- Write a concise contract as this epic's follow-up notes (append to this plan, or a short
  `research/202607/memory_readme_contract.md`) capturing:
  - The exact README section order and headings (parallel to `SDD_README_CONTENT`): title + intro, embedded infographic,
    "How Memory Files Are Used" (tiers + frontmatter schema + linking), an auto-generated "Memory Notes" section
    (per-note: name, type, description, line count, approx tokens, parent), a "Statistics" summary (counts + totals),
    and a "Commands" reference (`sase memory list | init | read | write | review | log`).
  - The precise, deterministic statistics list and how each is computed from `src/sase/memory/inventory.py` primitives.
  - The stable sort order for the per-note section.
- Create `src/sase/memory/assets/` and commit a **valid placeholder PNG** at
  `src/sase/memory/assets/memory-directory-map.png` **and** a byte-identical copy at `memory/assets/memory-directory-map.png`
  so downstream code and drift checks have a real target. Add the sidecar
  `src/sase/memory/assets/memory-directory-map.prompt.md` containing the full brief from "Visual Direction" above.
- Do not change Python generator logic in this phase; do not generate the final image in this phase.

Exit criteria:

- The README contract (sections, stats spec, sort order) is written down and unambiguous.
- Placeholder PNG exists at both asset paths, byte-identical; the sidecar prompt exists and is complete.
- No canonical memory notes, `AGENTS.md`, or provider shims are edited by hand.

## Phase 2: README Generator, Statistics & Asset Machinery (Python)

Owner: distinct agent instance. Depends on Phase 1.

Objective: implement the rich, data-driven, deterministic README render and the packaged-asset copy/drift machinery, wired
through the existing memory-init plan/apply paths so `sase memory init` and the `sase validate` drift gate stay in
lockstep.

Scope:

- Enrich `_render_memory_readme()` (`src/sase/main/init_memory/roots.py:141`) into the full section structure from the
  Phase 1 contract. It will need the discovered memory notes for the root; thread them in (via
  `discover_memory_notes()` / `MemoryInventory`) so the per-note section and statistics are computed deterministically.
  Update `_render_expected_memory_files` (`roots.py:169`, README entry at `roots.py:191-195`) accordingly.
- Compute statistics using `src/sase/memory/inventory.py` primitives (`_MemoryStats`, `_stats_for_text`,
  `MemoryInventory` counts). Per note: type, line count, approx token count, description, parent. Aggregate: short/long
  counts, total notes, total lines, total approx tokens.
- Add the packaged-asset machinery mirroring `src/sase/sdd/_init_files.py:117-200`: read the PNG from package resources
  via `importlib.resources`, add it as an expected file with a **bytes** drift check, copy it to
  `memory/assets/memory-directory-map.png` during init, and embed it in the README. Treat the PNG as **read-only input** —
  this phase must not rewrite the image bytes (the image phase owns those).
- Extend the memory-init tests (`tests/main/test_init_memory_*`, `tests/test_memory_inventory.py`): assert the README
  contains the new sections, the per-note stats, and the embed; assert the asset copy + bytes-drift behavior; assert
  **idempotency** (init then `--check` reports no drift). Do **not** pin exact PNG bytes in tests — verify the copy
  mechanism generically (package copy matches repo copy) so the image phase can swap the bytes without breaking tests.

Exit criteria:

- `sase memory init` renders the full README, embeds `assets/memory-directory-map.png`, and copies the asset.
- `sase memory init` immediately followed by `sase memory init --check` reports **zero drift**; `sase validate` is green.
- The render is deterministic (no timestamps / machine-specific content) and survives `format_generated_memory_markdown`.
- Tests cover sections, stats, asset copy/drift, and idempotency, and pass.

## Phase 3: GPT-5.5 Infographic (GPT Image)

Owner: distinct agent instance. **Model: `codex/gpt-5.5`.** Depends on Phase 1.

Objective: produce the real, beautiful `memory-directory-map.png` infographic with GPT image generation, per the Phase 1
brief and the "Visual Direction" section.

Scope:

- Using GPT image generation, create a **text-free structural base** with generous blank label zones per the brief
  (light background, restrained multi-accent palette, crisp flat panels/arrows/swimlanes, ~16:9 ~1600x900).
- Add short, exact labels **deterministically** afterward (SVG/ImageMagick with a pinned font), following the documented
  post-processing pattern (see `docs/images/*.prompt.md` sidecars for examples). Model-generated text must not appear in
  the final raster.
- Replace the placeholder at **both** `src/sase/memory/assets/memory-directory-map.png` and
  `memory/assets/memory-directory-map.png` with the final image, **byte-identical** between the two locations.
- Finalize `src/sase/memory/assets/memory-directory-map.prompt.md` with the exact final prompt, intended alt text, and
  post-processing notes so the image is regenerable without chat history.
- This phase touches only the PNG asset and its sidecar — **no Python code** — so it can run concurrently with Phase 2.

Exit criteria:

- The final infographic is readable at GitHub Markdown width, matches the house style, and clearly conveys the
  short-vs-long tier flow, the fan-out to provider shims, `sase memory read`, and the `sase memory init` / `sase validate`
  loop.
- Both asset copies are byte-identical; the sidecar records the final prompt and post-processing.
- No hallucinated/misspelled labels; terminology matches the memory system (`type: short`/`type: long`,
  `sase memory read`, `AGENTS.md`, provider shims).

## Phase 4: Integration, QA & Docs

Owner: distinct agent instance. Depends on Phase 2 and Phase 3.

Objective: reconcile the code and the real infographic, verify the whole feature end to end, and land it clean.

Scope:

- Run `just install`, then `sase memory init` to regenerate the README with the real infographic and current stats;
  confirm both asset copies match and the embed resolves.
- Confirm the drift gate is clean: `sase memory init --check` and `sase validate` report no drift.
- Review the rendered `memory/README.md` for quality: sections read well, stats are accurate, the infographic renders and
  materially aids comprehension. Spot-check a couple of note stats by hand.
- If a user-facing docs surface for the memory system exists or is warranted (e.g. a `docs/` memory page or the doc
  infographic registry), embed/reference the infographic there consistently; keep this tightly scoped.
- Run `just check` (repo rule requires it after file changes). If it is blocked by a known pre-existing failure
  unrelated to this work, report the exact failure and run the narrower relevant gates (`sase validate`, targeted
  `pytest` over `tests/main/test_init_memory_*` and `tests/test_memory_inventory.py`).

Exit criteria:

- `sase memory init` produces a beautiful, accurate, drift-free README embedding the real infographic.
- `sase validate` is green; targeted memory-init tests pass; `just check` passes (or the only failures are documented
  pre-existing ones).
- No canonical memory notes, `AGENTS.md`, or provider shims were hand-edited; all generated changes flow from
  `sase memory init`.

## Verification Strategy

Each Python phase should run at least:

```bash
just install
sase memory init
sase memory init --check          # must report zero drift
rg -n "assets/memory-directory-map\.png|## (Memory Notes|Statistics|Commands)" memory/README.md
test -f src/sase/memory/assets/memory-directory-map.png
test -f memory/assets/memory-directory-map.png
cmp src/sase/memory/assets/memory-directory-map.png memory/assets/memory-directory-map.png
pytest tests/main/test_init_memory_*.py tests/test_memory_inventory.py -q
```

The final QA phase should additionally run `sase validate` and `just check`.

## Risks and Mitigations

- **Drift-gate flapping (highest risk):** any non-determinism in the render breaks `sase validate`. Mitigation: the
  Determinism Contract above; every phase proves `init` → `--check` is a no-op.
- **Formatter mangling structure:** `format_generated_memory_markdown` reflows to 120 cols. Mitigation: choose a
  formatter-stable Markdown layout; if tables are unstable, use bulleted/definition-list layout; test idempotency of the
  formatter on the rendered README.
- **Binary-asset merge conflicts:** Phase 2 (code) and Phase 3 (image) both live near the PNG. Mitigation: Phase 2 treats
  the PNG as read-only and never rewrites its bytes; only Phase 3 changes the image, so the merge is conflict-free.
- **Chicken-and-egg on the asset:** the code + drift check need a real PNG to exist. Mitigation: Phase 1 commits a valid
  placeholder at both paths; Phase 3 swaps the bytes later; tests never pin exact bytes.
- **Raster text quality:** prefer a text-free base + deterministic labels; inspect the final image; keep the sidecar so
  regeneration never depends on chat history.
- **Accidental memory-note edits:** the "no memory edits without permission" rule still applies to canonical notes,
  `AGENTS.md`, and shims. Only generated output changes, and only via `sase memory init`.
