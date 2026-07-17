---
tier: epic
title: README demo GIF overhaul — truecolor, captions, live GitHub fan-out
goal: 'The README''s flagship demo GIF shows a truecolor ACE session in which one
  alternation-shorthand prompt on the GitHub VCS workflow fans out to three models,
  actually launches three hermetic agents, and kills them live — with explanatory
  text overlays produced by new, reusable demo-captioning tooling that all five demo
  tapes can adopt.

  '
phases:
- id: truecolor
  title: Truecolor demo tapes + full artifact regeneration
  depends_on: []
  description: 'See "Phase `truecolor`": pin COLORTERM/TERM truecolor env in all five
    VHS tapes (mirroring the visual-snapshot conftest), regenerate demos/out/ with
    `just demos -y`, and add a saturation sanity guard to the demos pipeline.

    '
- id: override
  title: LLM execution-provider override + fakey demo scenario
  depends_on: []
  description: 'See "Phase `override`": add a provider-agnostic SASE_LLM_EXEC_PROVIDER
    env knob at the invoke_agent provider-lookup choke point so launches execute under
    fakey while displaying the requested provider/model, record the executing provider
    in run artifacts, audit launch preflights, and add a bundled long-streaming fakey
    `demo` scenario with tests.

    '
- id: captions
  title: Demo media post-processing and caption overlay tooling
  depends_on: []
  description: 'See "Phase `captions`": implement the research-recommended caption
    pipeline — per-tape captions.yml sidecars rendered to ASS and burned in with ffmpeg/libass,
    GIFs derived from the captioned MP4s via palettegen/paletteuse, replace-in-place
    output policy, Justfile integration, docs, and unit tests.

    '
- id: demo
  title: Live GitHub fan-out launch/kill demo + README refresh
  depends_on:
  - truecolor
  - override
  - captions
  description: 'See "Phase `demo`": provision a demo venv with sase-github, seed a
    hermetic gh project (acme/nova), rewrite the fanout tape so one alternation-shorthand
    prompt really launches three fakey-backed agents (CLAUDE/CODEX/AGY badges) that
    are killed on screen, add the captions sidecar, regenerate all README/blog media,
    and update README + demos docs.

    '
create_time: 2026-07-17 11:28:33
status: wip
bead_id: sase-6l
---

# Plan: README demo GIF overhaul

## Problem

The README's hero demo (`docs/images/blog/sase_ace_multi_model_fanout.gif`, rendered from
`demos/tapes/sase_ace_multi_model_fanout.tape`) has four problems:

1. **It is nearly grayscale.** All five demo GIFs render ACE without color. Root cause (verified empirically and by code
   inspection): the VHS/ttyd shell has `TERM=xterm-256color` but no `COLORTERM`, so Rich's color-system detection
   (`_detect_color_system` in the vendored `rich/console.py`) picks 256-color mode and Textual quantizes the flexoki
   theme's warm near-grays and the provider badge accents (`src/sase/ace/tui/provider_styles.py:25-90` — claude
   `#FF5F00`, codex `#10A37F`, agy `#6E5DE7`) down to flat gray. The terminal itself is fully truecolor-capable (a
   24-bit gradient renders smoothly inside VHS). The PNG visual-snapshot suite already pins exactly the env we need
   (`tests/ace/tui/visual/conftest.py:35-37`: `COLORTERM=truecolor`, `TERM=xterm-256color`, `FORCE_COLOR`/`NO_COLOR`
   unset). Measured: the current fanout GIF has mean saturation ~0.02 vs ~0.11 for the truecolor goldens.
2. **It demos an unrealistic flow.** It uses the bare-git VCS workflow (`#git:nova`) and a three-pane prompt stack with
   one `%model` per pane. The realistic flagship flow is the GitHub VCS workflow (`#gh:<owner>/<repo>`) with the
   alternation shorthand (`%{%m:a | %m:b | %m:c}`) fanning one prompt out to three models.
