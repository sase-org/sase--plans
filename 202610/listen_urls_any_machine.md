---
tier: epic
title: 'sase-listen: URL-to-podcast editions, published from any machine'
goal: 'From athena, apollo, or the Mac, `sase-listen render <URL> --edition brief|full`
  fetches a web article, writes a fidelity-checked narration script, renders it, and
  auto-publishes the episode to the single AntennaPod feed served from apollo. This
  is proven by a full edition of OpenAI''s harness-engineering post, rendered on athena,
  appearing in the live served feed.

  '
phases:
- id: feed-host
  title: Publish to one feed host from any machine
  depends_on: []
  size: medium
  description: 'feed-host: add feed.host/feed.host_ssh config, an SSH transport that
    streams a rendered episode to the host''s new `feed receive` endpoint (validated
    import, then a locked publish), proxy publish/unpublish/feed/doctor in remote
    mode, keep a retry outbox, and document the model.'
- id: url-acquire
  title: Fetch and extract web articles as render sources
  depends_on: []
  size: medium
  description: 'url-acquire: fetch pages with curl_cffi browser impersonation, extract
    with Trafilatura plus HTML outline repair, keep a per-source store, add the `article`
    kind, and accept http(s) URLs in `script`/`render` for the deterministic verbatim
    edition.'
- id: url-editions
  title: Brief and full article editions with a script writer
  depends_on:
  - feed-host
  - url-acquire
  size: medium
  description: 'url-editions: add a Gemini script writer, driven by the packaged guide
    plus article rules, with a lint-and-repair loop and cached scripts. Make brief
    the URL default and label edition coverage plus the original-article link in titles
    and feed items.'
- id: rollout-proof
  title: Roll out to every machine and publish the harness-engineering full edition
  depends_on:
  - feed-host
  - url-acquire
  - url-editions
  size: medium
  description: 'rollout-proof: install the new sase-listen on apollo, then athena
    (and the Mac if reachable), switch the chezmoi config to `feed.host: apollo`,
    render the full edition of the OpenAI harness-engineering post on athena with
    auto-publish, verify it in the served feed, and write field notes.'
proposed_by: bbugyi200.athena.0wl
create_time: 2026-10-04 19:02:06
status: done
bead_id: sase-1g7
---

- **PROMPT:** [prompts/202610/listen_urls_any_machine.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/listen_urls_any_machine.md)
- **BEAD:** [sase-1g7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1g7/README.md)

# Plan: sase-listen URL-to-podcast editions, published from any machine

## Context

### What the user asked for

1. `sase-listen` converts a URL into a podcast episode, with `brief` and `full`
   editions.
2. Proof: produce a `full` edition of `https://openai.com/index/harness-engineering/`
   and have it **auto-publish** to the AntennaPod feed.
3. Today the user can only produce published audio on apollo. Make it work from athena,
   apollo, and the Mac, choosing the most reliable approach.

### Research this plan builds on

- `research:202610/article_paper_audio_workflow/article_paper_audio_workflow.md` (read
  with `sase artifact read`). Its recommendations are adopted here:
  - Fetch HTML first, using Trafilatura.
  - Keep a reviewable intermediate (stored source plus script), with source identity
    (canonical URL, fetch hash, extractor version, writer model and prompt version).
  - On a retry, reuse the same script so cached TTS chunks hit.
  - Label coverage, because `full` means an adaptation within a 2,400-word budget, not
    the complete text.
  - Carry the original source URL and coverage label through the manifest into episode
    notes.
  - Fail explicitly on blocked pages, and offer a saved-HTML fallback.
  - Prefer tailnet-only Tailscale Serve.

  The research also says an optional standalone HTML importer inside `sase-listen` is
  "not an architectural violation". Its warning that the numeric lint guard only checks
  that a number occurs somewhere in the source still holds. So the writer adds explicit
  attribution and fidelity rules on top of lint.

- `research:202610/sase_listen_antennapod_setup.md` covers the apollo feed setup. It
  documents the current limitation: "Launch from apollo. The feed directory is local to
  apollo, so an audio agent dispatched to athena would publish to athena's feed, which
  nothing serves."

### Live findings at planning time (2026-10-04)

- **apollo** runs sase-listen 0.1.0, installed with uv from
  `git+https://github.com/sase-org/sase-listen`.
  - `tailscale serve` (tailnet only) serves the feed dir on `:8443` at the secret token
    path. The feed has 9 episodes.
  - ffmpeg comes from imageio-ffmpeg.
  - **Non-interactive SSH does not have `~/.local/bin` on PATH.**
    `ssh apollo sase-listen` fails with "command not found", while
    `~/.local/bin/sase-listen` and `~/.local/bin/uv` work.
- **athena** runs an _editable_ uv-tool install of a stale checkout:
  `~/projects/github/sase-org/sase-listen` at `d3cb303`, which reports version `0.0.0`
  and is behind master `3037d60`.
  - Its config is identical to apollo's. Its local feed dir exists, is empty, and
    nothing serves it, so anything auto-published on athena never reaches the phone.
  - `pass` holds both `gemini_cli_api_key` and `sase_listen_feed_token`.
- **Mac**: SSH alias `mac`, user `bbugyi`, chezmoi hostname `Kellys-MacBook-Pro`. It was
  offline while planning, so its sase-listen install and `pass` state are unknown.
