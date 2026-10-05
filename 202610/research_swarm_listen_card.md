---
tier: epic
title: Listen card for research-swarm reports
goal: 'When `#research_swarm(..., audio=true)` runs, the canonical `<name>.md` the
  linker publishes opens with a quiet listen card, its Highlights PDF carries a one-click
  "▶ Play" button, and its Obsidian reference note embeds a native audio player —
  with no MP3 in the public research repo, and with a failed TTS render never blocking
  publication.

  '
phases:
- id: swarm-listen-card
  title: Swarm topology, audio contract, and linker listen card
  depends_on: []
  size: medium
  description: 'swarm-listen-card: in sase-research-artifacts make audio imply the
    linker, have audio wait on image and the linker wait on audio, give the audio
    agent a complete-on-failure var/artifact contract, teach the linker to write the
    listen card plus `audio:` frontmatter, flip the locking tests, and fix the docs
    drift in sase-research-artifacts and sase-listen.'
- id: bob-audio-companion
  title: bob highlights create discovers and copies companion audio
  depends_on: []
  size: medium
  description: 'bob-audio-companion: add audio discovery (flag, frontmatter episode
    id, narration-script content hash against the sase-listen library), atomic same-stem
    MP3 copy beside the intake PDF, collision guards, config, output, tests, and docs
    to `bob highlights create`.'
- id: bob-listen-banner
  title: Listen-card banner and Play button in the Highlights PDF
  depends_on:
  - bob-audio-companion
  size: small
  description: 'bob-listen-banner: render `listen` Divs as a dependency-free LaTeX
    callout and, when create bound a companion, prepend a "▶ Play" button linking
    to a configurable `obsidian://open` URI for the vault copy.'
- id: bob-scan-audio
  title: Scan carries audio into the library and embeds the player
  depends_on:
  - bob-audio-companion
  - bob-listen-banner
  size: medium
  description: 'bob-scan-audio: move same-stem audio with its PDF, late-pair orphan
    audio to an already-scanned PDF, add bob-managed `audio` frontmatter, insert the
    `![[lib/<type>/<stem>.mp3]]` player once below the PDF task, report orphans in
    doctor, and document it.'
proposed_by: bbugyi200.apollo.research.0b.linker.w0
create_time: 2026-10-04 18:40:03
status: done
bead_id: sase-1g6
---

- **PROMPT:** [prompts/202610/research_swarm_listen_card.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/research_swarm_listen_card.md)
- **BEAD:** [sase-1g6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1g6/README.md)

# Plan: Listen card for research-swarm reports

## Context

Bryan wants the narrated audio edition that `#research_swarm`'s audio agent produces to
be one click away while he reads the swarm's final report. Reports are read as
Highlights PDFs (rendered by the `research-highlights` file hook, which runs
`bob highlights create --include-id <abs-path>` on the `ADD` of a published
`<YYYYMM>/<name>/<name>.md` in the research sidecar) and their Obsidian reference notes
(`ref/chat/<stem>.md`, written by `bob highlights scan` on the Mac).

The design follows the research report
`research:202610/research_swarm_audio_listen_card__grk.md` (read it with
`sase artifact read file:explicit:f9b6d916654debc9d3068674 "<why>"`), whose
recommendations Bryan explicitly agreed with:

- `audio=true` implies the linker; the linker waits on the audio agent; only then is the
  one hook-eligible `<name>.md` written (the hook is `ADD`-only and fires once).
- Swarm audio narrates the lead's `<name>__final.md`, waits on the lead and on the image
  agent when `image=true` (so the cover matches the infographic), and never waits on the
  linker.
- The public research repo never gets the MP3, a relative `.mp3` href, a host path, or a
  feed URL. The report gets a listen card (human) plus `audio:` frontmatter (machine).
- A failed TTS render completes the audio agent with `audio.ok=false`; the linker then
  publishes without a card. A _crashed_ audio agent parks the linker exactly like a
  crashed image agent does today (same recovery).
- `bob highlights create` and `scan` grow companion-audio support; the hook command and
  its 120s timeout stay unchanged; no `MODIFY` hook; `audio` stays opt-in.