3. **Nothing actually launches.** The tape fabricates a LaunchApproval notification via `plan_fake_fanout` with no
   `dispatch` payload (approving it would crash in `dispatch_approved_launch_request`,
   `src/sase/agent/launch_request.py:245-285`), so the tape opens the Launch Review modal and quietly Escapes. No agents
   run; none are killed.
4. **No explanatory text overlays.** There is no way to put timed captions on the rendered media. The consolidated
   research in the `sase--research` sidecar repo (`202607/vhs_text_overlay_captions_consolidated.md`, open via
   `/sase_repo`) already recommends the design: per-tape caption sidecars rendered to ASS subtitles and burned in with
   ffmpeg/libass, with the GIF derived from the captioned MP4.

## Product outcome

A ~35-second README hero GIF, in full color, that shows the real end-to-end UX:

1. ACE opens on the Agents tab showing seeded prior runs (colorful badges, statuses).
2. The user presses `space`, and types one prompt on the GitHub workflow:
   `#gh:acme/nova %{%m:opus | %m:gpt53 | %m:pro31h} Harden the command parser against malformed input and add regression tests.`
3. `Enter` submits — this is ACE's real prompt-bar launch path
   (`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_submit.py` → `launch_agents_from_cwd`), which spawns real
   detached agent subprocesses. Three rows appear in the Agents tab with three distinctly colored provider badges:
   CLAUDE(opus) in orange, CODEX(gpt-5.3-codex) in green, AGY(Gemini 3.1 Pro) in indigo.
4. The agents visibly run (elapsed time ticking, then streamed output accumulating).
5. The user kills them on screen (`x` → ConfirmKillModal → `y`), rows flip to killed state.
6. Lower-third captions narrate the four beats.

Design decision — **no Launch Review modal in this demo**: the prompt-bar submit path does not raise a LaunchApproval
gate (gates are created only when launching from inside an agent context or out-of-band via `sase launch request`).
Faking a gate again would keep the demo dishonest; choreographing an out-of-band `sase launch request` would mean the
typed prompt is not what launches. Authenticity wins: type → Enter → live agents → kill, exactly as a user experiences
it. The README copy and alt text change accordingly (the launch-preview story can get its own demo later).

Everything on screen is hermetic: no network, no API keys, no real model calls. The agents are real sase agent processes
executed by the bundled `fakey` runtime while displaying the models the user actually requested (see the `override`
phase).

## Phase `truecolor` — Truecolor demo tapes + full artifact regeneration

Scope: make every demo tape render ACE in truecolor and regenerate all committed demo media.

- In each of the five tapes under `demos/tapes/`, pin the same env the visual snapshot suite pins
  (`tests/ace/tui/visual/conftest.py:35-37`). VHS v0.11.0 supports the `Env` command (verified with `vhs validate`), so
  add near the top of each tape: `Env COLORTERM "truecolor"` and `Env TERM "xterm-256color"`. (Exporting in the hidden
  setup block is an acceptable fallback if `Env` interacts badly with `Set Shell`.)
- Regenerate everything with `just demos -y` (mandatory after tape edits per `demos/tapes/CLAUDE.md`) and commit the
  refreshed `demos/out/` artifacts.
- Add a saturation sanity guard so grayscale regressions cannot silently return: a small executable check (e.g.
  `demos/scripts/check_demo_media`) that samples frames from each rendered GIF and fails if mean saturation is below a
  threshold (calibrate against the regenerated colorful output; the broken artifacts measure ~0.02, healthy truecolor
  ~0.10). Wire it into the `just demos` recipe after rendering. Pillow is already provisioned by the visual-test setup
  (`_setup-visual`); reuse that provisioning pattern rather than adding a new dependency mechanism.
- Update `demos/README.md`: document the truecolor pin and the guard.

Note: `docs/images/blog/` copies are refreshed in the `demo` phase (which builds the deterministic optimization
pipeline); this phase only touches tapes, the guard, and `demos/out/`.

Verification: `just demos -y` runs clean; the guard passes; spot-extract a frame with ffmpeg and confirm colored
provider badges and status colors; `just check`.