- **chezmoi** `home/dot_config/sase-listen/config.yml` is the same on every machine:
  - `narrator: gemini`, with the key from `pass show gemini_cli_api_key`
  - `feed.dir: /home/bryan/.local/share/sase-listen/feed` (hard-coded and wrong on
    macOS)
  - `base_url: https://apollo.tail297af1.ts.net:8443`
  - `token_command: pass show sase_listen_feed_token`
  - `auto_publish: true`

  The config loader **rejects unknown keys**, so new keys break older installs. Upgrade
  order matters.

- **SSH hosts**: chezmoi's `~/.ssh/tailnet.conf` defines `apollo` (tailnet) and
  `apollo-do` (public-IP fallback) on every machine.
- **Fetching openai.com**:
  - curl or httpx get **403 `cf-mitigated: challenge`** from openai.com with any
    User-Agent.
  - `curl_cffi` with `impersonate="chrome"` gets **200** (about 573 KB).
- **Extraction**:
  - Trafilatura 2.3.0 extracts about 2,835 words with good metadata: title "Harness
    engineering: leveraging Codex in an agent-first world", date 2026-02-11, sitename
    OpenAI, description "By Ryan Lopopolo, Member of the Technical Staff", author
    `None`.
  - **Trafilatura drops all 11 section `<h2>` headings**, even with
    `favor_recall`/`include_formatting`. Their markup is
    `<h2 class="text-h3 …"><span>We started with an empty git repository</span></h2>`,
    and a hidden TOC `<nav>` repeats them in `<button><span>`. readability-lxml drops
    them too.
- **Gemini text models** visible to the key include `gemini-pro-latest`,
  `gemini-3.1-pro-preview`, and `gemini-3.8-flash`.
- **URL vs artifact ref**: `https://…` matches the artifact-ref regex `_REF_RE` in
  `pipeline.py`, so it would currently be sent to `sase artifact read`.
- **Lint and frontmatter**: lint ignores unknown frontmatter keys, and a URL in the
  frontmatter `source:` raises no URL warning. The URL rule applies to the body only.
- **Current publish behavior**:
  - `feed.publish_episode` copies into the feed dir and rebuilds `feed.xml` with no
    cross-process lock.
  - `render` auto-publishes only `kind: research`.
  - Publish output prints the **unmasked** token in `item_url`.
- **Overlapping epic**: in-flight epic `sase-1g6` (research listen card) edits
  sase-listen `docs/sase-integration.md`. It expects the apollo library to hold research
  episodes, because bob discovers MP3s there. This plan keeps that true: the host
  imports every received episode into its own library. If that epic lands first, rebase
  cleanly.

## Design decisions

### Render anywhere, publish on one feed host over SSH

**Chosen design:**

- Every machine renders locally: fetch, write the script, synthesize, master.
- When the config names a feed host that is not this machine, publishing streams the
  finished library episode as a tar over SSH to `sase-listen feed receive` on the host.
- The host validates the tar, imports it into its own library, and publishes it under a
  feed lock, using the same code as a local publish.
- The phone's subscription, the token, and the `tailscale serve` setup do not change.

Why this is the most reliable option:

- It reuses what already works: tailnet SSH between all three machines, the apollo
  public-IP fallback `apollo-do`, the served feed dir, and AntennaPod's existing
  subscription.
- No new cloud account, daemon, or credentials. The token never leaves apollo.
- Only one writer ever touches `feed.xml`, so there are no merge races.
- The host stays the system of record (its library mirrors every published episode).
- Failures are explicit and retryable through a local outbox.

Rejected alternatives:

- **Object storage (S3/R2/Spaces) as the feed**: new cloud credentials on every machine,
  public-internet exposure behind a bearer URL, phone re-subscription, and `feed.xml`
  concurrency control.
- **Upload daemon on apollo**: a new long-running service to maintain, when SSH already
  provides authentication and transport.
- **Shared filesystem or Syncthing of the feed dir**: `feed.xml` conflicts and
  partial-sync states.
- **Always render on apollo**: apollo is a small droplet with an OOM history. Local
  files and `research:` refs would have to be shipped there, and a laptop that sleeps
  would kill a remote session. It stays available as a documented manual fallback for
  URLs, which are location-independent:
  `ssh apollo '~/.local/bin/sase-listen render <URL> …'`.
- **One feed per machine**: the phone would need several subscriptions, and athena or
  the Mac are not always reachable.

### URL editions live inside sase-listen, with an API script writer

- The user asked for this in sase-listen, and the research endorses a standalone HTML
  importer there.
- Acquisition (`web/`) and script writing (`writer/`) are separate modules ahead of the
  unchanged renderer.
- The writer returns only the script body. sase-listen builds the frontmatter itself,
  lints against the extracted source, and runs bounded repair rounds.
- Scripts are stored next to the source for review and reuse, and re-renders are
  idempotent.
- The no-sase-import rule still holds.
- PDFs, arXiv specifics, logins, batches, and a SASE `#listen` macro are out of scope.

## Shared rules for every code phase

- Work in the linked `sase-listen` repo: open it with `sase repo open sase-listen` and
  read its `AGENTS.md`.