A sibling report,
`research:202610/research_swarm_listen_link/research_swarm_listen_link.md`, recommended
committing the MP3 to git instead. That conflicts with the listen-card report Bryan
endorsed and is **not** followed; only its verified rendering facts are reused below.

### Design refinements (deliberate deviations from the listen-card report)

Each was verified while planning; call them out in commit messages and docs.

1. **`<div class="listen">`, not `::: listen`.** GitHub (cmark-gfm) has no fenced-div
   syntax and prints `::: listen … :::` literally (verified with `pandoc -f gfm`).
   Pandoc 3.1.3's default Markdown reader (what bob uses) parses an HTML
   `<div class="listen">` as the same native `Div` with class `listen` (`native_divs`),
   while GitHub renders it as an invisible wrapper. Same semantics, clean everywhere.
2. **No `[Play](highlights://<stem>#audio)` in git.** `highlights://<doc>#page=N` is the
   Highlights app's own document URL scheme (seen in its exports), so `#audio` would at
   best reopen the PDF, and GitHub strips the href. The Markdown card is therefore
   purely informational; bob adds a real **▶ Play** button in the PDF banner that
   targets an `obsidian://open?vault=<vault>&file=<lib path>` URI (verified to survive
   as a PDF `/URI`), through a configurable template, and scan puts Obsidian's native
   player on the reference note.
3. **The card sits directly under the research query, above the infographic.** The
   infographic is a full-width figure that LaTeX may float; placing the one-line card
   first keeps Play on the first content page and groups "what was asked" with "listen".
4. **No `audio` marker key.** bob derives a bob-managed `audio` frontmatter field
   (`"[[lib/<type>/<stem>.mp3]]"`) from the companion file's presence instead. One
   writer, works for backfilled audio whose PDF predates this feature, and lets scan
   insert the player exactly once, crash-safely.
5. **Narration fallback uses a content hash.** sase-listen episode ids hash the script's
   _absolute render-time path_, which differs from the hook's checkout, so bob matches
   the sibling `<stem>_narration.md` bytes against manifest `source.sha256` instead of
   recomputing `compute_episode_id`.
6. **Backfill is "drop the MP3 into intake".** `create --force` refuses whenever the
   library PDF already exists (and replacing it would discard annotations), so the
   backfill path is copying the MP3 to `~/bob/xlib/<type>/<stem>.mp3`; scan's late
   pairing attaches it. `bob highlights attach-audio` stays deferred.
7. **No new LaTeX packages.** The banner uses xcolor primitives only (a running `\vrule`
   inside an `\hbox` plus `\colorbox`/`\parbox`), so a missing `framed`/`tcolorbox` can
   never break every PDF render.

### What the reader sees

Markdown (GitHub, sase pager), written by the linker:

```markdown
# A Listen Card for Research Reports

> **Research query:** How should …?

<div class="listen">

♫ **Brief audio edition** · 4 min · 3 chapters ·
[Narration script](research_swarm_listen_card_narration.md)

</div>

![Infographic …](research_swarm_listen_card_infographic.png)

## Bottom line
```

Highlights PDF (bob): a light-blue callout with a 2.5pt left rule, sans-serif small
text: a filled **▶ Play** button (white on the rule color) followed by "**Brief audio
edition** · 4 min · 3 chapters". The leading `♫` is replaced by the button; relative
links inside the card (the narration script) are dropped with their `·` separator
because they are dead in a PDF. With no bound audio the card stays, with `♫` and no
button. This was prototyped with pandoc 3.1.3 + xelatex and renders cleanly.

Obsidian reference note (bob scan):

```markdown
---
…
audio: "[[lib/chat/research_swarm_listen_card.mp3]]"
---

# A Listen Card for Research Reports

- [ ] #task #ref [[lib/chat/research_swarm_listen_card.pdf]] #hide ^ref

![[lib/chat/research_swarm_listen_card.mp3]]

## Highlights

<!-- highlights:begin -->
<!-- highlights:end -->
```

### End-to-end flow

