---
tier: tale
title: Add an audited `sase bead read` command and surface bead-read reasons
goal:
  Agents read beads with `sase bead read <id>... -r "<why>"`, which prints exactly what
  `sase bead show` prints after recording one reasoned, agent-attributed read per bead.
  The Agents-tab `Beads:` rows and `sase bead touched` show when and why each bead was
  read. Automated tooling stops producing false `viewed` rows.
size: medium
proposed_by: bbugyi200.athena.0on
create_time: 2026-09-21 11:21:02
status: wip
---

# Plan: Audited `sase bead read` with visible read reasons

## Background and findings

- `sase bead show` is the only bead-reading command. Since da766b87a, when an agent
  identity is present it appends one machine-local `viewed` row per shown bead to
  `~/.sase/projects/<key>/bead_views.jsonl` (`src/sase/bead/bead_views.py`, called at
  the end of `handle_bead_show` in `src/sase/bead/cli_query.py`). These rows have no
  reason. The panel and `sase bead touched` fold them in as the weak `viewed` verb.
- **Most `viewed` rows are false positives.** The `0om--code` agent's `viewed ×3` for
  `sase-14j` (screenshot) came from three rows at 14:09:00, 14:09:06, and 14:09:11 UTC.
  Those match three `sase bead show sase-14j` probes run by `just check`. The Justfile
  at 95cb3ab06 had exactly three `--epic-symbol "sase-14j(...)"` entries, and
  `just _lint-symvision` runs symvision with `BD_COMMAND=tools/sase_bead`. symvision
  calls `tools/sase_bead show <epic-id>` once per epic symbol, and that subprocess
  inherits the agent's identity. The `sase-14j.land` agent shows the same pattern: its
  `viewed` rows line up with its symvision runs at 13:15–13:16 UTC. Its real
  `sase bead show sase-14j` at 13:07 UTC was not recorded, because the installed `sase`
  was older.
- Two other automated callers do the same thing inside agent environments.
  `_resolve_bead_issue` in `src/sase/workflows/commit/bead_hooks.py` runs
  `sase bead show <id> --format json`, which explains the single "agent viewed its own
  bead" rows. `show_bead` in `tools/check_feature_flags` also runs `BD_COMMAND show`.
- Audited bead reads are already half-wired. `sase artifact read bead:<id> "<reason>"`
  appends a row to the audited artifact-read log (`artifact_reads.jsonl`,
  `src/sase/artifact_read_log.py`) and queues a `read` link edge. The panel's
  `merge_bead_touch_entries` (`src/sase/ace/tui/bead_touches.py`) and
  `sase bead touched` (`_load_read_touches` in `src/sase/bead/cli_touched.py`) already
  turn `bead:` reads into the `read` verb. However, **both drop the reason**, and no
  agent is told to use that path.
- The bare `↳` under the `sase-14j` row in the screenshot is a rendering bug.
  `append_agent_bead_touch_rows` always calls
  `append_context_reason(text, item.title, ...)`, and that prints a lone glyph when the
  title is empty. This happens for viewed-only and own-only rows.

## Design decisions

1. **Reuse the audited artifact-read log; do not add a new log.** `sase bead read`
   appends one `ArtifactReadEvent` per resolved bead with `ref="bead:<full-id>"` and the
   authored reason. Every consumer (the panel `read` verb, `sase bead touched -v read`,
   link suggestions) already understands that row shape. As a result,
   `sase bead read X -r why` and `sase artifact read bead:X why` produce identical audit
   rows. Always use the resolved full ID (`entry.issue.id`), never a shorthand.
   Canonicalize the ref the same way `sase artifact read` does, e.g.
   `parse_artifact_ref(f"bead:{id}").rendered` as in `src/sase/artifact_ref_entries.py`.