- Verify with `sase tool run check`. `just check` is guarded and refuses a raw agent
  run. Keep `mypy --strict`, ruff, codespell, and the 90% coverage floor green.
- No `sase` imports.
- Every new CLI option gets clear `--help` text. Where it does not collide, also give it
  a short alias (`-e/--edition`). List options alphabetically in help.
- Never print, log, commit, or return the feed token or API keys. Mask the token as
  `****`.
- Never commit third-party article text as a test fixture. Build synthetic HTML that
  imitates the structure instead.

## Phase feed-host: Publish to one feed host from any machine

### Config (`src/sase_listen/config.py`)

New `FeedConfig` fields. Add them to `_FEED_KEYS`, `masked_snapshot`, and the docs.

- `host: str = ""`: the short hostname of the machine that owns and serves the feed dir.
  Empty means this machine, so existing configs behave exactly as before.
- `host_ssh: list[str] = []`: SSH destinations for reaching the host, tried in order.
  - Empty means `[host]`.
  - YAML may give a string or a list.

Env override: `SASE_LISTEN_FEED_HOST` overrides `feed.host`. An empty value means "this
machine".

### New module `src/sase_listen/feedhost.py`

- **`local_hostname()`**: `socket.gethostname()`, cut at the first `.`, lowercased.
- **`feed_role(cfg)`**: returns `"local"` or `"remote"`.
  - It is local when `host` is empty or its short, lowercased form equals
    `local_hostname()`.
  - Example: apollo is local for `host: apollo`; athena and the Mac are remote.
- **Remote-call guard**: remote invocations set `SASE_LISTEN_REMOTE_CALL=1`.
  - A process with that variable never forwards again.
  - If it would be remote, it refuses with exit 3: "this machine (<h>) is not the feed
    host (<host>); fix feed.host".
  - This prevents recursion and silent misrouting, for example a destination that points
    at athena.
- **`ssh_argv(dest, remote_args)`**: builds
  `[ssh, -o BatchMode=yes, -o ConnectTimeout=10, -o ServerAliveInterval=15, -o ServerAliveCountMax=4, dest, CMD]`.
  - `ssh` comes from env `SASE_LISTEN_SSH`, default `ssh`. This is the test seam.
  - `CMD` is
    `export PATH="$HOME/.local/bin:$PATH" SASE_LISTEN_REMOTE_CALL=1; exec sase-listen <shlex.join(remote_args)>`.
    It is POSIX, so it works in the remote zsh and bash, and it fixes apollo's
    non-interactive PATH.
- **`run_remote(cfg, args, *, stdin=None, timeout_s)`**:
  - Tries each destination in turn. It moves to the next destination only on a transport
    failure: ssh exit 255, `OSError`, or a timeout before any output.
  - All remote calls use `--json`. The function parses the single JSON object on stdout.
  - A non-zero remote exit raises `SaseListenError`, prefixed "on <host>:", carrying the
    remote `error` and `hint` (the existing `error_to_json` shape) and the remote exit
    code.
  - If argparse rejects `receive` (exit 2 with "invalid choice"), raise: "sase-listen on
    <host> is too old to receive episodes; upgrade it there:
    `~/.local/bin/uv tool install --force git+https://github.com/sase-org/sase-listen`".
  - If every destination fails, raise `FeedHostUnreachable` (a `SaseListenError`). It
    names the destinations tried and the last stderr line.
- **`pack_episode(episode_id, library)`**:
  - Builds an uncompressed tar of the regular files directly inside the library episode
    dir (`<slug>.mp3`, `script.md`, `cover.jpg`, `chapters.json`, `manifest.json`), in
    sorted order.
  - Refuses when the manifest is missing.
- **`receive_episode(episode_id, data, cfg, …)`** (host side):
  1. Validate everything before writing:
     - the episode id matches `^[a-z0-9][a-z0-9-]{0,100}$`
     - every member is a regular file named `^[A-Za-z0-9][A-Za-z0-9._-]{0,127}$`
     - at most 64 members and 256 MiB in total
     - `manifest.json` parses and its `episode_id` equals the argument
     - the manifest's `audio.file` is among the members
  2. Read members with `extractfile`, never `extractall`.
  3. Commit with `library.atomic_commit` under `library.episode_lock`.
  4. Publish under the feed lock.
  5. Run `mark_manifest_published` in the host library.
  6. Return JSON with a **masked** `item_url`.
- **`publish_any(episode_id, cfg, library=None)`**:
  - Local role: lock, then `publish_episode`.
  - Remote role: flush the outbox (best-effort), then `pack_episode`, then
    `run_remote(["feed", "receive", episode_id, "--json"], stdin=tar, timeout_s=600)`.
  - Returns the host JSON plus `host` and `via` (the destination used).
- **Outbox** (`state_dir()/outbox/<episode-id>.json`, holding `episode_id`, `queued_at`,
  `attempts`, `last_error`):
  - Functions: `queue_publish`, `pending_publishes`, and `flush_pending(cfg)`, which
    returns the published and failed lists.
  - An entry is deleted on success.
  - Re-queueing an id updates the existing entry.

### Feed (`src/sase_listen/feed.py`)

