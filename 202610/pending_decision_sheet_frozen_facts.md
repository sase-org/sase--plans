---
tier: tale
title:
  Pending plans show the gate's frozen decision facts, not a false "quote not found"
goal:
  A pending plan's Decision Sheet looks the same on every surface (ACE's PLAN lane,
  `sase plan show`, `sase bead read`) as it does in the Plan Review modal. A memory
  decision whose `requested:` quote was verified when the gate was built never shows "⚠
  quote not found · off" or an unchecked box just because the reader runs in a different
  environment.
size: small
proposed_by: bbugyi200.athena.10t
create_time: 2026-10-16 17:19:20
status: wip
---

# Plan: Pending plans render the gate's frozen decision facts

## Context

### The symptom (your suspicion is confirmed)

In the screenshot, the SASE_CONTEXT PLAN lane for the E4 epic
(`plan:202610/e4_readable_agents_md.md`) shows its memory decision `memory_nested_notes`
as `⬜` with `⚠ quote not found · off`. That is wrong. The decision's `requested:` quote
is copied word for word from the RAW PROMPT shown in the same screenshot ("Can you help
me implement the E4 epic for the "Memory-Built Instruction" work …"). The host did
verify it:

- `sase plan propose` runs strict quote checks (`strict_quotes=True` in
  `src/sase/main/plan_propose_handler.py`), so an unverified `default: true` memory
  decision could not have been proposed.
- The gate's frozen request
  (`~/.sase/interaction_requests/epic_plan/8662a065-…/request.json`,
  `payload.decisions`) records `memory_nested_notes` as `provenance: asked`,
  `requested_verified: true`, `effective_default: true`.
- The accepted sibling written at approval (`e4_readable_agents_md.plan-decisions.json`)
  says the same thing.

Only the PLAN lane disagreed. The screenshot was taken at 15:15:00, ten seconds after
the plan was stamped at 15:14:50, so the lane was still showing the sheet it cached
while the plan was pending (gate created 15:13:08). The next enrichment pass would have
switched to the accepted render. The pending render was wrong for the whole review
window.

### Root cause

`load_stamped_decisions()` (`src/sase/sdd/plan_decision_handoff.py`) builds the sheet
for every reader surface: ACE's PLAN lane (through `_cache_associated_plan_sheet` in
`src/sase/ace/tui/models/_agent_associated_plan_summary.py`), `sase plan show`,
`sase bead read`, and the memory-commit guard. It has two branches:

- **Accepted plans** (`decided_by` set) read the frozen sibling. This branch was made
  environment-independent in 6828ed3836.
- **Pending plans** still call `build_definitions(validation, "")`. That rebuilds host
  facts in the reader's process, and the empty `artifacts_dir` falls back to
  `artifacts_dir_from_env()`, which reads `SASE_ARTIFACTS_DIR`. The TUI (and any
  terminal command) has no planner artifacts dir. `human_authored_texts()` therefore
  finds nothing, and `_build_host_facts()` fails closed to
  `provenance: quote_not_found`. The core binding then clamps the effective default to
  `false`, which is where the `⬜` comes from. A reader running inside some _other_
  agent checks the quote against that agent's prompt chain instead, which is just as
  wrong in the other direction.

Reproduced on an unstamped copy of the E4 plan:

| Reader env                                  | `memory_nested_notes`                         |
| ------------------------------------------- | --------------------------------------------- |
| `SASE_ARTIFACTS_DIR` unset (the TUI's case) | `default=False`, `provenance=quote_not_found` |
| `SASE_ARTIFACTS_DIR` = planner's dir        | `default=True`, `provenance=asked`            |

`docs/reference/work-units/plan-format.md` already says "Decision definitions are frozen
when the gate is created". The gate's `payload.decisions` carries those frozen
definitions, and the Plan Review modal and `sase gate show` render from it. The pending
branch of `load_stamped_decisions()` is the one reader that ignores that freeze and
recomputes the facts.

### Fix shape

Freeze the gate's definitions next to the plan when the gate is built, and have pending
readers use that freeze. The frozen sibling (`<plan>.plan-decisions.json`,
`src/sase/sdd/plan_decision_freeze.py`) is exactly the file approval already writes from
the same gate payload. Writing it at gate creation gives pending and accepted readers
one source, and that source matches what the reviewer approves. Re-proposals get a fresh
archive name from `_fresh_proposal_destination`, so a sibling always belongs to one
proposal. Two cases keep their current behavior:

- Approval-time `write_frozen_definitions_if_missing` becomes a no-op on gated plans
  because the content is identical.
- Pending plans with no sibling still take the live `build_definitions` path. These are
  plans that were never gated, or gates created before this change. The live path is
  correct inside the planner's own session (`sase plan validate`/`show` there), and
  authorization paths (gateless `sase plan approve`, `-D`) deliberately keep failing
  closed.

## Implementation

### 1. Gate-time freeze helper (`src/sase/sdd/plan_decision_freeze.py`)

Add `write_gate_frozen_definitions(plan_path, definitions) -> bool` and export it in
`__all__`:

- It returns `False` without writing when `definitions` is empty.
- It returns `False` without writing when the plan's frontmatter already has
  `decided_by`. Accepted freezes stay immutable. Parse with
  `sase.sdd.frontmatter.parse_frontmatter`, and treat a read or parse failure as "do not
  write".
- Otherwise it writes `{"schema": SCHEMA, "definitions": [...]}` with the same JSON
  formatting as `write_frozen_definitions_if_missing`. It **overwrites** any existing
  sibling, because a newer gate for the same unstamped plan supersedes an older freeze.
  The write is atomic: write a temp file in the same directory, then `os.replace`. The
  TUI enrichment worker may read the sibling at the same moment, and a torn read would
  fall back to the false warning.
- It never raises. On `OSError`/`UnicodeError`/`ValueError` it returns `False` and
  removes the temp file.

Update the module docstring. The sibling now holds the gate's frozen host facts from
gate creation onward, and acceptance keeps it.

### 2. Write the freeze when the plan gate is built (`src/sase/plan_gate_turn/create.py`)

In `create_plan_gate_turn`, add one step after
`spec["turn"] = plan_gate_turn_block(tier)` and **before** `create_gate_turn(...)`. Read
`spec.get("payload", {}).get("decisions")`. When it is a non-empty list, call
`write_gate_frozen_definitions(plan_file, decisions)` inside a
`try/except Exception: pass`, because the freeze must never block gate creation. Writing
before creation means an auto-settled gate's stamp finds the sibling already present, so
the accepted sibling is byte-identical to the pending one. Keep `_build_plan_gate_spec`
in `src/sase/plan_gate.py` free of side effects. Add a short comment saying why the
freeze happens here: pending readers render from it, never from their own environment.

### 3. Pending readers prefer the freeze (`src/sase/sdd/plan_decision_handoff.py`)

In `load_stamped_decisions`, change the pending branch (`decided_by is None`) to call
`read_frozen_definitions(path)` first and use the result when it is non-empty. Only when
there is no usable sibling should it fall back to the existing
`build_definitions(validation, "")`. Keep the accepted branch unchanged. Hoist the
`read_frozen_definitions` import so both branches share it. Rewrite the docstring: a
pending gated plan renders the definitions its gate froze, and only a never-gated
pending plan rebuilds host facts live. The value-filling loop already handles both
cases. Pending plans have no `answer`s, so the values come from the frozen
`effective_default`.

No TUI code changes are needed. `_cache_associated_plan_sheet` already keys its cache on
the sibling's signature, so a sibling that appears or changes triggers a reload.

### 4. Docs (`docs/reference/work-units/plan-format.md`)

Read the `docs_writing` reference memory first. After "Decision definitions are frozen
when the gate is created.", add one or two sentences. They should say that SASE writes
the frozen definitions beside the durable plan as `<plan>.plan-decisions.json` and keeps
that file on approval. They should also say that every surface showing a pending or
accepted gated plan renders those frozen facts, so a decision's provenance (for example
"you asked") never depends on the environment of the process reading it.

