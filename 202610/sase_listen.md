---
tier: epic
title: 'sase-listen: narrated, chaptered audio editions of Markdown for the commute'
goal: 'Any SASE research report, or any other Markdown file, becomes a chaptered,
  loudness-normalized MP3 audio edition from one CLI command or one Telegram message.
  The episode reaches the phone through Telegram''s music player and a private podcast
  feed. The new public sase-org/sase-listen repo ships with a real description, excellent
  documentation linked back to the originating research and this plan, CI lint/type/test
  gates, and automated release-please plus PyPI trusted publishing.

  '
phases:
- id: scaffold
  title: Repo foundation, packaging, CI, and release automation
  depends_on: []
  size: medium
  description: 'scaffold: build the sase-listen package skeleton. That covers every
    runtime dependency declared up front, the config loader, the CLI command registry
    with stubs, and the docs site skeleton. Add CI, PR-title, docs-deploy, and release-please
    plus PyPI publish workflows, then set the GitHub description, topics, homepage,
    Pages, and workflow permissions.'
- id: telegram-audio
  title: sase-telegram delivers MP3s through sendAudio
  depends_on: []
  size: small
  description: 'telegram-audio: add send_audio to sase-telegram''s client and route
    .mp3/.m4a attachments to it in the outbound loop. Read title, performer, and duration
    from ID3, guard the 50 MB limit, fall back to a document send, and add tests and
    docs.'
- id: script
  title: Narration script contract, deterministic normalizer, lexicon, lint, and guide
  depends_on:
  - scaffold
  size: medium
  description: 'script: implement the narration-script v1 model and parser and the
    markdown-it AST normalizer with an omissions report and golden fixtures. Add the
    pronunciation lexicon, `lint` (including the --source number-fidelity check),
    and the packaged authoring guide behind `guide`.'
- id: audio
  title: Mastering, MP3 packaging, chapters, and cover art
  depends_on:
  - scaffold
  size: medium
  description: 'audio: resolve ffmpeg (bundled imageio-ffmpeg fallback). Trim silence
    and assemble PCM with gaps, then run two-pass loudnorm to a 64 kb/s mono MP3.
    Write ID3v2.3 tags, CHAP/CTOC chapters, and cover art, including the generated
    title-card design.'
- id: engines
  title: TTS engines, narrator profiles, credentials, retries, cache, and pricing
  depends_on:
  - scaffold
  size: medium
  description: 'engines: add the Engine protocol and the Gemini, OpenAI-compatible,
    and offline tone adapters. Add narrator profiles, secret resolution from env or
    command, retry with backoff, the content-addressed LRU chunk cache, and dated
    pricing estimates. Verify Gemini with a single live call.'
- id: pipeline
  title: Render orchestration, quality gates, manifest, and episode library
  depends_on:
  - script
  - audio
  - engines
  size: medium
  description: 'pipeline: wire `render`, which takes a script, Markdown, or artifact
    ref and produces an MP3 with chunk planning, concurrency, intro and outro, chunk-
    and episode-level quality gates with targeted re-synthesis, the manifest, an atomic
    library commit, locks, `--json`, and exit codes. Prove it end to end with the
    tone engine.'
- id: cli
  title: Beautiful CLI experience and companion commands
  depends_on:
  - pipeline
  size: medium
  description: 'cli: build the rich terminal experience, covering the dry-run plan
    table, live per-chapter progress, and summary panels. Add the `audition`, `ls`,
    `doctor`, `cache`, and `config` commands, polished help, NO_COLOR and non-TTY
    behavior, and SVG terminal captures for the docs.'
- id: feed
  title: Private podcast feed for AntennaPod
  depends_on:
  - cli
  size: medium
  description: 'feed: add `feed init`/`feed`/`publish`/`unpublish`. Publishing writes
    into a served-only feed directory with an RSS 2.0 + iTunes + Podcasting 2.0 feed.xml,
    per-episode chapters JSON, and generated channel art. Add retention, auto-publish
    for research episodes, a token-path URL with a QR code, and Tailscale Funnel serving
    docs.'
- id: research-audio
  title: '#research/audio xprompt and research_swarm audio stage'
  depends_on:
  - cli
  size: medium
  description: 'research-audio: in sase-research-artifacts, add the #research/audio
    xprompt, which writes <stem>_narration.md via `sase-listen guide`/`lint`, renders,
    and registers the MP3. Add the opt-in research_swarm `audio` stage after the linker,
    the @audio model alias, the narration companion exclude glob, tests, and docs.'
- id: docs
  title: Documentation polish and provenance links
  depends_on:
  - feed
  - research-audio
  - telegram-audio
  size: medium
  description: 'docs: finish the README and mkdocs site, covering the hero, quickstart,
    how it works, narration scripts, CLI reference, configuration, narrators, pronunciation,
    feed, SASE integration, reliability, and troubleshooting. Include background pages
    with permalinks to the originating research and this epic plan, plus CONTRIBUTING
    and AGENTS.md.'
- id: release
  title: First releases to PyPI
  depends_on:
  - docs
  size: small
  description: 'release: verify master CI and the built wheel. Propose merging the
    sase-listen 0.1.0 release-please PR, plus the sase-telegram and sase-research-artifacts
    release PRs, through gates, then confirm that trusted publishing put each version
    on PyPI.'
- id: rollout
  title: Install, configure, field-test, and turn on delivery on apollo
  depends_on:
  - release
  size: medium
  description: 'rollout: install sase-listen and upgrade the plugins on apollo, and
    add the config through chezmoi. Run doctor, deliver voice auditions and the first
    real audio edition (the commute-audio research itself) to Telegram, and initialize
    the feed. Record field notes and a docs sample, then propose Funnel exposure through
    a gate.'
proposed_by: bbugyi200.apollo.3z
create_time: 2026-10-01 14:42:34
status: wip
bead_id: sase-1e3
---

- **PROMPT:** [prompts/202610/sase_listen.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/sase_listen.md)
- **BEAD:** [sase-1e3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1e3/README.md)

# Plan: sase-listen — narrated audio editions of Markdown

## Context

### Where this comes from

- **Research (primary input):**
  `research:202610/commute_audio_from_markdown/commute_audio_from_markdown.md`.
  - Permalink:
    <https://github.com/sase-org/sase--research/blob/617bce562f5a01dfae5999fc0caf3f6a12baf1aa/202610/commute_audio_from_markdown/commute_audio_from_markdown.md>.
  - It consolidates five researcher reports in the same directory. `__cdx` has the
    strongest renderer engineering and `__cld` has the measurements and pipeline.
  - It builds on the earlier `research:202606/sase_audio_generation_consolidated.md`.
  - Read these with `sase artifact read`, not by opening files.
- **The user's answers to that report's open questions:**
  - They commute and walk _very often_.
  - The default edition is **`full`** (about 16 minutes).
  - Scope is research reports only, with **no special handling for plans or Obsidian
    notes**. The CLI still accepts any Markdown file.
  - The narrator-style question went unanswered. This plan uses one narrator and leaves
    out a two-host mode.
- **Repo state:**
  - `sase-org/sase-listen` is public. It contains only an empty `README.md` from commit
    `e70e585abac3d0cbe856cfa04092055ba8b0513d` ("feat: First commit").
  - The description is `TODO`, and there are no topics, homepage, Pages, environments,
    or secrets.
  - Workflow permissions are `read` with `can_approve_pull_request_reviews: false`.
  - PyPI has a pending trusted publisher: repository `sase-org/sase-listen`, workflow
    `publish.yml`, environment `pypi`.

### Facts verified while planning

- apollo has no `ffmpeg`/`ffprobe`. `imageio-ffmpeg` 0.6.0 bundles a static **ffmpeg
  7.0.2** (johnvansickle build) that includes `libmp3lame`, `loudnorm`, `silenceremove`,
  and `ebur128`. A pip dependency therefore gives every machine a working media engine
  with no sudo.
