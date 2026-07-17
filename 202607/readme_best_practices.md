---
tier: tale
title: README best-practices overhaul
goal: 'The top-level README states the project''s status and platform facts, routes
  readers to contributing/support, ships a lighter demo gallery, and renders correctly
  on PyPI — implementing the high-confidence recommendations from the consolidated
  README research report.

  '
create_time: 2026-07-17 10:13:28
status: done
prompt: 202607/prompts/readme_best_practices.md
---

# Plan: README best-practices overhaul

## Context

A three-researcher consolidated report on GitHub README best practices was written specifically for this project. It
lives in the `research` sidecar repo (open it with `sase repo open research -r "..."` and read
`202607/github_readme_best_practices_consolidated/github_readme_best_practices_consolidated.md`). Read it before
starting — especially §1 (verified repo ground truth), §3 (resolved disagreements), and §4 (the ranked recommendation
list this plan implements).

The report's headline: the current 109-line README has the right landing-page shape; it is missing **facts, not
sections**. This plan adds those facts, restructures the demo media, and fixes PyPI rendering. It deliberately does NOT
reorder sections, add a manual TOC, add social proof, or restore pre-rewrite content (all explicitly recommended
against).

Verified ground truth the plan relies on:

- `pyproject.toml` declares `Development Status :: 3 - Alpha` and `Operating System :: POSIX`; the README states
  neither. Build backend is hatchling (`requires = ["hatchling>=1.27"]`), and `readme = "README.md"` makes the README
  the verbatim PyPI long description.
- PyPI rendering is broken today: PyPI does not rewrite relative URLs, and the sdist (`only-include = ["src/sase"]`)
  excludes `demos/` and `docs/`, so every image and relative link (`INSTALL.md`, `LICENSE`, `docs/acknowledgements.md`)
  404s on the live PyPI page.
- Media payload is ~7 MB: `docs/images/sase_overview.png` (1.35 MB) + three 1920×1080 GIFs from `demos/out/` displayed
  at `width="830"`. Smaller unused assets already exist: `docs/images/blog/sase_ace_multi_model_fanout.gif` (1.8 MB,
  1280×720), `docs/images/blog/sase_ace_agents_observability.gif` (469 KB), and
  `docs/images/blog/agents_observability_still.png` (178 KB). There is no blog-size or still variant of the PRs-pipeline
  demo. Every README GIF has a smaller `.mp4` twin in `demos/out/`.
- `CONTRIBUTING.md` exists (setup, `just` workflow, tests, beads) and is referenced by zero files in the repo.
  `SECURITY.md` does not exist (out of scope here).
- CI workflow: `.github/workflows/ci.yml` (workflow name `CI`). The badge row currently has four badges (Docs, PyPI,
  pyversions, License) and no CI badge.
- The docs site has a blog (`https://sase.sh/blog/`) and a CI-tested PDF handbook
  (`https://sase.sh/downloads/sase-handbook.pdf`); the README links neither.
- `INSTALL.md` documents two prerequisites Quick start omits: `git` (all core VCS operations) and a text editor
  (`$EDITOR` → `nvim` → `vim` for commit messages).
- The publish workflow builds with `uv build` and validates with `twine check`, so a build-system metadata hook
  integrates cleanly.

## Changes

### 1. README copy changes (`README.md`)

Work top to bottom; keep the current section order and overall minimalism.

**Header block**

- Expand the acronym in the lede: `**sase** (Structured Agentic Software Engineering, pronounced "sassy") turns …`. The
  expansion currently exists only in the diagram image and package metadata; the name is the project's thesis.
- Add a CI badge to the badge row (workflow status badge for `ci.yml`, linking to the Actions page). This brings the row
  to five badges — do not add more.
- Add a one-sentence text equivalent for the overview diagram, as a caption line directly under the image (inside the
  centered block is fine): what the diagram encodes — one prompt fans out to parallel agents in isolated workspaces; ACE
  supervises; AXE schedules; durable state tracks ChangeSpecs/beads/artifacts; the output is reviewed PRs.

**Status and limits note**

Immediately after the lede paragraph, add a short (2–3 sentence) honest status note covering:

- sase is **alpha** software — interfaces and workflows are still evolving.
- Platform: POSIX only (Linux and macOS); Windows is not supported.
- The compact anti-sell (report item 12): sase assumes you already use and pay for at least one agent CLI, and it is
  opinionated about git-based, workspace-per-agent workflows. If you want a standalone agent rather than a coordination
  layer, use the agent CLIs directly.

Keep this as a note/paragraph, not a new full section — the README's shape should not grow.

**Demo gallery ("See it in action")** — keep the section and its position; cut its mass:

- Keep exactly **one** animated demo: the multi-model fan-out hero, switched from
  `demos/out/sase_ace_multi_model_fanout.gif` (3.8 MB, 1920×1080) to `docs/images/blog/sase_ace_multi_model_fanout.gif`
  (1.8 MB, 1280×720).
- Replace the other two GIFs with static stills that link to their `.mp4` twins in `demos/out/`, each with a caption
  noting it links to a short video and its duration (get durations with `ffprobe`):
  - Observability: use the existing `docs/images/blog/agents_observability_still.png`.
  - PRs pipeline: no still exists — generate one by extracting a representative frame from
    `demos/out/sase_ace_prs_pipeline.mp4` with `ffmpeg` (pick a frame that shows the PRs tab populated, not the opening
    blank state) and commit it alongside the observability still in `docs/images/blog/`. Fallback if frame quality is
    poor: regenerate media from `demos/tapes/sase_ace_prs_pipeline.tape` via the Justfile `demos` recipe (heavier; avoid
    if the ffmpeg still looks good).