2. **Mirror `sase memory read` / `show` at the CLI level.** `read` accepts exactly the
   `show` options, adds a required `-r/--reason`, and records before printing. A
   non-empty reason is validated up front: a whitespace-only reason fails with
   `sase bead read: ...` on stderr and exit 1, and prints nothing.
3. **Identity follows the artifact-read contract, not the memory-read one.** An
   interactive identity is allowed, because `build_artifact_read_event` uses
   `resolve_audit_identity` and the row always records who read. The `read` graph edge
   (`agent:<name> -read-> bead:<id>` in the read-link outbox) is queued only inside a
   SASE agent run with an identity, exactly as `sase artifact read` does. Outside an
   agent run, print the same one-line "not recorded as a graph edge" note on stderr,
   once per invocation.
4. **Record before printing, and refuse to print on audit failure.** After batch
   resolution and before `page_or_print`, append one audit row per resolved entry. If an
   append raises `ArtifactReadError`, print the error, print no bead output, and exit 1.
   Link-edge queueing happens after the audit and stays best-effort: a failure prints
   `Error: could not record read link: ...` but the beads still print. Unresolved IDs
   keep `show` semantics: stderr errors and exit 1, while resolved beads still print,
   and only resolved beads are audited. Duplicates collapse after resolution, so each
   bead is audited once per invocation.
5. **`sase bead read` writes no `viewed` row**, which avoids a double `read · viewed`
   count. `sase bead show` keeps its current `viewed` recording, since it is still a
   useful "read without a reason" signal.
6. **No artifact-consumption event for bead reads.** Consumption exists for
   artifact-file protection and retention; beads are not stored artifact files.
7. **Automation opts out of the view log.** Add an environment opt-out,
   `SASE_BEAD_SKIP_VIEW_LOG=1`, exposed as a constant in `bead_views.py`. When it is
   set, `record_bead_show_views` writes nothing. Every internal, non-agent- intent
   `sase bead show` caller sets it.
8. **Panel `↳` line prefers the "why".** For a bead with audited reads, the `↳` line
   shows the newest read reason. This matches `Reads:` and `MEMORY` rows, whose `↳` line
   is always the authored reason. Otherwise it falls back to the bead title. When both
   are empty, no `↳` line is emitted. The bead ID and its page hint stay on the row.
9. **No feature flag.** The command is additive, and `sase bead show` is unchanged for
   humans.
10. **Rust-core boundary.** The view/read merge and fold already live in the Python
    facade (`src/sase/core/bead_touch_index_facade.py`) by the sase-14j epic's choice.
    This plan extends that merge with reasons and does not add new core-owned reduction
    logic. `sase bead read` must never be handled by the Rust bead fast path, because
    Rust would not audit it.

## Implementation steps

### 1. Share the read-link recording between `sase artifact read` and `sase bead read`

- Move `_should_record_link` / `_in_agent_run` and `_record_read_link` out of
  `src/sase/artifact_cli/read.py` into a small neutral module, for example
  `src/sase/artifact_read_links.py`, with public names:
  - `should_record_read_link() -> bool`.
  - `record_read_link(target_ref: str, *, reason: str) -> None`. It takes the canonical
    target ref string instead of a `ResolvedArtifactReference` and canonicalizes it with
    `canonicalize_artifact_link_ref`.
  - The `_READ_NOT_RECORDED` note text as a public constant.
- `sase artifact read` calls the moved helpers, and its behavior and output do not
  change. Existing artifact-read tests must pass unchanged.

### 2. Bead-read recording

- Add `src/sase/bead/bead_reads.py`. It should be about as small as `bead_views.py` and
  use its module-docstring style, explaining that reasoned reads go to the audited
  artifact-read log and views go to the machine-local log. It exposes:
  - `bead_read_ref(bead_id) -> str`, which returns the canonical `bead:<full-id>`.
  - One entry point that validates the reason, builds and appends one
    `ArtifactReadEvent` per bead ID (with `recorded_link=should_record_read_link()` and
    `log_path=artifact_read_log_path(event.project)`), and returns the refs. It raises
    `ArtifactReadError` on failure.
  - A best-effort function that queues the read-link rows for those refs.