- `pass` holds `gemini_cli_api_key` and `chatgpt_sase_api_key`. `GEMINI_API_KEY`,
  `GOOGLE_API_KEY`, and `OPENAI_API_KEY` are not set in agent shells.
- `sase plugin install` adds a plugin to sase's `uv tool` environment with `--with` and
  does **not** put the plugin's console scripts on `PATH`. A CLI that agents and humans
  invoke should therefore be its own `uv tool install`.
- sase-telegram (`src/sase_telegram/scripts/sase_tg_outbound.py`) has image, animation,
  video, and PDF branches. An `.mp3` currently falls through to `md_to_pdf` and then
  `send_document`.
  - `_format_workflow_complete` keeps MP3 attachments.
  - Explicit artifacts (`sase artifact create`) ride the agent's completion notification
    `files`.
  - `telegram_client.py` wraps python-telegram-bot and has no `send_audio`.
- sase-research-artifacts:
  - `#research/image` is the precedent for a forked post-lead stage
    (`%wait:research.{@1}.final … #fork:research.{@1}.final #research/image`).
  - With the linker on, the lead writes `<name>__final.md` and the linker publishes
    `<name>.md`.
  - `_COMPANION_MARKDOWN_EXCLUDE_GLOBS` in `provider.py` keeps companion Markdown out of
    the `@research` inventory and the Highlights hook.
- Sibling plugin repos use hatchling, uv, just, ruff, mypy, pytest, release-please
  (`release-type: python`), Conventional Commit PR titles, and the same `publish.yml`
  shape: release-please, build, install smoke, then publish with `environment: pypi` and
  `id-token: write`. Their release PRs use a repo secret named `SASE_RELEASE_TOKEN`, and
  sase-listen has no secrets.

## Design decisions

These are owned by this plan and are not up for re-litigation by phase workers.

1. **sase-listen is a standalone tool, not a sase plugin.**
   - It never imports `sase`, declares no `sase_*` entry points, and does not get the
     `sase--plugin` topic.
   - It installs with `uv tool install sase-listen`, so the CLI is on `PATH`, and CI
     needs no sase or Rust checkouts.
   - SASE integration lives where its precedents live:
     - `#research/audio` and the swarm stage go in **sase-research-artifacts**, next to
       `#research/image`.
     - Audio delivery goes in **sase-telegram**.
   - No `sase` or `sase-core` changes. Rendering audio is not shared domain behavior
     that another frontend must match, which is the Rust-boundary litmus test.
2. **The narration script is the contract, and the CLI owns it.**
   - For research, an agent writes an "audio edition" script. Any other Markdown goes
     through a deterministic AST normalizer.
   - The renderer never knows which producer wrote the script.
   - `sase-listen guide` prints the authoring rules and `sase-listen lint` checks them.
     Both ship in the same package as the parser and renderer, so the rules never drift
     from the code that enforces them.
   - xprompts reference the CLI instead of copying the rules.
3. **One narrator per episode.**
   - A _narrator_ is a named profile: engine, model, voice, and style.
   - An episode never mixes engines. A failure never falls back silently to another
     narrator; the error tells the user how to re-render the whole episode with another
     one.
4. **Engines:**
   - The default is **Gemini 3.8 Flash TTS** (`gemini-3.8-flash-tts`, voice `Charon`
     until the audition picks one), using a fixed calm technical-briefing style.
   - An **OpenAI-compatible** adapter covers OpenAI `gpt-4o-mini-tts-2025-12-15` and,
     through `base_url`, a future Kokoro-FastAPI.
   - An offline **`tone`** engine makes deterministic, speech-paced synthetic audio for
     tests, CI smoke, and demos.
5. **Output format:**
   - MP3, mono, 24 kHz, 64 kb/s CBR.
   - Two-pass `loudnorm` to −16 LUFS integrated and −1.5 dBTP.
   - ID3v2.3 tags with CHAP/CTOC chapters, one per `##`, and an embedded cover.
   - About 7 MB per 15 minutes, well under Telegram's 50 MB bot limit.
6. **Delivery:**
   - Telegram `sendAudio`, through the completion notification's explicit MP3 artifact.
   - A private RSS feed for AntennaPod, served by Tailscale Funnel on `:8443` at a
     secret path. Port 443 stays tailnet-only for `sase_gateway`.
   - Audio never goes into git. Narration scripts are committed next to their reports as
     `<stem>_narration.md`.
7. **Beautiful is a requirement, not polish.**
   - **Sound:** a spoken AI-disclosure intro and a short outro, spoken chapter headings,
     measured pauses (0.5 s between chunks, 1.2 s between chapters), consistent
     loudness, and long silences compressed.
   - **Visuals:** a generated, consistent cover art system.
   - **Terminal:** rich tables, panels, and live progress, with clean `--json` for
     agents.
   - **Docs:** a styled mkdocs site with real terminal SVG captures and an embedded
     sample episode.
8. **Out of scope** (each is a follow-up idea, not part of this epic):
   - the daily digest and its file hook;
   - the ASR (Whisper) quality gate;
   - a two-host mode;
   - deploying Kokoro on athena;
   - plans and Obsidian scope;
   - an `audio` artifact kind;
   - registering sase-listen as a linked repo in the sase project. That would regenerate
     `sase/memory/sase.md`, so it needs explicit user approval as a separate change.

## Shared contracts

Every phase must honor these. Parallel phases depend on them to stay compatible.

### Narration script v1

```markdown
---
narration: 1 # required schema marker
title: One Updates Tab # required; episode title (ID3 TIT2, feed item title)
source: research:202609/admin_center_updates_tab_unification.md # ref or path; optional
source_blob: 3f9c2e1… # optional `git hash-object` of the source the script came from
date: 2026-09-14 # optional source date, spoken in the intro
kind: research # research | document (default: document)
edition: full # full | brief | digest | verbatim
producer: agent # agent | deterministic
target_minutes: 15 # optional
cover: one_updates_tab_infographic.png # optional; path relative to the script file
---

## The question

Plain spoken prose. Paragraphs are separated by blank lines…

## The short answer

…
```

- **Body:** only `##` headings (chapters) and plain paragraphs.
  - Every `##` is an ID3 chapter and a synthesis boundary.
  - Text before the first `##` is a structural error.
- **Spoken intro:** the renderer adds the AI disclosure. Scripts must not.
  - `full`, `brief`, and `digest`: "This is an AI-narrated audio edition of _Title_,
    SASE research from _September 14, 2026_." Omit the kind phrase when `kind` is not
    research, and omit the date phrase when there is no `date`.
  - `verbatim`: "This is an AI-narrated reading of _Title_ …".
- **Outro:** "That's the end of this audio edition of _Title_." Both templates are
  configurable.
- **Edition budgets** at 150 wpm:

  | Edition    | Word budget              | About  |
  | ---------- | ------------------------ | ------ |
  | `full`     | ≤ 2,400 words            | 16 min |
  | `brief`    | about 600 words          | 4 min  |
  | `digest`   | about 250 words per item | —      |
  | `verbatim` | no budget                | —      |

- **Files:** agent scripts are named `<stem>_narration.md`, where `<stem>` is the report
  stem with any trailing `__final` removed.

### CLI surface

The console script is `sase-listen`, and `python -m sase_listen` also works. It uses
argparse with `rich-argparse`.