## Phase `override` — LLM execution-provider override + fakey demo scenario

Scope: let a launch execute under a different provider than the one the user requested, without changing what the UI
displays — the seam that makes "real models on screen, fakey underneath" possible for demos and cheap end-to-end tests.

- New env knob, provider-agnostic (working name `SASE_LLM_EXEC_PROVIDER`): when set to a registered provider name, the
  invocation layer executes through that provider while the requested model/provider continue to flow into all metadata
  and UI surfaces. The single choke point is `invoke_agent`'s provider lookup (`src/sase/llm_provider/_invoke.py:256`,
  `get_provider(provider_name)`); implement the override there (or inside `registry.get_provider` with an explicit
  opt-in flag from `invoke_agent`) so every runtime is treated uniformly — no per-runtime branches (see the `gotchas`
  memory: Uniform Agent Runtimes).
- Display metadata is untouched: `agent_meta.json` keeps the resolved `llm_provider`/`model` from
  `src/sase/axe/run_agent_directives.py:371-374`, so ACE badges show CLAUDE/CODEX/AGY with the requested models. For
  honesty and debuggability, record the actual executing provider in the run artifacts (e.g. an `exec_llm_provider`
  field in `done.json`/invocation records) — not surfaced in badges.
- Audit launch-time preflights: if any launch/readiness path checks the resolved provider's CLI presence or auth
  (doctor-style checks, `llm_auth_evidence`, autodetect), make the override path validate the _executing_ provider
  instead (fakey requires no auth and ships in the venv: `pyproject.toml:120` console script, `SASE_FAKEY_PATH` override
  honored in `src/sase/llm_provider/fakey.py:36-38`).
- New bundled fakey scenario `demo` (`src/sase/fakey/scenarios/demo.yml`): a short initial `delay` (~2s), then a long
  line-streamed reply (~40 plausible work-log lines, `chunk_delay` ~1s) so a demo agent stays visibly Running for ~45s
  while producing real output, and dies cleanly on SIGTERM (fakey already exits 143 and records
  `outcome.status="terminated"`, `src/sase/fakey/cli.py:220-259`). Scenario body must stay free of `---` lines
  (multi-prompt separator collision).
- Tests: unit coverage for the override resolution (set env → fakey invoked, metadata retains requested provider/model;
  unset → unchanged behavior; unknown provider name → clear error), an e2e through the existing fakey harness
  (`tests/fakey/harness.py`) launching with a real model directive (e.g. `%model:opus`) plus the override and asserting
  the invocation ran fakey while `agent_meta.json` shows `claude`/`opus`, and scenario-format coverage for `@demo`
  alongside the existing bundled scenarios (`tests/fakey/test_scenario.py`).
- Read the `cli_rules` long-term memory first if any CLI surface is added (the env-only design needs none). Document the
  knob where LLM provider env vars are documented.

This is Python provider-layer behavior (execution environment concern), not shared domain logic — no `sase-core`
changes.

Verification: new tests pass; `just check`; manual: `FAKEY_SCENARIO=@demo SASE_LLM_EXEC_PROVIDER=fakey` launch of a
`%m:opus` prompt in a scratch SASE_HOME shows a CLAUDE(opus) badge in ACE while a fakey process runs, and `x`→`y` kills
it cleanly.

## Phase `captions` — Demo media post-processing and caption overlay tooling

Scope: implement the research-recommended caption pipeline as reusable tooling (no tape changes yet). Follow
`202607/vhs_text_overlay_captions_consolidated.md` in the `sase--research` repo.

- Sidecar format `demos/tapes/<name>.captions.yml`, per the research schema: `version`, `defaults` (font "Fira Code",
  size, `position: lower-third` plus `top-right` etc., margins, `fade_ms`, box color/opacity, text color) and `cues`
  (`at`/`until` absolute timestamps, optional per-cue `position`, `text`). Absolute timestamps only (Phase-1 timing
  strategy from the research); marker-based anchoring is explicitly deferred.
