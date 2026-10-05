---
tier: tale
title: Fix athena research audio Gemini key shadowing and finish research.3n.audio
goal: research.*.audio agents on athena render with the working pass-managed Gemini
  key regardless of stale ambient GEMINI_API_KEY values, sase-listen classifies and
  explains credential failures, and the research.3n.audio episode is rendered, registered,
  carded in the published report, and verified in the AntennaPod-subscribed apollo
  feed.
size: medium
proposed_by: bbugyi200.athena.0wp
status: done
---

# Plan: Fix athena's stale Gemini key shadowing for research audio, then finish research.3n.audio

## Problem

`research.3n.audio` (codex, athena, 2026-10-05) wrote and linted
`202610/sase_md_instruction_delivery/sase_md_instruction_delivery_narration.md` in the
research repo. Its render then failed:

```
sase-listen render … --json  ->  exit 4
"Synthesis failed: Error code: 400 - … 'reason': 'API_KEY_INVALID' … generativelanguage.googleapis.com"
hint: "Re-run to resume from the chunk cache, or try another narrator."
```

No MP3 was produced. The linker (`research.3n.linker`) published
`sase_md_instruction_delivery.md` without a listen card.

### Root cause (verified)

- `sase-listen` resolves the Gemini key from **env vars first**
  (`SASE_LISTEN_GEMINI_API_KEY`, `GEMINI_API_KEY`, `GOOGLE_API_KEY`), then
  `engines.gemini.api_key_command` (`src/sase_listen/engines/secrets.py`, `config.py`
  defaults).
- The chezmoi-managed `~/.config/sase-listen/config.yml` only sets
  `api_key_command: pass show gemini_cli_api_key`. That pass entry holds the working
  key: a direct `GET /v1beta/models` call returned HTTP 200.
- athena alone also exports a stale 39-character `AIza…` `GEMINI_API_KEY`, and the same
  call with it returns HTTP 400 `API_KEY_INVALID`. It comes from `~/.profile.local`,
  which is athena-local and not managed by chezmoi; that line is the file's only
  content. It spreads from there:
  - **SASE service host:** it was captured into `~/.sase/service/env` on 2026-09-23. The
    host loads that file with `override_existing=True`, so **every agent** it spawns
    inherits the stale key.
  - **Other live processes:** the tmux global env, the running TUI, and the service-host
    processes all carry it too.
- apollo has the same pass key and no `GEMINI_API_KEY` anywhere (shell, service env), so
  its renders use the pass key. That's why the user sees this work on apollo.
- The sase-listen rollout already hit this exact problem. `docs/field-notes.md` §
  Credentials in the sase-listen repo records it, but only as a manual
  `env -u GEMINI_API_KEY …` workaround. The `#research/audio` prompt never applies that
  workaround, so every athena audio agent fails.
- **Secondary bug:** the TTS path misclassified the error, producing exit 4 and a
  misleading "re-run / try another narrator" hint instead of exit 3 config/credentials.
  - Gemini TTS uses `client.interactions.create`. Its errors are google-genai's
    _interactions compat errors_: `google.genai._gaos.lib.compat_errors.APIStatusError`
    subclasses such as `BadRequestError`, carrying an integer `status_code`.
  - These are **not** subclasses of `google.genai.errors.APIError`, so
    `engines/gemini.py`'s `except errors.APIError` never catches them, and they fall
    through to the generic `except Exception`.
  - As a result:
    - `API_KEY_INVALID` (and 401/403) is not reported as a credentials error.
    - 429 and 5xx from the TTS path are **not retried** (they never become
      `TransientEngineError`).
  - The existing engine tests only simulate `errors.APIError(code, …)`, so they don't
    catch this.

The fix doesn't depend on any running process's environment. The config pins Gemini key
resolution to the tool-specific env var, so the pass key wins regardless of ambient
generic keys. This takes effect for the next `sase-listen` invocation without restarting
the service host.

## Ground rules for the implementer

- Open every non-workspace repo with `/sase_repo` (`sase repo open chezmoi`,
  `sase repo open sase-listen`, `sase repo open research`) and use only the printed
  paths. Read each repo's `AGENTS.md` when `sase repo open` names one.
