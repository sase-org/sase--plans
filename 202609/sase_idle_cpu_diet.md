---
tier: epic
title: 'Idle CPU diet: stop sase from burning ~4 cores at rest'
goal: 'An idle sase host (ace open, lumberjacks running, no agent work) drops from
  roughly four sustained cores of sase CPU to well under half a core — by eliminating
  per-chop interpreter boot waste, skipping chop subprocess spawns when their inputs
  are provably unchanged, replacing ace''s unconditional full-disk refresh with cheap
  per-surface change tokens, caching immutable axe status reads, and reusing one sase-core
  release build across workspaces — all without changing chop cadence, subprocess
  isolation, timeout semantics, or user-visible TUI freshness.

  '
phases:
- id: chop-import-slim
  title: Slim chop subprocess imports
  depends_on: []
  size: medium
  description: 'chop-import-slim: cut the ~0.6s/1,251-module import cost every sase_chop_*
    subprocess pays before doing any work, by making sase.axe lazy, trimming the chops
    SDK import chain, and adding an import-budget regression test; no behavior change.'
- id: chop-trigger-provider
  title: Filesystem change-token trigger provider (sase-core)
  depends_on: []
  size: medium
  description: 'chop-trigger-provider: add an fs change-token trigger provider to
    the sase-core axe_chop preflight engine (paths -> cheap state token, fire on change,
    fail-open, max_quiet fallback), with Rust tests, binding surface, and Python plumbing;
    no shipped chop uses it yet.'
- id: chop-guard-defaults
  title: Wire pre-spawn guards into shipped chop defaults
  depends_on:
  - chop-trigger-provider
  size: medium
  description: 'chop-guard-defaults: give every high-frequency shipped chop in default_config.yml
    an fs change-token trigger (or run_every where inputs are time-based), so an idle
    tick costs a few stat() calls instead of 8-14 interpreter boots; per-chop input
    maps verified against chop source, with fire/skip tests for each.'
- id: chop-incremental-scans
  title: Make wait_checks and bead_claim_checks incremental
  depends_on: []
  size: medium
  description: 'chop-incremental-scans: invert both scan chops so their cheap short-circuit
    runs before the full O(all-artifacts) walk - wait_checks consults waiting markers
    first and resolves only referenced dependencies (via the agent artifact index
    where it suffices), bead_claim_checks runs its owner pre-pass before scanning;
    identical outputs on the non-skip path.'
- id: ace-refresh-tokens
  title: Per-surface change tokens for ace auto-refresh
  depends_on: []
  size: large
  description: 'ace-refresh-tokens: replace ace''s unconditional full reconcile every
    refresh_interval with cheap per-surface change tokens (agents, axe, notifications,
    patches, procs) that work without an fs watcher, restoring the dirty-gate design
    on macOS and under Linux churn, with a periodic full-sanity reconcile and a sunset
    flag keeping the old unconditional path reachable.'
- id: ace-axe-status-cache
  title: Cache immutable axe status reads in ace
  depends_on:
  - ace-refresh-tokens
  size: medium
  description: 'ace-axe-status-cache: stop collect_axe_status_data re-parsing ~600
    files per tick - cache immutable chop run records by (path, mtime), tail logs
    only when they grew, and collect full chop snapshots only when the Axe tab needs
    them.'
- id: ace-idle-render-cache
  title: Stop multi-second idle re-renders of the prompt panel
  depends_on: []
  size: medium
  description: 'ace-idle-render-cache: make prompt-panel rendering (section-navigation
    strips/heights, lazy syntax highlighting, frontmatter lexing) cache-stable across
    refreshes of unchanged content, so an idle ace stops logging 1.5-4s main-thread
    stalls re-rendering the same document.'
- id: ace-io-hygiene
  title: Small ace I/O fixes (agents-sync reads, bead N+1)
  depends_on: []
  size: small
  description: 'ace-io-hygiene: replace the byte-at-a-time _read_until_nul in agents_sync
    git blob reads with buffered reads, and batch the per-bead show() N+1 in the prompt-panel
    detail header summary.'
