---
tier: epic
title: Fix the read-model cache race, cut sase-core-rs, and ship sase v0.18.0
goal: 'The sase-core read-model cache survives concurrent readers and writers on every
  platform, a sase-core-rs release carrying every binding sase needs is complete on
  PyPI, Master Gate and Full CI are green on the sase master tip, release PR 299 merges,
  and `pip install sase==0.18.0` works from PyPI.

  '
phases:
- id: core-cache-race
  title: Make the sase-core read-model cache safe under concurrent access
  depends_on: []
  size: medium
  description: 'core-cache-race: stop unlinking or implicitly recreating a live SQLite
    read-model cache, keep cache faults from failing mutations, prove it with a stress
    run, and drop the now-redundant lock_wait_ms golden helper.'
- id: sase-gate-fixes
  title: Clear the remaining sase Master Gate failures
  depends_on: []
  size: medium
  description: 'sase-gate-fixes: fix the CI-only prompt-key perf smoke failure, resolve
    the declared_commands Symvision residual unless sase-1if already has, and fix
    any other Master Gate red on the tip.'
- id: full-ci-fixes
  title: Clear the Full CI-only failures
  depends_on: []
  size: medium
  description: 'full-ci-fixes: triage and regenerate or repair the drifted PNG goldens
    that keep the visual-test lane red, and fix any other Full CI-only red on the
    tip.'
- id: core-release
  title: Cut and publish the sase-core-rs release
  depends_on:
  - core-cache-race
  size: medium
  description: 'core-release: wait for green sase-core CI on the race fix, dispatch
    the urgent release-plz cut, and verify the new sase-core-rs is complete on PyPI.'
- id: ship
  title: Prove the release gates green, merge PR 299, and publish v0.18.0
  depends_on:
  - core-release
  - sase-gate-fixes
  - full-ci-fixes
  size: medium
  description: 'ship: ratchet PR 299 onto the new core floor, drive Master Gate, Full
    CI, and the PR checks green, merge it, publish, and verify the PyPI install.'
proposed_by: bbugyi200.athena.sase-1io.land
parent_bead: sase-1io
create_time: 2026-10-09 06:48:47
status: wip
bead_id: sase-1io.7
---

