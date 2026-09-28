---
tier: tale
size: small
title: Finish and land epic sase-1bf (bounded agent scratch)
goal:
  "The epic-caused leftovers are fixed: zombies no longer make launch-scratch liveness
  incomplete, nested registered roots no longer double-reap or double-count, and the CI
  core pin builds managed-tmp wire 4. Then epic sase-1bf is verified, closed, and its
  plan is marked done."
proposed_by: bbugyi200.apollo.sase-1bf.land
bead: sase-1bf
status: done
---

- **PARENT:**
  [202609/bounded_agent_scratch.md](https://github.com/sase-org/sase--plans/blob/main/202609/bounded_agent_scratch.md)
- **BEAD:**
  [sase-1bf](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1bf/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.sase-1bf.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.apollo.sase-1bf.land.md)
- **COMMITS:**
  - [7880b16](https://github.com/sase-org/sase--plans/commit/7880b164a7f72a7c26a8b0e65b91dcb6ee30d4aa)
    — docs(plan): mark bounded-agent-scratch epic done

# Plan: Finish and land epic sase-1bf (bounded agent scratch)

Epic `sase-1bf` ("Bound agent scratch by ownership, not by environment luck", plan
`plan:202609/bounded_agent_scratch.md`) has all six phases closed. The land agent
checked every phase in source and commits: sase 40295eaf54, 7e4482a62f, c2eb318d88,
5db68f77f2 and d99f4789e9, plus sase-core 924884e, 297bc1e and 90d141e. Host acceptance
(`sase-1bf.6`) passed on apollo and athena. The land agent already triaged every
`PROPOSED FOLLOW-UP` and recorded each outcome as a note on `sase-1bf`. Do not
re-triage.

Three small defects caused by the epic remain, followed by the epic closeout. This tale
has no land agent of its own: **step 4 is the landing, and it must be done in this
tale.**

Read `sase bead read sase-1bf -r "<why>"` first for the epic context, the phase list and
the land agent's triage note.

## Step 1 — Zombie processes must not make launch-scratch liveness incomplete (sase-core)

Work in the linked sase-core repo (`sase repo open sase-core -r "<why>"`, then use the
printed path). Read that repo's `AGENTS.md` before editing.

File: `crates/sase_core/src/launch_scratch_liveness.rs`,
`observe_launch_scratch_liveness`.

The problem: the observer scans same-uid `/proc/<pid>` entries. When `environ` or `cwd`
of a process cannot be read, and the process started after the candidate's birth,
`observe_unreadable` marks the candidate `complete = false`. A zombie (or dead) process
has already released its address space, fs struct and cwd, so it cannot hold a launch's
environment or working directory. Acceptance on apollo still saw zombie pid 635512
(orphaned under a stale `sase tui`) keep dead-launch entries incomplete. The same false
"incomplete" also blocks runner-exit cleanup.

Change:

- After the uid filter and before the `environ`/`cwd` reads, read the process state from
  `/proc/<pid>/stat`: the first whitespace token after the last `)`, because comm can
  contain spaces and parentheses. If the state is `Z` (zombie) or `X` (dead), skip the
  pid entirely (`continue`). It is neither a holder nor an incomplete observation.
- Parse `stat` once. Factor a small helper that returns both the state and the start
  time, so `process_start_epoch` and the new state check do not read the file twice. An
  unreadable or malformed `stat` keeps today's behavior: no state, fall through.
- Count skipped zombies. When the count is non-zero, push one diagnostics line, for
  example `zombie exemption: N exited processes skipped`, next to the existing
  `pre-launch exemption` line.
- **Do not change the wire.** `LAUNCH_SCRATCH_LIVENESS_WIRE_SCHEMA_VERSION` stays 1, and
  no request/result fields or Python bindings change. sase therefore needs no new pin
  for this behavior.
- Tests, in the same file's `mod tests`: extend the `write_process_stat` helper, which
  hardcodes state `"R"`, to take a state. Add a test in which a pid has state `Z`, an
  unreadable `environ` (create `environ` as a directory, as the existing tests do) and a
  start time after the candidate's birth. The candidate must stay `complete` and not
  `live`, with `unreadable == 0`. Keep `later_unreadable_process_stays_incomplete`
  passing unchanged for the non-zombie case.
- Docs, in the sase repo: in `docs/axe.md`, the `managed_tmp_reap` paragraph describes
  the dead-launch backstop's "incomplete-observation entries". Add one short clause
  saying that zombie processes are not counted as holders or as incomplete observations.
- Verify in sase-core with `just test -p sase_core launch_scratch_liveness` and
  `just fmt`, then the repo gate through `sase tool run check`. That gate is currently
  red on pre-existing clippy denies in other files, tracked by task `sase-1an`
  (`manual_range_contains`/`nonminimal_bool` in `agent_runtime.rs`, `provider_usage`,
  and others). Confirm that the new code adds no clippy deny in
  `launch_scratch_liveness.rs`, and record any remaining pre-existing failure rather
  than fixing unrelated files.

## Step 2 — Collapse registered roots nested inside another covered root (sase)

File: `src/sase/core/managed_tmp_roots.py`, `effective_managed_tmp_roots`.

The problem: on apollo, the registry enrolled a per-launch directory as its own root
(`~/.cache/sase/tmp/agent-tmp/<key>/tmpXXXX/managed`), inside the already-registered
`~/.cache/sase/tmp`. De-duplication is by resolved path only, so every reader handles
the nested root twice:

- the housekeeping chop and `sase disk reap` reap it twice;
- `sase disk list`, via `src/sase/core/disk_footprint_inventory.py`, emits managed-tmp
  rows for both the outer and the nested root and double-counts those bytes against
  `df`.

Change:

- After the existing resolved-path de-duplication, drop any candidate whose resolved
  path is strictly inside another candidate's resolved path. Keep the outermost root and
  keep the original order. The outer root's reaper already owns the nested tree at
  launch-key granularity: the whole `agent-tmp/<key>` entry is removed by runner-exit
  cleanup, the dead-launch backstop or the age horizon. Use `Path.is_relative_to` on the
  resolved paths.
- Keep registration unchanged. Missing roots are already pruned on the next registry
  write, so a nested per-launch root disappears once its launch directory is removed.
- Update the function docstring. Update the registry sentence in `docs/axe.md` ("reaps
  the effective root plus every registered root that still exists …") and the matching
  `managed_tmp` registry sentence in `docs/configuration.md` to say that a root nested
  inside another covered root is folded into it.
- Tests, in `tests/core/test_managed_tmp_roots.py`: register an outer root and a nested
  `agent-tmp/<key>/tmpX/managed` root under the same test `SASE_HOME`. Then
  `effective_managed_tmp_roots(...)` returns only the outer root. Also cover sibling
  roots: two unrelated roots are both kept, and a sibling whose name only shares a
  string prefix, such as `/a/tmp` vs `/a/tmp2`, is not treated as nested.

## Step 3 — Move the CI core pin past the epic's sase-core commits (sase)

sase master already requires managed-tmp reap wire schema 4
(`MANAGED_TMP_REAP_WIRE_SCHEMA_VERSION = 4` in `src/sase/core/managed_tmp_reaper.py`).
`sase-core-revision.txt` still pins `0e8981a1f131d2dd040c4887ae949edf19fbeef6`, which
builds wire 3, so CI's `build-core`/`core-wheel` jobs build a core that sase rejects.
See "The CI source revision pin" in `docs/rust_backend.md`.

- Run `just ratchet-core-revision`. Exit 2 means "bump applied" and is expected. It
  moves the pin to sase-core's current remote HEAD, which must contain sase-core
  `90d141e` (dead-launch wire 4) and `924884e` (roots registry bindings). Confirm with
  `git -C <sase-core path> merge-base --is-ancestor 90d141e <new pin>`.
- Step 1's sase-core change is committed by the host finalizer after this turn, so the
  pin cannot point at it. That is fine: step 1 adds no binding and no wire change.
  `.github/workflows/core-pin-ratchet.yml` moves the pin forward later.
- Run `./.venv/bin/python tools/check_sase_core_rs_bindings`. It must report no missing
  bindings.

## Step 4 — Verify, then close epic sase-1bf (the landing)

1. Verification in sase: read the `lint_and_test` reference memory
   (`sase memory read lint_and_test.md -r "<why>"`) and run `just check` exactly as it
   directs, through `sase tool run`. Also run the focused suites:
   - `tests/core/test_managed_tmp_roots.py`
   - `tests/core/test_disk_footprint_inventory.py`
   - `tests/test_managed_tmp_reaper*.py`
   - `tests/test_run_agent_runner_scratch_cleanup.py`

   Do not run `just check-full`. A known pre-existing `just symvision` failure on
   `rail_panel_title`, `rail_tooltip_text` and `rail_urgency` in
   `src/sase/ace/tui/widgets/_agent_list_render_rail.py` belongs to the in-progress epic
   `sase-1bn`. It is already recorded there as a DISCOVERED ISSUE, so do not fix it and
   do not treat it as a blocker. Any other failure caused by steps 1–3 must be fixed
   before closing.

2. Run `sase bead epic-symbols sase-1bf`. It currently reports no entries. If any
   appear, resolve each one: wire it up, privatize it, add a non-test pragma or delete
   it, per the Symvision epic-whitelist policy in the `symvision` reference memory.
   Re-key a Justfile `--epic-symbol` line only to a still-open later bead that genuinely
   needs the exemption.
3. Close the epic, never with `--force`:

   ```bash
   sase bead close sase-1bf --note "<verification>"
   ```

   The note must cover:
   - all six phases were verified in source and commits (sase 40295eaf54, 7e4482a62f,
     c2eb318d88, 5db68f77f2, d99f4789e9; sase-core 924884e, 297bc1e, 90d141e), and
     `sase-1bf.6` host acceptance passed on apollo and athena;
   - integration review: the only non-epic commit touching epic files, 965248789b,
     privatized unused registry and observation symbols without conflict. Epic note #1's
     clippy denies were fixed upstream in sase-core bc71eb2;
   - this tale's three fixes (zombie exemption, nested-root collapse, pin moved to the
     new SHA) and the verification results;
   - a pointer to the land agent's triage note on `sase-1bf`: sase-1an +1, new task
     sase-1bw, and the declined proposals with reasons.

   If the close is rejected because `--epic-symbol` entries remain, finish step 4.2 and
   close again. All six phases are already closed.

4. Run `just symvision`. Confirm that no `--epic-symbol` entry keyed to `sase-1bf` or
   its phases remains in the `Justfile`. The only acceptable remaining failure is the
   `sase-1bn` rail symbols above.
5. Set `status: done`, replacing `status: wip`, in the frontmatter of the epic's plan
   file: the PLAN path that `sase bead read sase-1bf` prints for
   `plan:202609/bounded_agent_scratch.md`. Change nothing else in that file.
6. `sase-1bf` has no `parent_bead`, so the landing ends here.