| Command                                                                                                                                | Owner phase        | Purpose                                                                                                               |
| -------------------------------------------------------------------------------------------------------------------------------------- | ------------------ | --------------------------------------------------------------------------------------------------------------------- |
| `render SOURCE [-o PATH] [-n NARRATOR] [--voice V] [--cover IMG] [--dry-run] [--publish/--no-publish] [--no-cache] [--force] [--json]` | pipeline (UX: cli) | Turn a narration script, plain Markdown (normalized deterministically), or a `kind:path` artifact ref into an episode |
| `script SOURCE [-o PATH] [--json]`                                                                                                     | script             | Deterministic Markdown → narration script (`edition: verbatim`), with an omissions report                             |
| `lint SCRIPT [--source REPORT] [--strict] [--json]`                                                                                    | script             | Validate a script against the contract and listenability rules                                                        |
| `guide [--edition full\|brief]`                                                                                                        | script             | Print the packaged authoring guide for agents                                                                         |
| `audition [--voices A,B] [-n NARRATOR…] [--text FILE] [--json]`                                                                        | cli                | Render the built-in sample passage once per voice or narrator                                                         |
| `ls [EPISODE] [--json]`                                                                                                                | cli                | Library table, or one episode's details and chapters                                                                  |
| `doctor [--online] [--json]`                                                                                                           | cli (feed extends) | Check config, ffmpeg capabilities, credentials, writable dirs, disk, and feed                                         |
| `cache [prune [--all\|--older-than 30d]] [--json]`                                                                                     | cli                | Cache size and pruning                                                                                                |
| `config [init\|path] [--json]`                                                                                                         | cli                | Show the effective config with each value's origin; `init` writes a commented starter file                            |
| `feed [init --base-url URL\|rebuild\|prune] [--qr] [--show-url] [--json]` and `publish EPISODE`, `unpublish EPISODE`                   | feed               | Private podcast feed                                                                                                  |

- **Exit codes:**

  | Code | Meaning                                                               |
  | ---- | --------------------------------------------------------------------- |
  | 0    | OK                                                                    |
  | 1    | Unexpected failure                                                    |
  | 2    | Usage                                                                 |
  | 3    | Configuration or credentials                                          |
  | 4    | Synthesis failed after retries                                        |
  | 5    | Quality gate failed                                                   |
  | 6    | Script has structural lint errors (`render` refuses unless `--force`) |

  `lint` exits 1 on errors, or on warnings with `--strict`.

- **`--json`** prints exactly one JSON object to stdout and nothing else there.
  - `render --json` success returns
    `{"ok": true, "episode_id", "title", "audio_path", "manifest_path", "duration_s", "size_bytes", "chapters": [{"title", "start_s"}], "narrator": {"name", "engine", "model", "voice"}, "chunks": {"total", "cached", "synthesized", "retried"}, "loudness_lufs", "cost_usd_estimate", "published", "warnings": []}`.
  - Failures return `{"ok": false, "error": {"code", "message", "hint"}}`.
- **Ref resolution:** a `SOURCE` like `research:202610/x.md` that is not an existing
  path is resolved with `sase artifact read <ref> "sase-listen render"`, so the read is
  audited and managed link tables are stripped. This needs `sase` on `PATH` and is never
  imported. If `sase` is missing, exit 3 with a hint.

### Engines, narrators, and cache

- **Engine protocol:**
  - `synthesize(SynthesisRequest(text, model, voice, style, speed)) -> SynthesisResult(pcm, sample_rate, usage)`,
    where `pcm` is s16le mono. The canonical rate is 24 kHz; mastering resamples
    anything else.
  - `limits(model) -> EngineLimits(max_chars, target_words, default_concurrency)`.
  - Errors: `TransientEngineError(retry_after)` covers 429, 5xx, timeouts, connection
    errors, and empty audio. `PermanentEngineError` covers 400 and invalid input.
    `CredentialsError` covers missing keys and 401/403.
- **Built-in narrators:**

  | Narrator               | Engine | Model                        | Voice    | Style                                                                                                                   |
  | ---------------------- | ------ | ---------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------- |
  | `gemini` (the default) | gemini | `gemini-3.8-flash-tts`       | `Charon` | "Calm, clear technical-briefing narrator. Moderate pace. Slight emphasis on numbers. Read the text exactly as written." |
  | `gemini-lite`          | gemini | `gemini-3.8-flash-lite-tts`  | —        | —                                                                                                                       |
  | `openai`               | openai | `gpt-4o-mini-tts-2025-12-15` | `marin`  | equivalent `instructions`                                                                                               |
  | `tone`                 | tone   | —                            | —        | —                                                                                                                       |
  - Users add their own profiles, for example `kokoro` as engine `openai` with a
    `base_url`.
  - Gemini style goes through the API's documented style channel, never inlined into the
    transcript text. The engines phase verifies this channel live.

- **Secrets:** first the env names (Gemini: `SASE_LISTEN_GEMINI_API_KEY`,
  `GEMINI_API_KEY`, `GOOGLE_API_KEY`; OpenAI: `SASE_LISTEN_OPENAI_API_KEY`,
  `OPENAI_API_KEY`), then `api_key_command`, for example `pass show gemini_cli_api_key`.
  - The command runs without a shell, has a 15 s timeout, and uses the first line of
    output.
  - Secrets never appear in logs, manifests, errors, or `config` output.
- **Chunk cache:**
  - The key is the sha256 of canonical JSON
    `{"v": CACHE_VERSION, engine, model, voice, style, speed, sample_rate, text}`, where
    `text` is **after** lexicon application.
  - Entries are `$XDG_CACHE_HOME/sase-listen/chunks/<k[:2]>/<k>.wav` plus a `.json` with
    duration, words, usage, and created time.
  - Writes are atomic, hits touch mtime, and LRU eviction enforces `cache.max_gb`
    (default 2) after each render.
  - The cache is also the resume mechanism: a killed render re-run pays only for missing
    chunks.

### Library, manifest, config, and paths

All paths follow XDG on every POSIX platform, including macOS, so dotfile users get
predictable locations.

- **Config:** `$SASE_LISTEN_CONFIG`, else `$XDG_CONFIG_HOME/sase-listen/config.yml`,
  else `~/.config/sase-listen/config.yml`.
  - Precedence: CLI flags > `SASE_LISTEN_*` env > file > built-in defaults.
  - Unknown keys are errors, with a did-you-mean suggestion.
- **Config sections:**
  - `narrator`;
  - `narrators.<name>.{engine, model, voice, style, speed, base_url}`;
  - `engines.gemini|openai.{api_key_env, api_key_command, base_url, concurrency (3), timeout_s (180), max_retries (4)}`;
  - `audio.{bitrate_kbps 64, sample_rate 24000, loudness_lufs -16, true_peak_db -1.5, chunk_gap_s 0.5, chapter_gap_s 1.2, intro_gap_s 0.9}`;
  - `intro` and `outro` templates (an empty string disables the outro);
  - `author`;
  - `lexicon` (a user file merged over the packaged default);
  - `cache.max_gb`;
  - `feed.*` (the feed phase).
- **Library:** `$XDG_DATA_HOME/sase-listen/library/<episode-id>/`.
  - Contents: `<slug>.mp3`, `manifest.json`, `script.md` (the exact script rendered),
    `cover.jpg`, and `chapters.json` (Podcasting 2.0 JSON chapters).
  - `<episode-id>` is `<slug(title)[:60]>-<sha256(source or absolute script path)[:6]>`.
    Re-rendering the same source atomically replaces its episode.
  - Auditions go under `library/auditions/<timestamp>/`.
- **State:** `$XDG_STATE_HOME/sase-listen/` holds `locks/<episode-id>.lock` (fcntl) and
  daily debug logs that never contain script bodies or secrets.
- **`manifest.json`:**
  - `schema_version`, `episode_id`, `title`, `sase_listen_version`, `created_at`;
  - `source {ref|path, sha256, blob}`;
  - `script {sha256, producer, edition, words}`;
  - `narrator {name, engine, model, voice, style_sha256}`;
  - `lexicon_sha256`;
  - `chunks[] {index, chapter, words, cache_key, duration_s, attempts, cached}`;
  - `chapters[] {title, start_ms, end_ms}`;
  - `audio {file, bytes, duration_s, bitrate_kbps, loudness_lufs, true_peak_db}`;
  - `gates[]`, `omissions` (deterministic producer), `cost_usd_estimate`, `published`.