- id: core-build-cache
  title: Reuse sase-core release builds across workspaces
  depends_on: []
  size: medium
  description: 'core-build-cache: add a host-level sase_core_rs wheel cache keyed
    by sase-core commit, toolchain, and ABI so ephemeral workspaces install a cached
    wheel instead of each running a multi-core-minute maturin release build; cache
    miss or dirty checkout falls back to today''s build path.'
- id: perf-guardrails
  title: Perf counters, budgets, and regression guardrails
  depends_on:
  - chop-import-slim
  - chop-guard-defaults
  - ace-refresh-tokens
  size: medium
  description: 'perf-guardrails: land the counters that prove the wins (chop spawns/min
    and no-op ratio in lumberjack metrics, per-tick reload counters in ace), lock
    in import-time and spawn-rate budgets as tests, and update the perf runbook.'
proposed_by: bbugyi200.kellys_mbp.o.f0
create_time: 2026-09-04 12:10:54
status: wip
bead_id: sase-wn
---

- **BEAD:** [sase-wn](https://github.com/sase-org/sase--beads/blob/main/pages/sase-wn/README.md)

# Idle CPU Diet: Stop sase From Burning ~4 Cores At Rest

## Problem

A 16-core Linux host running a normal sase fleet sits at load ~9 around the clock. Live
profiling (process sampling, `/proc` counters, the TUI stall watchdog's own records, and
lumberjack run histories) attributes the _idle_ portion — roughly four sustained cores
that have nothing to do with actual agent work — to sase itself:

1. **ace burns ~0.9–1.0 cores continuously while completely idle.** A restarted
   `sase ace` settles at ~95–105% CPU with no user input, waking ~230–350 times/sec. The
   same pathology reproduces on macOS (~74% of a core, 18 minutes of CPU in a 24-minute
   session). Thread sampling on macOS shows one persistent worker thread at ~80% doing
   `stat`/`open`/`readall`/JSON in a continuous loop, and
   `~/.sase/logs/tui_agent_loads.jsonl` records the broad agent disk load at **median
   3.23s, max 66.65s per run**, re-triggered by auto-refresh every 10s. The stall
   watchdog logged 99 main-loop stalls of 1.5–4.3s in the first 10 idle minutes of a
   Linux session, and stalls up to 144s on macOS.
2. **Lumberjack chops spawn ~109 fresh Python subprocesses per minute** (~1.8/sec) on an
   idle host, each paying **~0.6s of interpreter+import CPU** before any work — about
   one full core of pure boot tax. Their results are overwhelmingly `status: no_op`
   (`reason=no_hooks`, `reason=waiting_markers_already_ready`,
   `reason=no_claim_reconciliation_candidates`).
3. **`wait_checks` parses every `agent_meta.json` on the host every cycle** — 4,597
   artifacts across 5 projects on the Linux host, 2,899 on the Mac — burning up to 12+
   seconds of a full core per run every ~11–25s, to usually conclude "no_op".
4. **Every ephemeral workspace rebuilds sase-core in release mode from scratch**
   (`maturin develop --release`, `opt-level=3`, `codegen-units=1`, per-workspace
   `target/`), so the same sase-core commit is rebuilt several times a day at multiple
   core-minutes each; two such builds frequently run concurrently.

## Root causes (verified, with locations)

### ace: the dirty-gate design is dead, so every tick is a full reconcile

- The auto-refresh timer (`src/sase/ace/tui/actions/_startup_mount.py:136-138`, default
  `refresh_interval` 10s from `src/sase/main/parser_ace.py:48-53`) runs
  `_on_auto_refresh` →
  `src/sase/ace/tui/actions/event_refresh/_auto_refresh.py:113-125`, where
  `_should_refresh()` returns `True` for **every** surface whenever `_watcher_active()`
  is false.
- The fs watcher can never start on macOS: `src/sase/ace/tui/util/fs_watcher.py:196-203`
  returns `None` from `_libc()` on any non-Linux platform, so `ArtifactWatcher.start()`
  returns `False` and `actions/event_refresh/_watcher.py:150-152` reports the watcher
  permanently inactive. On Linux the watcher may start, but background chop/agent churn
  dirties the watched state every few seconds, which produces the same full-reload
  cadence.
- Each unconditional tick then runs: the broad agent disk load
  (`actions/agents/_loading_disk.py:328-340`, measured median 3.23s), the full
  `collect_axe_status_data` walk (below), and the notification snapshot read. The
  debounce floor `AGENTS_LOAD_MIN_INTERVAL_SECONDS = 5.0`
  (`actions/event_refresh/_constants.py:19`) is half the refresh interval and never
  trips; the comment above it already names the loader as "the dominant cost on every
  refresh".
- `collect_axe_status_data` (`src/sase/ace/tui/actions/axe_display/_data.py:263-410`)
  reads **~600 files per tick** on a 7-lumberjack/28-chop host: per chop it re-reads the
  run index plus up to 10 immutable run-record JSONs _and_ a ≥64KB log tail per run
  (`_data.py:203-211`, `src/sase/axe/_state_chops.py`, `src/sase/axe/state.py:80`),
  every 10 seconds, on every tab.
- `ProcObserver` polls the proc store at 2Hz unconditionally
  (`src/sase/ace/tui/proc_observer.py:52,183,203-233`) — the signature check only
  suppresses delivery, not the read.
- Heavy `to_thread`/`run_worker` jobs (agent load, axe status, bead cache warmup, family
  plan previews, agents-sync git reads) pile onto one GIL simultaneously; stall captures
  show 3–5 CPU-bound workers at once, which is why the _main_ thread stalls up to 144s
  while "idle".
- Idle main-thread stalls on the Linux host also capture pure render work:
  `widgets/prompt_panel/_section_navigation.py:89,142,165` (strips/anchors/height),
  `util/lazy_syntax.py:122` re-rendering, and `util/frontmatter_syntax.py:28` markdown
  lexing — multi-second renders 300+ seconds after the last keypress.
- `agents_sync` status checks (600s cadence) do real `git fetch` +
  `git cat-file --batch` reads where `_read_until_nul`
  (`src/sase/agents_sync/git_objects.py:246-252`) reads **one byte at a time**; the
  prompt-panel detail header does a per-bead `show()` N+1
  (`widgets/prompt_panel/_agent_display_header_summary.py:512` →
  `src/sase/bead/store_locator.py:110`).

### Chops: one fresh interpreter per chop per tick, and nothing ever short-circuits

- Every eligible chop is spawned as its own console-script subprocess every tick:
  `src/sase/axe/lumberjack.py:217-225` (unbounded `ThreadPoolExecutor`, one thread per
  chop, each blocking on `Popen`) → `src/sase/axe/chop_script_runner.py:147-159`.
- Shipped cadences (`src/sase/default_config.yml`, `axe:` block from line 769): `hooks`
  interval **5s** with 8 chops, `waits` **10s** with 4, `comments` 60s, `checks` 300s,
  `external_mirror` 900s, `housekeeping` 3600s. The `schedule` library re-arms from the
  _end_ of each tick, so effective cadence is interval + tick duration (hundreds of
  "Tick overrun" warnings, `src/sase/axe/lumberjack.py:245-250`).
- Each chop subprocess imports **1,251 modules (871 sase.\*)** in ~0.53–0.70s:
  `src/sase/chops/sdk.py:19` imports `sase.axe.chop_script_context`, which executes
  `src/sase/axe/__init__.py`, whose lines 7–9 eagerly import `Lumberjack`,
  `read_recent_lifecycle_events`, and `Orchestrator` — dragging in `sase.ace`,
  `sase.agents_sync`, `sase.xprompt`, `sase.llm_provider`, `sase.notification`, and more
  into every chop that will never schedule anything. `src/sase/chops/builtin.py:9-18`
  adds `sase.ace.patch` and friends. There is no fast path for chop entrypoints
  (`src/sase/main/entry.py:14-26` fast-paths only `sase bead` and completion).
- A complete pre-spawn guard surface already exists and is evaluated in the parent
  before `Popen`: `run_every` (`lumberjack.py:206-215`), `enabled`, declarative
  `inhibit_if` guards and `trigger` providers via `evaluate_chop_preflight`
  (`src/sase/axe/chop_runner_script.py:118-136`, `src/sase/axe/chop_policy.py:96-179`,
  Rust engine in `sase-core/crates/sase_core/src/axe_chop/{config.rs,decision.rs}`),
  plus live-run dedupe. **Not one shipped or user-configured chop declares any of them**
  — every chop takes the default `always` trigger. The only trigger providers today are
  `always` and `git.commits_since`; there is no filesystem/change-token provider.

### Scan chops: the expensive walk runs before the cheap check

- `wait_checks` (`src/sase/scripts/sase_chop_wait_checks.py:73-108`) iterates every
  project, every `ace-run` artifact dir, and `json.load`s every `agent_meta.json`
  _before_ looking at `waiting.json`/`ready.json` markers. The 21MB sqlite agent
  artifact index (`query_agent_artifact_index`,
  `src/sase/core/agent_scan_facade.py:364-380`) already serves six other callers but no
  chop.
- `bead_claim_checks` (`src/sase/scripts/sase_chop_bead_claim_checks.py:392,404-413`)
  has a cheap "no candidates" early return — but computes it _from_ the full
  `scan_agent_artifacts` filesystem walk, so the expensive part is unconditional.

### Workspace provisioning: N workspaces × same release build

- Workspace install runs `maturin develop --release` against the workspace's own linked
  sase-core checkout with a per-workspace cargo `target/`; the same commit rebuilds per
  workspace (and again after re-clones). A prebuilt-wheel injection point already
  exists: the `SASE_CORE_WHEEL` env var branch in the install recipe (justfile) — but
  nothing populates or reuses it automatically.

## Design principles (the "don't break anything" contract)

1. **No cadence changes.** Shipped intervals stay as they are; ticks get cheap instead
   of rare. Hook/wait reaction latency is unchanged.
2. **No execution-model changes.** Chops stay isolated subprocesses with today's
   timeout, kill, dedupe, and result semantics. We do not run chop bodies in the
   lumberjack process, and we do not touch the `schedule` re-arm semantics.
3. **Every new skip path fails open.** A trigger that cannot compute its token, a cache
   that cannot prove freshness, a wheel cache that cannot prove an exact match — all
   fall back to today's behavior (fire / full reload / full build).
4. **Skips are bounded by sanity re-fires.** Change-token triggers carry a `max_quiet`
   fallback; ace keeps a periodic full reconcile; nothing can be suppressed forever by a
   missed event.
5. **Old paths stay reachable while confidence builds.** The ace refresh rework lands
   behind a `sunset`-kind feature flag (created with `sase flag new` by that phase, per
   the flags policy) so the unconditional-refresh branch can be re-enabled instantly.
6. **Identical outputs on the non-skip path.** The scan-chop rework must produce
   byte-identical results/markers whenever it does run the full path; tests assert
   parity between old and new implementations on fixture trees.

## Phases

### Phase `chop-import-slim` — slim chop subprocess imports (no deps)

Make `import sase.scripts.sase_chop_<x>` cheap:

- Break the eager import chain: `src/sase/axe/__init__.py:7-9` must stop importing
  `Lumberjack`/`Orchestrator`/lifecycle at module import (module-level `__getattr__`
  lazy re-export is the established pattern); audit `src/sase/chops/sdk.py` and
  `src/sase/chops/builtin.py:9-18` so the chop SDK pulls only what chop bodies need;
  push remaining heavy imports (`sase.ace.*`, `sase.xprompt`, `sase.llm_provider`,
  `sase.agents_sync`) behind function-local imports on the paths that actually use them.
- Verify with `-X importtime` before/after; target **< 0.2s / < 400 modules** for a
  representative chop script import on the dev host.
- Add a regression test asserting the chop-SDK import closure excludes named heavy
  packages (e.g. importing `sase.chops.sdk` in a subprocess and asserting
  `sase.xprompt`, `sase.ace.tui`, `sase.llm_provider` are absent from `sys.modules`).
  Budget-by-module-set is deterministic; wall-clock budgets belong in `perf-guardrails`.
- Symvision and import-cycle gates must stay green; behavior of `sase axe` commands,
  orchestrator, and all chop entrypoints unchanged.

### Phase `chop-trigger-provider` — fs change-token trigger provider (no deps)

Add the missing cheap trigger to the existing preflight engine. **This crosses the Rust
core boundary**: implement the provider in `sase-core/crates/sase_core/src/axe_chop/`
(config parse + decision evaluation + unit tests), expose it through the binding, then
wire the thin Python side — open the sase-core repo via `sase repo open` / the
`/sase_repo` skill.

- Semantics: a trigger declares a small list of watch specs (absolute-or-state-root
  relative paths, optional shallow glob). Each preflight computes a cheap state token
  (per path: exists/mtime*ns/size, plus child count for dirs — no recursion, no content
  reads) and fires when the token differs from the last _fired* token, persisting the
  token in the chop's lumberjack state alongside existing run state.
- Required `max_quiet: <duration>` field: fire unconditionally when that long has passed
  since the last fire, so a missed/unobservable change can only delay work, never lose
  it.
- Fail open: token computation error, missing persisted state, schema mismatch, or
  unreadable path ⇒ `fire`.
- Schema: extend `src/sase/config/sase.schema.json` trigger definitions; document in the
  same places `git.commits_since` is documented; `sase axe chop-doctor`
  (`src/sase/axe/chop_doctor.py`) should surface the evaluated decision.
- Ships dark: no shipped chop config uses it yet (that is `chop-guard-defaults`), so
  this phase is pure additive surface with both-repo tests.

### Phase `chop-guard-defaults` — wire guards into shipped defaults (deps: chop-trigger-provider)

For each high-frequency shipped chop in `src/sase/default_config.yml`, either attach an
fs change-token trigger naming its actual inputs, or `run_every` where inputs are
inherently time-based. The phase must read each chop's source and verify its real input
surface before writing the trigger — the table below is the starting map, not the
finished one:

- `hooks` lane (8 chops, interval 5): patch/hook state under the project stores and the
  lumberjack tick files (`hook_checks`, `mentor_checks`, `workflow_checks`,
  `pending_checks_poll`, `comment_zombie_checks`, `suffix_transforms`, `orphan_cleanup`,
  `stale_running_cleanup` — several key off agent artifact dirs and checks state).
- `waits` lane: `wait_checks` keys off waiting/ready markers and agent artifact dirs;
  `bead_claim_checks` off claim artifacts and bead stores.
- Leave `comments`, `checks`, `external_mirror`, `housekeeping` (≥60s lanes) and
  anything network-polling on `always` — the win there is small and network inputs have
  no fs token.
- Choose `max_quiet` per chop ≈ a few multiples of today's effective cadence (e.g.
  60–300s for the 5s/10s lanes) so worst-case reaction latency under a missed token is
  still minutes, not never.