- **PROMPT:** [prompts/202610/finish_release_v0_18_0.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/finish_release_v0_18_0.md)
- **PARENT:** [202610/release_v0_18_0.md](https://github.com/sase-org/sase--plans/blob/main/202610/release_v0_18_0.md)
- **BEAD:** [sase-1io.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1io/sase-1io.7.md)

# Plan: Fix the read-model cache race, cut sase-core-rs, and ship sase v0.18.0

This plan finishes epic `sase-1io` ("Turn CI green and ship sase v0.18.0 to PyPI"). That
epic's six phases closed overnight, but its last phase recorded `RELEASE NOT SHIPPED`.
PyPI still shows `sase` 0.17.1 and `sase-core-rs` 0.37.0. The user's request, quoted
exactly: "Can you help me do whatever needs to be done to get all tests green and
release version v0.18.0 of the sase package to PyPI?" They are asleep and expect the
release when they wake. Make reasonable calls yourself and record them in bead notes. Do
not stop to ask.

## Escalation Rule: Hand Off To `opus/opus@xhigh`

The user explicitly asked for this rule in the parent epic, and it carries over. If you
get stuck, or for any reason think you cannot close your assigned phase bead, do not end
your turn with the bead open. Hand off with the `/sase_handoff` skill to the model
`opus/opus@xhigh`. The user's instruction is your explicit authorization to use that
skill. Write a self-contained successor prompt that includes the phase id, what you
found, what you tried, the current evidence (run ids, PR numbers, failing tests), and
what is left.

- If `sase pipe` rejects that model spec, retry with `--model 'claude/opus@xhigh'`. A
  non-zero exit means no handoff happened, so you are still running.
- Fence any literal percent-sign or hash-sign text in the successor prompt. Both are
  live syntax for the successor.
- A successor already on opus at xhigh hands off again only when its context window is
  spent, using `--fresh`. Otherwise it keeps working.
- Waiting on CI is **not** being stuck. Wait with `/sase_monitor`.

## State Established By The Land Agent (2026-10-09 ~10:45 UTC)

Re-verify everything at the current `origin/master` of each repo before you act. Other
agents push to both masters all night.

**sase-core** (open it with `sase repo open sase-core -r "<why>"`, work only in the
printed path, and read its `AGENTS.md`)

- Master tip `6faaa653`. Release PR **#323** `chore: release v0.37.1` is open on
  `release-plz-2026-10-06T18-16-25Z`. Release-plz merges it only on the daily
  `41 7 * * *` UTC cut or through the urgent-cut path in `docs/pypi-retention.md`:
  `gh workflow run release-plz.yml --repo sase-org/sase-core -f dry_run=false`. Either
  path waits for the PR's checks.
- Every recent master and release-PR CI run is red on
  `crates/sase_core/tests/bead_read_model_parity.rs::concurrent_readers_see_consistent_snapshots`.
  Four failure modes have been seen:
  - `no such table: meta` from the final `read_model_verify_cache` assertion (master
    runs 37830391615 through 37886308902, and PR run 37904345831).
  - SIGBUS crashing the whole test binary, on macOS (37904251133) and once on ubuntu
    (37908024251).
  - `append_issue_note(...).unwrap()` failing with
    `unable to open database file: .../bead-read-model-*.sqlite` (37912585861). The
    reader threads' ENOENT panics in that run are fallout: the main-thread panic dropped
    the `TempDir`.
  - `concurrent read served a state no replay ever produced` (PR run 37912989918).
- The land agent reproduced it on Linux: 1 failure in 240 runs, `append_issue_note`
  failing with `no such table: issues`. So the race is not macOS-only; macOS just loses
  it more often. It is a product bug in unreleased code. It first appears around
  2026-10-08 19:14 UTC (`e411a392`, after the read-model write-through work), well after
  `v0.37.0` (2026-10-06), so 0.37.1 would ship it.
- **Root cause.** Every read-model failure path ends in `drop_cache_file`
  (`crates/sase_core/src/bead/read_model/store.rs`). It is called from about 35 sites in
  `store.rs`, `tail.rs`, `queries.rs` and `publish.rs`, including lock-free read paths
  such as `rebuild_from_replay`'s post-commit re-sweep. It unlinks the cache database
  and its `-wal`/`-shm` companions while other connections in other threads or processes
  still hold them open. Readers take no `beads.db` flock.
  - `ensure_schema` treats `cache_path.is_file()` as "schema exists".
  - `open_read_write` uses `Connection::open`, which silently creates an empty database
    at the path.
  - So a concurrent opener can leave a valid but schema-less SQLite file in place.
    `read_meta` then fails with `Fault::Cache` on every later read, and the cache never
    heals.
  - Deleting a live WAL database's companions also explains the torn snapshot and the
    SIGBUS (a mapped `-shm` replaced under live connections).
- **Mutation side.** The cached `MutationView` backing
  (`crates/sase_core/src/bead/mutation/view.rs`: `open_cached`, `read_cache_witness`,
  and the row loads) reopens the cache read-only after admission. If a reader drops or
  recreates the file in that window, the mutation fails with a `BeadError` of kind `io`.
  That contradicts the publish-side rule that a cache problem never fails a mutation.
- **Duplicate helper.** Phase `sase-1io.1` added `normalize_lock_wait_ms` (plus its unit
  test `lock_wait_normalization_pins_zero_regardless_of_measured_wait`) in
  `crates/sase_core/src/bead/mutation/tests/replay_goldens.rs`. The later commit
  `d2a954b0` zeroes `lock_wait_ms` at the source in `outcome_string`, and every golden
  `act` helper in `replay_golden_cases.rs` returns through `outcome_string`. So the
  helper is dead weight.

**sase** (this repo)

- Master tip `7e87589fb5`. Master Gate run 37915170978 is red on two things:
  - `lint` (`_lint-symvision`): unused public `fetch_upstream_pyproject`,
    `get_declared_commands`, `parse_declared_commands`, `read_declared_cache`, and
    `write_declared_cache` in `src/sase/plugins/declared_commands.py`. They came from
    `bd68ee4942` (phase `sase-1if.6` of in-progress epic `sase-1if`). DISCOVERED ISSUE
    notes on `sase-1if` already record it, including a coordination note from this
    landing.
  - `test (7)`:
    `tests/ace/tui/test_prompt_key_perf_smoke.py::test_prompt_key_perf_harness_records_space_and_cycle`
    fails with `assert 'prompt_space' in ['prompt_cycle_ctrl_p']`. It also failed on
    both the 3.13 and 3.14 legs of Full CI run 37909515949. It passes locally (Python
    3.14 workspace venv). No bead tracks it.
  - The previous tip `6c783599c1` was red only on the Symvision item.
- Full CI run 37909515949 (at `e2efd56242`, older than the newest fixes):
  - `visual-test` is red with 26 drifted PNG goldens. They include 11
    `agents_final_*_120x40` (0.005-0.02% each), 3 `agents_retry_e2e_*`, 3
    `agents_waiting_epic_follow_*`, `agents_jump_panel_expanded_two_sections` (0.59%),
    `agents_renamed_generic_session_root` (0.25%), `artifacts_beads_idle_query`,
    `custom_gate_task_triage` (7.6%), `mini_macro_location_flow_finder` (6.0%),
    `model_picker_usage_hints`, two `models_panel_alias_picker_reordered_*`, and
    `timeband_past_light` (1.8%).
  - The parent phase `sase-1io.4` closed without touching any golden.
  - `test (3.13)` also failed
    `tests/ace/tui/widgets/test_prompt_next_word_inline_tail.py::test_typing_space_inside_parens_shows_tail_ghost`.
    That is known flake bead `sase-1ib`, corroborated by this landing.
  - The 3.12 coverage leg was still running at planning time.
- Release PR **#299** `chore(master): release 0.18.0` (branch
  `release-please--branches--master`) fails `release-core-floor-smoke`. The published
  floor `sase-core-rs 0.37.0` lacks 5 of 807 bindings: `bead_probe_target_owner`,
  `classify_auto_directive`, `instruction_manifest_wire_schema_version`,
  `normalize_instruction_manifest`, and `wait_epic_follow_reduce`. All five are on
  sase-core master. Only a new core release plus the `sync-release-metadata` ratchet in
  `publish.yml` clears it. Never hand-edit the window.