- **`feed_lock(timeout_s=120)`**:
  - Takes `fcntl.flock` on `locks_dir()/feed.lock`, polling until it times out. On
    timeout: exit 1, "the feed is busy".
  - flock is per open file description, so nested acquisition in one process would
    deadlock. Make it re-entrant with a thread-local depth counter, or take it once
    around each public operation (publish, unpublish, rebuild, prune, receive).
    `publish_episode` must never self-deadlock through `rebuild_feed`.
- **`feed_status` JSON** gains `host` (local hostname), `sase_listen_version`, and
  `receive_protocol: 1`.
- **Token masking**:
  - Publish results mask the token in `item_url` unless the new `publish --show-url` is
    given.
  - Remote results are always masked: the token never leaves the host.

### CLI

- `publish EPISODE | --latest | --pending [--show-url] [--json]`:
  - Dispatches through `publish_any`.
  - In remote mode, a failure queues the episode and exits 1 with the hint
    `sase-listen publish --pending`.
  - `--pending` flushes the outbox in either role.
- `unpublish EPISODE`: in remote mode, runs `unpublish <id> --json` on the host.
- `feed`, `feed rebuild`, `feed prune`:
  - In remote mode they run on the host with `--json` (plus `--show-url` when given),
    and the result is printed with the existing human printer.
  - `--qr` builds the QR locally from the URL the host returns with `--show-url`.
  - Status adds `Host: apollo (via apollo)` and the local outbox count.
- `feed init` in remote mode exits 2: "This machine publishes to feed host apollo; run
  `sase-listen feed init` on apollo."
- New host-only `feed receive EPISODE_ID --json`:
  - Reads the tar from stdin, capped at the size limit.
  - Refuses unless `feed_role` is `local`.
  - Document it in `cli.md` as the internal transport endpoint. Keep the `feed` action
    choices sorted (`init`, `prune`, `rebuild`, `receive`).

### Pipeline and doctor

- **`pipeline.render`**: call `publish_any` instead of `publish_episode`.
  - Auto-publish failure: in remote mode, queue the episode and warn "Auto-publish to
    <host> failed (<error>); queued — run `sase-listen publish --pending`". Do not
    change the exit code.
  - Explicit `--publish` failure: in remote mode, queue it, then raise as today.
  - `RenderResult` and JSON gain `publish_queued` and `publish_host`.
- **`doctor`**:
  - Remote role: skip the local feed-dir and token checks. Add a `feed:host` check that
    runs `feed --json` remotely. It passes when `configured` is true and
    `receive_protocol >= 1`. Detail: "apollo via apollo · sase-listen X · N episodes".
  - Warn on pending outbox entries.
  - Local role: add an informational `feed:host` line reading "this machine".

### Docs

- New `docs/multi-machine.md`, placed in the mkdocs nav after "Podcast feed". It covers:
  - the model
  - the config snippet
  - a per-machine checklist: install, credentials via `pass` or env, and
    `ssh -o BatchMode=yes apollo true`
  - what travels over SSH: MP3, cover, chapters, script, manifest — never the token
  - the outbox and `publish --pending`
  - the upgrade order (host first)
  - the thin-client fallback `ssh apollo '~/.local/bin/sase-listen render <URL> …'`
- `podcast-feed.md`: the live setup is tailnet-only `tailscale serve`; Funnel becomes
  the alternative.
- Also update `configuration.md`, `cli.md`, `troubleshooting.md` (host unreachable, host
  too old, "not the feed host"), `architecture.md`, and the `AGENTS.md` architecture
  map.

### Tests (new `tests/test_feedhost.py`, plus updates)

- **Role detection**: empty host, matching host, case and FQDN differences, env
  override, and the remote-call refusal.
- **pack/receive round trip** into a separate temp "host" XDG tree. Rejection cases:
  traversal names, directory or symlink members, id mismatch, oversize, missing
  manifest, missing audio file.
- **`run_remote`** through a fake `SASE_LISTEN_SSH` script: fallback to the second
  destination on exit 255, propagation of a remote JSON error, and detection of a host
  that is too old.
- **End-to-end**: the fake ssh runs the remote command locally under the host's XDG env.
  A tone-engine `render` in a "client" tree auto-publishes `kind: research` into the
  host feed. Check that `feed.xml` has the item and the client manifest has
  `published: true`.
- **Outbox**: an unreachable host leads to a queued episode and a warning, then
  `publish --pending` succeeds once the host is reachable.
- **Feed lock**: two concurrent publishes both land.
- The existing `feed.xml` goldens stay byte-identical.

## Phase url-acquire: Fetch and extract web articles as render sources

### Dependencies

- Add `trafilatura>=2.0`, `curl_cffi>=0.10`, and `lxml>=5` to `pyproject.toml`. `lxml`
  is imported directly for outline repair.
- Run `uv lock`.
- Add mypy `ignore_missing_imports` overrides only where stubs are missing.
- Confirm in `uv.lock` that wheels exist for Linux x86_64 and macOS arm64.
- Note in the commit that this phase owns the dependency change. The old AGENTS.md
  "don't edit pyproject" rule was for the original parallel scaffold phases.

### New package `src/sase_listen/web/`

**`fetch.py`**

- `fetch_page(url, *, timeout_s=30, max_bytes=20 MiB) -> FetchedPage`, with
  `requested_url`, `final_url`, `status`, `content_type`, `body`, `fetched_at`.