```text
researchers ─► lead (__final.md, registered)
                 ├─ image?  waits lead            → <name>_infographic.png
                 ├─ audio?  waits lead [+ image]  → <name>_narration.md (git),
                 │                                   MP3 in sase-listen library,
                 │                                   `sase var set audio`, artifact audio:<id>
                 └─ linker  waits lead [+ image] [+ audio]
                              → <name>.md with listen card + audio: frontmatter
                                 └─ ADD hook: bob highlights create --include-id
                                      → xlib/chat/<name>.mp3 (copied first)
                                      → xlib/chat/<name>.pdf (banner + ▶ Play)
                                         └─ Mac bob_xlib_pull (rsync whole xlib/)
                                              → bob highlights scan
                                                 → lib/chat/<name>.{mp3,pdf}
                                                 → ref note with native player
```

Execution matrix (default researchers cdx + cld):

| `audio` | `image` | `linker` arg | Agents | Lead writes  | Hook-eligible file                 |
| ------- | ------- | ------------ | ------ | ------------ | ---------------------------------- |
| false   | false   | false        | 3      | `<name>.md`  | lead's `<name>.md`                 |
| false   | false   | true         | 4      | `__final.md` | linker's `<name>.md`               |
| false   | true    | (implied)    | 5      | `__final.md` | linker's `<name>.md` + infographic |
| true    | false   | (implied)    | 5      | `__final.md` | linker's `<name>.md` + listen card |
| true    | true    | (implied)    | 6      | `__final.md` | both companions                    |

## Phase swarm-listen-card: Swarm topology, audio contract, and linker listen card

Repos: `sase-research-artifacts` (open with `sase repo open sase-research-artifacts`;
read its `AGENTS.md`; run its guarded check as `sase tool run check`) and `sase-listen`
(docs only). No sase-core or sase changes.

### `src/sase_research_artifacts/xprompts/research_swarm.md`

- `{%- set run_linker = linker or image or audio -%}`.
- Audio segment header:
  `%wait:research.{@1}.final {% if image %}%wait:research.{@1}.image {% endif %}%q(...)`,
  unchanged `#fork:research.{@1}.final #research/audio(edition={{ audio_edition }})`.
  Never a linker wait.
- Linker segment header gains `{% if audio %}%wait:research.{@1}.audio {% endif %}`
  after the optional image wait. The bare agent name resolves through the session
  container, so a `/sase_monitor` handoff inside the audio agent keeps the linker
  waiting until the follow-up turn settles.
- Both layout trees already list `<name>_narration.md` when `audio`; keep them, and keep
  `provider.py` unchanged (narration scripts are already excluded from the `@research`
  inventory and the Highlights hook, and no MP3 enters the sidecar).
- Linker prompt, only when `audio`, add an "audio agent's narrated edition" input block
  inside `{% raw %}` (deferred expressions must stay outside code spans), rendering:
  - every `agents` entry that carries an `audio` variable, e.g.
    `{% if agents is defined %}{% for key, outputs in agents.items() if outputs.audio is defined %}- agent={{ key }} audio={{ outputs.audio }}{% endfor %}{% endif %}`
    (iterate rather than hard-coding the key: keyed-template agents may be keyed by
    their template base, and the value prints as compact JSON);
  - every `wait.artifacts` entry with `kind == "file"` and a label starting with
    `audio:` (`wait_name`, `label`, `path`, `ref`).