- Release mechanics are unchanged from the parent plan:
  - `publish.yml` runs on cron (`17 */3 * * *`) or by `workflow_dispatch`. With
    `publish_existing=false` it regenerates the release PR and ratchets the
    `sase-core-rs` window to the newest published core.
  - After PR 299 merges, the next generation run creates the tag and the GitHub release.
    It then runs `build`, `install-smoke`, `install-smoke-core-floor`, and `publish`.
  - The `ci_watch` AXE routine merges a fully green release-please PR (merge method
    `merge`) when three things hold: Master Gate is green for the master tip, Full CI
    was green within the last 6 hours, and the PR's own checks are green.

## Guardrails For Every Phase

- Fix root causes. Never weaken an assertion, skip a test, add a `xfail`/retry, or raise
  a timeout to get green. Update an expectation only after you confirm the product
  change behind it was intentional (`git log -S`, the responsible commit). Then make the
  stale side match current truth.
- Never hand-edit release-owned files: `CHANGELOG.md`, any `version` field,
  `.release-please-manifest.json`, the `sase-core-rs` window in `pyproject.toml`, or any
  sase-core `version`, pin, or `CHANGELOG.md`.
- Do not run `just install` or `just install-dev`; use `just install-venv`.
- Verify with `sase tool run check` in every repo you change. Run it from the opened
  checkout for sase-core. Run `just fix` (sase) or `just fmt` (sase-core) first. Only
  `full-ci-fixes` may run `just check-full`.
- Wait for CI, releases, and PyPI only through `/sase_monitor`, with a bounded timeout
  and a `--next` that says exactly what to do with the result. Never end a turn
  promising to come back.
- The only PRs you may merge are sase PR 299 (in `ship`) and the sase-core release-plz
  PR, which merges only through the documented `release-plz.yml` dispatch. Leave every
  other open PR alone.