- Tests per chop: (a) idle tick ⇒ no spawn; (b) touching a watched input ⇒ next tick
  spawns; (c) `max_quiet` elapsed ⇒ spawns. Plus one end-to-end lumberjack test
  asserting an idle tick performs zero `Popen` calls.
- Expected effect (with `chop-import-slim`): idle chop spawn rate drops from ~109 to
  single digits per minute; a quiet 5s tick costs a handful of `stat()` calls.

### Phase `chop-incremental-scans` — invert the scan chops (no deps)

- `wait_checks`: read waiting markers first; exit no_op before any artifact walk when
  there are no waiting entries (or all are already satisfied per existing ready
  markers). When resolution is needed, resolve only the referenced dependencies — via
  `query_agent_artifact_index` if the index provides the needed fields (verify; it
  serves six other callers today), else via targeted per-artifact reads. Keep a
  full-walk fallback path selectable for parity testing; counters in the result JSON
  (`artifacts=` etc.) must keep reporting truthfully.
- `bead_claim_checks`: compute the cheap candidate pre-pass from marker/tombstone state
  _without_ the full `scan_agent_artifacts` walk, and only walk when candidates exist.
  Preserve the reconciled-tombstone semantics exactly.
- Parity tests on fixture trees: new implementation's markers/results/launch proposals
  byte-identical to the old on populated and empty stores; measure and record
  before/after wall time in the phase notes.

