---
tier: epic
title: 'Memory history: a time axis for SASE memory and agent instruction files'
goal: 'Every committed version of every SASE memory note, web, strand, and agent instruction
  file (AGENTS.md plus its provider shims, project and home) can be browsed quickly
  and understood at a glance. The pager is the single place where history is read,
  and it can be reached from the Memory panel, a cross-file changes feed, and `sase
  memory history` (which also has JSON output for agents). Git remains the only store.
  A disposable, incremental metadata index in sase-core provides the speed. Tracking
  gaps and dirty states are always visible, never hidden.

  '
phases:
- id: capture
  title: Tracking guarantees and as-seen evidence capture
  depends_on: []
  size: medium
  description: 'capture: make untracked or ignored managed memory and instruction
    files a `sase memory init --check` failure, make publish fail loudly when an intended
    file was not committed, and start recording what each agent saw: the workspace
    HEAD and instruction blob OIDs at launch, and blob OIDs on audited memory reads.'
- id: prose-diff
  title: Prose-aware comparison engine in sase-core
  depends_on: []
  size: medium
  description: 'prose-diff: add a pure sase-core `prose_diff` module and binding.
    Given two Markdown texts, it returns per-line change marks, reflow-insensitive
    word operations, hunks with heading paths, a bidirectional line map, a YAML-aware
    frontmatter delta, word stats, and unified diff text.'
- id: file-history
  title: Generic git file-history index in sase-core
  depends_on: []
  size: medium
  description: 'file-history: add sase-core `file_history`. It runs a bounded, lock-free
    git runner and parses a first-parent `--raw -M` log over explicit pathspecs. It
    builds rename-aware lineage and path aliases, updates incrementally by tip ancestry,
    detects shallow or incomplete history, reads blobs in batches, reports a path''s
    worktree, index, and tracked state, and persists a serializable snapshot.'
- id: memory-history-core
  title: Memory history semantics, cache, and query bindings in sase-core
  depends_on:
  - prose-diff
  - file-history
  size: large
  description: 'memory-history-core: build the semantic layer over file_history. This
    covers subjects (note, web, strand, instructions, asset), per-version shim aliasing
    by blob equality, version classes and summaries, commit-footer provenance, cause
    attribution for instruction renders, changesets and the feed, sparkline volumes,
    the persisted per-scope snapshot, and the GIL-releasing query bindings. It is
    tested against a fixture corpus modelled on real SASE history.'
- id: history-cli
  title: Python history service and the sase memory history CLI
  depends_on:
  - memory-history-core
  size: medium
  description: 'history-cli: add the thin facade and wire types, a scope builder for
    project and chezmoi home repos from the agent-docs inventory and generated-note
    list, a thread-safe history service, and one shared visual-vocabulary table. Ship
    `sase memory history` with colored text and JSON output for subjects, versions,
    diffs, and the feed. Measure real-repo performance.'
- id: pager-axis
  title: Pager time axis and read view
  depends_on:
  - history-cli
  size: large
  description: 'pager-axis: create the temporary beta flag. Add a generic section-history
    provider seam to the pager and the memory provider. Add the history mixin with
    the `(` `)` `{` `}` keys, version-pinned sections and trail entries, scroll anchoring,
    neighbour prefetch, and generation-guarded workers. Add the read-view change gutter,
    the subject chip, and the past accent. Links in a past version resolve at that
    revision. The CLI opens this pager by default on a TTY.'
- id: time-band
  title: Time band chrome, sparkline, and honest states
  depends_on:
  - pager-axis
  size: medium
  description: 'time-band: add the time band. At now it is a one-row life strip; in
    the past it has two rows, a meaning row and a sparkline time row. Its provenance
    items are label targets. Add the instruction-file cause row, alias and diverged
    chips, honest state labels (untracked, ignored, no VCS, shallow, template, indexing,
    unavailable), and the upstream-ahead marker. Degrade by height and width together
    with the trail band. Add visual goldens.'
- id: diff-view
  title: Word-diff view and change navigation
  depends_on:
  - pager-axis
  size: medium
  description: 'diff-view: add the `=` read/diff toggle. The diff view shows inline
    word insertions and struck-through deletions, a frontmatter semantic block, and
    folds of unchanged runs that expand in place from a label. `[` and `]` jump between
    changes in both views. The default view depends on how the user arrived and then
    sticks for the session. `yy` copies a unified diff, and the CLI `-d` opens this
    view. Add visual goldens.'
- id: timeline-picker
  title: Timeline picker with two-point compare
  depends_on:
  - diff-view
  size: medium
  description: 'timeline-picker: add the `@` modal timeline over all versions, including
    worktree and staged rows and a hidden-versions summary row. It has vim-style list
    keys, a `/` filter across section, agent, bead, and words, `⏎` jumps that push
    a trail entry, `=` to compare the row with the open version, and `.` to toggle
    hidden versions. It stays fast on timelines with hundreds of versions. Add visual
    goldens.'
- id: changes-feed
  title: Cross-file memory changes feed
  depends_on:
  - diff-view
  size: medium
  description: 'changes-feed: running `sase memory history` with no selector opens
    a pager feed. It has one section per day. Each changeset lists its authored subjects
    as labels that open subject@version in the diff view. Generated consequences fold
    under their cause, regen-only changesets collapse into an expandable count, and
    home changes interleave with a ⌂ tag. `r` resyncs the feed. Add visual goldens.'
- id: memory-panel
  title: Memory panel entry points and History row
  depends_on:
  - time-band
  - changes-feed
  size: medium
  description: 'memory-panel: add the Memory panel `H` binding, which opens the selected
    note, web, or strand in the pager at now, and the `C` binding, which opens the
    changes feed. Add a History card row with a mini sparkline, loaded off-thread
    after paint. Wire the keymap everywhere the gotchas require, including default_config.yml,
    and add visual goldens.'
- id: launch
  title: Unflag, document, and verify end to end
  depends_on:
  - capture
  - time-band
  - timeline-picker
  - memory-panel
  size: small
  description: 'launch: remove the beta flag by deleting its off branches and closing
    the flag bead. Write the user guide and link it from the memory and pager docs.
    Verify the performance budgets with TUI tracing, review the complete visual golden
    set, and record the listed follow-ups as proposed follow-up notes.'
proposed_by: bbugyi200.apollo.3o
create_time: 2026-09-30 19:09:10
status: wip
bead_id: sase-1dr
---

- **PROMPT:** [prompts/202609/memory_history.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/memory_history.md)
- **BEAD:** [sase-1dr](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1dr/README.md)

# Plan: Memory history, a time axis for SASE memory and agent instruction files

## 1. Summary

SASE memory is the policy layer loaded into every agent turn. A one-word edit to a core
note, or a note promoted from `reference` to `core`, changes how every later agent
behaves. Every one of those edits is already committed to git. What is missing is a way
to read that history that understands SASE's file model:

- notes, webs, and strands
- generated files
- provider shims
- home templates
- renames and deletions

It should answer "when did this rule appear, who added it, and why?" in seconds, and it
should look good.

This epic ships **memory history** in three parts:

1. **Git is the store.** Every committed version is durable and comes from git by blob
   OID. There is no save-level journal, no SQLite, and no dedicated memory repo.
2. **A disposable, incremental metadata index in sase-core provides the speed.**
   - It holds lineage across renames, shim aliasing, classification, word-level
     summaries, provenance from SASE commit footers, and cause attribution.
   - It is built in one first-parent `git log --raw -M` pass over an explicit pathspec,
     updated incrementally by tip ancestry, and persisted as a cache that can never
     disagree with git: any doubt triggers a rebuild.
   - It never stores file bodies.
3. **The pager is where history is read.**
   - A modeless time axis appears on any section that shows a memory or instruction
     file.
   - `(`/`)` step through versions, `=` switches between the read and diff views, and
     `@` opens a timeline picker.
   - A time band above the body carries the story of each version and a sparkline of the
     file's whole life.
   - There are three ways in: the Memory panel (`H` for the selected item, `C` for all
     changes), `sase memory history` (`--format json` for agents), and simply opening
     any memory file in the pager.

The design follows the consolidated research report
`memory_and_instruction_file_history.md` in the research sidecar. Section 2 records
every place this plan deliberately departs from that report or from the original
request.