### 3. CLI surface

- `src/sase/main/parser_bead_queries.py`: factor the `show` arguments into a shared
  helper, such as `_add_bead_view_arguments(parser, *, require_reason=False)`, modeled
  on `_add_memory_view_arguments` in `src/sase/main/parser_memory.py`. Add
  `register_bead_read_parser`, whose required `-r/--reason` gets help text like
  "Non-empty reason for the audited bead read". `-r` is unused by `show`, so there is no
  conflict. Keep options listed alphabetically.
  - Help: "Read one or more beads after recording an audited read".
  - Description: say that the output is identical to `sase bead show` and that the
    command records one audited, agent-attributed read per resolved bead before
    printing. Say that agents consulting beads to do work must use it, while `show` is
    the unaudited human command.
  - Epilog: examples in the style of the `memory read` epilog, including `..` expansion,
    `-f json`, and `-P`.
  - Update `sase bead show`'s description to point agents at `sase bead read`.
- Register it in `src/sase/main/parser_bead.py`, keeping the alphabetical order (between
  `ready` and `ref`).
- Handler: refactor `handle_bead_show` in `src/sase/bead/cli_query.py` so `show` and
  `read` share one body. For example, a private runner can take an optional audited
  reason: `None` records views as today, while a reason records audited reads before
  printing and skips views. Add `handle_bead_read`. `cli_query.py` is about 500 lines;
  if the shared body plus the read glue would push it toward the 700-line toobig
  threshold, put the read handler in a new `src/sase/bead/cli_read.py`.
- Export `handle_bead_read` from `src/sase/bead/cli.py` and
  `src/sase/bead/cli_basic.py`, as `handle_bead_touched` is. Add
  `"read": handle_bead_read` to `_BEAD_HANDLERS` in `src/sase/main/entry.py`, and add
  `read` to the usage string there.
- `src/sase/main/bead_fast_path.py`: add `"read"` next to `{"list", "show"}` in
  `try_handle_bead_fast_path`, so the command always reaches the Python handler.
- Completion: add `(("bead", "read"), "ids")` in `src/sase/completion/kinds.py`.
  Regenerate `tests/completion/snapshots/cli_spec.json` with the repo's snapshot-update
  path, and update any routing and completion tests that enumerate bead subcommands
  (e.g. `tests/test_bead/test_cli_command_routing_acceptance.py`).
- `sase bead onboard` quick start (`src/sase/bead/cli_admin.py`, around line 552): add a
  `sase bead read <id> -r "<why>"` line described as the audited agent read, and label
  the `show` lines as human viewing.

### 4. Stop automated `viewed` noise

- `src/sase/bead/bead_views.py`: add the `SASE_BEAD_SKIP_VIEW_LOG` constant.
  `record_bead_show_views` returns 0 and creates no file when the variable is `"1"`.
  Update the module docstring.
- `tools/sase_bead`: export `SASE_BEAD_SKIP_VIEW_LOG=1` before both the status-only path
  and the final `exec`, with a comment naming the Python constant. This covers symvision
  and `tools/check_feature_flags`, which both use `BD_COMMAND=tools/sase_bead`. Confirm
  that `check_feature_flags`' `_bead_env` copies `os.environ` so the variable survives.
- `src/sase/workflows/commit/bead_hooks.py`: `_resolve_bead_issue` runs its
  `sase bead show` subprocess with that variable set, passing `env` through
  `_run_bead_command`.
- Do not touch existing rows in `bead_views.jsonl`. Historical false positives stay;
  this is a non-goal.

### 5. Surface read reasons (panel and `sase bead touched`)