### Repository rules for parallel phases

`script`, `audio`, and `engines` run in parallel against the same repo.

- `scaffold` therefore declares **every runtime dependency** and commits `uv.lock`.
- It also pre-creates each phase's module stubs, test directories, CLI command stubs,
  and docs pages, so parallel phases edit disjoint files.
- Parallel phases must not edit `pyproject.toml`, `uv.lock`, `mkdocs.yml` nav, or
  another phase's modules. If that becomes unavoidable, keep the edit minimal and
  mention it in the phase notes.
- Every phase updates its own docs page and tests. Workers open the repo with
  `sase repo open gh:sase-org/sase-listen -r "<why>"`, read its `AGENTS.md`, and use
  Conventional Commit subjects.

## Phase: scaffold

Open `gh:sase-org/sase-listen`. Bootstrap it to the sibling-plugin standard, minus the
sase coupling.

1. **Packaging**
   - `pyproject.toml`: hatchling with a `src/sase_listen` layout,
     `requires-python >=3.12`, MIT license, version `0.0.0` (release-please owns it
     afterwards), and a strong description and keywords.
   - `[project.urls]`: Homepage (the docs site), Documentation, Repository, Issues, and
     Changelog.
   - `[project.scripts] sase-listen = "sase_listen.cli:main"`.
   - Runtime dependencies, all declared now:
     - `rich`, `rich-argparse`, `PyYAML`;
     - `markdown-it-py`, `mdit-py-plugins`;
     - `google-genai`, `httpx`;
     - `mutagen`, `imageio-ffmpeg`, `numpy`, `Pillow`;
     - `segno` (QR codes).
   - Dev group: `ruff`, `mypy`, `types-PyYAML`, `pytest`, `pytest-cov`, `codespell`,
     `build`, `twine`, `mkdocs-material`.
   - Commit `uv.lock`. `__version__` comes from `importlib.metadata`.
2. **Skeleton**
   - Package modules, each a typed stub with a docstring naming its owner phase:
     - `cli/` (an `app.py` registry plus one module per command; each stub exits 1 with
       a friendly "not implemented yet" message);
     - `config.py`, `paths.py`, `errors.py` (the exit-code enum);
     - `script/`, `normalize/`, `lexicon.py`;
     - `engines/`, `cache.py`, `pricing.py`;
     - `audio/`;
     - `pipeline.py`, `library.py`, `manifest.py`;
     - `feed.py`, `ui.py`.
   - Package data dir: `data/` (guide, lexicon, sample passage, font, pricing).
   - Implement `config.py` and `paths.py` fully now, since every phase shares them: XDG
     resolution, YAML loading, typed dataclass defaults for **every section in Shared
     contracts**, env overrides, unknown-key errors, and origin tracking for `config`
     output. Add tests.
   - `--version` works.
3. **Tooling**
   - Justfile recipes:
     - `install` (`uv sync --locked --all-groups`), `fmt`, `lint`, `test`;
     - `test-live` (marker `live`, skipped without `SASE_LISTEN_LIVE=1`);
     - `golden` (regenerate golden fixtures);
     - `docs`, `docs-build`, `build`, `demo` (tone engine);
     - `check`, guarded like the siblings: `sase/sase.yml` with a `tools.check` entry
       plus a `tools/require_tool_run` POSIX guard.
   - `lint` runs `ruff check`, `ruff format --check`, `mypy --strict src`, and
     `codespell`.
   - ruff selects `E,W,F,B,C4,UP,I,SIM,RUF` with line length 88.
   - pytest uses `--strict-markers` and coverage with `fail_under = 90`, enforced from
     the pipeline phase onward. Until then, set it to the current level and leave a
     comment pointing to the pipeline phase.
   - Also add `.gitignore`, `LICENSE` (MIT, "Copyright (c) 2026 Bryan Bugyi"),
     `CLAUDE.md` (`@AGENTS.md`), and an initial `AGENTS.md`: overview, just recipes,
     architecture map, parallel-phase rules, the `sase tool run check` guidance (with
     `SASE_TOOL_BYPASS='<why>' just check` as a fallback when no catalog resolves), and
     the no-sase-import rule.
4. **Workflows** in `.github/workflows/`:
   - **`ci.yml`:**
     - Triggers: push and PR to `master`, plus a weekly `schedule`.
     - Matrix: ubuntu-latest × Python 3.12/3.13/3.14, plus macos-latest × 3.12.
     - Steps: `astral-sh/setup-uv`, `extractions/setup-just`, `just install`,
       `just lint` (one leg), `just test`, `just docs-build` (one leg), and `uv build`
       followed by `twine check --strict dist/*`.
     - The scheduled run adds a `latest-deps` leg using `uv sync --upgrade` to catch
       google-genai and other upstream breakage early.
   - **`pr-title.yml`:** the siblings' Conventional Commits regex.
   - **`docs.yml`:** on push to `master`, `mkdocs build --strict`, then
     `actions/upload-pages-artifact` and `actions/deploy-pages`.
   - **`publish.yml`:** the sibling shape: `release` (release-please-action@v5, token
     `secrets.SASE_RELEASE_TOKEN || secrets.GITHUB_TOKEN`), then `build` (`uv build`),
     then `install-smoke`, then `publish` (`environment: pypi`,
     `permissions: id-token: write`, `pypa/gh-action-pypi-publish@release/v1`). Keep the
     `workflow_dispatch` `publish_existing` escape hatch.
     - `install-smoke` installs the wheel into a fresh venv and runs
       `sase-listen --version` and `sase-listen doctor --json` (offline).
     - Phases extend `install-smoke` with a tone-engine `render` of a packaged fixture,
       asserting chapters with mutagen.
   - **release-please:** `release-please-config.json` with `release-type: python`,
     `bootstrap-sha: e70e585abac3d0cbe856cfa04092055ba8b0513d`,
     `initial-version: 0.1.0`, `include-v-in-tag`, `bump-minor-pre-major`,
     `bump-patch-for-minor-pre-major`, and component `sase-listen`. Manifest:
     `{".": "0.0.0"}`.
   - **Nobody merges the release PR** until the release phase.
5. **mkdocs-material site**
   - Theme palette chosen to match the cover-art system: dark and light, one accent.
   - Nav stubs: Home, Getting started, Narration scripts, CLI, Configuration, Narrators
     & voices, Pronunciation, Podcast feed, SASE integration, Architecture, Reliability,
     Troubleshooting, Background, Changelog.
   - Expand `README.md` to a short "coming soon" description with badges: CI, PyPI,
     Python, License, Docs.
6. **GitHub settings** (`gh`; these are outward-facing but explicitly requested):
   - Description:
     `Turn Markdown into chaptered, loudness-normalized MP3 audio editions — narrated by Gemini TTS, built for listening to SASE research on a commute or walk.`
   - Topics: `text-to-speech tts markdown podcast audiobook gemini cli python sase`.
   - Homepage: `https://sase-org.github.io/sase-listen/`.
   - Enable Pages with `build_type=workflow`.
   - Create the `pypi` environment.
   - Set workflow permissions with `can_approve_pull_request_reviews=true`, so
     release-please can open PRs with `GITHUB_TOKEN`.
   - Record a phase note for the user: adding a `SASE_RELEASE_TOKEN` repo secret makes
     release PRs trigger CI, as in the sibling repos.
7. **Verify:** `sase tool run check` passes locally, CI is green on the pushed commit,
   and the docs site deploys.

## Phase: telegram-audio

Open `sase-telegram` with `sase repo open sase-telegram -r "<why>"` and read its
`AGENTS.md`. Use commit `7dcec6c` ("feat: route completion media attachments") as the
template; it touched the same four areas.