- Before fixing a failure, look for an existing bead (`sase bead search`,
  `sase bead list -T ci`, `sase bead list -T flake`). Note on that bead that you are
  fixing it rather than duplicating the work.
- Land code only through the host finalizer at the end of your turn. A phase cannot
  watch CI for its own commit, which is why fix phases and verify phases are separate.
- Phase workers never create beads. Record `PROPOSED FOLLOW-UP:` notes on your own phase
  bead instead.

## Phase core-cache-race: Make the sase-core read-model cache safe under concurrent access

Work in the opened sase-core checkout. Read the read-model module facade
(`crates/sase_core/src/bead/read_model/mod.rs`) and `publish.rs`'s header comment first;
they state the invariants (freshness token, generation and content-generation CAS,
"after a durable append a cache problem never fails the mutation").

Required outcomes:

1. **Never pull a live database out from under a connection.** No path may unlink,
   replace, or truncate the cache database or its `-wal`/`-shm` while another connection
   may hold it open.
   - Replace `drop_cache_file`'s role on every concurrent path with in-place
     invalidation inside a SQLite write transaction. For example, reset the schema and
     meta, or mark the cache unusable so the next freshness pass rebuilds.
   - Keep the generation monotonic so any in-flight writer's CAS loses rather than
     committing stale rows.
   - Unlinking may remain only where the file is provably not a SQLite database
     (`cache_file_is_not_a_database`), since no WAL connection can exist there.
   - Audit every caller in `store.rs`, `tail.rs`, `queries.rs` and `publish.rs`.
2. **A cache file always carries its full schema.**
   - Stop implicit creation: open existing caches without `SQLITE_OPEN_CREATE`
     everywhere except creation itself.
   - Create the cache atomically: build the schema in a sibling temp file and publish it
     with a no-clobber step such as `std::fs::hard_link` into place and then remove the
     temp. A losing racer just uses the winner's file. It must work on macOS and Linux.
   - A valid-but-schema-less or partially initialized file (left by older builds) must
     heal on the next freshness pass instead of staying `Fault::Cache` forever.
3. **Cache faults never fail a mutation.** If the cached `MutationView` backing cannot
   open or read the cache before any durable append, fall back to the replay backing
   instead of returning a `BeadError`. Faults after the append keep publish's existing
   skip or invalidate behavior. Never append twice.
4. **Keep everything else byte-identical.**
   - All existing read-model, mutation-suite, parity, and replay-golden tests pass.
   - `replay_golden_bytes_are_pinned` and `cached_golden_bytes_match_replay` stay
     byte-identical; no golden is regenerated.
   - Wire schemas and Python bindings do not change.
   - Rebuild and serve telemetry may move only where a removed unlink changes it, and
     tests asserting those counters must still pass.

Add deterministic regression tests that need no timing luck, beside the existing
read-model and mutation tests:

- invalidation while another connection holds the cache open, for example a read-only
  connection kept open across an invalidating path;
- a pre-existing schema-less SQLite file at the cache path heals;
- a mutation whose cache disappears or is invalidated between admission and its cached
  reads still succeeds through the replay fallback.

Integration cleanup in the same commit: delete `normalize_lock_wait_ms` and
`lock_wait_normalization_pins_zero_regardless_of_measured_wait` from
`crates/sase_core/src/bead/mutation/tests/replay_goldens.rs`, call `(case.act)(...)`
directly, and keep the module doc's `lock_wait_ms` paragraph accurate. `outcome_string`
already pins it to zero.

Verification:

1. Linux stress recipe. Build with
   `just test -p sase_core --test bead_read_model_parity --no-run`. Then run the printed
   binary with `concurrent_readers_see_consistent_snapshots --exact --test-threads 1`
   from 16 parallel shell workers × 60 iterations (960 runs, about 13 minutes on the
   shared host). Before the fix this fails about 1 in 240. After the fix it must be 0
   of 960. Run it through `/sase_monitor` if it will not fit your synchronous limit.
   Record the before and after counts.
2. `sase tool run check` in sase-core passes.
3. The `mac` tailnet host is usually offline, so CI is the macOS proof. Record in a bead
   note how the fix removes each of the four failure modes, so `core-release` can judge
   the CI result.