- **Never print, log, commit, or paste any secret.** This covers the API keys, the pass
  output, and the feed token (including the token path segment of the feed URL). When
  checking keys, print only presence, length, or prefix class (`AIza` vs `AQ.`).
- Do **not** restart, stop, or re-init the SASE service host. Agents are running on it.
- Do **not** work around the bug with `env -u GEMINI_API_KEY`. The verification render
  must run in this agent's normal inherited environment, which still contains the stale
  key, to prove the fix.

## Step 1 — Durable fix: pin sase-listen's Gemini key sources (chezmoi)

1. In the chezmoi checkout, edit `home/dot_config/sase-listen/config.yml` so the
   `engines.gemini` block becomes:

   ```yaml
   engines:
     gemini:
       # Only the tool-specific env var may override the pass-managed key. Generic
       # GEMINI_API_KEY / GOOGLE_API_KEY exported for other tools must never shadow
       # it: a stale one on athena broke every research audio render with
       # HTTP 400 API_KEY_INVALID.
       api_key_env: [SASE_LISTEN_GEMINI_API_KEY]
       api_key_command: pass show gemini_cli_api_key
   ```

   Leave every other key unchanged.

2. Apply it to athena now. This is the same targeted apply the sase-listen rollout used.
   ```bash
   chezmoi apply --source <chezmoi checkout> ~/.config/sase-listen/config.yml
   ```
   Then confirm two things:
   - `diff` shows the target equals the source file.
   - `sase-listen config` lists `api_key_env: [SASE_LISTEN_GEMINI_API_KEY]` for
     `engines.gemini`.
3. Prove the resolution offline with the installed tool's Python. That is the
   interpreter on the shebang line of `~/.local/bin/sase-listen`.
   - Load the config and call
     `sase_listen.engines.secrets.resolve_api_key(engine="gemini", env_names=…, api_key_command=…)`.
   - Assert that the resolved key is the `AQ.`-prefixed pass key, i.e. **not** equal to
     the ambient `GEMINI_API_KEY`.
   - Print only lengths, prefixes, and the equality boolean.
   - Before this step, confirm that `GEMINI_API_KEY` is still present in this agent's
     env. Check presence and length only; that presence is what makes the test
     meaningful.

## Step 2 — Remove the stale key from athena's environment sources

The stale key is rejected by Google, so nothing can depend on it. nvim codecompanion's
Gemini adapter reads `GEMINI_API_KEY` and is already broken by it. Removing it brings
athena in line with apollo:

1. `~/.profile.local`: delete the `export GEMINI_API_KEY=…` line. Check the file first;
   it should be the only line. Keep the (now empty) file, since `~/.profile` sources it
   when it exists.
2. `~/.sase/service/env`: remove only the `GEMINI_API_KEY` entry. Keep every other line
   byte-for-byte.
   - The file uses the strict `NAME=<json string>` format from `sase.service.env`.
   - Rewrite it atomically and keep mode `0600`.
   - Then confirm the remaining keys are exactly `PATH`, `SASE_TMPDIR`, `SSH_AGENT_PID`,
     `SSH_AUTH_SOCK`. Print key names only.
   - The running host keeps the old value in memory until its next restart. Step 1
     already makes that harmless.
3. `tmux set-environment -g -u GEMINI_API_KEY`, so new tmux panes stop inheriting it.

## Step 3 — Finish research.3n.audio's work

Use your own research checkout (`sase repo open research`).

- **Report:**
  `202610/sase_md_instruction_delivery/sase_md_instruction_delivery__final.md`
- **Script:**
  `202610/sase_md_instruction_delivery/sase_md_instruction_delivery_narration.md`
  - Committed as `1abe649`; `edition: brief`, 3 chapters.
  - Its `cover` already points at `sase_md_instruction_delivery_infographic.png`.

1. Re-run `sase-listen lint <script> --source <report>` and make sure it is clean. Reuse
   the script as-is; don't rewrite it.