- Implementation:
  `curl_cffi.requests.get(url, impersonate="chrome", allow_redirects=True, …)`.
- Accept only http(s), and only `text/html` or `application/xhtml+xml`.
- `application/pdf` is a usage error: "PDF sources are not supported yet — convert to
  Markdown and render the file".
- A non-2xx status raises an error that includes the status.
- Bot-challenge detection:
  - Signals: a `cf-mitigated: challenge` header, or a 403/429/503 whose body carries
    challenge markers.
  - Raise exit 1 with the hint "Save the page from a browser (HTML only) and re-run with
    `--html FILE`".
- Never use a hosted fetcher: pages are fetched locally.

**`extract.py`**

`extract_article(html, url) -> Article`, with `markdown`, `title`, `author`, `date`,
`site`, `canonical_url`, `description`, `words`, `outline`, `restored_headings`, and
`missing_headings`.

- **Trafilatura call**:
  `extract(html, url=url, output_format="markdown", include_formatting=True, include_tables=True, include_links=False, include_images=False, include_comments=False, with_metadata=False)`
  plus `extract_metadata(html, default_url=url)`.
- **Author fallback**: when the author is missing, take it from a leading byline
  (`^By ([A-Z][^,\n]{1,60})`) in the description or the first paragraph. Example: "By
  Ryan Lopopolo, Member of the Technical Staff" gives "Ryan Lopopolo". Drop that byline
  paragraph from the body.
- **Outline repair**, `repair_outline(markdown, html) -> (markdown, restored, missing)`:
  1. Parse with `lxml.html`. Scope to the first `<article>`, else `<main>`, else
     `<body>`.
  2. Collect non-empty `h2` and `h3` text (`text_content()`, whitespace collapsed). Skip
     headings inside `nav`, `header`, `footer`, `aside`, `button`, or
     `[role=navigation]`.
  3. For each heading, find the first later `p`, `li`, or `blockquote` in document order
     with at least 40 characters of text.
  4. Normalize text the same way on both sides: NFKC, casefold, collapsed whitespace,
     and markdown emphasis or backticks stripped.
  5. Find the first extracted-Markdown line that starts with the normalized prefix
     (about 80 characters). Search forward from the previous insertion point so order is
     kept.
  6. Insert `## Heading` (for h2) or `### Heading` (for h3) before that line, unless the
     heading already appears as a heading.
  7. Report unmatched headings in `missing_headings`. Footer boilerplate such as
     "Author" or "Keep reading" is expected to land there.
- **Short pages**: reject extractions under 150 words (exit 1). The hint names `--html`
  and suggests checking for a login, abstract, or error page.
- **Lost outline**: warn when the page had at least 3 outline headings but none were
  restored.
- **Canonical URL**:
  - Use the metadata URL (canonical or og:url) when it is http(s), else the final URL.
  - Drop the fragment and the `utm_*`, `fbclid`, `gclid`, `mc_cid`, and `mc_eid` query
    params.
  - Lowercase the scheme and host.

**`store.py`**

- Store root: `$XDG_DATA_HOME/sase-listen/sources/`. Add `sources_dir()` to `paths.py`.
- Each source gets a directory `<slug(title)>-<sha256(canonical)[:6]>/` containing:
  - `page.html`: raw bytes
  - `source.md`: repaired Markdown with an H1 title
  - `source.json`: `url`, `canonical_url`, `final_url`, `title`, `author`, `site`,
    `date`, `description`, `fetched_at`, `http_status`, `html_sha256`, `source_sha256`,
    `words`, `extractor {name, version}`, and `outline {found, restored, missing}`
- `sources/index.json` maps both the normalized requested URL and the canonical URL to
  the directory.
- All writes are atomic.
- `acquire(url, *, html_file=None, refresh=False)` reuses the stored source unless
  `--refresh` is given. Reuse means offline re-renders work and the same script is
  reused.

### Script and pipeline integration

- **Script model and parser**:
  - Add `article` to `Kind` and `VALID_KINDS`.
  - Add optional `ScriptMeta.author` and `ScriptMeta.site`. The parser reads them, and
    `dumps` writes them when set.
- **`pipeline.looks_like_url(source)`**: a case-insensitive `^https?://` test, checked
  **before** `looks_like_ref`. `research:…` stays a ref.
- **Verbatim URL path**:
  - Run `normalize_markdown(source.md)`, then set:
    - `title`
    - `source` = the canonical URL
    - `date`, `author`, `site`
    - `kind: article`
    - `edition: verbatim`
    - `producer: deterministic`
  - Save the result as `verbatim_narration.md` in the source dir. Reuse it while the
    source SHA is unchanged.
- **`LoadedSource`** gains `source_url` and `source_meta`. For URL sources:
  - `source_label` is the canonical URL.
  - `source_key` is `url:<canonical>#<edition>`.
  - `source_sha256` is the SHA of `source.md`.
  - Any script with `kind: article` and an http(s) `source` uses the same key and URL.
    So rendering the stored script file directly yields the same episode id.
- **Manifest `source` for articles**:
  `{url, sha256, title, author, site, date, fetched_at, script_path}`. Other sources
  keep today's `ref`/`path` shape.