Use a Conventional Commit subject such as `fix(bead-read-model): ...`. Do not touch
versions or changelogs.

**Done when** the fix and regression tests are ready to land, the stress run shows 0
failures, and `sase tool run check` passes in sase-core.

## Phase sase-gate-fixes: Clear the remaining sase Master Gate failures

Run `just install-venv` first. Then `git fetch`, and list the failures of the newest
completed Master Gate run on the tip (`gh run list --workflow master-gate.yml`). Fix
every one:

- **`test_prompt_key_perf_harness_records_space_and_cycle`**: it fails on GitHub runners
  (Python 3.12/3.13/3.14, Textual from the lockfile) but passes locally.
  - Find why no `prompt_space` key-to-paint sample is recorded there. Start from where
    `action_start_agent_from_patch` and the hot-spare prompt bar
    (`sase-1ex.11`/`1ex.12`) record that sample, and whether the test can observe the
    bar mount before the paint that records it.
  - Mirror CI as closely as you can: the Python 3.12 venv, `pytest -p xdist -n` with the
    shard's neighbours, and host load.
  - Fix the product race, or the test's readiness wait if the test merely reads too
    early. Never delete the assertion.
- **`declared_commands.py` Symvision residual**: first check `git log` for a `sase-1if`
  commit that already resolved it, and read `sase-1if`'s notes. If it is still red, read
  the Symvision memory note and apply its hierarchy per symbol:
  - privatize the symbols used only inside `declared_commands.py`, updating in-file
    callers, `__all__`, and tests;
  - add `--epic-symbol sase-1if.7(<symbol>)` Justfile rows only for symbols the
    `updates-tab` section of the `sase-1if` plan (`plan:202610/plugin_commands.md`) says
    the detail-view preview worker will consume, and only while `sase-1if.7` is still
    open.
  - Record what you did on `sase-1if` with `sase bead note`.
- Any other Master Gate failure present on the tip when you start.

Run every named test, `just _lint-symvision`, and `sase tool run check`.

**Done when** every Master Gate failure on the starting tip passes locally and the fixes
are ready to land.

## Phase full-ci-fixes: Clear the Full CI-only failures

This phase owns failures that appear only in Full CI (`full.yml`): the `visual-test`
lane, the 3.12 coverage leg, and Full CI-only flakes. Failures `sase-gate-fixes` owns
count as KNOWN here.

1. `git fetch`. Take the newest completed Full CI run, or dispatch one on the tip with
   `gh workflow run full.yml --repo sase-org/sase` and wait through `/sase_monitor`.
   List its red jobs and failures.
2. **Drifted PNG goldens.** Use the `visual-test` artifacts and failure report, plus
   `git log` on the renderers involved, to group the drifted goldens by cause. Decide
   for each group whether the new rendering is intended.
   - Read the `tui` memory note's screenshot guidance first.
   - Regenerate intended ones with `just fix-tui-screenshots`, with selectors where
     possible, through `/sase_monitor` because it is long. Inspect every changed PNG.
   - Fix real rendering regressions in the product instead.
   - Commit every dirty golden per `/sase_final`'s screenshot rule.
   - Goldens keep drifting as other agents push UI work, so regenerate against the tip
     you start from and list the final set in your bead note.
3. **Coverage leg and flakes.** Reproduce any 3.12 coverage-leg failure with a
   `just test-cov`-style run of only those tests. For known flake `sase-1ib` or any
   other red that recurs in the run, fix the root cause if you can reproduce it under
   load. If you cannot, record the evidence on its bead.
4. This bead explicitly authorizes `just check-full` through `/sase_monitor`, with the
   `verify` profile, as final verification.

**Done when** every Full CI-only failure on the starting tip passes locally, every
intended golden is regenerated, and the fixes are ready to land.

## Phase core-release: Cut and publish the sase-core-rs release

1. Wait (through `/sase_monitor`) for sase-core master CI on the commit that contains
   the `core-cache-race` fix, on both ubuntu and macOS. Then wait for the release-plz
   PR's own CI after release-plz refreshes it.
   - If `concurrent_readers_see_consistent_snapshots` or anything else is still red,
     reproduce it and land a fix in the opened sase-core checkout. Then close this bead
     with a note that starts `RELEASE NOT CUT:` and gives the reason and the run ids;
     `ship` picks the cut up.