2. Render it in the normal, unmodified agent environment:

   ```bash
   sase tool run -- sase-listen render <script> --json
   ```

   - A brief edition should finish inline. If it approaches the inline ceiling, hand it
     to `/sase_monitor` and finish the remaining steps in the follow-up turn.
   - `feed.auto_publish: true` publishes it to the apollo feed over SSH.
   - Capture these fields from the `--json` object:
     - `episode_id`, `title`, `edition`, `duration_s`
     - `chapters` (chapter count = `len(chapters)`)
     - `audio_path`, `published`, cost
   - If the render fails, stop and report the exact code, message, and hint. Do not
     switch narrators.

3. Register the MP3:
   ```bash
   sase artifact create -p <audio_path> -k file -l "audio:<episode_id>"
   ```
4. Add the listen card the linker would have written to the published report,
   `202610/sase_md_instruction_delivery/sase_md_instruction_delivery.md`. Follow the
   listen-card rules in sase-research-artifacts'
   `src/sase_research_artifacts/xprompts/research_swarm.md` (linker step 3):
   - **Frontmatter:** the report currently has none, so create a block. It holds only
     this `audio:` mapping, with numbers copied verbatim from the render JSON:
     ```yaml
     audio:
       edition: brief
       duration_s: <duration_s>
       chapter_count: <len(chapters)>
       episode_id: <episode_id>
     ```
   - **Placement:** insert the card once, directly below the `> **Research query:**`
     blockquote and above the infographic image. Keep the blank lines:

     ```markdown
     <div class="listen">

     ♫ **Brief audio edition** · <max(1, round(duration_s/60))> min · 3 chapters ·
     [Narration script](sase_md_instruction_delivery_narration.md)

     </div>
     ```

   - **Public repo:** don't add any MP3 path, library path, feed URL, token, or artifact
     id.

## Step 4 — Verify the episode is in the subscribed feed

AntennaPod subscribes to the URL that `sase-listen feed` reports (apollo, tailnet
`:8443/<token>/feed.xml`).

1. `sase-listen feed --json`: confirm that `episode_ids` contains the new `episode_id`
   and that `last_build` is newer than the render start.
2. Fetch the real subscribed feed without echoing the URL:
   - Run `URL=$(sase-listen feed --json --show-url | jq -r .url)`, then
     `curl -fsS "$URL"`.
   - Confirm the XML has an `<item>` for the new episode: its title is
     `SASE Instructions Delivered Once`, and its GUID or enclosure references the
     `episode_id`.
3. Request the enclosure URL from that item with `curl -sSI`. Confirm all three:
   - HTTP 200
   - an `audio/mpeg` content type
   - `Content-Length` equal to the local MP3's size

   Redact the token in anything you print.

## Step 5 — Harden sase-listen so this failure is classified and self-explaining

These changes go in the sase-listen repo, following its `AGENTS.md`. sase-listen never
imports `sase`.

1. **Classify interactions-API errors** in `src/sase_listen/engines/gemini.py`.
   - In `synthesize`, also catch google-genai's interactions compat errors. Duck-type on
     an integer `status_code` from a `google.genai` exception class, rather than
     importing the private `_gaos` module.
   - Map them with the same table as `errors.APIError`; teach `_map_api_error` to read
     `code` or `status_code`:
     - 401/403 → `CredentialsError`
     - 429 → `TransientEngineError`, honoring `retry-after`
     - ≥500 → `TransientEngineError`
     - other → `PermanentEngineError`
   - Connection and timeout compat errors (`status_code` is `None`) →
     `TransientEngineError`.
   - Treat HTTP 400 whose detail contains `API_KEY_INVALID` or "API key not valid" as
     `CredentialsError`, for both error families. Share one helper with
     `writer/gemini.py`'s existing `_is_invalid_api_key` rather than duplicating it.
2. **Name the credential source** without revealing the value.
   - Add a variant of `resolve_api_key` that also returns the source: `env <NAME>` or
     `engines.<engine>.api_key_command`. Keep `resolve_api_key` working for existing
     callers.
   - When synthesis fails with `CredentialsError`, the exit-3 `SaseListenError` hint
     names the source.
   - If an env var won while an `api_key_command` was configured, the hint says so and
     suggests unsetting the variable or pinning `engines.<engine>.api_key_env`.