- New executable `demos/scripts/postprocess_demo_media` (Python, style-matched to `demos/scripts/seed_sase_ace_demo`'s
  role as a checked-in build tool). Responsibilities per tape name:
  1. Probe the VHS-rendered MP4 with `ffprobe` for real fps/duration (tapes say `Set Framerate 30` but artifacts measure
     25fps — never hard-code).
  2. If `<name>.captions.yml` exists: validate cues (monotonic, within duration, non-empty text), generate a temporary
     `.ass` file (escape text properly; never build `drawtext` strings), and burn it into the MP4 with ffmpeg's `ass`
     filter (`libx264 -crf 18 -pix_fmt yuv420p -movflags +faststart`).
  3. Regenerate the GIF from the (captioned or plain) MP4 with `palettegen`/`paletteuse` at the measured fps.
  4. Optional flags for README/blog derivatives used by the `demo` phase: an optimized downscaled GIF (lanczos scale +
     reduced palette, targeting roughly the current blog-copy size budget) and a deterministic still PNG extracted at a
     named timestamp.
  5. Fail loudly if Fira Code is not resolvable via fontconfig (`fc-match "Fira Code"`), so caption typography cannot
     silently drift from the terminal font.
- **Output policy (decision): replace-in-place.** The committed `demos/out/<name>.mp4` and `demos/out/<name>.gif` are
  the final, captioned artifacts; uncaptioned intermediates are temp files. No `*.captioned.*` twins — this keeps
  exactly one committed artifact set, and keeps README/PyPI URLs stable. Wire the script into the `just demos` recipe
  between the `vhs` runs and the commit prompt, running for every tape (no-op captioning when no sidecar exists, so GIF
  derivation is uniform); fold the `truecolor` phase's saturation guard into or after this step so the guard checks the
  final artifacts.
- Caption visual language (design intent for authors): lower-third placement, Fira Code, white text on a ~70%-opacity
  dark box, ~250ms fade in/out, sparse cues (≤5 per demo, 2.5–4s each), ASS style shape as in the research note.
- Docs: a "Captions" section in `demos/README.md` (schema, replace-in-place policy, retiming guidance after tape edits,
  the fps caveat).
- Tests: unit tests for the script's pure parts — sidecar parsing/validation errors, ASS generation and escaping, fps
  handling (subprocess calls mocked). Follow the repo's existing patterns for testing executable scripts; keep ffmpeg
  integration exercised via `just demos` rather than pytest.

This is build-time presentation tooling; it belongs in this repo, not `sase-core` (research note's conclusion,
consistent with the Rust core boundary memory).

Verification: unit tests pass; `just check`; run the script against an existing rendered MP4 with a scratch sidecar and
eyeball the burned captions and derived GIF.

## Phase `demo` — Live GitHub fan-out launch/kill demo + README refresh

Scope: rebuild the fanout demo end-to-end on the three foundation phases, and refresh the README-facing media. Keep the
artifact base name `sase_ace_multi_model_fanout` — the PyPI readme pipeline substitutes
`sase_ace_multi_model_fanout_still.png` and rewrites paths to `raw.githubusercontent.com/master` URLs
(`pyproject.toml:174-190`), and already-published package READMEs reference these paths forever.

**Demo venv.** `#gh:` resolution is 100% plugin-provided (`sase_workspace` entry points,
`src/sase/workspace_provider/_registry.py:39-57`); this repo registers only `bare_git`. Installing `sase-github`
(published on PyPI, currently 0.1.7) into the main dev venv would change entry-point-sensitive behavior for the whole
test suite, so the `just demos` recipe instead provisions a dedicated demo venv (e.g. `.venv-demos`, gitignored) via
`uv venv` + `uv pip install -e . sase-github`, and the tapes prepend that venv's `bin` to `PATH` instead of `.venv/bin`.
Network to build the venv is acceptable (demo regeneration already requires vhs/ ttyd/ffmpeg); the _recording_ stays
offline.

**Seeder additions (`demos/scripts/seed_sase_ace_demo`).** Add a GitHub-flavored project next to the existing bare-git
`nova` (which other tapes keep using):