### Phase `ace-refresh-tokens` — per-surface change tokens (no deps, large)

Restore the dirty-gate design without depending on an fs event watcher:

- Introduce cheap per-surface **change tokens** computed by stat-level probes (no
  content reads): agents (project artifact roots), axe (lumberjack status/metrics
  mtimes), notifications, patches, procs (`procs.jsonl` mtime/size). A surface reloads
  on a tick only when its token changed since the last completed load; a configurable
  sanity interval (default ~5 min) forces a full reconcile so a token blind spot can
  only delay, never lose, an update.
- On watcher-active hosts, tokens complement events (events set dirty flags as today;
  tokens replace the `not watcher_active ⇒ reload everything` branch at
  `actions/event_refresh/_auto_refresh.py:113-125`).
- Gate `ProcObserver.poll_once` reads on the proc-store token (2Hz stat is fine; 2Hz
  full read+parse is not).
- Bound worker pile-up: the refresh path must not launch a new broad load while the
  previous one is still running (extend the existing coalescing flags), and heavy
  warmups (bead cache, family previews) run only after their surface actually changed.
- Create a `sunset` feature flag via `sase flag new` (this phase, not by hand) whose Off
  branch restores today's unconditional-refresh behavior; both states tested per the
  flags policy.
- Record before/after: idle ace CPU (target **< 10% of a core**), stall-watchdog record
  count over a 30-minute idle session (target ~0), and RSS trajectory — the current
  build climbs past 1.2GB within minutes of startup; if growth persists once idle
  reloads stop, file a follow-up with the findings.