1. **`telegram_client.py`**
   - Add
     `send_audio(chat_id, audio, caption=None, title=None, performer=None, duration=None)`
     after `send_video`. Follow the same `@_with_retry` and `_get_bot()` pattern.
   - Give it a generous `write_timeout` for media, so a slow upload does not trigger a
     duplicate retry upload.
2. **`scripts/sase_tg_outbound.py`**
   - Add `_is_audio_file` (`.mp3`, `.m4a`) after `_is_video_file`.
   - Add an audio branch right after the video branch:
     - Read `title` (TIT2), `performer` (TPE1), and `duration` with `mutagen`, which
       becomes a new small dependency; any read failure falls back to `None`.
     - Send through `_send_media_with_document_fallback`.
     - Files over 50 MB skip the upload and send a short text note with the size and
       path.
3. **Tests**
   - Predicate tests.
   - `TestRunOutboundAttachments`: the MP3 goes through `send_audio` with the tag
     metadata and not through a document send; a send failure falls back to a document.
   - The oversize note.
   - `test_telegram_client` adds `bot.send_audio = AsyncMock()`.
   - `test_formatting` preserves an `.mp3` on workflow-complete notifications.
4. **Docs:** the README "Media attachments" bullet and `docs/outbound.md` "Attachments".
5. Run `sase tool run check`. Commit as
   `feat(outbound): send MP3/M4A attachments with sendAudio`.

## Phase: script

Implement `script/`, `normalize/`, `lexicon.py`, `data/guide.md`, `data/lexicon.yml`,
and the `script`, `lint`, and `guide` commands.

1. **Script model**
   - Parse and serialize narration script v1: frontmatter validation with precise
     line-numbered errors, then chapters and paragraphs.
   - Apply a safety-net cleaner that strips residual Markdown (emphasis, backticks, link
     syntax to anchor text, HTML) and reports what it touched.
2. **Deterministic normalizer** (markdown-it-py plus mdit-py-plugins: front matter,
   footnotes, tasklists, dollarmath):
   - **Title:** frontmatter `title`, else the first H1, else the humanized filename.
   - **Headings:** H2 becomes a chapter. H3–H6 become a spoken sentence followed by a
     paragraph break.
   - **Lists:** items become separate sentences in order, without saying "bullet".
   - **Links, images, and URLs:** links become anchor text. Images with useful alt text
     become "Figure: …". Bare URLs are dropped.
   - **Inline code:**
     - drop paths, `file:line` citations, SHAs, URLs, and SASE refs (`kind:YYYYMM/…`);
     - humanize identifiers (`snake_case` becomes "snake case", `CamelCase` is split);
     - say single-key keybindings as "the X key".
   - **Cleanup:** remove orphaned `()`/`[]` and dangling commas left behind by removals.
     Turn `§6` into "section 6".
   - **Dropped silently but recorded in the omissions report:** fences, Mermaid, math,
     HTML comments, footnote markers, and "Sources"/"References" sections.
   - **Tables:** tables with ≤ 6 rows and ≤ 4 columns are spoken row by row as
     "<first cell>:
     <header> <cell>, …". Larger tables become one sentence naming the table's columns and row
     count, plus an omission entry.
   - **Output:** a lint-clean `edition: verbatim`, `producer: deterministic` script.
3. **Golden tests** in `tests/fixtures/markdown/*.md`, with `*.narration.md` and
   `*.omissions.json` beside each:
   - a synthetic everything-fixture;
   - a trimmed excerpt of a real public research report that reproduces the residue the
     research quoted: empty `()`, letter keybindings, `§6`, and mid-argument code;
   - a CommonMark edge-case set.
   - `just golden` regenerates them.
   - A property test asserts that every fixture's output lints clean.
4. **Lexicon**
   - YAML of `term: spoken form`, matching case-sensitive whole words.
   - The packaged default is merged under the user file.
   - Seed entries: `TUI`, `xprompt`/`xprompts` (ex-prompt), `chezmoi` (shay-mwah), `uv`
     (you-vee), `PyPI` (pie-pee-eye), `mypy` (my-pie), `LUFS` (loofs). Leave `SASE` out;
     see Open decisions.
   - Expose `apply(text) -> text` and a `sha256` for the manifest.
   - Apply the lexicon to spoken text only, never to ID3 or feed chapter titles.
5. **Lint rules**, each with an id, severity, `line:col`, and fix hint:
   - **Structural errors:** frontmatter, no chapters, empty chapter, non-`##` headings,
     text before the first chapter.
   - **Residue errors:** list markers, tables, fences, backticks, link/image syntax,
     HTML, emphasis.
   - **Warnings:**
     - URLs, paths, `file:line`, hex SHAs ≥ 7 characters, refs, `§`;
     - the symbols → ≈ × ≥ ≤ ±;
     - chapters over 900 words, paragraphs over 180 words, sentences over 45 words;
     - edition budget overruns (above 115% of the budget) or underruns (below 50%).
   - **`--source REPORT` number fidelity:** warn on every multi-character, decimal, or
     percent number in the script that does not appear, normalized, in the source text.
   - Output is a rich grouped table, or `--json`. `render` reuses the structural/residue
     split: structural refuses, residue is cleaned and warned.
6. **Guide (`data/guide.md`, printed by `guide`)**
   - Who the listener is: walking or commuting, 1.25–1.5× speed, cannot glance back.
   - The contract and frontmatter, including `source_blob` via `git hash-object`, and
     `cover: <stem>_infographic.png` when one exists.
   - **Shape:** question and answer first, then signposted reasons, costs and caveats,
     and a recap of what to do. Use 4–8 chapters whose speakable titles echo the
     report's section names.
   - **Fidelity rules:**
     - no new claims;
     - keep every argument-carrying number exactly, in digits;
     - preserve stated uncertainty;
     - never soften or strengthen the recommendation;
     - say tables as comparisons: the winner, the runner-up, and the deciding numbers;
     - code becomes one sentence of purpose, or nothing;
     - never say paths, SHAs, line numbers, URLs, or refs.
   - **Listenability:** short sentences, expand acronyms on first use, write symbols as
     words, no "see section 6".
   - Edition budgets. Do not write the AI disclosure.
   - The final step is `sase-listen lint <script> --source <report>`, repeated until
     clean.
   - A short worked example.
   - `--edition brief` swaps in the brief budget and shape.
7. Update `docs/narration-scripts.md` and `docs/pronunciation.md`.

## Phase: audio

Implement `audio/` as a pure library with no engine or CLI knowledge.

- **Input:** `ChapterAudio(title, segments: list[PCM])` sequences plus `EpisodeMeta`.
- **Output:**
  `MasteredEpisode(path, duration_s, chapters[start_ms, end_ms], loudness_lufs, true_peak_db, bytes)`.

1. **`resolve_ffmpeg()`:** `$SASE_LISTEN_FFMPEG`, else `ffmpeg` on `PATH`, else
   `imageio_ffmpeg.get_ffmpeg_exe()`. Probe for `libmp3lame` and `loudnorm` once, cache
   the result, and report the source for `doctor`.
2. **PCM utilities (numpy):**
   - RMS over 20 ms windows with a −50 dBFS threshold.
   - Trim each chunk's leading and trailing silence to 80 ms.
   - Compress internal silences longer than 1.5 s to 0.7 s, and report any longer than 4
     s so gates can request re-synthesis.
   - Exact-length gap insertion: intro, chunk, and chapter gaps from config.
   - Chapter start offsets computed from sample counts after trimming.
3. **Mastering:**
   - Assemble a temporary WAV.
   - Pass 1: `loudnorm=I=-16:TP=-1.5:LRA=11:print_format=json`.
   - Pass 2: apply the measured values with `linear=true`, then
     `-ar 24000 -ac 1 -c:a libmp3lame -b:a 64k`, CBR, Xing header, `-map_metadata -1`.
   - Parse and return the pass-2 output loudness.
   - Write to a temp file and `os.replace` it into place.