- Fictional repo `acme/nova`: a local `git init` checkout at the exact path the plugin derives,
  `$HOME/projects/github/acme/nova/` (fake seeded `$HOME`), reusing the existing nova source tree content. Set `origin`
  to a hosted-looking URL (`git@github.com:acme/nova.git`) — never fetched — because `ws_detect_workflow_type`
  classifies gh by a hosted origin URL and the absence of `BARE_REPO_DIR`. Seed local remote-tracking refs so
  default-branch resolution works offline (`git update-ref refs/remotes/origin/main <sha>` and
  `git symbolic-ref refs/remotes/origin/HEAD refs/remotes/origin/main`).
- Project record `$SASE_HOME/projects/gh_acme__nova/gh_acme__nova.sase` with `PROJECT_NAME`, `PROJECT_STATE: active`,
  `WORKSPACE_DIR` pointing at that checkout, **no `BARE_REPO_DIR`**, and a small ChangeSpec section for visual variety.
  Pre-seeding the record plus the checkout makes the typed `#gh:acme/nova` form resolve fully offline
  (`_resolve_repo_path_ref` only clones when the workspace dir is missing).
- Add one or two `#gh:acme/nova` prompts to the seeded prompt history.

**Tape rewrite (`demos/tapes/sase_ace_multi_model_fanout.tape`).** Delete the fake-notification Python heredoc entirely.
New choreography (~35s target; current renders run 20–32s at 25fps):

1. Hidden setup: seed, demo-venv PATH, truecolor env, `FAKEY_SCENARIO=@demo`, `SASE_LLM_EXEC_PROVIDER=fakey`,
   `FAKEY_STATE_DIR` under the demo root, then `sase ace --refresh-interval 0 --tab agents -x`... but note: live agent
   status needs refresh — verify the row updates with `--refresh-interval 0` (manual-refresh keymap or a small interval
   like `--refresh-interval 2` are the fallbacks; pick whichever keeps the recording deterministic).
2. `space` → prompt bar. Type the single-pane prompt with completion beats (pauses on the `#gh:` and `%{`/`%m:`
   completions):
   `#gh:acme/nova %{%m:opus | %m:gpt53 | %m:pro31h} Harden the command parser against malformed input and add regression tests.`
   Model aliases resolve to registered providers: claude `opus`, codex `gpt53` → `gpt-5.3-codex`, agy `pro31h` →
   `Gemini 3.1 Pro (High)`. (If the `#pr:` rollover xprompt from sase-github expands hermetically, append
   `#pr:harden_parser` for extra realism — verify during implementation; drop if it complicates the recording.)
3. `Enter` submits (single pane → direct real launch). `Wait+Screen` for the three new rows / provider badges (`/opus/`,
   `/gpt-5.3/`, `/Gemini 3.1/`).
4. Let them run ~8–10s: elapsed time ticks, `@demo` scenario streams work-log lines; optionally focus one agent's output
   panel briefly.
5. Kill beat: `x` → ConfirmKillModal renders → `y`; then kill the remaining two (individually, or the kill-all flow if
   it choreographs more cleanly). `Wait+Screen` for killed status.
6. Short hold on the final state; hidden teardown.

Known caveat to embrace: a single-pane `%{...}` prompt does not show "· 3 agents" in the prompt-bar title (that counter
counts `---` panes) — the caption narrates the fan-out instead. (A slot-count preview for alternation prompts is noted
under Future work; it needs Rust fan-out planning per keystroke, so it must clear the `tui_perf` memory before anyone
attempts it.)

**Captions sidecar (`demos/tapes/sase_ace_multi_model_fanout.captions.yml`).** Four cues, lower-third, timed against the
rendered MP4 (draft copy — tune at implementation):

1. While typing the alternation: "One prompt, three models — %{ %m:a | %m:b | %m:c }"
2. On submit: "Enter launches all three — real agents, isolated workspaces"
3. While running: "Claude, Codex, and Antigravity, side by side"
4. On the kill: "Kill any run instantly — you stay in control"

