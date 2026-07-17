---
tier: tale
title: README redesign with new GPT-image hero and demo GIFs
goal: 'The repo-root README.md becomes a concise, visually striking landing page that
  hooks new users, gets them installed and running fast, showcases the best demo GIFs,
  embeds a redesigned GPT-image hero infographic, and funnels readers to sase.sh for
  everything else.

  '
create_time: 2026-07-17 08:39:29
status: wip
prompt: 202607/prompts/readme_redesign.md
---

# Plan: README redesign with new GPT-image hero and demo GIFs

## Context

The current `README.md` (~354 lines) reads like an operations manual, not a landing page. It front-loads a 14-bullet
"Operational model" section, a 22-command dump, and a 40-link "Keep reading" list — content that already lives on the
docs site. The one visual is `docs/images/sase_overview.png`, a competent but busy architecture diagram whose workspace
pills still say "Gemini CLI" even though the supported-agent table now lists Antigravity CLI (`agy`). Meanwhile
`demos/out/` contains five polished, hermetic VHS demo GIFs of the ACE TUI that the README never shows.

Goals, in priority order:

1. Draw readers in and show them within seconds how sase is useful to them.
2. Get them installed and running (`uv tool install sase` → `sase doctor` → first run → `sase ace`).
3. Point them to **[sase.sh](https://sase.sh/)** with a small curated set of links, not an index dump.
4. As concise as possible, but not more so.

Established conventions this plan follows:

- **Infographic pipeline** (`docs/images/infographic-style-brief.md` and existing `*.prompt.md` / `*.critique.md`
  sidecars): GPT-image base generated via the `imagegen` skill's default built-in GPT-image path; deterministic label
  pass with ImageMagick (DejaVu Sans) when generated text is unreliable; a sidecar prompt file updated on every
  regeneration; 16:9 landscape PNG normalized to the family size (1672x941).
- `docs/index.md` (docs hero) and `mkdocs.yml` (`og:image`) both reference `docs/images/sase_overview.png`. Keeping the
  same filename means the docs site inherits the redesign automatically.
- Demo GIFs are checked-in artifacts under `demos/out/` (see `demos/README.md`); embedding them with repo-relative paths
  is safe.

## Non-goals

- No changes to the demo tapes, seeds, or generated GIFs/MP4s.
- No redesign of the five documentation infographics governed by the style brief (`docs/xprompt.md`,
  `docs/workflow_spec.md`, etc.). Only the README hero image is regenerated.
- No restructuring of the docs site or `docs/index.md` beyond inheriting the new PNG and, if needed, a matching alt-text
  update.
- No INSTALL.md rewrite (the README links to it).

## Design: the new README

Target length: roughly 100-130 lines. Overall shape (top to bottom):

### 1. Centered hero block

```markdown
<div align="center">

# sase

**One developer. A team of coding agents. Tracked, reviewable, repeatable work.**

[Docs badge: sase.sh] [PyPI version] [Python versions] [License: MIT]

<img src="docs/images/sase_overview.png" alt="..." width="830">

</div>
```

- Trim the badge row to badges a _user_ cares about: Docs (sase.sh), PyPI version, Python versions
  (`img.shields.io/pypi/pyversions/sase`), and License MIT. Drop the Ruff/mypy/pytest/tox tool badges — they are
  contributor trivia and dilute the hero.
- The redesigned hero infographic (see next section) sits directly under the badges.

### 2. The pitch (one short paragraph + bullets)

Two or three sentences: **sase** (pronounced "sassy") turns coding agents — Claude Code, Codex, Antigravity, Qwen Code,
OpenCode — into a coordinated engineering team. One developer supervises many parallel agents, each in its own isolated
workspace, with every run tracked, reviewable, and repeatable.

Then a tight "Why sase" list (keep to five bullets, reworded from the current ones):

- Launch, monitor, resume, and archive agent runs from one keyboard-driven TUI (**ACE**).
- Run agents in parallel, each in an isolated numbered workspace clone.
- Keep prompts and multi-step workflows reusable (**XPrompts**) instead of trapped in shell history.
- Track every PR-sized unit of work with status, commits, comments, and review state (**ChangeSpecs**).
- Schedule background and recurring agent work with the **AXE** daemon.

Close the section with the one-liner that already works well: sase does not replace coding agents; it makes agent-driven
engineering dependable.

### 3. "See it in action" — three demo GIFs

Embed exactly three of the five GIFs, in this order, each introduced by a bold one-line caption (heading + single
sentence, then the image). Chosen for story arc — fan out, supervise, land:

1. `demos/out/sase_ace_multi_model_fanout.gif` — **One prompt, three agents, three models.** A single prompt fans out to
   Claude, Codex, and Gemini agents with per-agent model directives and a launch preview.
2. `demos/out/sase_ace_agents_observability.gif` — **Supervise every run.** The Agents tab: live status, retry chains,
   per-agent diffs, chats, and artifacts from one control surface.
3. `demos/out/sase_ace_prs_pipeline.gif` — **Land tracked changes.** The PRs tab: ChangeSpec lifecycle from WIP to
   Submitted, with grouping, search, and per-PR commits and diffs.

Leave out `sase_ace_prompt_input.gif` and `sase_ace_prompt_history_stash.gif` — good demos, but more niche; three GIFs
is the right density for a concise page. Use `<img ... width="830">` (or plain Markdown images) with meaningful alt
text; repo-relative paths.

### 4. Quick start

Condense the current Quick start. Prerequisites as one line (Python 3.12+, [uv], one authenticated agent CLI: Claude
Code, Codex, Antigravity CLI (`agy`), Qwen Code, or OpenCode). Then:

```bash
uv tool install sase   # add a plugin too: uv tool install sase --with sase-github
sase doctor            # readiness gate: install, config, provider auth
sase run "#git:home summarize what this repository does; do not change files"
sase ace               # open the interactive control surface
```

Keep, in one short closing paragraph: `sase doctor` re-run guidance when a provider is missing (link
[Agent Providers](https://sase.sh/agent_providers/)), the [INSTALL.md](INSTALL.md) link for the full install guide, and
the [Getting Started](https://sase.sh/getting_started/) link as the guided path.

### 5. Works with your agents

Keep the existing five-row supported-agents table verbatim — it is compact and scannable.

### 6. Learn more (curated, not exhaustive)

Replace the 40-link "Keep reading" section with: one bold line pointing at **[sase.sh](https://sase.sh/)**, then a
curated list of about eight sase.sh links with a few words of "why click this" each:

- [Getting Started](https://sase.sh/getting_started/) — the guided beginner path
- [ACE TUI](https://sase.sh/ace/) — the interactive control surface
- [XPrompts](https://sase.sh/xprompt/) — reusable prompts and multi-step workflows
- [ChangeSpecs](https://sase.sh/change_spec/) — tracked PR-sized units of work
- [AXE Automation](https://sase.sh/axe/) — scheduled and background agent work
- [Spec-Driven Development](https://sase.sh/sdd/) — plans, epics, and beads
- [Plugins](https://sase.sh/plugins/) — GitHub, Telegram, editor, and provider integrations
- [CLI Reference](https://sase.sh/cli/) — every command

Drop the parallel `(local)` links; GitHub readers should land on the docs site.

### 7. Development (short)

Keep the source-install snippet (clone, `uv venv`, `just install`, `sase core health`) and a one-line pointer to
`just check` plus the [Development](https://sase.sh/development/) page. Cut the full verification-recipe list and
pytest-worker details — they live in the docs.

### 8. Acknowledgements and License (short)

Compress Acknowledgements to one short paragraph: Boris Cherny's parallel-agents demonstration, Steve Yegge's beads, and
the two papers, each linked inline, with a pointer to [docs/acknowledgements.md](docs/acknowledgements.md) for the full
version. Keep the MIT license line.

### Content being cut — disposition

The big cuts are "Operational model" (all 14 bullets), "Common commands", "Core pieces", and the long link dump. Before
deleting each, the implementer must spot-check that the topic is covered on the docs site (it should be: workspaces →
`docs/workspace.md`, prompt authoring → `docs/ace.md`/`docs/prompt.md`, retries/model routing → `docs/llms.md`, SDD
storage → `docs/sdd_storage.md`, commit finalization → `docs/commit_workflows.md`, project lifecycle →
`docs/project_spec.md`, Rust core → `docs/rust_backend.md`, artifacts/index → `docs/architecture.md` + `docs/cli.md`,
updates → `docs/init.md`/Admin Center docs). If a specific fact exists **only** in the README, move that fact into the
most fitting docs page in the same change rather than losing it; keep such ports minimal and surgical. Do not rewrite
docs pages beyond these insertions.

## Design: redesigned hero infographic

Replace `docs/images/sase_overview.png` (same path/filename so `docs/index.md` and the `og:image` in `mkdocs.yml`
inherit it) with a bolder, simpler hero that matches the dark ACE terminal aesthetic of the demo GIFs, so the README
reads as one visual system.

**Design direction (decided, not open):**

- Dark hero banner: near-black slate background (GitHub-dark-friendly, ~#0d1117), thin light strokes, and the ACE accent
  palette already used by the TUI and the retired tabs infographic: teal `#00D7AF`, light blue `#87D7FF`, coral
  `#FF5F5F`, plus a restrained amber and green for status accents.
- Radically simpler story than the current diagram — five beats read left to right:
  1. **You** (one developer node) with an ACE TUI chip (interactive) and AXE chip (scheduled).
  2. **One prompt** (Prompt / XPrompt / Workflow pill).
  3. **Parallel agents** — three workspace cards labeled Workspace 1/2/3 containing agent pills "Claude Code", "Codex",
     "Antigravity CLI" (fixes the stale "Gemini CLI" label; plain text pills, no logos).
  4. **Durable state** rail: ChangeSpecs · Beads · Commits · Artifacts.
  5. **Outcomes**: Reviewed PRs · Tracked runs · Scheduled work.
- Title "SASE" with subtitle "Structured Agentic Software Engineering"; drop the current footer sentence,
  planner-handoff sub-diagram, and plugin strip — the simplified story is the point. (The full architecture remains
  available in docs infographics.)
- 16:9 landscape, normalized to 1672x941 PNG like the rest of the family, legible at ~830px README width. No logos, no
  fake screenshots, no decorative gradients, no watermark.

**Generation workflow (follow the family convention):**

1. Author the final GPT-image prompt from the direction above. Instruct a _mostly text-free_ base (blank label zones, no
   pseudo-text) since generated text is the family's known weak point.
2. Generate with the `imagegen` skill's default built-in GPT-image path. Inspect at full resolution for cropping, arrow
   correctness, pseudo-text, and contrast; iterate until the base is clean.
3. Add the deterministic labels listed above with ImageMagick (DejaVu Sans), matching the sase_tui_tabs post-processing
   precedent. If a generation reliably renders the short labels correctly, GPT-rendered labels are acceptable — judge
   from the actual output.
4. Normalize to 1672x941, overwrite `docs/images/sase_overview.png`.
5. Update the sidecar `docs/images/sase_overview.prompt.md` (final prompt, target insertion points, alt text,
   post-processing notes) and record the review pass in a new `docs/images/sase_overview.critique.md`, mirroring the
   sibling critique files.
6. Update alt text at all three reference sites if it changes: `README.md`, `docs/index.md`, and keep `mkdocs.yml`
   `og:image` untouched (path unchanged). Suggested alt text: "One developer using SASE to run parallel coding agents in
   isolated workspaces with tracked, reviewable results".

## Verification

- `just install` first (ephemeral workspace), then `just check`. Note: in sandboxed runs the full test phase can be
  killed with exit 144 (environment SIGTERM, not a real failure) — if that happens, run the static gates (`just fmt`,
  `just lint`, `sase validate`) plus a targeted pytest subset and say so in the handoff rather than retrying the full
  suite.
- Render-check the new README locally (any Markdown preview) for: heading hierarchy, centered hero block, image/GIF
  paths resolving, and total length in the ~100-130 line band.
- View the final `docs/images/sase_overview.png` at 830px width and at full size; confirm label spelling, no
  pseudo-text, and that it still works as the docs-site hero in `docs/index.md`.
- Confirm the three GIF paths exist and render (they are checked-in; no regeneration).
- `just docs-check` to make sure the strict docs build is unaffected by any alt-text edits.

## Risks and notes

- **`imagegen` skill availability**: prior approved plans in this project (e.g. `202607/canonical_sase_directories.md`,
  phase sase-6d.8) used the `imagegen` skill successfully. If the implementing runtime does not expose it, stop and
  surface that instead of substituting an ad-hoc image pipeline.
- **PyPI rendering**: `pyproject.toml` uses `README.md` as the package long description, and relative image paths do not
  render on pypi.org. This is a pre-existing limitation (the current README has it too). Keep repo-relative paths for
  GitHub/PR-review correctness; converting media links to absolute `raw.githubusercontent.com` URLs is a possible
  follow-up, not part of this change.
- **GIF weight**: the three chosen GIFs total ~5.6 MB, which GitHub renders fine; no compression work in this change
  (demos/README.md already notes future gifsicle work).
- **Dark hero on light GitHub theme**: a dark banner-style hero is intentional and reads as a designed card on both
  GitHub themes, and it matches the dark demo GIFs beside it. Do not generate separate light/dark variants; one asset
  keeps the docs `og:image` and `docs/index.md` consistent.