## 2. Design decisions

| #   | Decision                                                                                                                                                                                                                                                                                              | Why                                                                                                                                                                                                                                                                                                                                                                                          |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | **Git is the only store.** Every committed version is durable. Uncommitted worktree and staged states appear as honest pseudo-versions labelled "not durable until committed". An untracked or ignored managed file is a `sase memory init --check` failure.                                          | Git cannot recover intermediate saves that were overwritten before a commit, and claiming otherwise would make the feature unreliable. A save journal is a different feature with its own retention and privacy problems.                                                                                                                                                                    |
| D2  | **The index is persisted from v1** as a disposable snapshot (`~/.sase/cache/memory_history/…`). _This departs from the report_, which kept the index in process until a measured trigger fired.                                                                                                       | Every `sase memory history` call is a fresh process, and the user asked specifically for very fast navigation. The snapshot keeps warm CLI calls in the tens of milliseconds and makes the Memory panel's History row instant across ACE restarts. The wire is serializable anyway, so persisting adds only an atomic write under a lock. It never stores bodies and is never authoritative. |
| D3  | **History is per subject, not per file.** A subject keeps its identity across renames, the legacy `memory/` → `sase/memory/` migration, and deletion (a tombstone). A provider shim aliases into its `AGENTS.md` **per version, by blob equality**; only diverged shim versions are shown separately. | Since 2026-02-21 every `CLAUDE.md` version is byte-identical to `AGENTS.md`, but the Feb 15–21 hand-written era is real history. A rule based on file names would hide it.                                                                                                                                                                                                                   |
| D4  | **Shim paths are included explicitly in the walk.** _This departs from the report_, which left shims out to save about 150–250 ms.                                                                                                                                                                    | Without the shims in the walk, the diverged era cannot be detected at all. With the snapshot persisted (D2), that cold cost is paid once per checkout.                                                                                                                                                                                                                                       |
| D5  | **Only explicit, enumerated pathspecs, never globs.** Memory roots are passed as directory pathspecs, and instruction files are enumerated from the `sase memory agent-docs` inventory.                                                                                                               | A `**/AGENTS.md` glob turns a 0.3 s pass into 8–10 s.                                                                                                                                                                                                                                                                                                                                        |
| D6  | **The timeline is the first-parent history of the owning checkout's HEAD.** A cheap `⇡N newer on origin/<default>` marker is computed from the local remote-tracking ref and never fetches.                                                                                                           | History then matches exactly what is on disk, and the marker tells you when your checkout is behind.                                                                                                                                                                                                                                                                                         |
| D7  | **sase-core owns the index's semantics and its git IO.** Bounded, non-interactive, `--no-optional-locks` git runners already exist in core (`artifact_file/vcs.rs`). Python supplies the scope inputs (repo roots, the instruction inventory, the generated-note list) and owns presentation.         | This is the rust-core boundary litmus test: a web app or editor would need identical history. Keeping git IO in core avoids moving about 9 MB of blobs across PyO3 on a cold build.                                                                                                                                                                                                          |
| D8  | **The time axis is modeless and uses punctuation keys only**: `( ) = @ [ ] { }`.                                                                                                                                                                                                                      | There is no "history mode" to enter or leave. Letters and digits are the pager's jump-label alphabet, and `(`/`)` already means prev/next version in ACE.                                                                                                                                                                                                                                    |
| D9  | **v1 is read-only.** There is no restore.                                                                                                                                                                                                                                                             | Memory writes are authorized and digest-checked by `memory/mutation.py`, and restoring a generated file makes no sense. Restore is a follow-up.                                                                                                                                                                                                                                              |
| D10 | **Viewing history is never an audited read.**                                                                                                                                                                                                                                                         | `memory_reads.jsonl` stays a record of what agents read. `sase memory read` stays current-only.                                                                                                                                                                                                                                                                                              |
| D11 | **Scope is SASE memory only.** Provider-native memories are out, for example Claude Code's `~/.claude/projects/*/memory/`.                                                                                                                                                                            | SASE neither writes nor versions them.                                                                                                                                                                                                                                                                                                                                                       |
| D12 | **Home instruction history is the chezmoi template source**, shown with a `TEMPLATE` chip. The deployed `~/sase/memory/*` and `~/AGENTS.md` map to their chezmoi source subjects.                                                                                                                     | The deployed home files are not in any repo. Their sources are.                                                                                                                                                                                                                                                                                                                              |
| D13 | **A temporary beta flag, `memory_history`**, hides the partially built TUI surfaces between phases. The `launch` phase removes it.                                                                                                                                                                    | Required by the flags convention, because intermediate phases would otherwise expose a half-finished UI. The text and JSON CLI is complete on its own and is not flagged.                                                                                                                                                                                                                    |
| D14 | **The name is `sase memory history`.**                                                                                                                                                                                                                                                                | `sase memory log` is the read-audit viewer, and `sase file-history` is prompt file-reference recency.                                                                                                                                                                                                                                                                                        |
| D15 | **"As seen by agent" evidence is captured now; its UI is a follow-up.**                                                                                                                                                                                                                               | Evidence that is not captured now can never be reconstructed.                                                                                                                                                                                                                                                                                                                                |

## 3. Concepts

- **Scope.** One owning repository plus the paths that hold memory in it.
  - `project:<name>` is the project checkout, with `sase/memory`, legacy `memory`, and
    every `AGENTS.md` plus its shims.
  - `home` is the chezmoi source repo, with `home/sase/memory`, `home/memory`, and the
    `home/*.md.tmpl` instruction templates.
  - Without chezmoi, home has state `NO VCS` and no history.
- **Subject.** A logical document whose identity survives renames.
  - Ids: `note:<scope>/<name>`, `web:<scope>/<web>` (the descriptor note),
    `strand:<scope>/<web>/<slug>`, `instructions:<scope>/<dir>` (an `AGENTS.md` plus its
    shim aliases), and `asset:<scope>/<path>` (non-Markdown, listed only).
  - Each subject carries `generated` (from `generated_memory_note_relative_paths`, the
    same source of truth that refuses direct edits) and `managed` (a SASE-rendered
    instruction file versus a hand-written subdirectory `AGENTS.md`).
- **Version.** One committed state of a subject. Fields:
  - ordinal (1 is the oldest)
  - commit, committer time (when it landed), and author time when that differs
  - path at that commit, `blob_oid`, and `prev_blob_oid`
  - change kind: `created | edited | moved | deleted`
  - class (§4.10)
  - summary: heading paths touched, `+w/−w`, and frontmatter semantics
  - provenance: SASE footer fields such as agent and bead, the subject line, and the
    conventional-commit type
  - cause (instruction subjects only)
  - sparkline volume
- **Pseudo-versions.** `STAGED` appears only when the index differs from both HEAD and
  the worktree. The live document is **now**. When the worktree differs from HEAD, now
  is labelled `◌ uncommitted`.
- **Changeset.** One commit's versions across all subjects. It is the unit of the feed.
  Generated consequences fold under their authored causes: the generated notes and the
  rendered `AGENTS.md`, `README.md`, and roster lines.

## 4. UX specification

The mockups are illustrative: SHAs, ages, counts, and names are placeholders. Width and
height degradation is specified in §4.4.

### 4.1 Principles

1. **Time is an axis of the document, not a mode.** If a section has history, the time
   verbs appear in the availability-driven footer. If not, they do not appear.
2. **You always know when you are in the past.** Past versions use a dedicated accent on
   the subject chip, the band, and the current sparkline cell. That accent is never
   amber, because amber already means uncommitted or unpublished. `E` always edits
   _now_, and the footer says so.
3. **You keep your place.** Stepping between versions keeps the same passage in view,
   anchored through the line map and then clamped. A search query persists across
   versions.