**README-facing media + copy.**

- Regenerate `demos/out/sase_ace_multi_model_fanout.{mp4,gif}` (captioned, truecolor) via `just demos -y`.
- Use `postprocess_demo_media`'s derivative flags to deterministically produce
  `docs/images/blog/sase_ace_multi_model_fanout.gif` (optimized, roughly the current ~1.8MB budget; hard ceiling well
  under GitHub's 10MB) and `docs/images/blog/sase_ace_multi_model_fanout_still.png` (frame at a named timestamp showing
  the three colored badges). Also refresh the other stale blog copies/stills now that renders are colorful
  (`agents_observability_still.png`, `sase_ace_prs_pipeline_still.png`, and the blog GIF copies referenced by the draft
  blog post).
- Update the README "See it in action" fanout blurb + alt text: one prompt fans out via the alternation shorthand on the
  GitHub workflow into three live agents across Claude, Codex, and Antigravity, which are then killed from the same
  surface. Keep `docs/images/blog/...` as the img src. Confirm the `pyproject.toml` fancy-pypi-readme substitution
  patterns still match.
- Update `demos/README.md`'s fanout-tape description (it currently documents the stop-at-the-modal behavior) plus the gh
  seed and demo-venv notes.

Verification: `just demos -y` end-to-end (render → postprocess → saturation guard → commit prompt); frame-extract review
of the final GIF (colors, captions present at cue midpoints, badges CLAUDE/CODEX/AGY, killed states);
`demos/out/sase_ace_multi_model_fanout.gif` and the blog derivative within size budgets; `just check`; full `just test`
(the seeder and tape are not under pytest, but the override/captions/fakey changes are).

## Risks and mitigations

- **Live-agent nondeterminism in a recording.** Real subprocesses make timing less scripted than seeded artifacts.
  Mitigate with `Wait+Screen` on badge/status regexes (the existing tape pattern), the deterministic `@demo` scenario
  pacing, and generous `WaitTimeout`. If the Agents tab needs a refresh tick to show live rows, prefer the smallest
  deterministic mechanism (manual refresh keymap at fixed beats, or a small `--refresh-interval`).
- **Launch preflight rejects uninstalled real CLIs.** The `override` phase explicitly audits and covers this with tests;
  if an unexpected preflight surfaces during tape work, stub binaries on the demo venv PATH as a last resort and file
  the gap.
- **`#gh:acme/nova` triggers a network clone.** Only happens if the seeded checkout path or project record is wrong; the
  seeder pre-creates both, and the recording environment can be verified offline-safe by running the seed + a resolution
  smoke in a network-less shell during implementation.
- **Caption cue drift after tape edits.** Accepted Phase-1 cost per the research: cues are sparse and retimed against
  the rendered MP4; the sidecar lives next to the tape to keep them reviewed together. Marker-based anchoring is
  deferred until drift is a demonstrated maintenance burden.
- **Artifact size growth.** Truecolor + captions increase GIF palette pressure; the fanout demo also gets longer.
  `palettegen`/`paletteuse` with measured fps plus the optimized blog derivative keep the README asset near its current
  budget; the saturation guard script can also warn on size regressions.
- **sase-github version skew.** The demo venv installs the published `sase-github`; if master needs unreleased plugin
  behavior, pin or install from a local checkout opened via `/sase_repo` — decide at implementation and document in
  `demos/README.md`.

## Explicit non-goals / future work

- No VHS fork and no dependence on upstream caption PRs (#719/#735); revisit if native `Overlay` ships.
- No marker/SSIM-based caption anchoring (Phase 2 of the research) yet.
- No prompt-bar slot-count preview for `%{...}` fan-out (needs `tui_perf` review).
- No Launch Review / approval-gate beat in this demo; a future demo can showcase gates using a real
  `sase launch request` gate (the fake-notification pattern should not be reused).
- No changes to `sase-core` (alternation parsing and fan-out planning are untouched) and no changes to `sase/memory/`
  files.