3. **`doctor` credentials detail.** Report which source would supply the key (env-var
   presence only; never run the command), e.g.
   `narrator 'gemini': env GEMINI_API_KEY (overrides engines.gemini.api_key_command; presence only)`.
   A shadowing env var keeps `ok: true`, but the detail must say it.
4. **Tests** in `tests/test_engines.py`, plus doctor/pipeline tests as appropriate:
   - A fake compat-style exception (integer `status_code`, body text) covering:
     - 400 + `API_KEY_INVALID` → `CredentialsError`
     - plain 400 → `PermanentEngineError`
     - 401, 403 → `CredentialsError`
     - 429, 500, 503 → `TransientEngineError`
     - connection error (`status_code=None`) → `TransientEngineError`
   - A 429 from the interactions path is retried by `synthesize_with_retry`.
   - A credential failure surfaces as exit 3, with a hint that names the env var and
     never contains the secret.
   - Doctor reports the shadowing source.
5. **Docs:**
   - `docs/configuration.md` § Credentials: document `api_key_env` and the "pin to
     `SASE_LISTEN_GEMINI_API_KEY`" recipe for machines whose shell exports unrelated
     generic keys.
   - `docs/troubleshooting.md`: add an `API_KEY_INVALID` / exit-3 entry.
   - `docs/field-notes.md` § Credentials: note that the `env -u` workaround is
     superseded by the pinned chezmoi config.
   - `docs/changelog.md`: add an entry.
6. Run `sase tool run check` in the sase-listen checkout until it passes.

Don't reinstall athena's `sase-listen` during this turn. The sase-listen commit lands
after the turn, and Step 1 alone prevents the failure.

## Repositories this turn will dirty

| Repo          | What changes                                                            |
| ------------- | ----------------------------------------------------------------------- |
| `chezmoi`     | `home/dot_config/sase-listen/config.yml`                                |
| `research`    | listen card + `audio:` frontmatter in `sase_md_instruction_delivery.md` |
| `sase-listen` | engine error classification, credential source, doctor, tests, docs     |

Declare a `commit` for each repo in `/sase_final`. Untracked machine-local edits aren't
part of any repo:

- `~/.profile.local`
- `~/.sase/service/env`
- the tmux env
- `~/.config/sase-listen/config.yml` (applied from chezmoi)

## Final report must include

- **Root cause:** one paragraph. Don't include key values.
- **Episode:** `episode_id`, title, duration, chapter count, approximate cost, and the
  artifact ref.
- **Feed evidence:** item present, enclosure HTTP 200 / `audio/mpeg` / size match, and
  `last_build`. Say that AntennaPod shows it on its next feed refresh.
- **Follow-ups for the user:**
  - Run `chezmoi update -a --force` on athena, apollo, and the Mac once the chezmoi
    commit lands.
  - Reinstall `sase-listen` from git on athena and apollo
    (`uv tool install --force --reinstall git+https://github.com/sase-org/sase-listen`)
    once the sase-listen commit lands. athena's current install points at an ephemeral
    workspace checkout.
  - The running service host keeps the stale value in memory until its next natural
    restart. This is harmless now.

## Acceptance criteria

- With the stale `GEMINI_API_KEY` still in the agent's inherited env, `sase-listen`
  resolves the pass key, and the research.3n.audio script renders successfully (exit 0,
  `published: true`).
- The new episode is in apollo's `feed.xml` (the AntennaPod-subscribed URL), and its
  enclosure downloads (HTTP 200, `audio/mpeg`, correct size).
- An `audio:<episode_id>` file artifact is registered, and the published report has the
  listen card plus the `audio:` frontmatter.
- None of `~/.profile.local`, `~/.sase/service/env`, or the tmux global env define
  `GEMINI_API_KEY`.
- `sase tool run check` passes in sase-listen with the new classification, source, and
  doctor tests.
- No secret appears in any commit, artifact, log line, or the final response.