4. **ID3 (mutagen, v2.3):**
   - Frames: TIT2, TPE1 (`author`), TALB ("Audio Editions"), TDRC, TCON "Podcast", COMM
     (description), TLEN, and `TXXX:SASE_LISTEN_EPISODE` / `TXXX:SASE_LISTEN_SOURCE`.
   - APIC front cover as JPEG.
   - CTOC (top-level, ordered) plus one CHAP per chapter with an embedded TIT2.
   - Round-trip tests read everything back.
5. **Cover art (Pillow; 1400×1400 JPEG, quality 88, progressive):**
   - **Generated title card:**
     - a two-tone gradient picked deterministically from the title hash out of a small
       curated palette;
     - the title in a bundled OFL font (Inter SemiBold + Regular with `OFL.txt`, under
       `data/fonts/`), auto-fitted to at most 5 lines;
     - a small-caps label (`SASE RESEARCH · AUDIO EDITION`, or `AUDIO EDITION` for
       non-research sources) and the date;
     - a waveform-bar motif along the bottom, generated from the title hash.
   - **Supplied images** (`cover:`, `--cover`, or an infographic) are letterboxed into
     the square over a blurred, darkened, scaled copy of themselves.
   - Snapshot-test the dimensions, determinism (same title gives the same bytes), and
     that text fits.
6. **Tests** use synthetic sine/noise PCM and the real bundled ffmpeg: integrated
   loudness lands within ±1 LU of −16, chapter offsets are exact, and the duration is
   the sum of segments plus gaps.
7. Update `docs/architecture.md` (audio section) and the format notes in
   `docs/reliability.md`.

## Phase: engines

Implement `engines/`, `cache.py`, and `pricing.py`, plus narrator resolution and secret
resolution on top of the scaffold's config.

1. **`gemini`:**
   - `google-genai` `Client(api_key=…)`, calling `models.generate_content` with
     `response_modalities=["AUDIO"]` and a `speech_config` prebuilt voice.
   - Style goes through `system_instruction`, or whatever channel the current
     speech-generation guide documents. Confirm it with the live call; never inline it
     into the text.
   - Parse the `inline_data` PCM and its rate from the mime type.
   - Map `errors.APIError` codes to the error taxonomy, and treat a missing or empty
     audio part as transient.
   - Limits: about 400 target words per chunk, under the 8,192-input /
     16,384-output-token caps.
2. **`openai` (OpenAI-compatible):**
   - `httpx` POST to `{base_url}/audio/speech` with
     `{model, input, voice, response_format: "pcm", instructions?, speed?}`; the
     response is 24 kHz s16le.
   - Limit: `max_chars` 3,500 by default, overridable per narrator for Kokoro-FastAPI.
   - Honor `Retry-After`.
3. **`tone`:** deterministic tone bursts at about 150 wpm, with a longer pause at
   sentence punctuation. It costs nothing and makes realistic durations for gates and
   e2e tests.
4. **`synthesize_with_retry`:**
   - Bounded exponential backoff with full jitter, honoring `retry_after` and
     `max_retries`.
   - Never retry permanent or credentials errors.
   - An injectable sleeper and clock for tests.
   - Per-request timeout from config.
5. **`ChunkCache`:** as specified in Shared contracts, with get, put, stats, and
   prune-to-size/age. It survives concurrent writers through atomic rename.
6. **`pricing`:**
   - A packaged `data/pricing.yml` of USD per audio minute with effective dates:
     - `gemini-3.8-flash-tts`: 0.0135, then 0.027 from 2027-01-01;
     - flash-lite: 0.009, then 0.018;
     - `gpt-4o-mini-tts-2025-12-15`: 0.015;
     - tone and Kokoro: 0.
   - `estimate(model, seconds, on=date)`. Every estimate is labeled "≈" in the UI.
7. **Narrator resolution:** built-ins merged with user `narrators`; `--voice` overrides
   the voice only; clear errors for unknown names.
8. **Tests:** fake-transport unit tests for both cloud adapters (with request-shape
   snapshots), retry classification, cache atomicity and LRU, and secrets never
   appearing in `repr` or logs.
   - One `live`-marked test synthesizes two sentences with Gemini.
   - **Run it once in this phase** with
     `SASE_LISTEN_GEMINI_API_KEY="$(pass show gemini_cli_api_key)"` to confirm the model
     id, response shape, sample rate, and style channel. Record the result in the phase
     notes. If the model id or API shape differs from this plan, adapt the adapter and
     record the deviation.
9. Update `docs/narrators.md` (engines, voices, the style prompt, adding a Kokoro
   profile) and the credentials part of `docs/configuration.md`.

## Phase: pipeline

Implement `pipeline.py`, `library.py`, `manifest.py`, and the `render` command with
plain human output and complete `--json`.

1. **Load the source:**
   - A narration script.
   - Plain Markdown: normalize deterministically and keep the omissions.
   - A ref: use `sase artifact read`.
   - Then lint: structural errors exit 6 unless `--force`; residue is cleaned and
     warned.
2. **Plan:**
   - Apply the lexicon to spoken text.
   - The intro chunk comes first. The first chunk of each chapter begins with the spoken
     heading followed by a paragraph break.
   - Pack paragraphs greedily up to the engine's `target_words` and `max_chars`,
     splitting an oversize paragraph at sentence boundaries.
   - The outro chunk comes last.
   - Compute cache keys, cached/uncached counts, and the estimated duration and cost.
   - `--dry-run` stops here and prints the plan; with `--json`, the plan object.
3. **Synthesize** uncached chunks with a thread pool sized to the engine concurrency,
   through `synthesize_with_retry`, and write each result to the cache immediately so
   the run can resume.
4. **Chunk gates** (wpm = words / duration):
   - **Hard:** a hard failure triggers re-synthesis with the cache bypassed, up to 2
     times, and then exit 5 with a per-chunk report. Hard failures are:
     - empty or undecodable PCM, or the wrong sample format;
     - wpm below 60 or above 300;
     - an internal silence longer than 4 s.
   - **Soft:** wpm outside 90–240, or outside 0.65–1.5× the episode median when there
     are at least 3 chunks, triggers one re-synthesis and then a warning, keeping the
     best attempt.
5. **Master:** through `audio/`, with cover resolution in order `--cover`, then
   frontmatter `cover`, then a sibling `<stem>_infographic.png` next to the source, then
   a generated card.