- **`RenderRequest`** gains `edition: str | None`, `html: str`, and `refresh: bool`.
  - In this phase, URLs accept only `verbatim`, which is also the default.
  - `brief` and `full` raise a clear usage error. The url-editions phase adds them.
- **CLI**:
  - `render SOURCE` help reads "Narration script, Markdown file, kind:path ref, or
    http(s) URL".
  - Add `-e/--edition`, `--html FILE`, and `--refresh` to both `render` and `script`.
  - `script URL --json` reports the source dir, words, outline found/restored/missing,
    and omissions.
- **Spoken intro** for `kind: article` (`build_intro_text`, including the verbatim
  intro):
  - The kind phrase is `, by {author} at {site}`, `, by {author}`, or `, from {site}`,
    depending on what is known.
  - The date phrase is ` published {date}`.
  - Example: "This is an AI-narrated reading of Harness engineering: …, by Ryan Lopopolo
    at OpenAI, published February 11, 2026."
- **Cover label** for articles: `<SITE> · AUDIO EDITION`.
- **Auto-publish** becomes `cfg.feed.auto_publish and kind in {"research", "article"}`.
  Update the docs wording to match.

### Tests

- **Synthetic HTML fixtures** imitating the OpenAI structure:
  - a hidden TOC `<nav>` repeating the headings in `<button><span>`
  - `<h2 class="…"><span>…</span></h2>` sections
  - a byline paragraph
  - footer `h2`s ("Author", "Keep reading")

  Assert that the repaired headings come back in order and that the footer headings are
  reported missing.

- Fetch failures, using a monkeypatched `fetch_page`: a short page is rejected; a
  challenge returns the `--html` hint; a PDF is a usage error.
- Canonical URL normalization.
- Store reuse versus `--refresh`, and the `--html FILE` path.
- URL versus ref detection.
- `script URL --json` in the verbatim edition.
- Tone-engine `render URL --dry-run` and a full render: check the manifest URL fields,
  the episode id, and that the id is stable when rendering the stored script file.
- The new intro phrasing.
- One `live`-marked test, skipped without `SASE_LISTEN_LIVE=1`: fetch the harness
  article and assert at least 2,000 words and at least 8 restored headings.

### Docs

- New `docs/web-articles.md`, placed in the nav after "Narration scripts". It covers:
  - what works (public HTML articles)
  - what does not (PDF, logins, JS-only pages; use `--html` for these)
  - where sources are stored and how they are reused
  - outline repair
  - what a verbatim reading omits (code, tables, math)
  - privacy: pages are fetched locally, and nothing goes to a hosted reader
- Also update `cli.md`, `configuration.md` (add the sources dir to the paths table),
  `narration-scripts.md` (`article` kind, `author`/`site`), `architecture.md`, and the
  `AGENTS.md` map.

## Phase url-editions: Brief and full article editions with a script writer

It depends on feed-host because it edits `feed.py` after that phase's lock and status
changes land.

### Writer

- **Config section `writer:`**: `engine: gemini` (the only supported engine for now;
  validate it), `model` (default below), `temperature: 0.3`, `max_attempts: 3`,
  `timeout_s: 300`.
  - Env override: `SASE_LISTEN_WRITER_MODEL`.
  - Update `_TOP_LEVEL_KEYS`, `masked_snapshot`, and the docs.
  - Credentials reuse `engines.gemini` through `engines.secrets.resolve_api_key`.
- **Package `src/sase_listen/writer/`**:
  - `base.py`: the `Writer` protocol,
    `write(system, user) -> WriterReply(text, model_version, input_tokens, output_tokens)`.
  - `gemini.py`:
    - Calls google-genai
      `client.models.generate_content(model=…, contents=user, config=GenerateContentConfig(system_instruction=system, temperature=…, max_output_tokens=16384))`.
    - Maps 429, 5xx, and timeouts to `TransientEngineError` and reuses the existing
      retry helper.
    - Maps auth failures to `CredentialsError` (exit 3).
  - `prompt.py` and `author.py`.
- **Default model**: confirm the model id live (`client.models.list()`) and pick a
  pro-class default. `gemini-pro-latest` was available at planning time. Record the
  response's `model_version`.
- **Prompt** (`WRITER_PROMPT_VERSION = 1`):
  - The system prompt is `cli.guide.render_guide(edition)` plus this article addendum:
    - The source is a published web article; read "report" as "article". Ignore the
      guide's file-naming, `source_blob`, `cover`, and lint-command steps.
    - Output **only** the body: `##` chapters and plain paragraphs. No frontmatter, H1,
      code fences, preamble, AI disclosure, intro, or outro.
    - Turn the article's first-person voice into third person ("the team", "the
      author"). The narrator is not the author. Keep the author's claims separate from
      any interpretation, and add no opinions.
    - **Brief**: 2–3 chapters and about 600 words: what the article is about, its
      central argument, the deciding evidence, and the takeaway.
    - **Full**: cover every major section in order. Use the article's own section
      headings as chapter titles where they are speakable, merging adjacent sections to
      stay within 4–8 chapters. Stay within the 2,400-word budget and never exceed about
      90% of the source's word count.
    - Follow the guide's rules for numbers, code, tables, and symbols.
  - The user prompt holds, in order:
    1. a metadata block: title, author, site, published date, and the canonical URL
       (with an instruction never to speak it)
    2. the outline
    3. the source Markdown between clear delimiters