- Also generate a still for the fan-out demo the same way (from `demos/out/sase_ace_multi_model_fanout.mp4`) — the
  README does not use it, but the PyPI substitution below does, so PyPI shows no autoplaying media (PyPI has no
  autoplay-off control, unlike GitHub).
- Keep the three existing bold one-line captions; keep existing alt text quality.

Net payload target: ~7 MB → roughly 3.5 MB per README view.

**Quick start** — complete the contract:

- Extend the prerequisites line: Python 3.12+, [uv], `git`, a text editor (`$EDITOR`, falling back to `nvim` then
  `vim`), and one authenticated agent CLI. Add a platform clause: Linux or macOS (POSIX); Windows is not supported.
- Explain the `#git:home` token in one sentence next to the `sase run` line (verify exact semantics against the xprompt
  docs at https://sase.sh/xprompt/ before writing the copy — it targets the built-in `home` project so the first run
  needs no project setup).
- State what success looks like: after the three commands, `sase ace` opens the TUI and the completed run is visible on
  the Agents tab.

**Works with your agents** (optional, low stakes — implementer's judgment): the five-row table repeats the lede's
provider list with identical "Supported" values. Either compress it to a single linked sentence or leave it as is; do
not invent a fake differentiating column.

**Learn more** — add three lines:

- Blog: `https://sase.sh/blog/` — announcements and deep dives.
- The SASE Handbook (PDF): `https://sase.sh/downloads/sase-handbook.pdf` — the full docs as a single document.
- Support: link GitHub Issues (`https://github.com/sase-org/sase/issues`) for bugs and questions. This answers GitHub's
  "where can I get help?" README question.

**Development** — two additions:

- Note that [`just`](https://github.com/casey/just) is required before the `just install` line (one clause; today the
  block fails on a fresh machine with no explanation).
- Link `CONTRIBUTING.md`: e.g. "Start with [CONTRIBUTING.md](CONTRIBUTING.md), then see
  [Development](https://sase.sh/development/) for the full contributor guide." This is the report's single
  strongest-evidence fix; the file is currently a complete orphan.

### 2. PyPI rendering fix (`pyproject.toml`)

Use [`hatch-fancy-pypi-readme`](https://github.com/hynek/hatch-fancy-pypi-readme) (a hatchling metadata hook — sase
already builds with hatchling) so the PyPI long description is generated from `README.md` at build time with relative
URLs rewritten to absolute ones:

- Add `hatch-fancy-pypi-readme` to `[build-system] requires`.
- Remove `readme = "README.md"` from `[project]`, add `"readme"` to `[project] dynamic`.
- Configure `[tool.hatch.metadata.hooks.fancy-pypi-readme]` with `content-type = "text/markdown"`, a fragment sourcing
  the whole `README.md`, and substitutions that:
  - rewrite relative image `src`s (`docs/...`, `demos/...`) to
    `https://raw.githubusercontent.com/sase-org/sase/master/...`;
  - rewrite relative file links (`INSTALL.md`, `LICENSE`, `CONTRIBUTING.md`, `docs/acknowledgements.md`) to
    `https://github.com/sase-org/sase/blob/master/...`;
  - swap the one animated hero GIF for its still PNG (absolute URL), so the PyPI page renders stills only.
- Branch-pinned (`master`) URLs are acceptable; tag-pinned URLs are optional hardening and not required.

Regex substitutions are order-sensitive and easy to get subtly wrong — verify against the real build output, not by
eyeballing the regexes (see Verification).

## Non-goals

- No section reordering, no manual TOC, no star/social-proof elements, none of the pre-rewrite content returns.
- No `SECURITY.md`, no repo About/topics/social-preview changes, no docs-site or `INSTALL.md` changes, no changes to the
  demo `.tape` sources (unless the ffmpeg-still fallback forces a regeneration).
- No README/docs section-marker embedding (ruff-style) — revisit only if drift becomes real.

## Verification

1. `just check` (required — this change touches repo files).
2. Confirm every image/link target referenced by the new README exists in the repo at the exact path used
   (case-sensitive).
3. Build-verify the PyPI description: run `uv build` (or `python -m build`), then inspect the long description in the
   built artifact's metadata (`PKG-INFO` in the sdist / `METADATA` in the wheel): no relative URLs remain, image URLs
   point at `raw.githubusercontent.com`, the hero image on PyPI is the still PNG, and
   `uvx --from twine twine check dist/*` passes (same check the publish workflow runs).
4. Sanity-check rendered output: view the modified README (e.g. `gh markdown-preview`, grip, or pushing a branch) to
   confirm the centered header block, caption, badge row, and still→mp4 links render as intended on GitHub.
5. Confirm the two new still PNGs are reasonable in size (well under ~300 KB each) and visually representative (not a
   blank first frame).

## Risks

- **Substitution regexes silently missing a link**: mitigated by step 3 above — grep the built metadata for `](docs/`,
  `](demos/`, `](INSTALL`, `src="docs`, `src="demos` and require zero hits.
- **Linked `.mp4`s on GitHub open the blob page rather than playing inline**: expected and acceptable — the blob page
  has a player; the caption should say "watch the demo" so the click-through is not surprising.
- **Blog-variant fan-out GIF differs slightly from the 1080p one**: it is the same demo at 1280×720; confirm visually
  that text remains legible at `width="830"`.
- **`ffmpeg`/`ffprobe` availability**: vhs-based demo tooling already implies ffmpeg on dev machines; if missing,
  install it or fall back to the vhs regeneration path.