- Linker steps, only when `audio`:
  - Step 3 opening order becomes: frontmatter, `#` title, research query, **listen
    card**, infographic (when image), bottom-line section. Update the step-3 order
    sentence and the step-6 re-check sentence to match.
  - New step-3 bullet **Listen card**:
    - Facts come from the `audio` variable of `research.{@1}.audio` when `ok` is true.
      If the variable is missing but an `audio:<episode_id>` artifact is listed, recover
      the facts with `sase-listen ls <episode_id> --json` (or `uvx sase-listen …`):
      `audio.duration_s`, chapter count, `script.edition`. If `ok` is false or nothing
      is listed, publish with no card and no `audio:` frontmatter, and say so in the
      final response (never write "audio pending").
    - Add this frontmatter mapping (create a frontmatter block if the lead's report has
      none), copying numbers verbatim, never inventing them:

      ```yaml
      audio:
        edition: brief
        duration_s: 250.34
        chapter_count: 3
        episode_id: a-listen-link-for-research-reports-d74298
      ```

      No library path, MP3 path, feed URL, artifact id, or `file:` ref — the research
      repo is public.

    - Insert exactly once, directly below the research-query blockquote and above the
      infographic, with the blank lines shown (they make GitHub and pandoc parse the
      inner Markdown):

      ```markdown
      <div class="listen">

      ♫ **Brief audio edition** · 4 min · 3 chapters · [Narration
      script](<name>_narration.md)

      </div>
      ```

      Rules: edition word capitalized (`Brief` / `Full`); minutes =
      `max(1, round(duration_s / 60))`; `1 chapter` singular; drop the chapters segment
      if the count is unknown; include the script segment only when
      `<name>_narration.md` exists beside the report in the research checkout; nothing
      else inside the div; not a heading; not inside the blockquote; never a link to an
      MP3, library path, feed URL, or `highlights://` URI.

  - Step 4 link validation already requires the narration-script href to resolve.

- Rewrite the `audio`, `linker`, and `audio_edition` input descriptions and the
  top-level `description`: audio implies the linker, which publishes a listen card;
  audio narrates the lead's report after the lead (and after the image agent when
  `image=true`, using the infographic as cover); `audio_edition` alone launches nothing.

### `src/sase_research_artifacts/xprompts/research_audio.md`

- Cover: use `<stem>_infographic.png` (with `__final` stripped) as `cover` when it
  exists beside the report after syncing the research checkout (in a swarm with
  `image=true` this agent starts after the image agent). Never poll or wait for it in
  the prompt; omit `cover` when absent. Replace the "Do not wait for the image or
  linker" sentence accordingly; keep "do not rerender automatically".
- Deliver (whichever turn finishes the render, including a `/sase_monitor` follow-up):
  1. `sase artifact create -p <audio_path> -k file -l "audio:<episode_id>"` (structured
     label so consumers filter exactly; Telegram still `sendAudio`s it from ID3 tags,
     which it reads independently of the label).
  2. `sase var set audio --json --value-file - <<'JSON' … JSON` with
     `{"ok": true, "episode_id", "title", "edition", "duration_s", "chapter_count", "script": "<YYYYMM>/<name>/<name>_narration.md", "audio_path", "published"}`
     taken from `sase-listen render --json` (chapter count = `len(chapters)`).
- Render failure:
  `sase var set audio --json --value '{"ok": false, "error": "<code>: <message>"}'`,
  register no artifact, report the error code and hint, never switch narrators, and
  **complete normally** so the linker publishes without a card.

### Tests (`tests/test_macro_loading.py`)

Flip the tests that lock the old graph and add coverage:

- `test_research_swarm_audio_does_not_imply_linker` → audio implies exactly one linker.
- `test_research_swarm_audio_opt_in_adds_segment_without_linker` → `audio=True` yields 5
  segments including the linker; lead writes `<name>/<name>__final.md`.
- `test_research_swarm_audio_opt_in_waits_only_for_lead` → audio waits on the lead, and
  on the image agent iff `image=True`; never on the linker.
- `test_research_swarm_audio_planner_edges_and_lead_source` → audio waits `[lead]` or
  `[lead, image]`; linker waits `[lead] + [image if image] + [audio]`; drop the "Do not
  wait for the image or linker" assertion in favor of the new cover wording.
- `test_research_swarm_dependency_graph_preserved` and
  `test_research_audio_covers_guide_lint_render_and_delivery`: assert the new linker
  audio wait, the `audio:<episode_id>` label, the `sase var set audio` contract, and the
  complete-on-failure sentence.
- New: linker prompt contains the listen-card instructions, the `<div class="listen">`
  template, and the `agents`/`audio:` deferred block only when `audio`; `audio_edition`
  alone adds no audio or linker segment; `linker=True` or `image=True` without audio
  renders no listen-card text and no audio wait.
