---
tier: tale
title: Agents sidecar infographic
goal: "Every agents sidecar initialized or refreshed by sase repo init includes a polished GPT Image infographic that
  remains visible in the root README before and after owner-sharded agent publication.

  "
create_time: 2026-07-28 12:52:40
status: done
---

- **PROMPT:** [prompts/202607/agents_sidecar_infographic.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/agents_sidecar_infographic.md)
- **AGENTS:**
  - [bbugyi200.athena.n6--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.n6.md#member-code)
- **COMMITS:**
  - [d4198f1](https://github.com/sase-org/sase/commit/d4198f1cc9b1e87b361fd80b6e0f99c94c5cec27) — feat: illustrate agents sidecar lifecycle

# Plan: Add the agents sidecar infographic

## Context

`sase repo init` currently seeds an agents sidecar with a privacy-forward `README.md`, `schema.json`, and empty
`agents/`, `families/`, and `users/` directories. Unlike the plans, research, and beads sidecars, it does not install an
illustrated guide asset. The first v2 publication then replaces the scaffold README with a deterministic root browsing
index, and later `repo init` runs intentionally preserve that manifest-derived README.

The infographic therefore has two consumers and one owner:

- The scaffold template must display it before any hood has been published.
- The manifest-derived root renderer must keep displaying it after publication.
- `sase repo init` must own installation and drift repair of the packaged image, including for already-populated
  sidecars whose derived README must not be overwritten.

This is presentation and generated-guide behavior, so it stays in the Python frontend/package rather than crossing the
Rust core backend boundary.

## Visual design and generation

Create `src/sase/sdd/assets/agents-directory-map.png` as a 1600x900 landscape documentation infographic consistent with
the existing plans, research, and beads directory maps: warm off-white canvas, crisp flat vector-like panels, thin
dark-slate strokes, restrained teal/blue/purple/amber accents, generous whitespace, and legibility at GitHub's
approximately 900px rendered width.

Use the built-in GPT Image tool through the `imagegen` skill. Generate a text-free structural base so model-rendered
text cannot introduce spelling or legibility defects, inspect the candidates, and iterate with one targeted composition
change at a time. The selected composition should communicate this left-to-right story:

1. An explicit privacy/creation gate for `sase repo init`, with public, private, and disabled outcomes.
2. One complete project-scoped top-level agent hood, visibly spanning active, waiting, failed, terminal, and dismissed
   runs and their prompt, optional chat, commit, and relationship artifacts.
3. Deterministic publication into an owner/machine shard, centered on `manifest.json` and a hood `snapshot.json`.
4. The hidden `<project>--agents` repository, explicitly not cloned into numbered workspaces.
5. Browsing from root to owner to machine to hood, then to agent and family pages, with a sync/refresh rail for commit
   outbox requests and `sase agent sync`.

Keep the amount of overlay copy disciplined. Candidate exact labels should include `AGENTS SIDECAR`, `EXPLICIT CONSENT`,
`sase repo init`, `PUBLIC · PRIVATE · DISABLED`, `COMPLETE HOOD`, `active · waiting · failed · terminal · dismissed`,
`OWNER SHARD`, `<username>/<machine>`, `manifest.json`, `hood snapshot`, `HIDDEN REPOSITORY`, `<project>--agents`,
`not cloned to workspaces`, `DETERMINISTIC BROWSING`, `root → owner → machine → hood`, `agent + family pages`,
`commit / outbox`, `sase agent sync`, and `refresh the same run`. Adjust wording or placement during visual review only
to improve accuracy and readability, not to change the underlying story.

Add all visible copy after generation with deterministic DejaVu Sans and DejaVu Sans Mono overlays, using the
established dark-slate text, white label panels, and light-gray borders. Resize/composite to exactly 1600x900, strip
metadata, retain an 8-bit sRGB PNG, and inspect both the full-size raster and a 900px-wide reduction. Reject or revise
any result with ambiguous arrows, cropped panels, pseudo-text, weak privacy emphasis, illegible labels, or a visual
implication that the agents sidecar is workspace-cloned.

Use this README alt text, refining only if the finished composition materially changes:

> Project-scoped agent hoods pass through explicit privacy consent into an owner-sharded agents sidecar, where
> deterministic sync publishes prompts, chats, commits, states, and browsable owner, machine, hood, family, and agent
> pages.

Record the final structured GPT Image prompt, source-generation provenance, exact deterministic labels, fonts/colors,
post-processing commands, alt text, and full-size/downscaled inspection checklist in
`src/sase/sdd/assets/agents-directory-map.png.prompt.md`. Do not retain discarded candidates in the repository.

## Guide bundle and README lifecycle

Extend `src/sase/sdd/_init_files.py` so the built-in `agents` guide bundle contains `assets/agents-directory-map.png` in
addition to its current README, schema, and scaffold directories. Load the PNG through the same packaged directory-map
asset path used by the other sidecars so production wheels, test-time asset overrides, drift planning, idempotent
writes, and sidecar commits all use the existing machinery.

Preserve the current publication-aware ownership rule:

- Before publication, `repo init` may create or refresh both the scaffold README and the infographic.
- After an owner manifest exists, `repo init` must continue to preserve the derived root README while independently
  creating or repairing a missing or stale infographic.
- Agent synchronization should continue to own only dynamic publication paths; do not add the static guide asset to the
  agents-sync payload or authority file set.

Update the scaffold at `src/sase/sdd/templates/sidecar-agents-README.md` to show the infographic near the introductory
privacy explanation. Update `src/sase/agents_sync/rendering_index_pages.py` so every manifest-derived root README
renders the same image link near its introduction, ahead of the dynamic owner/machine/hood/run counts. Nested browsing
pages should remain unchanged.

Teach the repository-init summary in `src/sase/main/repo_init_handler.py` to classify a generated directory-map PNG
action as a sidecar-guide refresh. This keeps an asset-only upgrade of an already-published agents sidecar user-facing
instead of falling back to the generic action-count summary.

## Documentation

Update the focused documentation to match the shipped behavior:

- In `docs/agents_sidecar.md`, include the infographic in the strict root layout and explain that the scaffold and
  derived browsing README both reference the `repo init`-managed static asset.
- In `docs/init.md`, describe agents as an illustrated built-in sidecar guide and call out that rerunning
  `sase repo init` upgrades the image without replacing a populated sidecar's derived index.
- Adjust `docs/sdd.md` only where its generated-guide wording needs to distinguish the illustrated built-in roles from
  generic custom sidecars.

## Tests and acceptance criteria

Expand focused coverage for the complete lifecycle:

- `tests/sdd_store/test_sidecar_init_files.py`: assert the agents bundle plans and writes the PNG, the scaffold
  references it, the real packaged file is a 1600x900 PNG, a second run is a no-op, and a populated sidecar preserves
  its derived README while repairing a missing or stale image.
- `tests/main/test_repo_init_plan.py`: include the asset in new-sidecar action paths and cover an asset-only
  populated-sidecar plan whose summary says it refreshes sidecar guide files.
- `tests/sdd_store/test_sidecar_init_creation.py`: verify a newly created agents sidecar receives the asset at the
  expected root-relative path.
- `tests/agents_sync/test_rendering.py`: verify the derived root page contains the exact image Markdown while nested
  index pages do not gain root-only content.
- `tests/test_markdown_template_packaging.py`: guard wheel inclusion of the agents README template, final PNG, and
  prompt/provenance Markdown.

Visually inspect the final image with the image-viewing tool at full size and at the 900px review size. Confirm exact
label spelling, hierarchy, contrast, arrow direction, privacy messaging, and that no model-generated text or watermark
remains.

Before verification, run `just install` as required for an ephemeral SASE workspace. Run the focused tests above, then
run the repository-mandated `just check`. Review the final diff and status to confirm that only the new
asset/provenance, guide/rendering integration, tests, and focused documentation changed.

The implementation is complete when a fresh `sase repo init` installs and shows the infographic, rerunning it is
idempotent, an older populated sidecar gets the asset without losing its generated root index, subsequent agent
publication keeps the infographic in that index, the packaged wheel contains all required sources, visual inspection
passes at both sizes, and `just check` succeeds.
