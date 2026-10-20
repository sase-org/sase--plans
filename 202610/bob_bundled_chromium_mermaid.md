---
tier: tale
title: Render Mermaid with Playwright's bundled Chromium and rebuild the broken PDFs
goal:
  bob ref create renders Mermaid figures on hosts like apollo whose only browser is
  Playwright's bundled Chromium, and the two library PDFs that lost their figures are
  regenerated with them.
size: medium
decisions:
  doctor_probe:
    ask:
      Should `bob ref doctor` do a one-diagram test render so its mermaid row proves the
      browser launches?
    choices:
      probe:
        Doctor launches the browser once (a few seconds slower) and warns with the real
        launch error
      skip:
        Doctor keeps its fast version-only ping; launch-time failures stay invisible to
        it
    default: probe
    why: doctor printed "mermaid ok" on apollo while every render there failed
    answer: probe
proposed_by: bbugyi200.athena.12p
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-20 17:37:46
status: wip
---

# Fix Mermaid rendering on hosts that only have Playwright's bundled Chromium

## Problem

`~/bob/lib/chat/capture_similar_tasks_band.pdf` has none of its 4 Mermaid figures. Each
diagram prints as source under the note "Mermaid diagram not rendered: browser launch
failed: BrowserType.launch: Executable doesn't exist at
~/.cache/ms-playwright/chromium_headless_shell-1234/chrome-headless-shell-linux64/chrome-headless-shell",
followed by Playwright's multi-line "playwright install" banner.
`~/bob/lib/chat/obsidian_z_image_inspection.pdf` (1 figure) failed the same way earlier
the same day. No other library PDF is affected: a `pdftotext` scan of every PDF since
2026-09 finds only these two. `ref_create_mermaid_figures.pdf` matches the string only
because it quotes it.

## Root cause (verified)

1. **apollo created both PDFs, not athena.** The `research-highlights` SASE file hook
   (`bob ref create --include-id -P "$SASE_FILE_HOOK_PROJECT" <report>`) runs on the
   host where the swarm's linker commits the report. In apollo's `~/bob` reflog the
   commit `bob ref create: ingest chat/capture_similar_tasks_band` is a local commit;
   athena only fast-forwarded to it.
2. **apollo has no Google Chrome.** Its only browsers are Playwright's revision-1248
   builds (`chromium-1248`, `chromium_headless_shell-1248`). They were installed at
   2026-10-20 02:45 UTC by `playwright install` from a bob-cli `tools/docs/.venv`
   (Playwright 1.64.0). That run's cleanup deleted revision 1234 (Chromium 151), which
   earlier notes in `docs/highlights-clip.md` record apollo having. The adapter's
   uv-managed environment never registers itself in `~/.cache/ms-playwright/.links`, so
   nothing kept 1234 alive.
3. **The adapter discovers one browser but launches another.** In
   `scripts/web_clip/web_clip_adapter.py`, `discover_browser()` falls back to
   `_bundled_chromium_candidates()` and finds `chromium-1248/chrome-linux64/chrome`, but
   returns `"launch": {}`. With no `executable_path`, Playwright 1.62.0 (the adapter's
   pinned version) launches its own default headless build: revision 1234, which no
   longer exists. Every `diagrams` op and every web-article `capture`/headed fallback on
   apollo fails. `diagrams.rs` then degrades softly to source blocks with a note, as
   designed. The file hook's terminal warning went unseen.
4. **`bob ref doctor` hid it.** On apollo it prints
   `mermaid: ok (bundled 12.2.0 · chrome 156.0.8078.4)` and names the chromium-1248
   path. The mermaid row is built from the ping only, and the ping never launches
   anything.

Reproduction on apollo, using the installed adapter with a one-item `diagrams` request:
it fails with exactly the PDF's error. The same request with
`BOB_CHROME=~/.cache/ms-playwright/chromium-1248/chrome-linux64/chrome`, which sends
discovery down the `executable_path` branch, renders a valid PDF with Playwright 1.62
driving Chromium 156. Its only warning is the existing sandbox retry. So the fix is to
launch the executable that discovery actually found.

Ruled out: athena renders correctly today through `/opt/google/chrome/chrome`. A "Socket
path too long" Chrome abort only shows up when `TMPDIR` is long and is handed to the
adapter directly. `ScratchDir::create` (`src/native/highlights_ref/workdir.rs`) already
falls back to `/tmp` when `TMPDIR` is longer than 40 bytes, and a real `bob ref create`
under a long SASE agent `TMPDIR` rendered fine. Do not change that path.

## Scope