4. **Small motions leave the trail alone; jumps push onto it** (like vim's jumplist).
   - `(` `)` `{` `}` push nothing.
   - Picker jumps, feed links, band links, and links followed from a past version push a
     trail entry.
   - Every trail entry records its version pin, so Backspace returns to the exact
     moment.
5. **Show meaning, not mechanics.** Show
   "`⇧ promoted reference → core · § Default Keymap Config · +31w −4w · sase-1au.5`",
   not `M sase/memory/gotchas.md`.
6. **Degrade gracefully and fail open.** If history cannot load, the pager shows today's
   document with a one-line notice. A keypress never crashes the pager, and chrome never
   pushes or wraps the body.

### 4.2 Pager keys

All keys are local to the pager and all are punctuation, so the jump-label alphabet is
untouched.

| Key       | Verb                                                                                                                                                                                                                                                |
| --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `(` / `)` | Older / newer version. Hidden versions are skipped. `)` from the newest committed version returns to **now**.                                                                                                                                       |
| `{` / `}` | First version / **back to now**. For a deleted subject, `}` goes to its tombstone.                                                                                                                                                                  |
| `=`       | Switch between the **read** and **diff** views.                                                                                                                                                                                                     |
| `@`       | Open the timeline picker.                                                                                                                                                                                                                           |
| `[` / `]` | Previous / next change, in either view.                                                                                                                                                                                                             |
| existing  | `/` searches, and the query persists across versions. `yy` copies `sha:path` in the past, or a unified diff in the diff view. `E` edits now. `r` refreshes, re-syncs the index, and follows HEAD. Backspace and Tab walk the trail. `?` shows help. |

- **Footer.** It shows only `( ) version · = diff · @ timeline · } now`, and only when
  they apply. The rest are listed under `?`.
- **Help.** `?` gains a "Time" group.
- **Keys pressed while indexing.** A time key pressed while the timeline is still
  loading records the intent, and the step runs as soon as the data arrives (the last
  intent wins). A time key is never silently ignored.

### 4.3 Read view in the past

```text
 ◆ sase/memory/gotchas.md                                  ⟲ PAST v8/9 · 8 days ago · 34% · md
 ⇧ promoted reference → core · § Default Keymap Config · +31w −4w      sase-1au.5 · athena.… · 1a2b3c4
 ▁▁▃▁▂▁▇▁▁▂▅▁▁▃▁▂▁▁▅▁▂▁▃▁▆▂▁▃▁▂▁▅▁█▁▂ → now ◌              Sep 22 2026 14:03 · ⇡2 on origin/master
 ─────────────────────────────────────────────────────────────────────────────────────────────────
   1│ ---
   2▌ type: core
   3│ ---
   5│ **Default Keymap Config**
   6▌ When changing keymaps, leader mode keys, or any configuration values, don't forget to
   7▌ update the keymap configuration in the `src/sase/default_config.yml` file if necessary.
    ╴
 ─────────────────────────────────────────────────────────────────────────────────────────────────
 ( ) version · = diff · @ timeline · } now · E edits now · ? keys · q close
```

- **Subject chip, right side of the subject line.**
  - In the past: `⟲ PAST v8/9 · 8 days ago`, in the past accent.
  - At now: `⟲ 9 versions` (dim), or `◌ uncommitted` in amber when the worktree is
    dirty.
- **Change gutter.** It is drawn in the rail column of the existing gutter.
  - Green `▌` marks a line added in this version.
  - A modified-accent `▌` marks a changed line.
  - A red `╴` tick marks where text was removed.
- **Tombstone.** A deleted subject shows its last content under a muted-red rule:
  `✖ deleted Jul 13 2026 by athena.… · last content shown`.

### 4.4 Time band

The band is a `#pager-time` region between the trail band and the chrome rule. The time
band is the feature's signature visual: it shows whether a note is stable or churning at
a glance.

- **At now: a one-row life strip.** It shows the sparkline, then
  `last changed 3d ago · athena.sase-1bc.12`, then `◌ uncommitted` when the worktree is
  dirty.
- **In the past: two rows.**
  - **The meaning row** shows the class glyph, section path, word delta, and frontmatter
    semantics. On the right it shows provenance: bead, agent, and short SHA. Each of
    these is a **jump-label target**: the bead opens the bead, the agent opens its chat,
    and the commit opens the commit or stitch view when a resolver exists (otherwise it
    is copy-only).
  - **The time row** shows the sparkline and the absolute date and time. It ends with
    `→ now`, `◌` when dirty, and `⇡N on origin/<default>` when the checkout is behind.
- **Sparkline.**
  - Each cell is one version, bucketed when there are more versions than cells.
  - Bar height is the log-scaled number of words changed (`▁▂▃▄▅▆▇█`).
  - Promotions and demotions are tinted with the accent, deletions in the error colour,
    and regenerations are dim.
  - Hidden versions are dim `·`.
  - The current version's cell is drawn in the past accent with reverse video.
- **Instruction subjects.** A **cause row** replaces the meaning row:
  - `⟳ rendered · sources: gotchas.md · dispatch.md`. Each source is a label that opens
    that note at the same commit in the diff view.
  - `⚙ config change · sase/sase.yml`
  - `⚙ regenerated · no source change in this commit (likely a sase upgrade)`
  - `◆ hand-edited` for unmanaged subdirectory `AGENTS.md` files
  - The chip reads `CLAUDE.md ≡ AGENTS.md` for aliased versions, or `⚠ diverged`
    otherwise.
- **Honest states** are shown in the band's own row and never block the body:
  - `UNTRACKED · commit this file to start its history` (amber)
  - `IGNORED` (amber)
  - `NO VCS · home memory is not in git`
  - `SHALLOW · history truncated at <date>`
  - `TEMPLATE · per-host rendering`
  - `indexing…` (dim)
  - `history unavailable: <reason>` (dim; fail open)
- **Degradation.** A single `chrome_row_budget(height, trail_visible, time_state)`
  decides the row counts for the trail band and the time band together.
  - The past band drops the time row below about 30 rows of height, or when the trail
    band is visible below about 40.
  - At 12 rows or fewer, the band folds into the subject chip.
  - As width shrinks, the meaning row sheds the SHA first, then the agent, then the
    bead, then the section path.
  - The time row sheds the absolute date first, then the `⇡N` marker.

### 4.5 Diff view (`=`)

```text
 ◆ sase/memory/lint_and_test.md                          ⟲ PAST v18/18 · diff vs v17 · 61% · md
 ◆ edited · § PNG Snapshot Tests · +1w −1w                                    apollo.… · 63d2bdc
 ─────────────────────────────────────────────────────────────────────────────────────────────────
     ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄  121 unchanged lines  a  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄
 125│ Local `just check-full` runs the update form after the other exhaustive gates and can
 126▌ modify goldens. SASE agent s̶h̶e̶l̶l̶s̶ processes export `CI=true`; that flag alone does not
     ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄  38 unchanged lines  b  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄
```

- **Styling.**
  - Inserted words use the success colour on a subtle tinted background.
  - Deleted words are struck through in dim error.
  - The word diff runs over whole paragraphs, so reflow disappears: `shells → processes`
    reads as one word, not two rewrapped lines.
  - Code fences are compared line by line.
- **Frontmatter changes** render as a semantic block above the body, for example
  `⇧ type: reference → core — now loaded by every agent`.
- **Unchanged runs fold** to 3 lines of context. Each fold is a jump-label target that
  expands in place, so no new key is needed.
- **Base.** By default the diff compares against the parent version. For a dirty now, it
  compares the worktree against HEAD. For a clean now, it shows the latest version's
  change. The picker's `=` sets any other version as the base.
- **Which view opens first.**
  - Arriving from the feed, a cause link, a band source link, or `-d` opens the **diff**
    view.
  - Arriving from a note, the Memory panel, or a plain file opens the **read** view.
  - An explicit `=` is sticky for the rest of the pager session.
- **Side-by-side** is not supported in v1: notes wrap at 88 columns, so two panes need
  about 190.

### 4.6 Timeline picker (`@`)

```text
╭─ gotchas.md · 9 versions · 2 hidden ───────────────────────────────────── / filter ─╮
│ now  ◌  worktree         not durable until committed          +2w                    │
│ v9   ◆  Sep 27   3d      § Default Keymap Config              +31w −4w   sase-1bc.12 │
│ v8   ⇧  Sep 22   8d      promoted reference → core                       sase-1au.5  │
│ v7   ◆  Sep 12   18d     § Default Keymap Config              +6w −6w                │
│ v1   ✚  Apr 12   5mo     created as memory/gotchas.md         412w                   │
│ ·· 2 hidden (↦ moved memory/ → sase/memory/ 100%, ≈ reflow) · . show                 │
│ ⏎ open · = compare with open version · . hidden · / filter · esc close               │
╰──────────────────────────────────────────────────────────────────────────────────────╯
```

- **List keys.** The picker opens in list mode: `j`/`k`/`g`/`G` move, `⏎` opens and
  pushes a trail entry, and `esc` closes.
- **Compare.** `=` compares the highlighted row with the version currently open. This
  gives VS Code Timeline-style two-point comparison without a mark key.
- **Hidden versions.** `.` toggles them.
- **Filter.** `/` starts a filter over section, agent, bead, subject line, and words.
- **Current version.** It is highlighted in the past accent.
- **Size.** Rows render lazily, so `AGENTS.md`'s roughly 260 versions open instantly.

### 4.7 Changes feed

Running `sase memory history` with no selector opens the feed, as does `C` in the Memory
panel.

```text
 ▤ Memory changes · sase + home · last 14 days                23 changesets · 5 regen-only hidden
 ━━ Mon Sep 28 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   11:03  feat(goals): complete G1 acceptance…                   sase-1bu.7 · athena.sase-1bu.7
          ✚ decisions:goal-ledger        new decision · 180w
          ✚ decisions:goals-host-binds   new decision · 150w
          ◆ glossary:artifact            § Definition · +20w −3w
            ⟳ AGENTS.md §3.1 (+2 entries) · README.md · glossary roster
 ━━ Sun Sep 27 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   14:19  feat(tabs): inherit agent tab across launches…     ⌂  sase-1bc.5 · athena.sase-1bc.5
          ◆ dispatch.md                  § Remote dispatch · +44w −10w
   ⋯ 5 regenerated-only changesets hidden  c
```

- **Structure.** The feed is an ordinary pager document with one section per day, so
  `ctrl+n`/`ctrl+p` jump between days and search, labels, and the trail work unchanged.
- **Subject rows.** Each is a label that opens `subject@version` in the diff view and
  pushes a trail entry.
- **Folding.**
  - `⟳` lines fold generated consequences under their cause.
  - Regen-only changesets collapse into one count line, which is a label that expands in
    place.
  - Home changesets interleave, tagged `⌂`.
  - Conventional-commit subjects are secondary context, dimmed for
    `chore: run sase init…` and `chore: initialize sase memory` boilerplate.

### 4.8 Memory panel

- **`H` (history)** opens the selected note, web descriptor, or strand in an in-ACE
  `PagerScreen` at now, in the read view.
- **`C` (changes)** opens the feed for the scope ring: all enabled scopes.
- **History card row.** Note and strand cards get a row in the property grid, loaded
  off-thread after paint through the existing debounced detail path:
  `History   ▁▂▁▅▁▃▇▂  9 versions · changed 3d ago · athena.sase-1bc.12`
  - While loading it shows `…`.
  - It shows amber `untracked` when the file has no history.
  - It is omitted while the flag is off.
- **Existing keys.** `Z` and `o` keep working. Any pager opened on a memory file gets
  the axis automatically.

### 4.9 CLI: `sase memory history [SELECTOR ...]`

```text
◆ gotchas.md · note · project sase · 9 versions (2 hidden) · TRACKED
  v9  ◆  2026-09-27  3d   § Default Keymap Config        +31w −4w   sase-1bc.12  athena.sase-1bc.12  1a2b3c4
  v8  ⇧  2026-09-22  8d   promoted reference → core                 sase-1au.5   athena.sase-1au.5   9f8e7d6
  …
  v1  ✚  2026-04-12  5mo  created as memory/gotchas.md   412w
  ·· 2 hidden (↦ 1 move, ≈ 1 reflow) · -a to show
```

- **Selectors.**
  - The memory selector grammar: `tui.md`, `sase/memory/tui.md`, `glossary`, and
    `glossary:stitch` (with strand keyword and alias lookup).
  - Instruction paths: `AGENTS.md`, `CLAUDE.md`, `src/sase/ace/AGENTS.md`, and
    `~/AGENTS.md`.
  - **Historical names**: `build_and_run.md` resolves to the renamed subject, with a
    notice.
  - Core notes, generated notes, and deleted notes are all accepted.
  - With no selector, the command shows the feed.
- **Options.** They are sorted, and every long option has a short alias (per
  `cli_rules`):
  - `-a/--all`: show hidden versions and regen-only changesets.
  - `-A/--at REV`: one version, given as `v7`, `~2`, a SHA prefix, or a date (the latest
    version at or before it).
  - `-d/--diff`: show the change instead of the body.
  - `-f/--format {json,pager,text}`: defaults to `pager` on a TTY, otherwise `text`.
  - `-l/--limit N`
  - `-p/--project PROJECT`: the same meaning as in `read`/`show`.
  - `-s/--since DATE`
  - `-S/--scope {all,home,project}`
- **Output.**
  - Text is coloured with the shared vocabulary on a TTY and plain when piped. A piped
    word diff uses git's `[-old-]{+new+}` form.
  - JSON emits the Rust wire unchanged: timelines, versions with provenance and
    summaries, bodies with `-A`, structured diff operations with `-d`, and the feed.
    Agents can use it to answer "why is this rule here?".
- **No audit.** The command never writes `memory_reads.jsonl`.

### 4.10 One visual vocabulary everywhere

The CLI, band, picker, feed, and Memory panel all use the same glyphs. They are defined
once in Python (`src/sase/memory/history/vocabulary.py`) and mapped from the core's
class enum.

| Glyph     | Class                                                                   | Default in a per-file timeline                          |
| --------- | ----------------------------------------------------------------------- | ------------------------------------------------------- |
| `✚`       | created (or recreated after a gap)                                      | shown                                                   |
| `◆`       | authored edit                                                           | shown                                                   |
| `⇧` / `⇩` | `type` promoted / demoted                                               | shown, highlighted: this changes what every agent loads |
| `▣`       | frontmatter-only                                                        | shown, dim                                              |
| `⟳`       | regenerated or rendered (generated subject or managed instruction file) | shown for those subjects; folded in the feed            |
| `⚙`      | config- or renderer-driven, or regen-only (instruction subject)         | shown, with its cause                                   |
| `≈`       | reflow or whitespace-only                                               | hidden (a dim dot in the sparkline)                     |
| `↦`       | pure move or rename                                                     | hidden (the path change shows in the band and picker)   |
| `✖`      | deleted                                                                 | shown, as a tombstone                                   |
| `◌`       | uncommitted or staged                                                   | shown when dirty (amber)                                |
| `⇡N`      | N newer versions on origin                                              | marker only                                             |

### 4.11 Colour

- **Palette.** Add a history palette, `history_palette_from_theme(theme)`, next to
  `syntax_palette_from_theme`. It defines:
  - the **past accent**, seeded from a violet hue (around `#9d7cd8`)
  - inserted-word foreground and tint
  - deleted-word style
  - gutter marks
  - the sparkline ramp
  - the tombstone rule
- **Contrast.** Every entry passes through the existing `_ensure_contrast`: at least 4.5
  for text and at least 3.0 for non-text marks.
- **Distinct from amber.** Tests assert that the past accent's hue differs from the
  theme's `warning` hue by at least 60° in every built-in theme. That keeps "past"
  visually distinct from "uncommitted" in both light and dark themes.

## 5. Architecture

### 5.1 Boundary

- **sase-core** (the linked repo; open it with `sase repo open sase-core` and follow its
  `AGENTS.md`):
  - `prose_diff` for comparison
  - `file_history` for generic git lineage and the index
  - `memory_history` for the semantics, the per-scope cache, and the queries
  - bindings in a new `sase_core_py` domain, each releasing the GIL so ACE worker
    threads never stall the event loop
- **sase (Python).** It owns:
  - scope assembly: repo roots, the instruction inventory from
    `src/sase/amd/inventory.py`, shim names from `src/sase/amd/constants.py`, and the
    generated-note list from
    `src/sase/main/init_memory/root_rendering_notes.py:generated_memory_note_relative_paths`
  - the facade and wire (`src/sase/core/memory_history_facade.py` and
    `memory_history_wire.py`)
  - the thread-safe service, the CLI, the vocabulary, text rendering, the pager seam and
    mixin, the chrome renderers, and the Memory panel wiring
- Python never re-implements lineage, classification, or diffing.

### 5.2 Index build

1. **One pass per scope.**

   ```text
   git -c core.quotepath=off -c diff.renames=true --no-optional-locks log --first-parent \
     --raw -z -M --no-abbrev --no-ext-diff --no-textconv --format=<pinned> <tip> -- <explicit pathspecs>
   ```

   The pathspecs are the memory root directories plus every enumerated `AGENTS.md` and
   shim path. There are never globs.

2. **Lineage** comes from the `-M` rename pairs across the whole pathspec, not from
   `--follow`, which costs 0.35–0.73 s and follows only one file. A delete followed by a
   re-add at the same path keeps the same subject and shows the gap.
3. **Blobs for classification** come from batched `git cat-file --batch` calls.
   Comparisons use `prose_diff`.
4. **Changed paths for cause attribution** are needed only for commits that touch an
   instruction file. Get them with whichever measures cheaper on the real repo:
   `--full-diff`, or a batched `git diff-tree --stdin` over just those commits.
5. **Incremental update.**
   - If the key mismatches (repo common dir, pathspec digest, generated, renderer, and
     config digests, or schema and classifier versions), rebuild.
   - Else if the cached tip equals HEAD, the index is fresh.
   - Else if `merge-base --is-ancestor cached_tip HEAD`, fold `cached_tip..HEAD`.
   - Otherwise (a rewrite, force-push, or branch switch), rebuild.
6. **Invariants**, which must be tested:
   - `blob_oid == rev-parse <commit>:<path>` for every version.
   - An incremental update produces exactly the same index as a full rebuild.
   - Lineage agrees with `git log --follow` on the renamed-note fixtures.

### 5.3 Cache

- **Location.** `~/.sase/cache/memory_history/v<schema>/<scope-key-digest>.json`, one
  file per scope and checkout (keyed by the git common dir).
- **Writes.** Atomic: a temporary file plus rename, under a lock. Readers never wait on
  the lock.
- **Contents.** Metadata only, with no bodies. The file is never committed and never
  placed in a project tree.
- **In-process reuse.** An in-process memo keyed by (path, mtime, size) spares ACE from
  reparsing the file on every query.
- **Trust.** If the file is corrupt or unreadable, rebuild silently and never surface an
  error for it.

### 5.4 Reliability rules

These rules are non-negotiable.

1. **No git or file IO on the keystroke or render path.**
   - `(`/`)` step through prefetched in-memory data.
   - Workers use `spawn_pump_free_task` with generation counters, so stale results are
     dropped during fast stepping.
   - UI state is re-captured after every await (the `tui_perf` rules).
2. **Background git never fights agents.**
   - Use `--no-optional-locks` and `GIT_OPTIONAL_LOCKS=0`, `GIT_TERMINAL_PROMPT=0`, argv
     arrays with `--`, NUL-separated output, pinned rename and quoting settings, and
     time and output budgets.
   - Never fetch, clone, or write a commit-graph from a keypress.
3. **Incomplete history is labelled, never silently short.** This covers shallow clones,
   missing objects, truncated budgets, no VCS, untracked files, and ignored files.
4. **Renames are heuristic, so the UI says so.** Show the old path, the new path, and
   the similarity (`↦ renamed 63%`).
5. **Shim aliasing is reversible.** A diverged version is shown with its own exact
   content.
6. **Links from past versions resolve at that past revision.** If a target did not exist
   at that revision, say so. Never fall back silently to today's file.
7. **Fail open to the live document.**

### 5.5 Performance budgets

- **First paint.** Unchanged for every pager document. History loads after paint, and
  meanwhile the band shows `indexing…`.
- **Warm version step.** At most 30 ms from key press to paint (prefetched blob, then
  comparison, then repaint), verified with `SASE_TUI_PERF=1` and `SASE_TUI_TRACE=1`.
- **Index, measured on the sase repo:**
  - cold build with shims and classification: at most 1.5 s, off-thread
  - incremental update over 100 commits: at most 100 ms
  - a warm "fresh" check against the snapshot: at most 30 ms
- **CLI.** `sase memory history <note>` against a fresh snapshot completes in at most
  300 ms end to end, including interpreter start-up overhead that the CLI already pays.

## 6. Phase: Tracking guarantees and as-seen evidence capture (`capture`)

Goal: "all changes are tracked" becomes a checked guarantee, and SASE starts recording
what each agent saw.

- **Trackedness checks.**
  - `sase memory init --check` reports each canonical memory file (`sase/memory/**`, and
    the home equivalents in the chezmoi source) and each managed instruction file or
    shim that is **untracked** or **ignored** in its owning repo. These count as drift
    and fail the check with a clear, actionable message.
  - Use one bounded `git ls-files -z` call plus one `git check-ignore -z --stdin` call
    per repo.
- **Publish guard.**
  - After the project and chezmoi commit sequences in
    `src/sase/main/init_memory_handler.py` (`_deploy_to_project_repo` and
    `_deploy_to_chezmoi`), verify that every intended memory and instruction path is
    tracked and clean.
  - If any is not, fail with a message naming the paths instead of reporting success.
  - `--no-commit` skips this guard.
- **Launch evidence** in `agent_meta.json`. Follow the `capture_sdd_base_sha` precedent
  in `src/sase/axe/run_agent_runner_setup_workspace.py` and its call site in
  `run_agent_runner_launch.py`, and write through the existing atomic meta writer.
  - Add `workspace_head`: the project repo HEAD at launch, from one bounded `rev-parse`
    with `--no-optional-locks`.
  - Add `instruction_snapshot`: a list of
    `{path, repo: project|chezmoi|none, blob_oid, tracked}` for the root instruction
    file the provider loads (its shim or `AGENTS.md`) and the home one.
  - Compute blob OIDs in-process with git's blob hashing (sha1 over `blob <len>\0` +
    bytes, matching the repo's object format), with no subprocess.
  - Store bytes that match no committed blob (per-host home renders, dirty files) once,
    content-addressed, under `~/.sase/instruction_snapshots/<oid>`: written atomically,
    skipped if already present.
  - Capture is **fail-open**. An error logs a warning and never blocks a launch.
- **Read evidence.**
  - `MemoryReadEvent` schema v3 (`src/sase/memory/_read_log_models.py`) adds `blob_oid`
    for single reads and `included_blob_oids` (path → OID) for batch reads.
  - Writers are `cli_read.py` and
    `ace/tui/modals/memory_panel_load.py:record_memory_panel_strand_read`.
  - The parser (`_read_log_events.py`, `_VALID_SCHEMA_VERSIONS`) accepts versions 1–3.
    Readers tolerate a missing field.
  - `sase memory log --json` passes the new fields through.
- **Tests.**
  - untracked and ignored fixtures fail `--check`
  - the publish guard triggers when a path is ignored
  - meta fields present in the launch tests
  - OID equality with `git hash-object`
  - v1, v2, and v3 events round-trip

## 7. Phase: Prose-aware comparison engine in sase-core (`prose-diff`)

This is a pure module, `sase_core::prose_diff`. It is generic and has no memory
knowledge.

- **Dependency.** Add `similar` as a workspace dependency, then regenerate hakari per
  the sase-core `AGENTS.md`.
- **API.**
  `compare_prose(ProseCompareRequestWire { base, target, format: markdown|plain, context_lines })`
  returns a `ProseComparisonWire` with a schema version and these fields:
  - `frontmatter`: a YAML-aware delta, as a list of `{key, before, after, kind}` plus a
    `type_change` (promoted/demoted) convenience field. Invalid YAML falls back to a
    line delta.
  - `line_marks`: for each target line, `unchanged | added | modified`, plus removal
    anchors (after target line _i_, _n_ base lines were removed).
  - `word_ops`: for each target line, spans of `equal | insert`, plus anchored `delete`
    spans carrying the deleted text for inline rendering.
    - Paragraphs, list items, and headings are tokenised with soft line breaks treated
      as whitespace, so reflow produces no word operations.
    - Code fences and tables are compared by line.
  - `hunks`: target line ranges, each with a `section_path` (the heading breadcrumb; for
    pure deletions, the base heading).
  - `line_map`: nearest-line maps in both directions, monotonic, used for scroll
    anchoring.
  - `stats`: `words_added`, `words_removed`, `lines_added`, `lines_removed`,
    `reflow_only`, `whitespace_only`, and `frontmatter_only`.
  - `unified_diff`: git-style text, used for copying.
- **Binding.** Add it to `sase_core_py` following the sase-core recipe. It releases the
  GIL.
- **Tests.**
  - comparing a text with itself produces no operations
  - a reflow-only fixture gives `reflow_only` with zero words changed
  - the `63d2bdceac` sentence (`shells → processes`) gives exactly one word removed and
    one added
  - a `type: reference → core` promotion is detected
  - a heading breadcrumb is attributed correctly
  - line maps are monotonic, checked with proptest if it is already available (otherwise
    with table tests)
  - a 400-line document compares in at most 5 ms (bench or timing test)
- Run `sase tool run check` in sase-core.

## 8. Phase: Generic git file-history index in sase-core (`file-history`)

This module, `sase_core::file_history`, has no memory semantics.

- **Git runner.** Reuse or generalize the bounded runner in `artifact_file/vcs.rs`,
  including its safe revision and path token checks. It must use:
  - argv arrays with `--`
  - `--no-optional-locks` and `GIT_OPTIONAL_LOCKS=0`
  - `GIT_TERMINAL_PROMPT=0`
  - pinned `-c` settings, so user config cannot change the output
  - timeout and output-size budgets that yield a `truncated` flag rather than an error
- **Raw log parser.** Parse the pinned `--raw -z -M` format, next to the `vcs_log`
  parser family:
  - commit, parents, committer time, author time, author, subject, full body
  - raw entries: status (`A/M/D/R<score>/T`), modes, old and new OIDs, old and new paths
- **Lineage fold.**
  - Stable lineage ids, with rename chains taken from R pairs across the whole pathspec.
  - Tombstones for deletions.
  - A re-add at the same path continues the same lineage, with an explicit gap.
  - A path-alias table records every path a lineage ever had.
  - Output is first-parent order, newest first, and never re-sorted by wall-clock time.
- **Incremental update.** Implement it as in §5.2, using `merge-base --is-ancestor`, and
  fall back to a full rebuild.
- **Health.**
  - shallow detection (`rev-parse --is-shallow-repository`, plus the shallow boundary
    date)
  - missing objects
  - budget truncation
- **Blob reads.** Batched `git cat-file --batch`, with an optional LRU cache keyed by
  OID. Content-addressed entries are never stale.
- **Path status.** Report `tracked | untracked | ignored | no_vcs`, the index OID
  (`ls-files -s -z`), the worktree OID (hashed in-process), and the HEAD OID.
- **Serialization.** The index serializes to and from JSON through the wire types, with
  schema versioning and atomic persistence helpers that take a caller-supplied path.
- **Tests.** Build temporary git repos in Rust tests (there is precedent in
  `artifact_ref/repository_resolution.rs` tests). Cover:
  - a renamed file (similarity around 63%), including a rename across directories
  - a delete and recreate
  - first-parent behaviour across merges
  - a shallow clone
  - a truncated budget
  - the three invariants in §5.2
- Run `sase tool run check` in sase-core.

## 9. Phase: Memory history semantics, cache, and query bindings in sase-core (`memory-history-core`)

This module, `sase_core::memory_history`, builds on `file_history` and `prose_diff`.

- **Scope wire.** `MemoryHistoryScopeWire` has these fields:
  - `scope_key` and `scope_kind: project|home`
  - `repo_root`
  - `memory_roots`, for example `["sase/memory", "memory"]` or
    `["home/sase/memory", "home/memory"]`
  - `instruction_files`: a list of `{dir, agents_path, shim_paths, template, managed}`
  - `generated_notes`
  - `renderer_prefixes` (non-empty only for the sase repo itself)
  - `config_paths`
  - `cache_dir`
- **Subjects** (§3).
  - Kinds are derived from path shape at the lineage's latest path: a note, a web
    descriptor (a note with a sibling strand directory), a strand, instructions, or an
    asset.
  - Each subject gets display names and a path-alias lookup.
- **Shim aliasing.** For each commit and directory, a shim version whose blob equals
  that directory's `AGENTS.md` blob aliases into the instructions subject. Any other
  shim version is a `diverged` version: it keeps its own exact timeline, and the subject
  carries a `diverged_count`.
- **Classification** of each version into the §4.10 classes, using `prose_diff` stats
  and the frontmatter delta:
  - `reflow` and pure `moved` are `hidden_by_default`.
  - `regenerated` applies to generated subjects.
  - Instruction subjects get `rendered | config | regen_only | hand_edited`.
  - Init boilerplate commits (`chore: run sase init…`, `chore: initialize sase memory`)
    are flagged for dimming and folding.
- **Summaries** for each version:
  - the heading paths touched (from hunks)
  - `+w/−w`
  - frontmatter phrases, for example "promoted reference → core"
  - the size in words at creation
  - a `volume` for the sparkline
- **Provenance.** Take `parse_commit_footer` fields (agent, bead, and the others it
  exposes), the subject line, and the conventional type from `vcs_log::commit_type`.
- **Cause attribution** for instruction versions.
  - Memory source subjects co-changed in the same commit, as a list of subject/version
    references.
  - Config paths.
  - Renderer prefixes.
  - `regen_only` when none of these apply.
  - Present these as "related source changes", not as proof of causation.
- **Changesets and feed.**
  - Group versions by commit, and fold generated and rendered consequences under their
    authored causes.
  - Flag regen-only changesets.
  - Merge several scopes by committer time.
  - Filter with `since`, `limit`, and `include_hidden`.
- **Cache.** Implement §5.3, keyed as in §5.2.
- **Upstream count.** `sync` also returns `upstream_ahead`: the count of in-scope
  first-parent commits on `origin/<default>` that are not in HEAD. It uses local refs
  only, and is `None` when the ref is absent.
- **Query bindings.** All carry a schema version, release the GIL, and have Python names
  starting with `memory_history_`.

  | Binding                                          | Returns                                                                                                |
  | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
  | `sync(scope)`                                    | Status (`fresh`, `folded`, or `rebuilt`), tip, counts, health warnings, `upstream_ahead`, `elapsed_ms` |
  | `subjects(scope)`                                | The subject list                                                                                       |
  | `resolve(scope, selector_or_path, at_commit?)`   | The subject, and the version as of that commit (used for historical names and links at a revision)     |
  | `timeline(scope, subject, include_hidden)`       | Versions, pseudo-versions (from path status), path status, and aliasing info                           |
  | `version(scope, subject, version, include_body)` | One version, optionally with its body                                                                  |
  | `compare(scope, subject, base, target)`          | A `prose_diff` comparison, with blobs fetched inside core                                              |
  | `feed(scopes, since, limit, include_hidden)`     | The feed                                                                                               |

- **Fixture corpus.** Build it programmatically, modelled on real SASE history:
  - a `build_and_run.md → lint_and_test.md` rename (around R063)
  - the legacy `memory/ → sase/memory/` migration
  - a reflow-only commit
  - a `type` promotion and a demotion
  - a deleted note, and a deleted-then-recreated note
  - a hand-written `CLAUDE.md` era followed by generated identical shims
  - an init-boilerplate commit
  - a regen-only `AGENTS.md`
  - a note edit that co-renders `AGENTS.md`
  - a strand rename inside a web
- **Tests.**
  - Class and summary goldens.
  - Aliasing and divergence.
  - Cause attribution.
  - Feed folding.
  - Cache round trip.
  - Corrupt-cache rebuild.
  - The key-mismatch rebuild.
  - Incremental equals rebuild on the corpus.
- Run `sase tool run check` in sase-core.

## 10. Phase: Python history service and the sase memory history CLI (`history-cli`)

- **Facade and pin.**
  - Add `src/sase/core/memory_history_facade.py` and `memory_history_wire.py` (thin;
    `require_rust_binding`), and update the module table in `docs/rust_backend.md`.
  - Move `sase-core-revision.txt` past the landed sase-core commits with
    `just ratchet-core-revision`, per `docs/rust_backend.md`.
- **Package `src/sase/memory/history/`.**
  - `scopes.py` builds a `MemoryHistoryScopeWire` for:
    - each project, using the existing project-root detection, the canonical and legacy
      memory roots, instruction files from `discover_project_agent_docs` plus
      `PROVIDER_SHIM_FILES`, `managed` detection that uses the renderer's own marker
      (find it in `src/sase/amd`), and generated notes from
      `generated_memory_note_relative_paths(include_project_memory=True)`
    - home, when `get_use_chezmoi()` is set: the `CHEZMOI_HOME` source repo, the
      `home/sase/memory` and `home/memory` roots, and the chezmoi `AGENTS.md.tmpl` and
      shim templates marked `template`. Otherwise home gets a `NO VCS` pseudo-scope.
    - It also maps deployed home paths (`~/sase/memory/*`, `~/AGENTS.md`) to their
      source subjects, for use by the pager provider.
  - `service.py` is a thread-safe wrapper around sync, resolve, timeline, compare, and
    feed. It keeps an in-process memo per scope and never runs on the event loop. The
    CLI, the pager provider, and the Memory panel all use it.
  - `vocabulary.py` defines the glyphs, labels, and style roles for §4.10.
  - `render_text.py` renders timelines, single versions, word diffs, and the feed as
    Rich text for a TTY, and as plain text when piped.
- **Selector resolution.**
  - Classify the selector with the memory grammar (`selector_models.classify_selector`,
    plus strand keyword and alias lookup in `memory/web/lookup.py`), then resolve it
    with `memory_history_resolve`.
  - Accept core notes, generated notes, deleted notes, historical names, instruction
    paths, and `~/AGENTS.md`.
  - A name matching more than one subject is an error that lists the candidates.
  - This must **not** reuse `validate_memory_read_path`, which rejects core notes on
    purpose for audited reads.
- **Command.**
  - Register `history` in `src/sase/main/parser_memory.py` and route it in
    `memory_handler.py`.
  - Implement the options from §4.9, except the `pager` format, which the `pager-axis`
    phase adds. In this phase `-f pager` is rejected and the TTY default is `text`.
  - Write excellent `-h` help with examples, following the `cli_rules` note.
  - Never write a read-audit event.
- **Measurement.** On the sase repo, measure cold, incremental, and warm sync and the
  CLI end-to-end time, and record the numbers in this phase's bead notes. If a §5.5
  budget is missed, fix the cause in core before closing the phase.
- **Tests.**
  - scope builder, using the chezmoi and no-chezmoi fixtures
  - selector resolution, including a historical name and an ambiguous name
  - JSON schema passthrough
  - text rendering goldens with colour off
  - `-A` forms (`v7`, `~2`, SHA, date)
  - no read-audit rows are written

## 11. Phase: Pager time axis and read view (`pager-axis`)

- **Flag.**
  - Create `memory_history` with `sase flag new memory_history` (kind `beta`), following
    the `sase_flags` note. Paste the registry entry it prints.
  - When the flag is **on**:
    - the memory history provider is registered, so any memory or instruction section
      gets the time axis
    - `sase memory history` defaults to `-f pager` on a TTY
    - the Memory panel entry points and History row are enabled (they land in later
      phases)
  - When the flag is **off**:
    - no provider is registered, so the pager behaves exactly as it does today
    - `-f pager` is rejected with a notice, and the TTY default is `text`
    - the Memory panel shows no history keys and no History row
  - Test both states.
- **Generic seam (`src/sase/pager/history/`).**
  - Add a `SectionHistoryProvider` protocol. It covers:
    - timeline loading
    - version bodies
    - comparisons
    - link resolution at a version
    - refresh and re-sync
  - Add the models `VersionPin` (subject, version ordinal, commit, blob OID, view,
    compare base) and `SectionTimeState`.
  - Add a provider registry with
    `history_provider_for_section(section) -> provider | None`, which is called
    off-thread after paint.
  - The pager core must not import memory modules.
- **Memory provider (`src/sase/memory/history/pager_provider.py`).** It maps file
  sections to subjects:
  - project memory and legacy memory
  - `AGENTS.md` and its shims, including subdirectory ones
  - chezmoi sources
  - deployed home files, via the scope mapping
- **`PagerHistoryMixin` (`src/sase/pager/_screen_history.py`).** The mixin is composed
  into `PagerScreen` next to the existing mixins, and the `( ) { }` bindings are added
  to `screen.py`.
  - It keeps a time state for each section identity.
  - A version step swaps in a derived `PagerSection` that carries a new optional
    `version_pin` field. The swap is refresh-style: `_apply_refreshed_document`
    semantics, with no trail push.
  - The swapped section's owner carries the pinned `revision`.
  - Syntax caches are keyed by `(identity, blob_oid)`. Today `_syntax_prepared` is keyed
    by identity alone, so fix that.
  - Scroll anchoring maps the top visible line through `line_map`, then clamps.
  - The search query is re-applied after each swap.
  - Blobs and comparisons for the ±2 neighbouring versions are prefetched.
  - All loads use `spawn_pump_free_task` with a generation counter, and stale results
    are dropped.
  - Time intents are queued while indexing, as in §4.2.
  - `r` re-syncs the index and follows HEAD.
- **Read view.**
  - Change-gutter marks are drawn in the rail column of `src/sase/pager/_gutter.py`
    (`apply_gutter`), using `line_marks` from the parent comparison.
  - A deleted subject shows the tombstone rule.
- **Chrome.**
  - The subject chip states from §4.3 go in `_chrome.py:subject_line`.
  - Footer verbs go in `footer_legend` / `_update_footer`, including the `E edits now`
    hint while in the past.
  - Help gains a "Time" group in `_trail_chrome_help.py`.
  - Add `history_palette_from_theme`, with the §4.11 contrast and hue-distance tests.
- **Trail.**
  - `PagerTrailEntry` (`src/sase/pager/trail.py`) gains `version_pins`.
  - Restoring a trail entry restores the exact version and view.
  - Jumps push a trail entry, and steps do not.
- **Links at a revision.** A label activated in a pinned section that targets a file
  path or memory selector in the same scope goes through the provider first, using
  `memory_history_resolve` with `at_commit`.
  - If the target existed at that commit, it opens pinned at that version and pushes a
    trail entry.
  - If it did not, a one-section notice document opens instead: "`X` did not exist at
    `1a2b3c4` (Sep 22). It was created on Sep 25." It offers labels to open the target
    at its creation or at now.
  - Other kinds, such as beads, agents, URLs, and plans, resolve normally.
- **Copy and edit.** `yy` in the past copies `<sha>:<repo-relative path>`. `E` always
  edits the live file.
- **CLI.** Add `-f pager`, the TTY default while the flag is on. It opens the subject at
  now, or at the `-A` version. A deleted subject opens at its tombstone.
- **Tests.**
  - Mixin state-machine unit tests: stepping, skipping hidden versions, the `)`-to-now
    boundary, `{`/`}`, intent queuing, and generation drops.
  - Trail-pin round trips.
  - Anchoring.
  - Links at a revision, including the did-not-exist notice.
  - Flag on and off.
  - PNG goldens in `tests/pager/visual/` at 120×40 and 60×30, in dark and light themes:
    the past read view, a dirty now, and a tombstone.
  - Run `just fix-tui-screenshots` for the new goldens, and inspect the report as the
    `lint_and_test` note requires.

## 12. Phase: Time band chrome, sparkline, and honest states (`time-band`)

- **Layout.** Add a `#pager-time` `Static` between `#pager-trail` and
  `#pager-chrome-rule` in `screen.py`, with styles in `_styles.py`.
- **Renderers.** Pure renderers go in `src/sase/pager/_time_band.py`, modelled on
  `_trail_chrome_band.py`:
  - the life strip
  - the past meaning row
  - the time row
  - the instruction-file cause row
  - the honest-state row
  - a `render_sparkline(volumes, classes, current, width)` function
  - `chrome_row_budget(height, trail_visible, time_state)`, which implements the §4.4
    degradation and replaces the trail band's standalone height rule
- **Band labels.** Extend the label layer so band provenance items (bead, agent, SHA)
  and cause sources are label targets. They are assigned before body targets, so their
  letters stay stable. They open through the existing resolvers:
  - beads through `pager/beads.py`
  - agents through the known agent-chat kinds
  - commits through the stitch or commit resolver when one exists, otherwise copy-only
  - cause sources open that note at the same commit, in the diff view once it exists and
    otherwise in the read view
- **Aliases.** Add the `≡ AGENTS.md` and `⚠ diverged` chips. Opening `CLAUDE.md`
  resolves to the instructions subject.
- **Upstream.** Show the `⇡N on origin/<default>` marker from `sync.upstream_ahead`.
- **Tests.**
  - Pure renderer tests at widths from 40 to 200 columns and heights from 10 to 50 rows,
    including the shedding order and every honest state.
  - PNG goldens: the past band, the life strip, the instruction cause band, untracked,
    and a narrow (60×30) layout with the trail band visible.

## 13. Phase: Word-diff view and change navigation (`diff-view`)

- **Toggle.** `=` switches the current section between the read and diff views.
  - The default view depends on how you arrived (§4.5). It is stored per `PagerScreen`
    session, and an explicit toggle makes the choice sticky.
- **Rendering.** Build the diff body from `word_ops`:
  - Inserted words use the success foreground on a tint background, and deleted words
    are struck through in dim error, using the §4.11 palette.
  - The frontmatter semantic block sits above the body.
  - Code fences are compared by line.
  - The search corpus and `styled_search_base` must match the rendered text character
    for character.
- **Folds.**
  - Unchanged runs longer than 2 × `context_lines` + 1 fold into `┄ N unchanged lines ┄`
    rows.
  - Each fold row is a label target of a new _local-action_ target kind that expands in
    place (it recomposes the body) without navigating or pushing onto the trail.
- **Change navigation.** `[` and `]` jump between hunks in the diff view, and between
  gutter-marked changes in the read view. Both wrap with a footer notice.
- **Copy.** `yy` in the diff view copies `unified_diff`.
- **Compare bases.** Support non-parent bases, which the picker will set, and show
  `diff vs vN` in the chip.
- **CLI.** `-d` with the pager format opens the diff view.
- **Tests.**
  - Rendering goldens with colour off.
  - Fold expansion.
  - Hunk navigation.
  - Sticky default.
  - PNG goldens: the diff view (dark and light), a frontmatter promotion block, and a
    long `AGENTS.md` diff with folds.

## 14. Phase: Timeline picker with two-point compare (`timeline-picker`)

- **Modal.** `@` opens `PagerTimelinePicker` (a modal screen) in
  `src/sase/pager/_timeline_picker.py`, with the rows, header, and footer shown in §4.6.
- **Contents.** Pseudo-rows for worktree and staged, a hidden-versions summary row, and
  the current version highlighted in the past accent.
- **Keys.**
  - `j`/`k`/`g`/`G` move.
  - `⏎` opens the row and pushes a trail entry.
  - `=` compares the row against the open version: it sets the compare base and switches
    to the diff view.
  - `.` toggles hidden versions.
  - `/` filters incrementally over section path, agent, bead, subject line, and word
    stats.
  - `esc` closes, and closes the filter first when it is open.
- **Performance.** Build the rows from the already-loaded timeline, with no IO. Render
  lazily or virtualized so that 300 versions open in under one frame budget.
- **Tests.** Key handling, filtering, compare wiring, the trail push, and PNG goldens
  for the picker over a read view and with a filter active.

## 15. Phase: Cross-file memory changes feed (`changes-feed`)

- **Builder.** Add a feed document builder at `src/sase/memory/history/feed_document.py`
  that turns `memory_history_feed` output into a `PagerDocument`.
  - A header gives the summary: scopes, window, counts, and how many changesets are
    hidden.
  - There is one section per day.
  - Changeset rows show time, subject line, `⌂` for home, and provenance labels.
  - Subject rows are labels that open `subject@version` in the diff view and push a
    trail entry.
  - `⟳` lines show folded consequences.
  - Regen-only changesets collapse into a count line that is a local-action label,
    reusing the fold mechanism from `diff-view`.
- **Refresh.** Pass `refresh_document_fn` so that `r` re-syncs every scope.
- **CLI.** `sase memory history` with no selector opens the feed on a TTY while the flag
  is on, and honours `-s`, `-S`, `-a`, `-l`, and `-p`.
  - Text and JSON output are already provided by `history-cli`. Reuse that data path.
- **Tests.** Builder goldens over the fixture scopes (with home interleaved), label
  targets, the regen-only expansion, and PNG goldens for the feed at both sizes.

## 16. Phase: Memory panel entry points and History row (`memory-panel`)

- **Bindings.**
  - `H` (`history`) pushes an in-ACE `PagerScreen` for the selected note, web
    descriptor, or strand at now, following the `_metadata_pager.py` push precedent.
  - `C` (`changes`) pushes the feed document for the enabled scopes.
  - Both are active only while the flag is on.
- **Keymap plumbing.** Update every required place:
  - `MemoryPanelKeymaps` in `src/sase/ace/tui/keymaps/app_keymaps.py`
  - `_MEMORY_BINDING_META` in `keymaps/metadata.py`
  - `ace.keymaps.memory` in `src/sase/default_config.yml` (per the gotchas note)
  - `config/sase.schema.json`
  - the keymap table in `docs/configuration.md`
  - the panel footer (`build_panel_footer`, conditional keys only) and help
- **History row.**
  - Add it to `_build_note_property_grid` in
    `src/sase/ace/tui/modals/memory_panel_rendering.py` for notes and strands, laid out
    as in §4.8.
  - Load it off-thread through the service after paint, through the existing debounced
    detail path.
  - Cache it by (scope, subject, index tip).
  - Show `…` while loading and amber `untracked` when the file has no history.
  - The mini sparkline reuses `render_sparkline`.
- **Scopes.** They come from the panel's scope ring (`MemoryScopeRef.content_root`),
  never from the current working directory.
- **Tests.**
  - Binding and footer tests.
  - A test that the History row loads without blocking.
  - A keymap config and schema test.
  - PNG goldens in `tests/ace/tui/visual/` for a note card with the History row, in dark
    and light themes.

## 17. Phase: Unflag, document, and verify end to end (`launch`)

- **Remove the flag.** Delete every off branch of `memory_history` and make the on
  branches unconditional. Remove the registry entry and close the flag bead in the same
  change, following the `sase_flags` removal rule. Update the tests to single-state.
- **Documentation.**
  - Write `docs/memory_history.md`. It covers:
    - the concepts (subjects, versions, aliasing, and honest states)
    - a band anatomy diagram
    - keys
    - the glyph table
    - the CLI with examples
    - what is and is not tracked (D1, D11, and D12)
    - performance notes
    - optional git maintenance: `git commit-graph write --changed-paths` and Git ≥ 2.51
      speed up cold builds. SASE only suggests this and never runs it automatically.
  - Link it from `docs/memory.md` and from the Keys section of `docs/pager.md`.
- **Performance verification.**
  - With `SASE_TUI_PERF=1` and `SASE_TUI_TRACE=1`, confirm that first paint is unchanged
    and that warm steps take at most 30 ms (p95).
  - Confirm that `~/.sase/logs/tui_stalls.jsonl` records no stalls during rapid `(`/`)`
    stepping on `AGENTS.md`.
  - Record the numbers in the bead notes.
- **Visual review.** Run the full `just fix-tui-screenshots`, inspect every created or
  updated golden in the report, and confirm that no unrelated goldens changed.
- **Follow-ups.** Record the items in §18 as `PROPOSED FOLLOW-UP:` notes on this phase's
  bead.
- Run `sase tool run check`.

## 18. Non-goals and follow-ups

These are out of scope for this epic. The `launch` phase records them.

- **"Memory as seen by agent" UI.** An Agents-tab link that opens `AGENTS.md` at the
  launch snapshot, and `sase memory log` rows linking to the exact version read. This
  uses the evidence captured in `capture`.
- **A review watermark in the feed.** "N new since you last reviewed" for each scope.
- **Restore as an unpublished draft** through `memory/mutation.py`, never for generated
  subjects.
- **An age lens.** A gutter tinted by line age, with labels back to the version that
  introduced each line.
- **A plain git-file history provider** for plans, skills, xprompts, and research docs,
  plugged into the same pager seam.
- **Other surfaces:**
  - `path@rev` refs for `sase pager`
  - side-by-side diffs at 190 columns or more
  - per-host rendered home instruction history
- **Tooling:**
  - doctor or upkeep advice for commit-graph and Git versions
  - a decision record ("memory history is git-derived; the index is a disposable
    cache"), proposed as a memory task bead because memory edits need their own
    authorization
  - `sase_memory_read` skill text that points agents at `sase memory history -f json`