- `src/sase/core/bead_touch_index_facade.py`:
  - Add a pure helper, `fold_read_reasons(pairs)`, over `(timestamp, reason)` pairs. It
    returns distinct, non-empty, whitespace-trimmed reasons, newest first, with undated
    pairs last and stable ordering.
  - Add `read_reasons: tuple[str, ...] = ()` to `BeadTouch` and `FoldedBeadTouch`. This
    is a Python-only field that wire conversion never sets.
  - `_FoldBucket` collects `(last_at, reason)` for each reason it receives and builds
    `read_reasons` with the helper.
- `src/sase/bead/cli_touched.py`:
  - `_load_read_touches` sets `read_reasons=(event.reason,)` on each synthesized `read`
    row.
  - JSON output adds `"read_reasons": [...]` to each touch.
  - The compact row keeps its current line. When reasons exist, it prints one extra
    indented line, `  ↳ <newest reason>`.
  - Update the `touched` help text in `src/sase/main/parser_bead_touched.py` and the
    module docstring: `read` comes from `sase bead read` / `sase artifact read bead:`
    with reasons, and automation never produces `viewed`.
- `src/sase/ace/tui/bead_touches.py`:
  - `BeadTouchEntry` gains `read_reasons: tuple[str, ...] = ()`.
  - `_BeadBucket` records `(read_display.event.timestamp, read_display.event.reason)`
    for each `bead:` read and builds `read_reasons` with the facade helper. This is a
    pure function and adds no new I/O on the render path.
  - Keep the file under the 700-line toobig threshold. If needed, move the pure merge
    (`_BeadBucket`, `merge_bead_touch_entries`, `_entry_rank`) into a sibling module and
    re-export it.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_bead_touches.py`: the `↳` text is
  `item.read_reasons[0]` when present, otherwise `item.title`. Skip
  `append_context_reason` entirely when that text is empty. Update the docstrings.

### 6. Instruct agents to use `sase bead read`

This step changes agent instructions, including one generated memory note. The user
asked for agents to be instructed to use the new command, and plan approval authorizes
these edits.

- `src/sase/default_config.yml`, built-in xprompts:
  - In the epic land prompt (around lines 1788, 1789, 1809, and 1820), replace each
    `sase bead show` instruction with `sase bead read ... -r "<why>"`.
  - In `bd/work_task` (around line 1876), do the same.
  - In `bd/work_phase_bead`, make "Read its description and design file" name
    `sase bead read {{ bead_id }} -r "<why>"`.
- `src/sase/xprompts/skills/sase_new_task.md` (generated-skill source, around lines 33,
  82, 99, and 100): use `sase bead read <id> -r "<why>"`. Do **not** run
  `sase skill init --force` or deploy to chezmoi from this unlanded tree. Preview with
  `sase skill init --diff` only; deployment happens after landing, per the
  generated-skills memory.
- `src/sase/main/init_memory/templates/memory-sase-beads.template.md`: this is the
  generator template for `sase/memory/sase_beads.md`; never hand-edit the generated
  note.
  - In "Reading And Repairing", add `read` as the audited agent command. Note that
    `show` is the unaudited human command, and that automation run inside agents never
    counts as a view.
  - In "Notes And History", point the note-ordinal guidance ("the ordinal
    `sase bead show` renders" / "re-read `show`") at `sase bead read`.
  - Keep it terse, since memory costs context.
  - Then run `sase memory init` to regenerate `sase/memory/sase_beads.md`, `AGENTS.md`,
    and the provider shims.
- `src/sase/main/plan_explain.py` (around line 72) and the note `--edit` / `--remove`
  help in `src/sase/main/parser_bead_lifecycle.py` (around lines 305 and 315): refer to
  `sase bead read` in place of, or alongside, `sase bead show`.
- Leave human-facing surfaces on `show`: `cli_work_from_plan_render.py`, the sidecar
  README template, and the getting started and blog docs.

### 7. Docs

- `docs/beads.md`:
  - Add a `sase bead read <id> [<id2> ...]` section next to the `sase bead show`
    section. Cover the reason, the audit row, the graph edge, the absence of a `viewed`
    row, and when to use `read` versus `show`.
  - Update the `sase bead touched` section (around line 1980) for read reasons,
    `read_reasons` in JSON, and the automation opt-out.
  - Add `read` to the command overview block near line 61.
- `docs/cli.md`: add a `sase bead read` table row.
- `docs/configuration.md`: add a `#### sase bead read` flag table next to
  `#### sase bead show`.