All code changes are in the **bob-cli** repo (`bobs-org/bob-cli`), a different SASE
project. Open it with `/sase_repo` (`sase repo open bob-cli -r "<why>"`), read that
checkout's `AGENTS.md`, and work only in the printed path. Before editing `docs/`, read
bob-cli's `docs.md` reference memory through `/sase_memory_read`, run from the bob-cli
checkout. Nothing changes in the sase repo.

## Changes

### 1. Launch the discovered bundled Chromium (`scripts/web_clip/web_clip_adapter.py`)

- In `discover_browser()`, the bundled-Chromium branch must return
  `"launch": {"executable_path": bundled}`, so Playwright launches exactly the binary
  that ping and doctor report. Leave the `BOB_CHROME` and Google Chrome branches alone;
  `channel: "chrome"` keeps working on athena and macOS.
- In `_bundled_chromium_candidates()`:
  - Order candidates **numerically** by the revision in `chromium-<N>`, newest first.
    The current reverse lexicographic sort would rank `chromium-999` above
    `chromium-1248`. Names without a number sort last.
  - Fix the stale macOS layout. Current Playwright, including the pinned 1.62.0,
    installs
    `chromium-<N>/chrome-mac-{arm64,x64}/Google Chrome for Testing.app/Contents/MacOS/Google Chrome for Testing`.
    The adapter only globs the legacy `chrome-mac/Chromium.app/Contents/MacOS/Chromium`.
    Match both layouts. Linux keeps `chrome-linux*/chrome`, which already covers
    `chrome-linux64`.
  - Pull the per-platform glob patterns into a small helper that takes the platform as a
    parameter, so the self-test can cover the macOS layout on Linux.
- Keep `PLAYWRIGHT_BROWSERS_PATH` handling as is. Do not start a Playwright driver
  during discovery; the docstring explains why.

### 2. Offline self-test coverage (same file, `self_test()`)

Add offline checks that need no real browser:

- Build a temporary fake `PLAYWRIGHT_BROWSERS_PATH` tree containing `chromium-999` and
  `chromium-1248`. Each holds an executable `chrome-linux64/chrome` shell stub that
  prints a version.
- With `BOB_CHROME` unset and `BROWSER_CANDIDATE_PATHS` temporarily emptied,
  `discover_browser()` must return the chromium-1248 path, kind `chromium`, and
  `launch == {"executable_path": <that path>}`.
- Feed the pattern helper a fake Chrome-for-Testing macOS tree and assert it finds the
  binary.
- Restore every patched global and environment variable in a `finally`, following the
  self-test's existing `discover_browser` monkeypatch around the fixture checks.

### 3. Compact the in-PDF fallback note (`src/native/highlights_ref/diagrams.rs`)

When the whole diagrams op fails (`fallback_pass` / `fallback_from_adapter`), the
`fallback_error` written to `results.json` becomes the note printed into the PDF. Today
that is the full multi-line adapter message, including Playwright's box-drawn banner.

- Store a compact reason for the PDF: the first non-empty line, trimmed, capped at 200
  characters on a char boundary. This mirrors the adapter's per-item
  `_first_line_error`.
- Keep the stderr warning (`unavailable_warning`) at full detail. Browser logs are
  useful for debugging in the terminal.
- Add a unit test next to the existing fallback-note tests: a multi-line adapter failure
  gives a single-line `fallback_error` of at most 200 characters, and the warning still
  carries the full text.

### 4. Doctor test render (decision `doctor_probe`)

> [!decision] doctor_probe = probe In `src/native/highlights_ref/clip.rs`, keep the
> ping. When diagrams are enabled and the ping reports a browser, also send one
> `diagrams` request:
>
> - Use a fixed tiny flowchart. Its item id is the lowercase SHA-1 hex of the source.
> - Put the workdir in a `ScratchDir`, which is removed on drop.
> - Resolve the client with a short fixed timeout, about 60 s.
>
> Results:
>
> - On success, print
>   `mermaid: ok (bundled <version> · chrome <version> · test render ok)`.
> - On failure, print
>   `mermaid: warn (test render failed: <compact first line>; diagrams print as source)`
>   and push a matching entry onto `warnings`.
>
> Doctor still writes nothing to the vault, so `writes: none` stays true.
> `BOB_HIGHLIGHTS_DIAGRAMS=off` skips the probe exactly as it skips the row today.
> Update the fake adapter (`tests/cli/highlights/fake_clip.rs`) and any doctor tests
> that pin the mermaid row, and add tests for one passing and one failing test render.
> Update the doctor row's description wherever docs list the doctor rows.