- This is the delicate phase: it touches the refresh spine described in the `tui_perf`
  memory note (which the implementing agent must read); `large` so the worker plans
  first.

### Phase `ace-axe-status-cache` — cache immutable axe reads (deps: ace-refresh-tokens)

- Chop run records are write-once: cache parsed run JSON keyed by (path, mtime_ns, size)
  across ticks; re-read the run index only when its mtime changed.
- Tail a lumberjack/chop log only when its size grew; skip per-run log tails entirely
  except for the runs actually rendered.
- Collect full per-chop snapshots only when the Axe tab is visible (or a zoomed axe
  panel needs them); other tabs need only the summary header fields. The sanity
  reconcile from `ace-refresh-tokens` still refreshes everything periodically.
- Target: a background tick with no axe changes performs zero JSON parses of
  previously-seen run records; measure per-tick file-open count before/after.

### Phase `ace-idle-render-cache` — stop idle re-renders (no deps)

- Diagnose why `_section_navigation` strips/anchors/heights and `lazy_syntax` re-render
  for unchanged content on idle refreshes (stall stacks:
  `widgets/prompt_panel/_section_navigation.py:89,142,165`, `util/lazy_syntax.py:122`,
  `util/frontmatter_syntax.py:28`) — likely cache keys tied to renderable identity that
  churns on refresh rather than content.