- Verify (by reading sase's `output_variable_context.py` / `resolve_resume_agent_name`
  or a render test) how a keyed-template producer's `agents` key is spelled and whether
  a `/sase_monitor` follow-up's `sase var set` reaches the waiting linker; record the
  answer in `docs/macros.md`. The artifact + `sase-listen ls` fallback covers a miss.

### Docs

- `docs/macros.md`: flag table (`audio` implies the linker), segment list items 8/9, the
  execution matrix above, a "Listen card" subsection (card format, frontmatter, failure
  behavior, crash recovery: rerun the named audio agent or kill the parked linker), and
  the order rule.
- `README.md`: the `#research_swarm` bullet's audio sentence.
- `sase-listen` `docs/sase-integration.md`: replace "after the linker when it runs" with
  the real topology (waits on the lead and on the image agent when present; the linker
  waits on audio and publishes the listen card; bob binds the MP3 for Highlights), and
  fix step 1 of the `#research/audio` section (swarm audio uses the lead's `__final.md`;
  an explicit `@research:` ref selects exactly that file).

## Phase bob-audio-companion: bob highlights create discovers and copies companion audio

Repo: `bob-cli` (open with `sase repo open bob-cli`; read its `AGENTS.md` and its
`cli_rules.md` reference memory before adding flags; verify with `just all`).

### New module `src/native/highlights_ref/audio.rs`

- `AUDIO_COMPANION_EXTENSIONS = ["mp3", "m4a", "ogg", "opus"]` (shared with scan).
- Library root: env `BOB_HIGHLIGHTS_AUDIO_LIBRARY` > config `highlights.audio_library`
  (extend `bob_config` highlights parsing next to `pre_scan_hook`) >
  `$XDG_DATA_HOME/sase-listen/library` > `~/.local/share/sase-listen/library`.
- Discovery, first hit wins; a miss is never an error:
  1. `--audio PATH` (explicit; missing file or non-allowlisted extension **is** an
     error). `--no-audio` (conflicts with `--audio`) disables discovery.
  2. Report frontmatter `audio.episode_id` (parse with serde*yaml like
     `frontmatter_title`; validate as one path component
     `[A-Za-z0-9.*-]+`, not `.`/`..`) → `<library>/<id>/manifest.json`→`audio.file`(must be a bare filename that exists). A frontmatter id that does not resolve prints a`warning:`
     and falls through.
  3. Sibling narration script `<md-dir>/<md-stem>_narration.md` (strip a trailing
     `__final` from the stem): sha256 its bytes and scan `<library>/*/manifest.json` for
     `source.sha256` (or `script.sha256`) equal to it; newest `created_at` wins;
     unreadable manifests are skipped.
- Return the source path plus a human origin string for output.

### `create.rs` / `stamp.rs`

- Companion destination: the PDF target with the source's (lowercased) audio extension,
  e.g. `xlib/chat/<stem>.mp3`.
- Guards (planning, before any write): an existing destination with identical bytes is
  reused; different bytes refuse unless `--force`. In the `Intake` workflow an existing
  `lib/<type>/<stem>.<ext>` refuses (scan would refuse to move over it), mirroring the
  PDF library-destination guard. The same-stem `.md` sidecar guard is unchanged.
- Order: render to the temp PDF → copy audio atomically (temp file in the destination
  directory + rename) → `stamp_and_install` the PDF. The PDF appearing therefore implies
  its audio is already present (rsync also sorts `.mp3` before `.pdf`). If installing
  the PDF fails, remove an audio file this run created (never a reused one).
- Output: `audio: <dest> (from <origin>)` after `pdf:`; `--dry-run` prints the planned
  copy (or `audio: none`) and still writes nothing. Warnings go to stderr as
  `warning: …` and never fail the render (file hooks are non-gating, but a PDF without
  audio beats no PDF).
- Update the `create` `after_help` with a short companion-audio paragraph.
- Keep the hook contract: `bob highlights create --include-id <md>` needs no new flags.