> [!decision] doctor_probe = skip Do not touch doctor. Change 1 already makes the
> reported browser the launched browser.

### 5. Docs

- `docs/highlights-clip.md` ("Hosts and browsers" plus the apollo verification note),
  `docs/reference/compatibility.md`, and the `BOB_CHROME` rows in
  `docs/reference/configuration.md` and `docs/highlights-clip.md`: state that
  Playwright's bundled Chromium is launched by its discovered path, newest installed
  revision first, so it works even when the pinned Playwright's own default revision is
  missing. Record the apollo revision-1248 situation as the motivating case.
- Follow bob-cli's `docs.md` memory for anything it requires, such as the docs check
  recipe.

## Verification

1. In the bob-cli checkout, run `just check` (fmt, clippy, all cargo tests) and
   `just check-web-clip-adapter`. The second does a `py_compile` and runs the self-test.
   On athena the browser-backed self-test checks run against Google Chrome.
2. **Prove the fix on apollo, the failing host, without installing anything.** Copy the
   changed `scripts/web_clip/` directory to a fresh `mktemp -d` on apollo (`ssh apollo`;
   tailnet SSH works without prompts). Then run, there:
   - `uv run --quiet --script <dir>/web_clip_adapter.py --self-test`. This must now pass
     its browser-backed Mermaid checks with bundled Chromium 1248.
   - A one-item `diagrams` request piped into the copied adapter. It must return
     `"ok": true` with a PDF path.

   Generate the request JSON with a small Python one-liner. Hand-escaped `printf` JSON
   breaks on the embedded newline. apollo's `uv` is at `~/.local/bin/uv`, outside
   non-interactive `PATH`. Remove the temp dir afterwards.

## Regenerate the two broken library PDFs

Neither reference has any highlights yet, and neither has a Highlights sidecar.
Regenerate them **on athena**, whose installed bob already renders Mermaid through
Google Chrome (verified). This step does not wait on the new bob being installed.

1. Get the report sources: open `gh:bobs-org/bob-cli--research` with `/sase_repo`.
   Athena's primary bob-cli research clone is behind origin and lacks one of the final
   reports. The sources are:
   - `202610/capture_similar_tasks_band/capture_similar_tasks_band.md`
   - `202610/obsidian_z_image_inspection/obsidian_z_image_inspection.md`
2. Check each `<stem>` before writing. Stop and report instead of overwriting if
   `~/bob/lib/chat/<stem>.md` (a Highlights sidecar) exists, or if
   `~/bob/ref/chat/<stem>.md` has anything under `## Highlights`.
3. Dry run:
   `bob ref create <source> --include-id -P bob -o ~/bob/lib/chat/<stem>.pdf --force -d`.
   Expect `id: <stem>`, `parent bob`, and
   `intake: would ingest into lib/chat/<stem>.pdf → ref/chat/<stem>.md`. Expect no `_2`
   stem, and a Mermaid fence count of 4 for `capture_similar_tasks_band` and 1 for
   `obsidian_z_image_inspection`. A dry run with these exact flags was accepted on
   athena for `obsidian_z_image_inspection`.
4. Run it for real (same command without `-d`). It must print
   `diagrams: rendered N of N Mermaid diagrams` with no "not rendered" warnings.
5. Verify each result:
   - `pdftotext` finds no "Mermaid diagram not rendered".
   - The ref note's `source_pdf_sha256` equals the new PDF's sha256.
   - The resulting `~/bob` commit touches only that PDF and its ref note, plus at most
     an in-place change to the existing reading-task line in `bob.md`. There must be no
     second reading task and no `_2` files.
   - `bob ref doctor` ends `result: ok`.
6. If the in-place create refuses or would duplicate identity, do not delete vault files
   by hand. Stop and report what you saw so Bryan can choose.

## Deploy to apollo

After the bob-cli change is on `master`, use `/sase_gate` to propose (not run)
reinstalling bob on apollo:
`ssh apollo 'cd ~/projects/github/bobs-org/bob-cli && git pull --ff-only && just install && bob ref doctor | grep -E "^(mermaid|web clip browser)"'`.
Say in the gate that it is meant to be approved once the fix has landed. Until then,
file-hook PDFs created on apollo will keep losing their figures.

## Out of scope

- Discovering `chromium_headless_shell-*` as a last resort. It cannot do the headed Xvfb
  fallback that web capture needs.
- Pinning or reinstalling a specific Playwright revision on apollo, or registering the
  adapter's uv environment in `.links`. Launching by discovered path makes revision skew
  harmless.
- Surfacing file-hook warnings to the user from SASE.
