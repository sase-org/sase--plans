---
tier: tale
size: medium
title: Land bob highlights create audio discovery and close sase-1g6
goal:
  bob highlights create discovers a sase-listen companion and copies it beside the
  intake PDF before the listen-card Play button is rendered, then close epic sase-1g6.
proposed_by: bbugyi200.apollo.sase-1g6.land
bead: sase-1g6
create_time: 2026-10-04 21:04:34
status: wip
---

- **PARENT:**
  [202610/research_swarm_listen_card.md](https://github.com/sase-org/sase--plans/blob/main/202610/research_swarm_listen_card.md)
- **BEAD:**
  [sase-1g6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1g6/README.md)

# Land bob highlights create audio discovery and close sase-1g6

## Outcome

`bob highlights create` discovers companion audio and copies it atomically beside the
PDF it renders, and the Play button already implemented for listen cards uses that copy.
Epic `sase-1g6` is then closed in the same turn.

This is one `medium` tale. The missing behavior is the create-time half of an
already-specified contract, the banner and scan halves are on origin, and one agent can
implement, verify, and close the epic directly.

## What the lander already verified

Do not redo these phases. Do not edit `sase-research-artifacts` or `sase-listen`.

- **sase-1g6.1 is on origin.** `sase-research-artifacts` `867222d` is `HEAD` and
  `origin/master`. Audio implies the linker (`run_linker = linker or image or audio`).
  The audio segment waits on the lead and, only when `image` is set, on the image agent.
  The linker waits on the lead, the optional image agent, and the audio agent. The
  linker prompt writes `<div class="listen">` and `audio:` frontmatter only when audio
  succeeded, and a failed TTS render sets `audio.ok=false`, registers no artifact, and
  completes normally. Locking tests in `tests/test_macro_loading.py` assert that graph.
  Docs in that repo and in `sase-listen` `docs/sase-integration.md` (`ec1196b`) match
  it. No commit on `sase-research-artifacts` is newer than `867222d`.
- **sase-1g6.3 is on origin.** bob-cli `99293a5` renders a `<div class="listen">` as a
  dependency-free LaTeX callout (`\BobListenCard`, `\BobListenPlay`) and, when
  `bound_audio_path` finds audio, passes `--metadata bob-listen-uri=...`. The URI
  template is `BOB_HIGHLIGHTS_AUDIO_LINK_TEMPLATE`, then
  `highlights.audio_link_template`, then `obsidian://open?vault={vault}&file={path}`,
  with percent-encoding outside `A-Za-z0-9-._~`. Create prints `audio_link: <uri>` when
  a button is rendered.
- **sase-1g6.4 is on origin and is current `origin/master`.** bob-cli `f7c268d` moves
  same-stem `.mp3`/`.m4a`/`.ogg`/`.opus` with PDF intake, late-pairs xlib audio onto an
  existing library PDF, writes command-managed `audio` frontmatter, inserts
  `![[lib/...]]` once, and makes `doctor` warn on orphan audio.
  `src/native/highlights_ref/audio.rs` on that commit is the scan helper module
  (`AUDIO_COMPANION_EXTENSIONS`, `discover_companion_audio` meaning "file beside this
  PDF"). Keep those scan functions working.
- **sase-1g6.2 did not land.** There is no commit whose message names `sase-1g6.2` on
  any bob-cli branch. `bound_audio_path` in `create.rs` returns a path only when
  `plan.target` or `plan.source` already has a sibling `.mp3`. There is no `--audio` /
  `--no-audio`, no `highlights.audio_library`, and no narration-script hash or
  `audio.episode_id` lookup. The phase bead stays closed. Do not reopen it. This tale is
  the missing commit.
- **Later sase-listen work does not change this contract.** After `ec1196b`, `sase-1g7`
  landed URL fetch (`e085c63`), SSH feed publish (`0e03944`), and article writer
  editions (`9f491ac`). Those commits do not edit `docs/sase-integration.md`.
  Research-swarm audio still narrates a Markdown script and reads
  `sase-listen render --json`. Leave both repos alone.
- **No bob-cli commit since the epic started, other than `99293a5` and `f7c268d`,
  touches highlights.** Nothing else to merge into this feature.
- **Follow-ups are already filed.** Do not create task beads. Do not edit
  `tests/cli/capture/pomodoro_name.rs`. `sase-1ga` tracks the pre-existing clippy deny
  (`|| true` at line 808). `sase-1gb` tracks the one-off
  `completion::zsh_adapter::real_zsh_first_tab_completes` parallel-suite flake.
  `just all` stops on `sase-1ga`. That is not a failure of this tale.

`sase bead epic-symbols sase-1g6` currently lists nothing. `sase bead read sase-1g6`
shows no parent bead.

## Where to work

Open bob-cli with
`sase repo open bob-cli -r "Land create-time companion audio discovery for sase-1g6"`
and use only the path it prints. Read that checkout's `AGENTS.md` before editing.

Before adding flags, read bob-cli CLI rules:

```bash
sase memory read cli_rules.md -p bob-cli -r "Need CLI flag rules before adding highlights create audio options"
```

The opened checkout may be behind `origin/master`. Fetch if the remote-tracking ref is
stale, then fast-forward. Do not edit until this is true:

```bash
git merge-base --is-ancestor f7c268dc6497ca0e739f908d41e144299507bc05 HEAD
```

`f7c268d` is the phase-4 scan commit and contains `99293a5`. A checkout at `4e510cf`
does not and must be fast-forwarded with `git merge --ff-only origin/master`. Do not
rebase, do not create a branch, and do not implement discovery against the older tree.

## Create-time discovery

Add discovery to the existing `src/native/highlights_ref/audio.rs` module. Do not
replace `discover_companion_audio`, `maybe_insert_audio_embed`, or the scan collectors.
Share `AUDIO_COMPANION_EXTENSIONS`.

Library root, first hit wins:

1. `BOB_HIGHLIGHTS_AUDIO_LIBRARY`
2. config `highlights.audio_library` (parse it beside `pre_scan_hook` and
   `audio_link_template` in `src/native/config/mod.rs`)
3. `$XDG_DATA_HOME/sase-listen/library`
4. `~/.local/share/sase-listen/library`

Discovery order. A miss is not an error.

1. `--audio PATH`. A missing file or an extension outside the allowlist is an error.
   `--no-audio` conflicts with `--audio` and disables discovery.
2. Report frontmatter `audio.episode_id`. Parse YAML the same way the existing
   frontmatter title helper does. Accept only one path component matching
   `[A-Za-z0-9.*-]+`, rejecting `.` and `..`. Read `<library>/<id>/manifest.json` and
   take `audio.file` only when it is a bare filename that exists in that episode
   directory. A frontmatter id that does not resolve prints `warning:` on stderr and
   falls through.
3. Sibling narration script `<md-dir>/<md-stem>_narration.md`, stripping a trailing
   `__final` from the stem. SHA-256 the script bytes and scan
   `<library>/*/manifest.json` for `source.sha256` or `script.sha256` equal to that
   digest. Newest `created_at` wins. Skip unreadable manifests.

Return the source path and a short origin string for the output line (`--audio`,
`episode <id>`, or `narration sha256`).

## Copy, guards, and the Play button

Destination is the PDF target with the source extension lowercased, for example
`xlib/chat/<stem>.mp3`.

Plan the copy before any write:

- An existing destination with identical bytes is reused.
- Different bytes refuse unless `--force`.
- In the intake workflow, an existing `lib/<type>/<stem>.<ext>` refuses, because scan
  would refuse to move over it. Mirror the existing PDF library-destination guard.
- Leave the same-stem `.md` sidecar guard unchanged.

Order in the non-dry-run path: render the temp PDF, copy audio atomically (temp file in
the destination directory, then rename), then `stamp_and_install` the PDF. The PDF
appearing means its audio is already present. If installing the PDF fails, delete an
audio file this run created. Never delete a reused file.

Replace `bound_audio_path`. It must report the library location of the companion this
run discovered or reused, not only a pre-existing sibling `.mp3`:

- Intake: `library_destination` with the discovered extension.
- Library: the destination itself.
- External, empty template, or no bound audio: no Play URI. Keep the existing
  `warning: listen card has no bound companion audio` when the markdown contains
  `class="listen"` and nothing was bound.

`play_uri` already percent-encodes `{vault}` and `{path}` and prints `audio_link:`. Keep
that. Point `{path}` at the vault-relative final library location.

Output, after the existing `pdf:` line: `audio: <dest> (from <origin>)`. Dry-run prints
the planned copy, or `audio: none`, and still writes nothing. Warnings go to stderr as
`warning: …` and never fail the render.

Keep the hook contract. `bob highlights create --include-id <md>` gains no required
flag. Update `create`'s `after_help` so it describes discovery and the copy, not only a
same-stem MP3 that was already beside the PDF.

## Tests and docs

Unit tests: each discovery tier, precedence, episode-id validation, `audio.file`
traversal rejection, newest-manifest tie-break, and library-root precedence.

CLI tests in `tests/cli/highlights/create.rs`, with a fake library under a temp dir via
`BOB_HIGHLIGHTS_AUDIO_LIBRARY`:

- dry-run reports the planned copy
- identical existing companion is reused
- different bytes are refused without `--force`
- library-destination audio is refused
- `--no-audio` skips discovery
- `--audio` error cases (missing file, bad extension)
- a pandoc-gated render copies the audio before the PDF

Extend the landed "Listen cards in PDFs" section of `docs/highlights-ref-sync.md` with
the create half: discovery order, library config, guards, and the backfill note (copy an
MP3 into `xlib/<type>/<stem>.<ext>` for an already-scanned PDF; scan late-pairs it).
State that create now binds audio from the sase-listen library, not only from a file
that was already beside the PDF. Keep the scan section that `f7c268d` added.

Do not add `bob highlights attach-audio`. Do not commit MP3s. Do not change scan's
late-pair or embed-once behavior except by sharing the extension constant.

## Verify

From the bob-cli checkout:

- `cargo fmt --check`
- `git diff --check`
- the new audio unit tests
- the highlights `create` CLI tests, including the pandoc-gated copy test when pandoc is
  installed

`just all` is expected to stop at the pre-existing clippy deny in
`tests/cli/capture/pomodoro_name.rs:808` (`sase-1ga`). Do not change that file to make
`just all` green. Do not run `just check-full` in the sase repo.

## Close sase-1g6 in this same turn

Do this after the code and the bob-cli checks above, in this turn. Do not wait for this
turn's commit SHA, push, or CI. Do not use `--force`.

1. Run `sase bead epic-symbols sase-1g6`. The lander saw no entries. If the
   implementation added any `--epic-symbol` line keyed to `sase-1g6`, resolve each one:
   wire the symbol, make it private, add a non-test pragma, or delete it under the
   Symvision epic-whitelist policy. Re-key a Justfile entry to a different bead only
   when that bead is still open and still needs the exemption. `sase bead close` refuses
   while any entry remains.
2. Close the epic:

```bash
sase bead close sase-1g6 --note "<what you verified>"
```

The note must record all of the following:

- Phase 1 is `867222d` on `sase-research-artifacts` `HEAD`: audio implies the linker,
  audio waits on lead and image only when image is on, the linker waits on audio, the
  listen card and `audio:` frontmatter are written only on success, and TTS failure
  completes the audio agent with `ok=false`.
- Phase 3 is `99293a5`: LaTeX listen-card banner and encoded Obsidian Play URI.
- Phase 4 is `f7c268d`: scan moves and late-pairs companion audio, writes
  command-managed `audio` frontmatter, and embeds the player once.
- Phase 2 had no commit. This tale added create-time discovery, the atomic copy, and the
  Play-button binding. Name the tests that passed.
- sase-listen commits `e085c63`, `0e03944`, and `9f491ac` (`sase-1g7`) do not change the
  narration-script render contract, so they were left in place.
- Follow-ups: filed `sase-1ga` (clippy `|| true` in `pomodoro_name.rs:808`, size small,
  ready) from `sase-1g6.2` / `sase-1g6.3` / `sase-1g6.4`, and `sase-1gb` (zsh
  `real_zsh_first_tab_completes` parallel flake, size large, ready) from `sase-1g6.4`.
  Declined a separate task for the missing phase-2 commit because that gap is this
  epic's work and this tale lands it.

3. Run `just symvision` in the sase checkout. If bob-cli has a `symvision` recipe, run
   that too.
4. Set `status: done` in the frontmatter of the epic plan file.
   `sase bead read sase-1g6 -r "Need the linked plan path"` prints it
   (`plan:202610/research_swarm_listen_card.md`). Edit that file. If it is outside the
   workspace checkout, open its repo with `sase repo open` first.
5. Confirm the epic has no parent:

```bash
sase bead read sase-1g6 -r "Need the parent link"
```

The lander saw no `PARENT` section. If that is still true, stop. Do not close any other
bead. If a parent appears and it is a phase, verify this work completed that phase and
close only that phase. If it is a plan bead, re-check its descendants and its own
epic-symbols before closing it, and stop at the first incomplete parent with a note on
that parent.

The host commits after the turn. Closing the epic in this turn is the landing. Do not
order any step after a wait for this tale's own commit.