6. **Episode gates:**
   - The MP3 decodes (mutagen), and its duration is within ±(1 s + 0.5%) of the sum of
     segments plus gaps.
   - Chapter count, titles, and order match the script.
   - Loudness is within ±1 LU of the target.
   - Size above 45 MB is a warning (Telegram's limit is 50 MB).
7. **Commit:**
   - Write the manifest, `script.md`, `cover.jpg`, and `chapters.json`.
   - Stage under `library/.staging/`, then atomically swap into `library/<episode-id>/`.
   - An fcntl lock per episode id prevents concurrent clobbering.
   - With `-o PATH`, also copy the MP3 there atomically.
   - Then enforce the cache LRU.
   - Expose an event-callback protocol: plan ready, chunk started, chunk finished, chunk
     retried, stage changed, done. The cli phase renders progress from it without
     touching the orchestration.
8. **Tests:**
   - An e2e tone-engine render of the golden fixtures.
   - Resume after a simulated crash mid-synthesis: the second run synthesizes only the
     missing chunks.
   - A gate-triggered re-synthesis.
   - Lint refusal and `--force`.
   - Ref resolution with a fake `sase` on `PATH`.
   - `--json` schema snapshots.
   - Exit codes.
   - Extend publish.yml's `install-smoke` to render a packaged fixture with `-n tone`
     and assert the chapters.
   - Raise coverage `fail_under` to 90.
9. Update `docs/reliability.md` (cache and resume, gates, manifest, atomicity, exit
   codes) and `docs/architecture.md` (the pipeline diagram).

## Phase: cli

Make every command a pleasure to use. Use one theme in `ui.py`: one accent color and a
single glyph set (♪ ✓ ⚠ ✗ ≈). Rich handles `NO_COLOR` and non-TTY output, and non-TTY
output degrades to clean line logs.

1. **`render`:**
   - **Plan panel:** title, source, narrator (engine · model · voice), edition and
     producer, words, ≈ duration, chunks (cached/new), and ≈ cost.
   - **Chapter table:** #, title, words, ≈ min, chunks.
   - **Omissions summary** for deterministic scripts.
   - **Live progress:** one row per chapter with spinner → ✓, duration, and retries,
     plus an overall bar with ETA and per-stage status (synthesize → gates → master →
     tag → commit).
   - **Summary panel:** the MP3 path (clickable), duration, size, chapter count, LUFS,
     actual ≈ cost, cache hits, warnings, and "published to feed" when it applies.
   - Errors render as a panel with a cause and a concrete next step. For example,
     missing credentials shows the env names and `api_key_command` with a config
     snippet.
2. **`audition`:**
   - A packaged ~45 s sample passage with numbers, an acronym, SASE jargon, and a
     comparison.
   - Renders one clip per voice or narrator under `library/auditions/<ts>/`, tags them
     "Audition — <voice>", and prints a table with paths, durations, and ≈ cost.
3. **`ls`:** newest-first table of date, title, duration, narrator, edition, size, and
   published. `ls EPISODE` shows details and a chapter list.
4. **`doctor`:**
   - A checklist covering config validity, ffmpeg (path, source,
     `libmp3lame`/`loudnorm`), credentials for the default narrator (presence only, or a
     one-word synth with `--online`), writable dirs, free disk, and version.
   - The feed phase adds feed checks.
5. **`cache`:** stats and `prune`.
6. **`config`:** the effective config with origins, with secrets masked. `config init`
   writes a commented starter file and refuses to overwrite.
7. **Help:** examples in every command's epilog, plus a top-level description that tells
   the whole story in three lines.
8. **Captures:** `just screenshots` uses rich's `Console(record=True)` with the tone
   engine to write `docs/assets/render.svg`, `dry-run.svg`, `doctor.svg`, and `ls.svg`,
   and exports a generated cover as `docs/assets/cover-example.jpg`. Commit them.
9. **Tests:** snapshot tests of the rendered output at a fixed width for plan, summary,
   error, `ls`, and `doctor`.
10. Update `docs/cli.md` as a full reference, generated where practical from the
    argparse definitions so it cannot drift, plus `docs/getting-started.md`.

## Phase: feed

Implement `feed.py` and the `feed`, `publish`, and `unpublish` commands.

1. **Config** (`feed.*`):
   - `dir` (default `$XDG_DATA_HOME/sase-listen/feed`, the **only** directory ever
     served);
   - `base_url`, `token` or `token_command`;
   - `title` ("SASE Listen"), `description`, `author`, `language`;
   - `retention_days` (90), `max_episodes` (200);
   - `auto_publish` (default `false`);
   - `source_url_templates` (default
     `research: https://github.com/sase-org/sase--research/blob/master/{path}`), so each
     episode links to the written report.
2. **`feed init --base-url URL`:**
   - Generates a `secrets.token_urlsafe(24)` token and creates the dir.
   - Writes the `feed:` section to the config, or prints it with `--print` for
     chezmoi-managed configs.
   - Prints the subscribe URL, the exact Tailscale command
     (`tailscale funnel --bg --https=8443 --set-path=/<token> <feed-dir>`), and a
     terminal QR code (segno) for the phone.
3. **`publish EPISODE`** (id, MP3 path, or `--latest`):
   - Copies or hard-links the MP3, cover, and chapters JSON into `dir/episodes/`.
   - Regenerates `feed.xml` atomically and applies retention.
   - `unpublish` removes the episode and regenerates.
   - `render --publish` publishes, and `--no-publish` suppresses it. With
     `auto_publish: true`, `render` publishes `kind: research` episodes automatically.
4. **`feed.xml`:**
   - RSS 2.0 with the `itunes` and `podcast` namespaces.
   - Channel: `itunes:block yes`, `podcast:locked yes`, `itunes:explicit false`, and a
     generated 1400×1400 channel cover from the same art system.
   - Items, newest first:
     - title;
     - an HTML description with the chapter list and the source link;
     - a `guid` (`isPermaLink=false`) of `<episode-id>@<audio sha256[:8]>`, so
       re-renders appear as fresh audio;
     - `pubDate`;
     - an `enclosure` with the real length and `audio/mpeg`;
     - `itunes:duration`, `itunes:image`, `itunes:episodeType full`;
     - `podcast:chapters` (`application/json+chapters`).
5. **`feed` status:** URL (masked unless `--show-url`), episode count, size, last build,
   and retention; `--qr`. `doctor` gains feed checks: dir writable, `base_url` set,
   token resolvable, `feed.xml` parses.
6. **Tests:** golden `feed.xml` (parsed with `xml.etree` and compared semantically),
   retention, guid changes on re-render, unpublish, auto-publish only for research
   episodes, and atomicity.
7. **`docs/podcast-feed.md`:**
   - AntennaPod setup: subscribe by URL, auto-download on Wi-Fi, add to queue, chapters.
   - Tailscale Funnel serving and why `:8443`. The tailnet policy needs the `funnel`
     node attribute.
   - The security model: secret path, `itunes:block`, only what you publish is public,
     and rotating the token.
   - A static-server alternative.

## Phase: research-audio

Open `sase-research-artifacts` with `sase repo open sase-research-artifacts -r "<why>"`
and read its `AGENTS.md`. The plugin must not depend on `sase-listen`; the xprompt calls
the CLI at runtime.

1. **`xprompts/research_audio.md`** (`name: research/audio`):
   - Inputs: `edition` (word, default `full`) and `rewrite` (bool, default `false`).
   - **Find the report.**
     - When invoked with a `@research:` ref, read the report with `sase artifact read`.
     - When forked from a swarm lead, use the report you wrote. Prefer the published
       `<name>.md`, and fall back to `<name>__final.md`.
     - The research checkout is `$(sase repo path research --ensure)`.
   - **Choose the CLI.** Use `sase-listen` if `command -v sase-listen` succeeds,
     otherwise `uvx sase-listen`.
   - **Write or reuse the script.**
     - The script is `<stem>_narration.md` next to the report, with `__final` stripped
       from the stem, following the `research_image.md` stem rule.
     - If it exists and `rewrite` is false, reuse it.
     - Otherwise run `sase-listen guide --edition {{ edition }}` and write the script
       following it exactly, with `source`, `source_blob`, `date`, `kind: research`, and
       `cover` when `<stem>_infographic.png` exists.
     - Run `sase-listen lint <script> --source <report>` until it is clean.
   - **Render** with `sase tool run -- sase-listen render <script> --json`. If the
     render approaches the inline ceiling, hand it to `/sase_monitor`.
   - **Deliver** with
     `sase artifact create -p <audio_path> -l "Audio edition: <title>"`. The MP3 rides
     the completion notification to Telegram.
   - **Report** the duration, chapters, ≈ cost, and whether it was published to the
     feed.
   - On a render failure, report the error code and hint, and never switch narrators
     silently.
2. **`research_swarm.md`:**
   - New inputs: `audio` (bool, default false, "Narrate the published report as an audio
     edition…") and `audio_model` (word, default `@audio`).
   - New segment after the linker:

     ```text
     %if(should_run={{ audio }}) %id(audio, clan=research.{@1}) %m:{{ audio_model }}
     %wait:research.{@1}.final {% if run_linker %}%wait:research.{@1}.linker {% endif %}%q(...same as image...) #fork:research.{@1}.final #research/audio
     ```

     - It waits for the linker so it narrates the published report and can use the
       infographic as its cover.
     - It forks the lead for full context.

   - `audio` does **not** imply the linker. Update the description, the layout tree (add
     `<name>_narration.md` when audio is on), and the docs execution matrix.

3. **`default_config.yml`:** a `custom.audio` alias (bucket `researchers`, description
   "Research-swarm audio-edition narrator script writer"). Its model pool should favor
   strong prose writers already used in this plugin's defaults, for example
   `claude/opus@high | codex/gpt-6.1-sol@high`. Validate it against the config schema
   test.
4. **`provider.py`:** add `"!20*/**/*_narration.md"` to
   `_COMPANION_MARKDOWN_EXCLUDE_GLOBS`, so scripts never become `@research` reports or
   Highlights PDFs.
5. **Tests:**
   - Filters: the narration file is excluded from the inventory and the highlights hook.
   - xprompt loading: `research/audio` has typed inputs and mentions
     `sase-listen guide`, `lint --source`, `render --json`, and `sase artifact create`.
   - The swarm with `audio=true` plans an extra unit with both waits and the fork, with
     and without the linker.
   - `audio=false` adds nothing.
   - The default config validates.
   - The wheel contract includes the new xprompt.
   - Update `test_ci_install_contract.py` and the `publish.yml` published-minimum smoke
     if the swarm expansion assertions need it.
6. **Docs:** README xprompt bullets and a "requires `uv tool install sase-listen`" note,
   a `docs/xprompts.md` section and swarm input rows, and the alias in
   `docs/configuration.md`. Run `sase tool run check`, then commit as
   `feat(research-audio): …`.

## Phase: docs

Make the sase-listen docs excellent; the README is also the PyPI page.

1. **README:**
   - Hero: name, tagline, badges, and `docs/assets/cover-example.jpg`.
   - "What you get": 3 bullets.
   - Quickstart: `uv tool install sase-listen`, `sase-listen doctor`,
     `sase-listen render notes.md`, and `sase-listen render my_script.md`.
   - "Audio editions of SASE research": `#research/audio @research:…` from the TUI or as
     a Telegram message, and `#research_swarm audio=true`.
   - How it works: a Mermaid pipeline of select → script → synthesize → master →
     deliver.
   - The terminal SVG capture.
   - Narrators and voices (`audition`), configuration snippet, podcast feed, reliability
     highlights, cost table.
   - Background, Development, License.
2. **Docs site:** fill every nav page and remove the stubs.
   - `background.md` holds the design rationale, a condensed form of Design decisions
     above.
     - It links the originating research by permalink, the 202606 prior research, and
       the five researcher reports.
     - It links **this epic plan's** permalink in `sase-org/sase--plans`. Resolve it
       from the plan ref on the epic bead, for example with
       `sase artifact show <plan-ref>`, and use a commit permalink.
   - `sase-integration.md` covers `#research/audio`, the swarm stage, Telegram delivery,
     the feed, and why sase-listen is a standalone tool.
   - `troubleshooting.md` covers credentials, quota and 429, ffmpeg, gate failures,
     Telegram size limits, feed not updating, and mispronunciations (lexicon).
3. Add `CONTRIBUTING.md` (setup, recipes, golden updates, live tests, commit and release
   flow). Finalize `AGENTS.md`.
4. **Verify:** `mkdocs build --strict` (no broken links), `codespell` and `twine check`
   clean, and every CLI example in the docs runs against the tone engine (a
   docs-examples test).

## Phase: release

1. Confirm that sase-listen `master` CI and docs deploys are green. Build the wheel
   locally and check that it contains the guide, lexicon, fonts, sample passage,
   pricing, and the `OFL.txt` license.
2. Release PR merges are outward-facing, so propose each one with `/sase_gate`, giving
   the PR URL and changelog excerpt:
   - the open release-please PR `chore(master): release 0.1.0` for sase-listen;
   - the pending release PRs of sase-telegram and sase-research-artifacts, which now
     include the audio work. Note any unrelated changes those PRs also carry.
3. After merge, confirm that the `publish.yml` runs succeeded and that
   `pip index versions sase-listen` (and the plugins' versions) show the new releases.
   Note in the phase notes that a `SASE_RELEASE_TOKEN` secret would let future release
   PRs trigger CI.

## Phase: rollout

This phase runs on apollo.

1. **Install.**
   - `uv tool install sase-listen`.
   - Upgrade sase-telegram and sase-research-artifacts in sase's tool env with the
     `sase plugin` update command (check `sase plugin --help`), and restart the Telegram
     outbound service proc if needed.
   - `sase-listen doctor`.
2. **Configure through chezmoi.**
   - Open it with `sase repo open chezmoi -r "<why>"` and follow its `AGENTS.md`.
   - Add `~/.config/sase-listen/config.yml` with
     `engines.gemini.api_key_command: pass show gemini_cli_api_key` and
     `narrator: gemini`.
   - Apply only that target, then run `doctor --online`.
3. **Auditions.**
   - `sase-listen audition --voices Charon,Kore,Iapetus,Sadaltager`.
   - Register each clip with `sase artifact create`, so they arrive on Telegram.
   - The final message tells the user how to switch voices (one config line).
4. **First real edition, dogfooding the design on its own research.**
   - Follow the `#research/audio` flow by hand for
     `research:202610/commute_audio_from_markdown/commute_audio_from_markdown.md`:
     - write and lint `commute_audio_from_markdown_narration.md` in the research
       sidecar;
     - render it;
     - register the MP3.
   - Measure wall time, actual cost against the ≈ $0.16 estimate, size, loudness, chunk
     consistency, and silences.
   - Fix small defects in place. File larger ones as follow-ups in the phase notes.
5. **Docs sample.**
   - Add `docs/field-notes.md` to sase-listen with those measurements.
   - Commit a short (≤ 45 s, ≤ 400 KB) Charon audition clip as `docs/assets/sample.mp3`.
   - Embed it with an `<audio>` player on the docs home page.
6. **Feed.**
   - `sase-listen feed init --base-url https://<apollo MagicDNS name>:8443` (from
     `tailscale status --json`), recorded through chezmoi with `--print`.
   - Publish the first edition.
   - Set `auto_publish: true`.
7. **Last, because a gate ends the turn:** propose the public exposure through
   `/sase_gate`, or `/sase_sudo` if root is required.
   - Command: `tailscale funnel --bg --https=8443 --set-path=/<token> <feed-dir>`.
   - The gate notes explain:
     - only the feed dir is exposed, under an unguessable path;
     - 443 stays tailnet-only;
     - the tailnet policy needs the `funnel` node attribute;
     - how to subscribe in AntennaPod.

## Verification

- **Each sase-listen phase:**
  - `sase tool run check` passes: ruff, format, mypy strict, codespell, and pytest with
    coverage ≥ 90% from pipeline onward.
  - New behavior has tests.
  - The phase's docs page is updated.
- **CI:**
  - Green on ubuntu (3.12–3.14) and macOS (3.12).
  - The publish `install-smoke` renders a real chaptered MP3 from the built wheel with
    the tone engine.
  - Docs build strictly.
- **End to end:**
  - In rollout, a real Gemini audio edition of the commute-audio report arrives on
    Telegram as a playable track with title, performer, and duration.
  - It appears in the feed with chapters.
  - `sase-listen ls` and the manifest record narrator, cost, and gates.
- **Plugins:**
  - sase-telegram and sase-research-artifacts checks pass.
  - `#research_swarm audio=true` expands to the new stage.

## Open decisions for the reviewer

- **Default voice:** `Charon` until the rollout auditions; switching is one config line.
- **How "SASE" should be pronounced:** the default lexicon omits it until you say
  ("sassy", "S-A-S-E", …).
- **Public feed:** the private feed needs Tailscale Funnel on `:8443`, which publicly
  exposes only the feed directory under a secret path. Rollout proposes it through a
  gate, so nothing goes public without your confirmation.