- **Loop**, `author_script(article, edition, cfg, writer) -> AuthoredScript`:
  1. Call the writer.
  2. Strip any stray fences, frontmatter, or H1.
  3. Build the frontmatter deterministically: `narration: 1`, `title`, `source` = the
     canonical URL, `date`, `kind: article`, `edition`, `producer: agent`, `author`,
     `site`, and `target_minutes` (4 for brief; for full,
     `min(16, ceil(0.9 × source_words / 150))`).
  4. Run `lint_text(script, source_text=source.md)`.
  5. Any error, or the W012 over-budget warning, triggers a repair turn. The repair turn
     sends the previous body plus the findings (`rule line: message — hint`) and asks
     for the full corrected body. Stop after `max_attempts` attempts in total.
  6. Other warnings are tolerated and recorded.
  7. If errors remain, save the script anyway and raise exit 6 with the hint "edit it,
     then `sase-listen render <path>`".
- **Persistence**:
  - Save `<edition>_narration.md` and `<edition>_writer.json` in the source dir. The
    JSON holds `model`, `model_version`, `prompt_version`, `prompt_sha256`, `attempts`,
    final findings, tokens, `source_sha256`, and `created_at`.
  - Reuse the saved script when the source SHA, prompt version, edition, and model all
    match, unless `--refresh` is given.
- **CLI wiring** (`render URL` and `script URL`):
  - `-e/--edition {brief,full,verbatim}`, with **brief as the URL default**. Update the
    help text.
  - `render --dry-run` writes the script (state the small writer cost) and prints the
    plan.
  - JSON gains `script_path` and a `writer` summary.
  - Manifest `script` gains `writer` (`model`, `model_version`, `prompt_version`,
    `attempts`).

### Labels and feed items (pipeline, `audio/tags.py` callers, `feed.py`)

- **Display title** for `kind: article`: `<title> (Brief)`, `<title> (Full)`, or
  `<title> (Reading)` for verbatim.
  - It is used for the manifest `title`, the ID3 title, and the feed item title.
  - The spoken intro keeps the bare title.
  - The episode id still comes from the bare title plus the source key.
- **Item description for URL sources** contains:
  - a coverage sentence:
    - Brief: "A short narrated briefing: the article's question, answer, and deciding
      evidence."
    - Full: "A full-length narrated adaptation of the whole article — not a
      word-for-word reading."
    - Reading: "The article text read aloud; code, tables, and figures are omitted."
  - a byline (author, site, published date)
  - the chapter list
  - `<a href="{url}">Read the original article</a>`
- `source_url_for` returns `source.url` first, then the ref templates. The ID3
  description uses the same coverage sentence.
- Research items stay byte-identical: the existing golden must not change. Add a new
  golden for an article item.

### Tests

- Use a fake `Writer`, with no network:
  - frontmatter assembly
  - fence and H1 stripping
  - a planted number-fidelity error fixed on the second round
  - exhausted attempts lead to exit 6 with the script saved
  - cache reuse versus `--refresh` versus a prompt-version bump
  - full-edition targets for short sources
- The Gemini adapter request shape (a kwargs snapshot against a stub client) and its
  error mapping.
- The feed golden for an article item and the display titles.
- End-to-end `render URL -e full` with the fake writer and the tone engine.
- Config keys and the masked snapshot.
- A `live`-marked brief script for the harness article.

### Docs

- `web-articles.md` editions section:
  - what brief and full mean (budgets; "full" is an adaptation, not the complete text)
  - the review workflow: `script URL -e full -o f.md`, then `render f.md`
  - costs
- Also update the `configuration.md` writer section, `cli.md`, `narration-scripts.md`
  (writer scripts are `producer: agent`), and `podcast-feed.md` (article items).

## Phase rollout-proof: Roll out to every machine and publish the harness-engineering full edition

- **Repos**: `sase-listen` (field notes and any bug fixes) and `chezmoi`. Open each with
  `sase repo open` and read its `AGENTS.md`.
- **Hosts**:
  - athena, where agents run
  - apollo (`ssh apollo`, fallback `ssh apollo-do`; use `~/.local/bin/...` paths there)
  - the Mac (`ssh mac`, best-effort)
- **Secrets**: never print the feed token or the full feed URL. Redact token paths in
  any captured output.

1. **Code is landed.**
   - Confirm the feed-host, url-acquire, and url-editions commits are on sase-listen
     `origin/master`.
   - If they are not, build a wheel from the checkout (`uv build --wheel`) and install
     that file in the steps below, copying it to apollo with `scp`.
2. **apollo first**, because it must understand `feed receive` before any client pushes.
   - Install:
     `ssh apollo '~/.local/bin/uv tool install --force --reinstall git+https://github.com/sase-org/sase-listen'`.
   - Then run `--version`, `doctor`, and `feed --json` there. Expect local role,
     `receive_protocol: 1`, and the existing episodes, still on the old config.
   - Confirm `tailscale serve status` still maps `:8443/<token>` to the feed dir.
3. **athena**:
   - Record the current receipt: an editable install of the stale
     `~/projects/github/sase-org/sase-listen`.
   - Install:
     `uv tool install --force --reinstall git+https://github.com/sase-org/sase-listen`.
     athena then runs the same code as apollo.
   - The field notes say how to restore an editable dev install.