- Key render caches by content hash + width + style token so an unchanged document
  renders exactly once regardless of refresh count; keep caches bounded.
- Verify with the stall watchdog: an idle 30-minute session on a host with a large
  prompt-panel document produces no render-path stall records; j/k latency stays within
  the existing bench budgets (`tests/ace/tui/bench_tui_jk.py`).

### Phase `ace-io-hygiene` — small I/O fixes (no deps, small)

- `src/sase/agents_sync/git_objects.py:246-252`: `_read_until_nul` reads one byte per
  syscall from the `git cat-file --batch` pipe; switch to a buffered reader preserving
  the NUL-delimited protocol (headers are newline-terminated; bodies are length-prefixed
  — read exactly, not byte-wise).
- `widgets/prompt_panel/_agent_display_header_summary.py:512` +
  `src/sase/bead/store_locator.py:110`: batch the per-bead `show()` calls into one store
  query (or a single Rust-side multi-get if the binding offers one; if it does not, note
  the boundary and keep the batch in Python).
- Straightforward wins; both covered by existing test surfaces plus targeted units.

### Phase `core-build-cache` — reuse sase-core release builds (no deps)

- Add a host-level wheel cache (e.g. `~/.sase/cache/sase-core-wheels/`) keyed by **exact
  sase-core commit SHA + dirty-state + rustc/toolchain version + python ABI tag +
  sase_core_py crate hash inputs**. The install path (justfile `rust-install` recipe and
  the `SASE_CORE_WHEEL` branch that already exists) consults the cache: exact hit ⇒
  `uv pip install` the cached wheel; any miss, dirty checkout, or key-computation
  failure ⇒ today's `maturin develop --release`, then (clean checkouts only)
  `maturin build --release` output is stored back to the cache.
