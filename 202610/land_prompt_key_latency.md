---
tier: tale
size: medium
title:
  Finish landing sase-1ex - cold macro-identity I/O on the cycle keys, dead facade, and
  closeout
goal:
  Warm-snapshot `<ctrl+n>` / `<ctrl+p>` never list project records on the event loop,
  even when the macro project identity registry is cold. The dead publication payload
  facade and retired MRU names are gone. Epic sase-1ex is closed, Symvision is clean,
  and the epic plan is marked done.
proposed_by: bbugyi200.athena.sase-1ex.land
bead: sase-1ex
create_time: 2026-10-03 12:26:42
status: done
---

- **PARENT:**
  [202610/prompt_space_and_project_cycle_latency.md](https://github.com/sase-org/sase--plans/blob/main/202610/prompt_space_and_project_cycle_latency.md)
- **BEAD:**
  [sase-1ex](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ex/README.md)

# Plan: Finish landing epic sase-1ex (prompt `<space>` / `<ctrl+n/p>` latency)

## Context

Epic `sase-1ex` makes the prompt `<space>` key and the `<ctrl+n>` / `<ctrl+p>`
project-cycling keys in-memory operations. Its plan is
`plan:202610/prompt_space_and_project_cycle_latency.md`; the PLAN path printed by
`sase bead read sase-1ex -r "<why>"` is that file. All 12 phases are closed. The land
agent verified the work against the source and the epic's commits and finished follow-up
triage; its outcomes are in the sase-1ex bead notes. Three problems block the close:

1. **Epic invariant broken on a cold identity registry.**
   - The epic's goal is that the cycle keys never list project records on the event
     loop.
   - `tests/ace/tui/test_launchable_mru.py::test_warm_cycle_performs_zero_main_thread_io`
     and `::test_warm_cycle_ctrl_n_performs_zero_main_thread_io` both fail when run
     alone with `list_project_records=1`. They pass only when an earlier test in the
     same process has warmed the process-wide `lru_cache`
     `sase.macro.project_identity._identity_registry`.
   - Main-thread stack:
     - `_handle_vcs_mru_cycle_key` (`src/sase/ace/tui/widgets/_vcs_mru_cycling.py`,
       after the edit, `if self._cursor_may_need_arg_hint():`)
     - → `_refresh_xprompt_arg_hint_from_cursor`
     - → `_detect_xprompt_arg_hint_from_cursor`
     - → `_get_xprompt_arg_assist_entries`
     - → `_xprompt_arg_assist_project_from_text`
       (`src/sase/ace/tui/widgets/_xprompt_arg_hints.py`)
     - → `canonical_macro_project`
     - → `_identity_registry()`
     - → `load_project_display_snapshot()`
     - → `list_project_records(...)`.
   - The `_cursor_may_need_arg_hint` pre-check from sase-1ex.7 does not help. Every
     cycled `#ref` contains `:`, so the check always passes after a cycle.
   - `#gh:widgets` is a real macro invocation with a positional argument, so the
     arg-hint refresh itself must stay.
   - Production hits the cold path in two windows:
     - right after startup, before anything warms the registry off-thread;
     - after every `invalidate_macro_project_identity()`. That runs from
       `invalidate_project_display_snapshot` and the `project_aliases` mutators, which
       TUI project enable, disable, rename, and alias actions trigger.
2. **Dead code landed under an epic commit.**
   - `src/sase/core/publication_payload_facade.py` and
     `tests/test_core_publication_payload.py` were swept into `5c7e7514ae` (sase-1ex.7).
   - The facade requires a `sase_core_rs.plan_publication_payload_batches` binding that
     no sase-core version exposes. Its only consumer is its own test.
   - It duplicates the real, wired `src/sase/core/agent_publication_batches.py`
     (`plan_agent_publication_batches`, used by `src/sase/agents_sync/v2_io.py`).
   - It turns two gates red:
     - Symvision: unused `PublicationPayloadFile` / `plan_publication_payload_batches`;
     - `tests/test_check_sase_core_rs_bindings_tool.py::test_dev_extension_exposes_every_collected_name`.
   - Per `symvision.md`, delete it with its test.
3. **Retired names left by the xprompt→macro rename (epic sase-1eq) in epic-authored
   files.**
   - `src/sase/ace/tui/launchable_mru.py` module docstring: it references
     `load_launchable_vcs_xprompt_mru` and `load_launchable_vcs_xprompt_mru_pairs`,
     which no longer exist. Use `load_launchable_vcs_macro_mru` and
     `load_launchable_vcs_macro_mru_pairs`.
   - `tests/ace/tui/_prompt_key_io_probes.py` module docstring: it references
     `_load_vcs_xprompt_mru` and `_save_vcs_xprompt_mru`. Use `_load_vcs_macro_mru` and
     `_save_vcs_macro_mru`.
   - Two fixtures seed the legacy `vcs_xprompt_mru.json`, which is now read only through
     the legacy fallback:
     - `tests/ace/tui/test_launchable_mru.py::_seed_mru`;
     - `tests/ace/tui/bench_prompt_bar_keys.py::_seed_prompt_key_home`.

     Seed the canonical file instead, using the constant
     `sase.legacy_xprompt_names.VCS_MACRO_MRU_FILENAME`, so the fixtures exercise the
     production path.

Keep these out of scope. They are already filed as tasks:

- sase-1fp: overlay-docked bar for the missed 60 ms `<space>` target;
- sase-1fq: `tui_perf.md` rules;
- sase-1fr: remaining `on_mount` doubling;
- sase-1fn, sase-1fo: unrelated flake and golden drift.

## Guardrails

- **Event-loop safety.** Read the `tui.md` / `tui_perf.md` reference memories with
  `/sase_memory_read`.
  - Key paths do no disk I/O, list no project records, and spawn nothing.
  - Off-loop work uses a pump-free thread task (`spawn_pump_free_task` +
    `asyncio.to_thread`, or `run_worker(thread=True)`). It is single-flight and is
    cancelled at teardown.
- **Parity.** Once identity is warm, arg hints, the assist-catalog project key, and the
  catalog warm target are byte-identical to today. Cycling text, cursor, the ring, and
  MRU behavior do not change.
- **Bare hosts.** A host without the new app hook keeps today's synchronous
  `canonical_macro_project` call. This matches the epic's existing bare-host fallback
  for `peek_launchable_mru_snapshot`.
- **No `sase-core` change** and no keymap change, so `src/sase/default_config.yml` stays
  as it is. Never reintroduce retired `xprompt` spellings in names you add.

## Steps

### 1. Memory-only identity peek and an off-thread warm

In `src/sase/macro/project_identity.py`, add:

- `macro_project_identity_ready() -> bool`. It is memory-only and returns whether the
  identity registry is already built. Prefer an explicit module-level slot or flag that
  `_identity_registry` and `invalidate_macro_project_identity()` maintain over probing
  `lru_cache` internals. Keep `canonical_macro_project` semantics unchanged.
- `warm_macro_project_identity() -> None`, for off-thread callers. It builds the
  registry and never raises.

Make both public only if a non-test `src/` caller uses them; Symvision ignores test
references.

### 2. Never build identity on the prompt keystroke path

In `_xprompt_arg_assist_project_from_text` (`_xprompt_arg_hints.py`), canonicalize both
the VCS-tag branch and the `ctx.project_name` branch like this:

- **Identity ready:** call `canonical_macro_project(...)` as today. It is memory-only
  once the registry is built.
- **Identity cold, and the app exposes the new warm hook:**
  - request the warm;
  - return the global namespace (`None`) for this refresh.

  Do not return the raw ref. `_ensure_prompt_catalog_project` would register a spurious
  catalog project key.

- **Identity cold, and the host has no hook:** keep the synchronous call (bare-host
  fallback).

When identity first becomes ready after a cold fallback was served to an active prompt,
re-resolve the visible prompt surfaces once. Reuse the existing visible-prompt catalog
surface refresh in `src/sase/ace/tui/actions/_startup_prompt_catalog.py`. That way
project-scoped hints and highlighting do not wait for the next keystroke.

### 3. App-owned single-flight warm

Add `request_macro_project_identity_warm()` on `AceApp`. A natural home is the epic's
launchable-MRU mixin, `src/sase/ace/tui/actions/_launchable_mru.py`, or a sibling mixin
registered the same way.

- One warm runs in flight. Requests made while one is in flight coalesce.
- The warm runs `warm_macro_project_identity()` in a pump-free thread task.
- The task is cancelled at teardown.

Also warm identity off-thread wherever the TUI already does off-thread work after a
project mutation, so the cold window stays short:

- the launchable-MRU build worker, which runs at startup, on token drift, and after TUI
  set-current, enable, disable, rename, and alias actions. Warm before its
  inputs-signature fast path, so the skip still warms.

### 4. Tests

- **Make the zero-I/O probes deterministic.**
  - `test_warm_cycle_performs_zero_main_thread_io` and its ctrl+n twin must pass when
    each runs alone. They must also pass with `invalidate_macro_project_identity()`
    called immediately before the probed press.
  - Give the `_SnapshotApp` test host the new hook (recording requests), or move these
    tests onto the real mixin.
  - Do not hide the cold path by pre-warming identity inside the probed tests.
- **Add coverage for each of these:**
  - A cold identity press requests exactly one warm. Repeated cold presses coalesce
    while the warm is in flight.
  - After the warm completes, the next cycle resolves the canonical project. For
    example, an aliased `#gh:` ref resolves to the same assist-catalog key as today.
  - A bare host without the hook still canonicalizes synchronously.
  - Teardown with a warm in flight leaves no unfinished tasks.
- **Keep everything green:**
  - `tests/ace/tui/test_space_prefill.py`;
  - `tests/ace/tui/test_prompt_key_perf_smoke.py`;
  - `tests/ace/tui/widgets/test_cycle_edit_coalesce.py`;
  - `tests/ace/tui/widgets/test_prompt_vcs_mru_cycling.py`;
  - the xprompt arg-hint and completion tests under `tests/ace/tui/widgets/`.
- **Verify isolation explicitly.** Run every test that uses `prompt_key_io_probe`
  (`rg -l prompt_key_io_probe tests`) once per node, alone, and once per file.

### 5. Delete the dead publication facade

- `git rm src/sase/core/publication_payload_facade.py tests/test_core_publication_payload.py`.
- Confirm that `rg -n "publication_payload" --glob '!sase/repos/**'` finds nothing else.
- Confirm that `tests/test_check_sase_core_rs_bindings_tool.py` passes.
- Leave `src/sase/core/agent_publication_batches.py` untouched.

### 6. Retired-name integration

Apply the docstring and fixture edits listed in Context item 3. Then confirm the two
fixture-backed files still pass:

- `tests/ace/tui/test_launchable_mru.py`, including its "build never writes" byte check,
  which now guards the canonical file;
- `pytest -m slow tests/ace/tui/bench_prompt_bar_keys.py`. The bench is optional to
  rerun, but it must import cleanly (`python -c` import or `--collect-only`).

### 7. Verification

- Read `lint_and_test.md` and `symvision.md` with `/sase_memory_read`.
- Run `just fmt`, then `sase tool run check`. Do not run `just check-full`.
- Symvision must no longer report `PublicationPayloadFile` or
  `plan_publication_payload_batches`.
- Treat any other failure as yours unless triage labels it KNOWN or FLAKY. The known
  load flakes are already tracked: sase-1a8, sase-1bl, sase-1bb, sase-1f0, sase-1fn.
- Optionally rerun `pytest -s -m slow tests/ace/tui/bench_prompt_bar_keys.py` and record
  the ctrl+p/ctrl+n rows in the epic note. The ctrl+n/p handler should stay at about 1
  ms.

### 8. Close out epic sase-1ex (final step, in the same turn as the code)

1. Run `sase bead epic-symbols sase-1ex`. When the land agent checked, it reported none.
   For each listed `--epic-symbol` entry, resolve the symbol: wire it up, privatize it,
   add a non-test pragma, or delete it. Re-key a Justfile entry only to a still-open
   bead that needs it.
2. Close the epic:

   ```bash
   sase bead close sase-1ex --note "<verification>"
   ```

   The note must summarize:
   - the land agent's verification: all 12 phases checked against code and commits;
     sase-1ez.6's `_active_prompt_bar` reused rather than duplicated; the GC work reused
     from sase-1ez; the 60 ms `<space>` miss recorded per plan with sase-1fp; follow-up
     triage recorded in the epic notes;
   - this tale's fixes: cold-identity key path, facade deletion, retired-name
     integration;
   - the `sase tool run check` result and the per-node isolation runs.

   Never use `--force` merely to make the close succeed.

3. Run `just symvision` and confirm it is clean of anything this epic introduced.
4. Set `status: done` in the frontmatter of the epic's plan file. That file is the PLAN
   path shown by `sase bead read sase-1ex -r "<why>"`
   (`plan:202610/prompt_space_and_project_cycle_latency.md`); it currently reads
   `status: wip`.
5. sase-1ex has no `parent_bead`, so the landing ends here.

## Acceptance

- With identity cold (fresh process, or right after
  `invalidate_macro_project_identity()`), a warm-snapshot `ctrl+p` / `ctrl+n` performs
  zero main-thread MRU reads or writes, `list_project_records` calls, `Popen`s, watcher
  start or stop calls, and thread joins. Every `prompt_key_io_probe` test passes when
  run alone.
- Arg hints and assist-catalog keys match today's once identity is warm.
- The publication facade and its test are gone, Symvision no longer reports them, and
  the bindings tool test passes.
- No epic-authored file references retired `vcs_xprompt_mru` names.
- `sase tool run check` passes, or only KNOWN/FLAKY items remain.
- sase-1ex is closed, `just symvision` is clean, and the epic plan file says
  `status: done`.