### Tests and docs

- Unit: each discovery tier, precedence, episode-id validation, manifest `audio.file`
  traversal rejection, newest-manifest tie-break, library-root precedence.
- CLI (`tests/cli/highlights/create.rs`, fake library under a temp dir via the env var):
  dry-run reports the planned copy; identical existing companion reused; different one
  refused without `--force`; library-destination audio refused; `--no-audio`; `--audio`
  error cases; pandoc-gated real render copies the MP3 before the PDF.
- `docs/highlights-ref-sync.md`: new "Audio companions" section (create half: discovery
  order, library config, guards, backfill note from refinement 6).

## Phase bob-listen-banner: Listen-card banner and Play button in the Highlights PDF

Repo: `bob-cli`. Builds on the companion plan from the previous phase.

- Extend the pandoc Lua filter (keep the inline-code break behavior) with a `Div`
  handler for class `listen` in LaTeX output:
  - Take the card's first `Para`/`Plain`; drop any `Link` whose target is a relative
    path (not `scheme:` and not `#anchor`) together with the preceding `·` separator and
    spaces.
  - When a Play URI is available, replace a leading `♫` (plus its space) with a
    `pandoc.Link` whose content is `RawInline("latex", "\\BobListenPlay{}")` and whose
    target is the URI (let pandoc escape the URL), followed by `\hspace{0.7em}`.
  - Emit
    `RawBlock("latex", "\\BobListenCard{" .. pandoc.write(pandoc.Pandoc({pandoc.Plain(inlines)}), "latex") .. "}")`.
  - Read the URI from metadata key `bob-listen-uri` (two-pass filter:
    `{ {Meta=…}, {Div=…} }`); pass `--metadata bob-listen-uri=<uri>` only when bound.
- Add to the header includes (no new packages):

  ```latex
  \definecolor{BobListenRule}{HTML}{3B6EA8}
  \definecolor{BobListenFill}{HTML}{EEF3FA}
  \newcommand{\BobListenCard}[1]{\par\medskip\noindent\hbox{{\color{BobListenRule}\vrule width 2.5pt}\setlength{\fboxsep}{6pt}\colorbox{BobListenFill}{\parbox{\dimexpr\linewidth-2.5pt-12pt\relax}{\sffamily\small\raggedright #1}}}\par\medskip}
  \newcommand{\BobListenPlay}{\colorbox{BobListenRule}{\textcolor{white}{\textbf{▶\,Play}}}}
  ```

- Play URI: template from env `BOB_HIGHLIGHTS_AUDIO_LINK_TEMPLATE` > config
  `highlights.audio_link_template` > `obsidian://open?vault={vault}&file={path}`.
  `{vault}` = percent-encoded basename of the bob dir; `{path}` = percent-encoded
  vault-relative path of the companion's _final library location_ (`Intake`: the library
  destination with the audio extension; `Library`: the destination itself). Encode
  everything outside `A-Za-z0-9-._~` (so `/` → `%2F`). An empty template, an `External`
  target, or no bound audio renders the card without a button (print a `warning:` when a
  card exists but no audio was bound).
- Print `audio_link: <uri>` in create output when a button was rendered.
- Tests (pandoc-gated like `code_break_filter_splits_long_inline_code_paths`): LaTeX
  output contains `\BobListenCard`, the `\href` to the encoded obsidian URI when bound,
  no `\href` and a kept `♫` when unbound, the relative script link removed; template
  precedence and encoding unit tests; one xelatex-gated render proving the PDF still
  builds with only the existing packages.
- Docs: extend "Audio companions" with the banner, the Play URI template, and the
  `<div class="listen">` authoring contract (any Markdown author can use it).

## Phase bob-scan-audio: Scan carries audio into the library and embeds the player

Repo: `bob-cli`. Serialized after the create phases to avoid edit conflicts in shared
highlights modules and docs.