### 5. Tests

- `tests/test_plan_decisions_handoff_frozen.py`: add a pending-plan fixture (the
  existing tale without `answer`/`decided_by`/`decided_via`). Add these tests, and list
  the new test names in this module's `__all__` and in the re-export block of
  `tests/test_plan_decisions_handoff.py`, matching the existing pattern:
  - `test_pending_frozen_definitions_ignore_reader_env`: write a sibling with
    `provenance: asked`, unset `SASE_ARTIFACTS_DIR`, `chdir("/tmp")`, then
    `load_stamped_decisions`. Assert `decided_by is None`, the row's `value is True`,
    and the memory provenance is `"asked"`. Assert the
    `pending_decisions_text(sheet).plain` contains `you asked` and does **not** contain
    `quote not found`. This is the exact regression from the screenshot.
  - `test_pending_without_sibling_keeps_live_host_facts`: with no sibling and no
    `SASE_ARTIFACTS_DIR`, the row still resolves through the live path
    (`quote_not_found`, value `False`). This pins the deliberate never-gated fallback.
- New tests for `write_gate_frozen_definitions` (in the frozen test module or a new
  `tests/sdd/test_plan_decision_freeze.py`):
  - It writes for an unstamped plan, and `read_frozen_definitions` round-trips the
    result.
  - It overwrites a stale sibling for an unstamped plan.
  - It leaves an existing sibling untouched and returns `False` for a stamped plan
    (`decided_by` present).
  - It returns `False` for empty definitions.
  - It leaves no temp files behind in the plan directory.
- `tests/plan_gate_turn/test_create.py`: extend the existing monkeypatched harness, or
  add a sibling test next to
  `test_create_plan_gate_turn_records_context_before_auto_settlement`. Assert:
  - When the spec payload carries `decisions`, the sibling exists with those definitions
    **by the time the fake `create_gate_turn` runs** (assert inside the fake).
  - A plan without decisions gets no sibling.
  - A failing freeze (monkeypatch the helper to raise) does not stop gate creation.
- `tests/ace/tui/models/test_agent_associated_plan_cache.py`: add a pending-plan case
  next to `test_accepted_sheet_cache_tracks_sibling_lifecycle`. A pending plan with a
  gate-written sibling yields a cached sheet with `decided_by is None` and the memory
  row's provenance `asked`. Writing the sibling after a first enrichment reloads the
  sheet.

## Verification

- Read the `lint_and_test` reference memory and run what it prescribes (`just check` via
  `sase tool run`). Also run the touched test modules directly:
  `tests/test_plan_decisions_handoff_frozen.py`, `tests/test_plan_decisions_handoff.py`,
  `tests/plan_gate_turn/test_create.py`,
  `tests/ace/tui/models/test_agent_associated_plan_cache.py`, and the new freeze tests.
  Also run the existing approval and stamp suites that touch the sibling:
  `tests/test_gate_finish_gaps_routes.py`,
  `tests/test_plan_approval_actions_archive.py`.
- Manual repro: copy the E4 plan to a temp dir, strip its `answer:`/`decided_by`/
  `decided_via` lines, copy its `.plan-decisions.json` beside it under the matching
  stem, and run `load_stamped_decisions(path, tier="epic")` with `SASE_ARTIFACTS_DIR`
  unset. `memory_nested_notes` must come back `value=True`, `provenance=asked`. Without
  the sibling it must still come back `quote_not_found`, which is the never-gated
  fallback.