4. **chezmoi config** (`home/dot_config/sase-listen/config.yml`). Keep `narrator` and
   `engines`, and set the feed section to:

   ```yaml
   feed:
     host: apollo
     host_ssh: [apollo, apollo-do]
     dir: ~/.local/share/sase-listen/feed
     base_url: https://apollo.tail297af1.ts.net:8443
     token_command: pass show sase_listen_feed_token
     title: SASE Listen
     auto_publish: true
   ```

   - Never commit secrets: the dotfiles repo is public.
   - Apply only this target on athena now, from the edited checkout:
     `chezmoi apply --source <chezmoi checkout> ~/.config/sase-listen/config.yml`. This
     honors the repo's `.chezmoiroot`.
   - Verify with `sase-listen config`.
   - apollo works with either config, because it is the host.
   - After the chezmoi commit lands, run `chezmoi update -a --force` on athena, apollo,
     and the Mac when it is online. The chezmoi repo's gotcha requires this; record it
     as the closing step.

5. **athena checks**: `sase-listen doctor` must show `feed:host` OK via apollo, and
   `sase-listen feed` must show apollo's episodes with `Host: apollo`.
6. **Brief check, no publish**:
   `sase-listen script https://openai.com/index/harness-engineering/ -e brief --json`.
   Expect lint clean and about 600 words; skim it for fidelity.
7. **Full-edition proof on athena**:
   1. Run
      `sase-listen render https://openai.com/index/harness-engineering/ -e full --dry-run --json`
      and inspect it:
      - at least 8 headings restored
      - lint clean
      - at most 2,400 words
      - the chunk plan and cost estimate
   2. Read the full script. Spot-check claims against `source.md`: "0 lines of
      manually-written code", "about 1/10th the time", "a million lines of code", "five
      months", and the third-person attribution.
   3. Fix any sase-listen bugs found, adding tests, and reinstall.
   4. Run
      `sase-listen render https://openai.com/index/harness-engineering/ -e full --json`
      **without** `--publish`. It must auto-publish: `published: true` and
      `publish_host: apollo`.
   5. The render takes minutes. Run it in the foreground with a generous timeout, or
      hand it to `/sase_monitor`. Exit 4 (TTS quota) is resumable: rerun the same
      command and the cached chunks are reused.
8. **Verify delivery without printing the token**:
   - Read the subscribe URL from `sase-listen feed --show-url --json` into a shell
     variable, then `curl -fsS` it.
   - Assert the newest item:
     - its title is "Harness engineering: leveraging Codex in an agent-first world
       (Full)"
     - its description holds the openai.com link and the full coverage sentence
   - `curl -fsSI` the enclosure: expect 200 and a `Content-Length` equal to the
     enclosure `length`. Also check the cover and the chapters JSON.
   - Confirm `ssh apollo '~/.local/bin/sase-listen feed --json'` lists the episode id.
   - AntennaPod fetches it on its next refresh; the user can pull to refresh to confirm.
9. **Mac (best-effort)**:
   - If `ssh -o ConnectTimeout=10 mac true` succeeds:
     1. Install uv if missing, then
        `uv tool install --force git+https://github.com/sase-org/sase-listen`.
     2. Check `pass show gemini_cli_api_key >/dev/null`. If that fails, document setting
        `SASE_LISTEN_GEMINI_API_KEY` from a local secret. Never commit it.
     3. Get the new config: `chezmoi update` after landing, or
        `SASE_LISTEN_FEED_HOST=apollo` for the check.
     4. Run `sase-listen doctor` (expect `feed:host` via apollo), then
        `sase-listen script <URL> -e brief`.
   - If the Mac is unreachable, record a `PROPOSED FOLLOW-UP:` note on this phase's bead
     with the exact commands. Do not block on it.
10. **Field notes**: append a dated section to sase-listen `docs/field-notes.md`.
    Include what was installed where, the proof episode id, duration, chunks, cost,
    extraction and outline diagnostics, writer attempts, and any issues. Include no
    token and no full feed URL.

Do not publish extra test episodes to the live feed. If one slips in, run
`sase-listen unpublish <id>`.

## Acceptance criteria

- On athena, `sase-listen render https://openai.com/index/harness-engineering/ -e full`
  auto-publishes a lint-clean, chaptered full edition.
- That episode is listed in the apollo-served `feed.xml` with:
  - the `(Full)` title
  - a coverage note
  - the original-article link
  - an enclosure that returns 200 with a matching length
- `-e brief` produces a lint-clean script of about 600 words.
- `doctor` on athena reports the apollo feed host as healthy. apollo still serves every
  earlier episode, and the phone subscription is unchanged.
- An unreachable host queues the publish instead of losing it, and `publish --pending`
  retries it.
- An old host gives a clear upgrade message.
- `sase tool run check` is green in sase-listen.

## Out of scope (possible follow-ups)

- PDF and arXiv ingestion
- Authenticated or JS-only pages beyond `--html`
- Batch or reading-list ingestion
- Hosted alternatives (Audioread, NotebookLM)
- Brief/full editions of local Markdown files (the writer API is source-agnostic)
- A SASE `#listen` macro
- The PyPI release (`sase-1eh`)
