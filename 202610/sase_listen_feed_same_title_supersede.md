---
tier: tale
title: Repair the duplicated arXiv feed episode and supersede same-title episodes
goal:
  apollo's feed holds only the PDF render of the Overseeing Agents paper, publish keeps
  one feed item per title, and every publish that replaces already-downloadable audio
  tells the user how to refresh it in AntennaPod.
size: medium
proposed_by: bbugyi200.athena.0xa.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0xa.f0](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.0xa.f0.md)
- **COMMITS:**
  - [f8154ad](https://github.com/sase-org/sase-listen/commit/f8154ad0708141e6cb7c674c1130015f262a8a2a)
    — feat(feed): supersede same-title episodes on publish with AntennaPod refresh
    notice

# Plan: Repair the duplicated arXiv episode in the AntennaPod feed, and supersede same-title episodes on publish

## Problem

Before the arXiv PDF fix (`eb05e7c feat(web): fetch arXiv paper URLs as their PDF`, see
`@plan:202610/sase_listen_arxiv_pdf_urls.md`), the paper _Overseeing Agents Without
Constant Oversight_ was rendered twice on 2026-10-06. The two renders used different
source keys, so they became two separate library episodes. Both auto-published to the
feed host `apollo`, and both are still in its `feed.xml` (checked live):

| Episode id (`overseeing-agents-without-constant-oversight-challenges-and-…`) | Source                                    | guid suffix | pubDate (UTC) | `itunes:duration` | bytes     |
| ---------------------------------------------------------------------------- | ----------------------------------------- | ----------- | ------------- | ----------------- | --------- |
| `…-c606b2` (stale)                                                           | `https://arxiv.org/abs/2602.16844v1` page | `@7bce71d4` | 13:49:14      | 118               | 1,100,105 |
| `…-09e39f` (correct)                                                         | `https://arxiv.org/pdf/2602.16844`        | `@db69b88a` | 13:54:09      | 529               | 4,388,966 |

Both items have the same title,
`Overseeing Agents Without Constant Oversight: Challenges and Opportunities (Full)`.

### Why AntennaPod kept the old audio (checked against AntennaPod source, commit `9c7ffa167`)

- `FeedItemDuplicateGuesser.seemDuplicates` treats two items as **one episode** if any
  of these hold:
  - same guid;
  - same enclosure URL;
  - **same title, same pubDate day, durations under 10 min apart, and same MIME type.**
- Our two items meet the last rule: same title, same day, 118 s vs 529 s (411 s apart),
  both `audio/mpeg`.
- `FeedDatabaseWriter.updateFeed` sorts items newest first, so the PDF item comes first:
  1. The PDF item does not match any saved guid. The guesser then matches it to the
     already-saved abs item. AntennaPod "repairs" that saved item: it copies over the
     new guid, title, pubDate, and enclosure URL/size/MIME (`FeedItem.updateFromOther`,
     `FeedMedia.updateFromOther`).
  2. It does **not** replace an already-downloaded file. It also does not overwrite a
     duration it measured itself. So the phone shows one episode that plays the 1m57s
     abstract-only audio.
  3. The abs item is then skipped on every refresh, because it looks like the PDF item
     added a second time. Each refresh logs "The podcast host appears to have added the
     same episode twice" in AntennaPod's download log.
- The normal refresh path (`FeedUpdateWorker`) passes `removeUnlistedItems=false`.
  Dropping an item from the feed therefore never removes or changes anything already on
  the phone.
- `MediaDownloadedHandler` measures the duration again after any new download.

Consequences:

1. **No change on the server can replace the file already on the phone.** The feed can
   be cleaned up, but the phone needs one step: delete the episode's download and
   download it again. AntennaPod's item already points at the `…-09e39f` MP3 URL, so the
   new download fetches the correct audio.
2. The arXiv fix stops abs/pdf renders of one paper from splitting again. Other cases
   still produce two same-title items in the feed, and AntennaPod quietly merges or
   drops them:
   - the same title rendered from a different source key, such as a Markdown file moved
     to a new path or two URLs for one article;
   - explicitly publishing an older same-title episode.
3. `docs/podcast-feed.md` says the guid `<episode-id>@<audio-sha256[:8]>` makes
   "re-renders show up as fresh audio". That is wrong for AntennaPod once the episode is
   downloaded:
   - A re-render keeps the same enclosure URL, so `FeedItemDuplicateGuesserPool` matches
     it by URL first.
   - The downloaded file is kept.

## Goals

- Remove the stale `…-c606b2` item from apollo's feed so it holds exactly one item for
  this paper (the PDF render).
- Make `publish` keep **one feed item per title**: publishing an episode removes other
  episodes with the same title from the feed. Use the same title comparison AntennaPod
  uses. Feed copies only; the library is never touched.
- Whenever a publish replaces audio a podcast app may already have downloaded, say so
  and tell the user exactly what to do in AntennaPod. This covers both a same-title
  replacement and a same-id re-render with different MP3 bytes.
- Correct the docs.

## Non-goals

- No re-render. `…-09e39f` is already the correct PDF render.
- Do not delete library episodes on athena or apollo. The library keeps every episode by
  design, and nothing reads the library manifest `published` flag. The stale `…-c606b2`
  library copies stay as history.
- No title-based cleanup inside `feed rebuild` / `feed prune`. Supersede happens only
  when an episode is published, so existing feed state only changes through an explicit
  publish or unpublish.
- No changes to guid, enclosure URL, or pubDate to get around AntennaPod's matching. A
  same-day item with the same title always merges in AntennaPod, so those tricks cannot
  replace a downloaded file.

## Where the work happens

All code, docs, and tests are in the linked **sase-listen** repo
(`sase repo open sase-listen`). Read its `AGENTS.md` first. Run checks with
`sase tool run check` there, not bare `just check`. sase-listen must not import
`sase`/`sase_core_rs`. Nothing in the sase repo changes.

athena's and apollo's `sase-listen` are editable installs from their primary checkouts.
Editing the linked workspace checkout does not change the installed binary during this
turn, so Step 1 runs against the currently deployed code.

## Step 1 — One-time feed repair (do this first, with the installed `sase-listen`)

Run on athena. Its config routes feed commands to apollo over SSH (BatchMode).

1. Precheck: `sase-listen feed --json`. Confirm `episode_ids` contains both
   `overseeing-agents-without-constant-oversight-challenges-and-c606b2` and
   `overseeing-agents-without-constant-oversight-challenges-and-09e39f`. If `…-c606b2`
   is already gone, skip to step 3.
2. `sase-listen unpublish overseeing-agents-without-constant-oversight-challenges-and-c606b2 --json`.
   It must return `ok: true` and `host: apollo`.
3. Verify with `sase-listen feed --json`: `…-c606b2` is absent, `…-09e39f` is present,
   and the episode count dropped by one.
4. Verify the served XML on apollo read-only:
   - Over `ssh -o BatchMode=yes apollo`, parse
     `~/.local/share/sase-listen/feed/feed.xml` with `python3` and `xml.etree`.
   - Assert exactly one `<item>` titled
     `Overseeing Agents Without Constant Oversight: Challenges and Opportunities (Full)`.
   - That item's guid must start with
     `overseeing-agents-without-constant-oversight-challenges-and-09e39f@`.
   - `itunes:duration` must be `529`, and the enclosure `length` must equal the size of
     the `…-09e39f` MP3 under apollo's `~/.local/share/sase-listen/feed/episodes/`.
   - `~/.local/share/sase-listen/feed/episodes/overseeing-agents-without-constant-oversight-challenges-and-c606b2/`
     must no longer exist.
   - Never print the feed token or an unmasked feed URL.
5. Put the before/after `episode_ids` delta and the XML check result in the final
   response.

## Step 2 — Same-title supersede and replacement notice (`src/sase_listen/feed.py`)

1. Add `canonical_title(title: str) -> str`. It mirrors AntennaPod's
   `FeedItemDuplicateGuesser.canonicalizeTitle`; cite it in the docstring:
   - `strip()`;
   - `“` `”` `„` → `"`;
   - `—` (U+2014 only) → `-`;
   - case-sensitive, nothing else.
2. Add a manifest-only title reader for feed episode dirs. Read only `manifest.json`; do
   **not** hash MP3s as `list_feed_episodes` does. Skip dirs with a missing or invalid
   manifest.
3. Change `publish_episode`. Under the existing `feed_lock()`:
   - **Before** copying the MP3, set `replaced = True` if `dest / src_mp3.name` already
     exists and its sha256 differs from the source MP3's. Otherwise `replaced = False`.
   - Copy the episode files exactly as today.
   - Then, for every other `<feed>/episodes/<id>/` (`id != episode_id`) whose manifest
     title, after `canonical_title`, equals the published manifest's non-empty canonical
     title: `shutil.rmtree` the feed copy and collect the id.
   - An empty or missing title never matches (AntennaPod's `sameAndNotEmpty`).
   - The episode being published wins whether it is newer or older. Publishing is an
     explicit choice.
   - Then call `rebuild_feed` as today.
   - Return two new keys: `superseded` (sorted list of removed ids) and `replaced`
     (bool). `removed` stays retention-only.
   - If the copy raises, nothing is superseded.
4. Add a shared constant and a formatter, both used by the CLI and the pipeline:
   - `STALE_DOWNLOAD_HINT`: "Podcast apps keep audio they already downloaded: in
     AntennaPod, delete this episode's download and download it again."
   - `replacement_notice(superseded: list[str], replaced: bool) -> str`:
     - with superseded ids: "Superseded N same-title feed episode(s): <ids>."
     - otherwise, when replaced: "Replaced this episode's earlier feed audio."
     - followed by the hint;
     - `""` when there is nothing to report.
5. `feedhost.receive_episode` already returns `publish_episode`'s dict, so the new keys
   reach remote callers without transport changes. `RECEIVE_PROTOCOL` stays `1`; the
   keys are additive. Every reader must default missing keys (`[]` / `False`), so an
   older feed host still works.

## Step 3 — Surface the notice

1. `src/sase_listen/cli/publish_cmd.py` `run_publish` (single-episode path, text mode):
   after the existing `Published …` / `Audio:` / `Host:` lines, print one
   `Superseded <id> (same title).` line per superseded id. If `replacement_notice(...)`
   is non-empty, print `note: <STALE_DOWNLOAD_HINT>`. JSON mode already spreads
   `result`, so `superseded`/`replaced` appear with no extra work. `--pending` output is
   unchanged.
2. `src/sase_listen/pipeline.py`, successful-publish branch (after `publish_any` returns
   `published_info`): if `published_info` has a non-empty `superseded` or a true
   `replaced`, append `replacement_notice(...)` to `warnings`. `cli/render.py`'s
   `_print_result` already lists warnings under `⚠ N warning(s)`. Leave the
   `Published to <host> — refresh the feed in AntennaPod…` line unchanged.

## Step 4 — Docs (sase-listen `docs/`)

- `podcast-feed.md`:
  - Fix the guid bullet. The guid changes on a re-render, but AntennaPod matches items
    by enclosure URL and by same title + same day, and keeps an already-downloaded file.
  - Add a short "Re-renders and same-title episodes" section covering:
    - the one-item-per-title rule on publish (feed copies only; the library is kept);
    - the replacement notice;
    - the AntennaPod recovery step (delete the download, download again);
    - `sase-listen unpublish <id>` for any same-title duplicate published before this
      change.
- `cli.md`, in the `feed`, `publish`, `unpublish` section: state that `publish`
  supersedes same-title feed episodes and reports `superseded`/`replaced` in `--json`.
- `troubleshooting.md`: new entry "AntennaPod still plays the old audio after a
  re-render" with the cause and the two-tap fix.
- `changelog.md`: one `## Unreleased` bullet.

## Step 5 — Tests

- `tests/test_feed.py`, using the existing `_feed_config` / `_write_library_episode`
  helpers:
  - Publish `ep-a-111111` titled "T", then `ep-b-222222` titled "T":
    - the second result's `superseded == ["ep-a-111111"]`;
    - `<feed>/episodes/ep-a-111111` is gone;
    - `feed.xml` has one item, whose guid starts with `ep-b-222222@`;
    - the library copy of `ep-a-111111` is untouched.
  - Canonicalization:
    - titles that differ only by surrounding whitespace, `“…”` vs `"…"`, or `—` vs `-`
      supersede;
    - titles that differ in case, by `–` (en dash), or by edition label (`(Brief)` vs
      `(Full)`) do not;
    - an empty title never supersedes.
  - Publishing the older same-title episode afterwards supersedes the newer one (the
    explicit publish wins).
  - `replaced`:
    - `False` on first publish;
    - `True` when republishing the same id with different MP3 bytes (extend or sit next
      to `test_guid_changes_on_rerender`);
    - `False` when republishing identical bytes.
  - CLI: text `publish` of a superseding episode prints `Superseded <id> (same title).`
    and the `note:` hint. `--json` carries `superseded` and `replaced`.
  - Render: two local scripts with the same title but different paths (so different
    episode ids) with `--publish`. The second render's warnings contain the replacement
    notice, and the feed holds only the second episode. Follow
    `test_render_publish_and_unpublish_cli` / `_write_feed_config`.
- `tests/test_feedhost.py`: a receive round-trip of a same-title second episode returns
  `superseded` in the receive result, so the remote JSON path is covered. A payload
  without the new keys (older host) does not break `publish_any` callers.

## Verification

- Focused: `uv run pytest tests/test_feed.py tests/test_feedhost.py` in sase-listen.
- Full: `sase tool run check` in sase-listen (ruff, format, mypy `--strict`, codespell,
  pytest). It must exit 0.
- Step 1's post-repair checks pass, and their results appear in the final response.

## Hand-off notes for the final response

- **Phone step (user):** in AntennaPod, open _Overseeing Agents Without Constant
  Oversight … (Full)_ → **Delete** the download → **Download** (or stream). The duration
  should read about 8:49 with 6 chapters. AntennaPod's item already points at the PDF
  render, so a feed refresh alone will not swap a file that is already downloaded.
- **Deploy (user):** apollo publishes from its editable checkout. The supersede and
  notice only take effect on apollo after the sase-listen commit lands and is pulled
  into apollo's sase-listen checkout (and athena's, for the CLI notice). Step 1 does not
  depend on this.