- If the Agents-tab `Beads:` sub-section is described in `docs/ace.md`, mention that the
  `↳` line shows the newest read reason.

### 8. Tests

Follow the existing test layout (`tests/test_bead/test_bead_views.py`,
`tests/test_bead/test_cli_touched.py`, `tests/test_bead/test_cli_show*.py`,
`tests/ace/tui/widgets/test_agent_bead_touch_rows.py`,
`tests/ace/tui/widgets/test_agent_bead_touches.py`, and
`tests/core/test_bead_touch_index_facade.py`). Redirect SASE home as those tests do, and
never write the real `~/.sase`.

- `sase bead read`:
  - The parser requires `-r`, and output is identical to `show` for full, compact, and
    json formats.
  - Each resolved bead gets exactly one audit row with ref `bead:<full-id>` (shorthand
    input still stores the full ID) and the trimmed reason.
  - `..` expansion audits each expanded bead once.
  - A whitespace-only reason prints nothing and exits 1.
  - An audit-append failure prints no bead output and exits 1.
  - Unresolved IDs print errors, audit only the resolved beads, and exit 1.
  - No `viewed` row is written.
  - A read-link outbox row is queued only when agent-run identity is present, and the
    stderr note appears otherwise.
  - The fast path does not handle `read`.
- `sase artifact read` link-recording behavior is unchanged after the helper move.
- The view log writes nothing when `SASE_BEAD_SKIP_VIEW_LOG=1`, and `tools/sase_bead`
  exports the variable (assert the script text, or run it with a stub `sase` binary).
- `fold_read_reasons`: ordering, dedup, and trimming. The facade fold and
  `sase bead touched` carry `read_reasons`, the JSON includes them, and the compact row
  prints the `↳` reason line.
- Panel:
  - `merge_bead_touch_entries` carries the newest-first reasons.
  - The renderer shows the newest reason over the title, falls back to the title, and
    emits no `↳` for rows without a reason or title. Update
    `test_own_only_bead_renders_without_verbs` and any `↳` count assertions accordingly.

### 9. Verification

- `just install` if the workspace venv is stale, then `just fix`, then
  `sase tool run check`. Fall back to `just check` if `sase tool` is unavailable. Do not
  run `just check-full`.
- Rendered TUI output changes, because own-only and viewed-only rows lose their bare
  `↳`. Run a targeted `just fix-tui-screenshots -- <selectors>` through `/sase_monitor`
  for the SASE-context Beads goldens in
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_sase_context.py`:
  `agents_phase_bead_context_120x40`, `agents_task_bead_notes_120x40`, and
  `agents_phase_bead_and_plan_context_120x40`. Inspect the report and every golden diff
  before finalizing.
- `sase memory init --check` reports no drift after regeneration.
  `sase skill init --diff` shows only the intended `sase_new_task` change.
- Manual smoke test from an agent shell:
  - `sase bead read <some-id> -r "smoke test"` prints the bead.
  - `sase bead touched <self> -v read -j` shows the reason.
  - `bead_views.jsonl` gains no row.
  - Running `tools/sase_bead show <id>` adds no `viewed` row.

## Non-goals

- Deploying generated skills (that happens only after landing), or changing
  `sase artifact read` output or semantics.
- Purging historical `bead_views.jsonl` rows, or moving the view/read merge into
  `sase-core`.
- Requiring an agent identity for `sase bead read`, or deprecating `sase bead show`.