- Concurrency-safe cache writes (write to temp + atomic rename; last writer wins on
  identical keys); bounded cache size with LRU pruning.
- Must not change behavior for: prebuilt `SASE_CORE_WHEEL` users, hosts without the
  sase-core checkout (published wheel path), stale-core floor validation
  (`tools/validate_test_environment` / `validate_sase_core_rs` still run and still
  gate). A dirty or ahead/behind checkout never serves from cache.
- Test via the existing install-validation tooling plus a shell-level test of the cache
  key function; verify on a real workspace pair that the second workspace installs from
  cache in seconds and produces an identical extension (hash compare).

### Phase `perf-guardrails` — counters, budgets, runbook (deps: chop-import-slim, chop-guard-defaults, ace-refresh-tokens)

- Lumberjack metrics gain `chops_skipped` (per skip reason: trigger, run_every,
  inhibited) alongside `chops_executed`, and per-tick spawn counts land in the existing
  status/metrics JSON so `sase axe status` surfaces spawn rate and no-op ratio directly.
- ace perf logging gains per-tick counters (surfaces reloaded, files opened by the axe
  collector) in the existing `SASE_TUI_PERF`/trace channels.
- Lock in budgets as tests where deterministic (module-set import budget from
  `chop-import-slim`; zero-spawn idle tick from `chop-guard-defaults`); wall-clock
  numbers go in `docs/perf_runbook.md` with the capture commands used here (`/proc`
  deltas, stall-watchdog counts, lumberjack run-history gap analysis).
- Update `docs/perf_runbook.md` with the idle-host measurement recipe so the next
  regression is a five-minute diagnosis, and propose updates to the `tui_perf` memory
  note through the proper memory-write channel (not direct edits).

## Non-goals

- **No shipped cadence/interval changes** and no `schedule` re-arm semantics changes —
  guards make ticks cheap instead of rare.
- **No in-process or batched chop execution.** Isolation/timeout semantics stay.
- **User-config lanes are out of scope.** The `telegram` lane (interval 5, Bot API
  polling) lives in the user's own config and the sase-telegram plugin; this epic makes
  its per-spawn cost small via `chop-import-slim`, and a follow-up recommendation to add
  `run_every`/trigger guards to that user config belongs in the dotfiles repo, not here.
- **Agent-runner streaming costs** (~5–11% of a core per active runner) are proportional
  to real agent work and are not addressed here.
- **No inotify/FSEvents watcher for macOS.** Change tokens make the watcher an
  optimization rather than a requirement; a native macOS watcher can be a later
  follow-up if token probes ever show up in profiles.

## Success criteria

- Idle host (ace open, lumberjacks running, no agent work): sase's sustained CPU drops
  from ~4 cores to **< 0.4 cores**, measured by the runbook recipe.
- Chop subprocess spawns on an idle host: from ~109/min to **< 10/min**.
- Representative chop import: from ~0.6s / 1,251 modules to **< 0.2s / < 400 modules**.
- Idle ace: **< 10% of a core**, zero stall-watchdog records in a 30-minute idle
  session, per-tick axe file opens near zero when nothing changed.
- Second workspace install of an already-built sase-core commit completes the Rust step
  in **seconds, not minutes**, with an identical extension artifact.
- `just check-full` green at every phase; no chop result/marker output changes on
  non-skip paths (parity tests).