2. When both are green, run
   `gh workflow run release-plz.yml --repo sase-org/sase-core -f dry_run=false`. Skip
   this if the daily cut already merged the PR. Monitor the dispatch run, then the
   push-triggered `Release-plz` run that tags the version and publishes the wheels.
3. Verify on `https://pypi.org/pypi/sase-core-rs/json` that the new version (expected
   `0.37.1`; release-plz decides) lists five unyanked distributions: linux x86_64, linux
   aarch64, macOS universal2, win_amd64, and sdist. Then, in a fresh `uv venv`,
   `uv pip install sase-core-rs==<version>` and import all five bindings named above.
4. If the publish job refuses because of PyPI storage quota, follow
   `docs/pypi-retention.md` in sase-core. If recovery needs a human, such as a PyPI web
   login, record exactly what is needed, send it through `sase notify create`, and hand
   off per the escalation rule.

**Done when** the new `sase-core-rs` is complete on PyPI with all five bindings, or the
bead closes with a `RELEASE NOT CUT:` note after landing a further fix.

## Phase ship: Prove the release gates green, merge PR 299, and publish v0.18.0

1. **Preconditions.**
   - The required `sase-core-rs` must be complete on PyPI. If `core-release` closed with
     `RELEASE NOT CUT`, run the `core-release` procedure first.
   - `git fetch` and confirm the `sase-gate-fixes` and `full-ci-fixes` commits are on
     `origin/master`.
2. **Start in parallel.**
   - `gh workflow run full.yml --repo sase-org/sase` on the current tip, unless a run
     already covers a tip containing the fixes.
   - `gh workflow run publish.yml --repo sase-org/sase -f publish_existing=false`. This
     regenerates PR 299 from current master and ratchets its floor.
   - Confirm PR 299 is still titled `chore(master): release 0.18.0`, that its
     `pyproject.toml` floor is the new core, and that `release-core-floor-smoke` passes.
3. **Monitor** Master Gate for the tip, the Full CI run, and PR 299's checks until they
   settle. If anything is red, reproduce it at `origin/master`.
   - If it needs code, land the fix and close this bead with a note that starts
     `RELEASE NOT SHIPPED:`. Give the failures fixed and the runs to re-dispatch; this
     plan's land agent finishes the release.
   - Hand off only if you cannot produce a fix.
4. **Merge.** When all three `ci_watch` conditions hold, let `ci_watch` merge PR 299. It
   runs one merge per tick. If it has not merged within about 30 minutes, merge it
   yourself with `gh pr merge 299 --repo sase-org/sase --merge`. The user explicitly
   asked for this release, and that is the method `ci_watch` uses.
5. **Publish.** Run
   `gh workflow run publish.yml --repo sase-org/sase -f publish_existing=false` so
   release-please creates the `v0.18.0` tag and GitHub release. Monitor `build`,
   `install-smoke`, `install-smoke-core-floor`, and `publish`. If the tag exists but
   publishing failed, fix the cause and use the `-f publish_existing=true` dispatch; its
   upload uses `skip-existing`.
6. **Verify.** `https://pypi.org/pypi/sase/json` reports `0.18.0` with a wheel and an
   sdist. In a fresh venv, `uv pip install sase==0.18.0`, then `sase version` and
   `sase core health --json` both succeed.
7. **Notify.** Send a `sase notify create` summary for the user: the PyPI URL, the core
   version, and the failures that were fixed.

**Done when** `sase==0.18.0` installs from PyPI and passes the health check, or the bead
closes with a `RELEASE NOT SHIPPED:` note after landing a further fix.

## Landing

The land agent confirms that `sase==0.18.0` is live on PyPI and that the final Master
Gate and Full CI runs are green. If any phase closed with `RELEASE NOT CUT` or
`RELEASE NOT SHIPPED` and the release is still not on PyPI, finishing the release **is**
the landing work. Follow the `ship` procedure and the escalation rule. Then triage the
phases' `PROPOSED FOLLOW-UP:` notes as usual. After this plan closes, resume the
interrupted landing of its parent epic `sase-1io` as the land prompt describes.