- Intake (`doctor.rs`): `intake_companion_moves` also moves same-stem audio
  (`AUDIO_COMPANION_EXTENSIONS`) with its PDF. Late pairing: audio files in `xlib/` with
  no same-stem PDF in `xlib/` but an existing `lib/<rel>.pdf` move beside that library
  PDF. Audio with no PDF anywhere stays in `xlib/` (next tick usually brings the PDF).
  Destination collisions join the existing pre-write conflict report. Intake reporting
  counts audio moves.
- Note metadata: `PipelineMetadata` gains `audio: Option<String>` (vault-relative path
  of a same-stem audio file beside the PDF, first by extension order). Render it as the
  command-managed frontmatter line `audio: "[[lib/<type>/<stem>.<ext>]]"` (quoted
  wikilink like `type: "[[ref]]"`), add `audio` to `COMMAND_MANAGED_FIELDS` so it never
  enters the marker projection, hash, or base, and omit it when no companion exists.
- Body:
  - `default_note_body` writes `![[lib/<type>/<stem>.<ext>]]` plus a blank line between
    the PDF task line and `## Highlights` when a companion exists.
  - Existing notes: insert the same embed once, directly below the `^ref` PDF task line
    (fallback: above `## Highlights`, then above the managed begin marker), only when
    the note's current frontmatter lacks `audio`, the new metadata has it, and the body
    does not already contain that embed. The field and the embed land in one note write,
    so the rule is crash-safe; once the field exists the body is never touched again (a
    user who deletes the player keeps it deleted). The managed Highlights region is
    untouched.
  - Dry-run reports the planned embed/frontmatter change like other note changes.
- Doctor: report audio in `xlib/` that has no PDF in `xlib/` or `lib/` as a warning line
  (not a failure).
- Tests (`tests/cli/highlights/scan.rs` and unit tests): PDF + MP3 intake → both moved,
  new note has frontmatter `audio` and the embed in place; PDF already scanned + MP3
  arrives → moved, embed inserted once, frontmatter added, second scan is a no-op;
  user-deleted embed with field present stays deleted; orphan MP3 stays and doctor
  warns; audio destination collision refuses; dry-run writes nothing; marker hash and
  `highlights_marker_fields` ignore `audio`.
- Docs (`docs/highlights-ref-sync.md`): finish "Audio companions" (scan half, late
  pairing, backfill), update "Generated Body Contract" and "Synced Properties" (`audio`
  is command-managed like `ref_type`), and note that the vault already tracks `*.mp3`
  (about 2 MB per brief edition).

## Verification

- `sase-research-artifacts`: `sase tool run check`.
- `bob-cli`: `just all` (fmt, lint, test); pandoc/xelatex-gated tests must run, not
  skip, on apollo where both are installed.
- Smoke (bob phases, never touching `~/bob`): with the binary built from the checkout
  and `--bob-dir` pointing at a temp vault, copy an existing narrated report and its
  `_narration.md` into a temp dir (for example
  `research:202610/research_swarm_listen_link/research_swarm_listen_link.md`, whose
  narration script matches library episode `a-listen-link-for-research-reports-d74298`);
  `bob highlights create --dry-run --include-id` shows `audio:` found via the narration
  hash, and a real render of a copy with a hand-written listen card produces the banner
  and the encoded Play URI.

## Rollout and acceptance (for Bryan after the epic lands)

1. `just install` in bob-cli on apollo (hook host) and on the Mac (scan host).
2. Run `#research_swarm(prompt=…, audio=true)` once with image off (exercises the new
   linker wait) and once with `image=true`; confirm the card, frontmatter, PDF banner,
   and the reference-note player, and click ▶ Play in Highlights on the Mac. If
   Highlights will not hand `obsidian://` to Obsidian, switch
   `highlights.audio_link_template` — no code change.
3. Optional backfill: copy a library MP3 to `~/bob/xlib/chat/<stem>.mp3` for an
   already-read report (first candidate: `commute_audio_from_markdown`); the next scan
   attaches the player.

## Out of scope

`bob highlights attach-audio`, a Highlights-app `#audio` handler, `MODIFY` hooks,
default-on audio, committing MP3s or feed URLs to the research repo, and read-along /
per-section timestamp links.
