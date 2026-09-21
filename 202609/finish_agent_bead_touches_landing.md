---
tier: tale
title: Finish landing the agent bead touches epic
goal:
  sase bead touched is on master, one row per bead, and agrees with the Agents-tab Beads
  sub-section. The CLI shares the panel's glyph vocabulary and includes views and
  audited reads. The sase-core pin exposes the touch-index bindings. symvision carries
  no sase-14j debt. The Beads goldens are current, so sase-14j's land agent can close
  it.
size: medium
proposed_by: bbugyi200.athena.sase-14j.land
bead: sase-14j
create_time: 2026-09-21 09:32:06
status: wip
---

- **PARENT:**
  [202609/agent_bead_touches.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_bead_touches.md)
- **BEAD:**
  [sase-14j](https://github.com/sase-org/sase--beads/blob/main/pages/sase-14j/README.md)

# Plan: Finish landing the agent bead touches epic (sase-14j)

## Why this plan exists

Epic `sase-14j` (plan `plan:202609/agent_bead_touches.md`) adds a `Beads:` sub-section
to the Agents-tab `ARTIFACTS` lane. It is backed by a sase-core touch index, a Python
facade, a `sase bead touched` CLI, and a machine-local `sase bead show` view log. Its
land agent found that every phase bead is closed but the epic is not landable. This tale
covers only the remaining work. The epic's own close, symvision confirmation, and
plan-file status update stay with the resumed land agent.

What is wrong on master (`da766b87a` or later):

1. **`sase bead touched` never landed.** Phase `sase-14j.3` was closed, but its host
   commit finalizer died (muse exec exit 143) after the bead-close commit, so the code
   commit was never made. There is no `touched` subcommand. The complete uncommitted
   diff survives as a host artifact:
   `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/20/20260920163305/commit_diffs/002.diff`.
   The diff was written before phases `.5` (panel render) and `.6` (bead views), so it
   does not integrate with them yet.
2. **Master Gate is red on this epic.** `sase-core-revision.txt` pins sase-core
   `1655a1298fc9`, which predates sase-core `9a5c568` (the touch index). The
   `Check pinned core bindings` lint step reports `bead_touch_index_query`,
   `bead_touch_index_refresh`, and `bead_touch_index_status` missing. Three fast-suite
   tests also fail on them: `tests/core/test_bead_touch_index_facade.py` (2) and
   `tests/test_check_sase_core_rs_bindings_tool.py::test_dev_extension_exposes_every_collected_name`.
3. **symvision is red.** `bead_touch_glyph` and `ordered_bead_verb_chips` in
   `src/sase/ace/tui/widgets/prompt_panel/_agent_bead_touches.py` are unused public
   symbols: only that file and tests use them. The Justfile's
   `--epic-symbol "sase-14j(...)"` entries (`BeadTouchIndexStatus`, `BeadTouchRefresh`,
   `query_touches_for_agent`) are keyed to an epic that is about to close.
4. **Three Agents-tab PNG goldens are stale.** `agents_task_bead_notes_120x40`,
   `agents_phase_bead_context_120x40`, and `agents_phase_bead_and_plan_context_120x40`
   have not been regenerated since the `Beads:` sub-section started rendering (commit
   `2b3b37e89`).

## Work

### 1. Recover the `sase bead touched` diff

Apply the artifact diff without the hunks master already carries. The
settlement-predicate privatization and its Justfile entry landed separately in
`d79525c55`.

```bash
git apply --exclude=Justfile \
  --exclude=src/sase/ace/tui/actions/agents/_notification_utils.py \
  --exclude=tests/ace/tui/test_agent_settlement_notification_match.py \
  ~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/20/20260920163305/commit_diffs/002.diff
```

This was verified to apply cleanly on `da766b87a` (one fuzz offset in
`tests/completion/snapshots/cli_spec.json`). It adds `src/sase/bead/cli_touched.py`,
`src/sase/main/parser_bead_touched.py`, and `tests/test_bead/test_cli_touched.py`, and
wires `touched` into `src/sase/bead/cli.py`, `cli_basic.py`, `src/sase/main/entry.py`
(handler map and usage string), `parser_bead.py`, the routing acceptance test, and the
completion snapshot. If the artifact is gone, re-implement the same surface from the
contract below; the epic plan's `bead-cli` section is the original spec.

### 2. Integrate the CLI with what landed after it

The recovered CLI lists raw index rows only. Bring it in line with the epic plan's
contract ("one row per bead", "same glyph vocabulary the panel uses", and
"`sase bead touched <agent>` agrees with the panel row for row" for touched beads):

- **One row per bead.** Index rows are keyed by `(actor, bead)`. One agent can have
  several actor spellings for the same bead: its globalized name, its bare local name,
  and legacy bare rows recovered by `touch_matches_agent`. Today that yields duplicate
  bead rows. Fold the agent's rows per bead id: sum verb counts, take the earliest
  `first_at` and newest `last_at`, and keep the first non-empty title with durable index
  rows first. Put the fold in one pure, tested place in the non-TUI layer, for example
  beside `merge_view_touches` in `src/sase/core/bead_touch_index_facade.py`. The CLI
  must not import `sase.ace.tui`.
- **Views (phase `.6`).** Merge the agent's machine-local `sase bead show` views behind
  the durable rows. Use `read_bead_view_events` and `view_touches_for_agent` from
  `src/sase/bead/bead_views.py`, plus the facade's `merge_view_touches`. `viewed` never
  promotes to `read`. Add `viewed` to the accepted `-v/--verb` values, after the durable
  verbs.
- **Audited reads.** The panel counts audited `sase artifact read bead:<id>` reads as
  `read`. For row-for-row agreement, fold the agent's audited `bead:` reads from
  `sase.artifact_read_log.read_artifact_read_events` in as `read` counts. Attribute them
  to the agent the same way the facade matcher does, canonicalize `bead:<id>` refs to
  bare ids, and accept `read` in `-v/--verb`. The panel additionally shows
  assigned-but-untouched beads with an `own` chip. That mark comes from the Agents-tab
  row's metadata, not from touches. Leave it out of the CLI and state the difference in
  `-h`.
- **Shared glyph vocabulary.** The CLI currently hard-codes its own glyph table and
  precedence, which duplicates `bead_touch_glyph` in the panel. It also lacks `read`
  (`←`) and `viewed` (`◇`) and uses `•` for an empty verb set where the panel uses `◇`.
  Move the verb-to-glyph precedence into one small non-TUI module both surfaces import,
  for example `src/sase/bead/touch_glyphs.py`. Precedence: created ✚ > closed ✓ >
  reopened ↻ > the edited group ✎ (updated, noted, ready, snoozed, dep, linked, ref, +1)
  > read ← > viewed ◇ > removed ⌫, with ◇ for no verbs. Make
  > `_agent_context_common.py`'s `BEAD_*_GLYPH` constants and the panel's glyph
  > selection come from that module. The panel's rendered output and its existing tests
  > must not change.
- **Drop `-a/--all`.** The core index drops email and `validate_agent_name`-rejected
  actors at reduction time. The sase-core test is
  `owner_email_and_invalid_actors_never_become_touchers` in
  `crates/sase_core/src/bead/touch_index.rs`. So `--all` can never add a row, and the
  plan's identity rule says those actors are not shown. Remove the flag,
  `_is_owner_actor`, the `by <actor>` suffix, and their tests. `sase-14j.1` note #1
  proposed exactly this.
- **Help text.** Keep `-h` complete per `sase/memory/cli_rules.md` (read it with
  `/sase_memory_read`): options sorted, short aliases, examples. Also state:
  - `viewed` comes from a machine-local log, so a remote agent's views are not visible
    while its mutations are. The epic plan's `bead-views` section requires this sentence
    in `sase bead touched -h`.
  - `linked` currently never appears. Bead link events are owner-attributed; task
    `sase-159` tracks that.
  - The CLI lists touched beads only; the panel also marks assigned beads `own`.
- **JSON.** Each row is one bead: `bead_id`, `title`, `issue_type`, `status`, `verbs`,
  `first_at`, `last_at`, plus `actors` (sorted distinct recorded actor spellings that
  contributed). Keep the envelope keys `agent`, `generation`, and `total`.
- **Docs.** Add `sase bead touched` to the bead command table in `docs/cli.md` and a
  short usage entry in `docs/beads.md` near the other read-only bead commands. No
  memory-file edits: the plan requires `-h` to be complete enough that `sase_beads.md`
  needs no new prose.

Update `tests/test_bead/test_cli_touched.py` for these behaviors. Cover: per-bead
folding of mixed actor spellings, view rows merged behind durable rows, audited `bead:`
reads counted as `read`, `-v viewed` and `-v read`, the removed `--all`, the JSON row
shape, and the help sentences above. Stub the index, view log, and read log the way the
existing tests stub the facade. Re-run the routing acceptance and completion snapshot
tests after the parser change, and refresh the snapshot through its normal regeneration
path if the option set changed.

### 3. Ratchet the pinned sase-core revision

Run `just ratchet-core-revision` (use `--report-only` first). It moves
`sase-core-revision.txt` to sase-core's current remote HEAD. That HEAD must contain
`9a5c568`; confirm with `git merge-base --is-ancestor 9a5c568 <new-sha>` in a sase-core
checkout opened through `/sase_repo` (`sase repo open sase-core -r '<reason>'`). Do not
touch `pyproject.toml` or `uv.lock`: the published-wheel floor is release workflow, and
`tools/probe_core_floor` is advisory. With the new pin, confirm locally that
`tools/check_sase_core_rs_bindings` and the three failing tests above pass against the
installed extension, running `just install` first if the workspace venv is stale.

### 4. Clear the symvision debt

- After the glyph move, `bead_touch_glyph` either has a real non-test consumer or is
  used only in its own file. `ordered_bead_verb_chips` is used only in its own file.
  Make file-local symbols private (`_`-prefix) and update their tests; do not whitelist
  them. Follow `sase/memory/symvision.md`.
- `query_touches_for_agent` gains a real consumer in the CLI. Remove its `--epic-symbol`
  entry.
- `BeadTouchIndexStatus` and `BeadTouchRefresh` are the public return types of
  `touch_index_status` and `_refresh_touch_index` in
  `src/sase/core/bead_touch_index_facade.py`. Resolve both without `--epic-symbol`. If a
  real non-test consumer should name the type, wire it: for example, annotate the
  `sase doctor` touch-index check in `src/sase/doctor/checks_beads.py` with
  `BeadTouchIndexStatus`, or type the refresh sites' result with `BeadTouchRefresh`.
  Otherwise privatize the type. No `sase-14j` `--epic-symbol` entry may remain in the
  Justfile; `sase bead epic-symbols sase-14j` must print none.

### 5. Regenerate the three stale goldens

Read `sase/memory/tui.md` and `tui_screenshot.md` with `/sase_memory_read` first. Run
the targeted update through `/sase_monitor` with the `TESTING`/`TESTED` pair:

```bash
just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_agents_sase_context.py
```

Then inspect the report and every golden diff. The only intended change is the new
`Beads:` sub-section in those three scenes. Generation is not approval. A top-bar `⚙`
proc chip appearing is known host leakage tracked by `sase-14q`, not an intended change.
If it appears, rerun when no procs are running rather than committing it.

## Verification

- `just fix` (or `just fmt`), then `sase tool run check`. It must be green, including
  symvision and the scoped test lane. Do not run `just check-full`.
- `just _lint-symvision` is clean with no `sase-14j` epic-symbol entries.
- Live smoke on this host: run `sase bead touched <agent>` for an agent that created,
  noted, closed, and audited-read beads, for example the epic launcher
  `bbugyi200.athena.0oa`. Check one row per bead, correct chips and glyphs, titles, and
  relative ages. Check `-j`, `-l 3`, `-v noted`, and `-v viewed`. Compare the rows
  against that agent's `Beads:` sub-section, ignoring `own`-only rows.
- The three goldens are regenerated and inspected, and the visual check passes for
  `test_ace_png_snapshots_agents_sase_context.py`.

## Out of scope

- Closing `sase-14j`, its symvision confirmation, and setting its plan file to
  `status: done`. The resumed land agent does these.
- Owner-attributed bead link events (`sase-159`) and bare local actor names in note/+1
  events (`sase-15a`).
- Non-hermetic visual captures (`sase-14q`).
- Unrelated Master Gate failures: `tests/service/test_service_host_scenarios.py` import
  error, shard-timing drift, `test_usage_header`, `test_notify_rules` help,
  `test_usage_config`, and the contract manifest.
