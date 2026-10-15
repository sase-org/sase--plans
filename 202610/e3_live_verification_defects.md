---
tier: tale
title: Fix E3's remaining live verification defects
goal:
  "E3's readout, verify, coverage, and conformance report the real delivery state, so
  the four flag soaks are judged on correct numbers: successful handoffs never count as
  misses, Codex evening rollouts are found, and sessions a run never launched are
  excluded."
size: medium
proposed_by: bbugyi200.athena.10i
create_time: 2026-10-15 06:15:47
status: wip
---

# Plan: Fix E3's remaining live defects: false readout misses, missed Codex evening sessions, and unrecorded sessions counted as the run's

## Context

The research report
`research:202610/instruction_bundles_vs_readable_agents_md/instruction_bundles_vs_readable_agents_md.md`
(written 2026-10-14 ~14:00Z, against `88acef6234`) named three live E3 defects. I
re-checked all three on 2026-10-15 at master `be73b6021a`. **None of them needs more
work. Do not redo them:**

- **Codex fallback drops the home layer.** Fixed in `d070e6b2dc`.
  `src/sase/llm_provider/_codex_home.py` now keys suppression to
  `active_explicit_codex_delivery()`, not to the flag.
- **Muse explicit runs show the bundle 2×.** This was an observer artifact: Muse 1.4.7
  stores one prompt in several records. It was fixed in `d070e6b2dc`
  (`instructions/muse.py::_prompt_prefixes` dedupe), and the landing agent verified
  `research.17.mus` at 1×. Muse has had no runs since (quota until 2026-10-19).
- **Claude runs still mostly legacy (install skew).** Resolved. athena and apollo both
  run editable installs at `be73b6021a`. The readout shows every Claude root run since
  2026-10-14T16:57Z as `explicit`.

The live re-check found **three other E3 defects**, all in E3's verification tooling.
They report healthy delivery as failures. They also block the removal conditions on the
four sunset flag beads (`sase-1kv` claude, `sase-1ks` codex, `sase-1kt` grok, `sase-1ku`
muse), which require "healthy instruction conformance, and accepted readout results".

### Defect 1: the readout scores successful Grok/Codex/Muse handoffs as misses

`sase instructions readout -s 2026-10-14T10:35:00Z` on athena reports these explicit
miss rates:

- Grok `≤7/36 (19%)`, against a pre-cutover baseline of about 7%. That is past the
  soak's "more than 2 points above baseline" disable trigger.
- Codex `≤1/44`.

**All 8 are successful monitor handoffs.** Each run's `agent_meta.json` has a
`lifecycle_phases` entry `{"phase": "handoff"}` plus `monitor_id` /
`monitor_member_agent_name`. For example, `10f--code`
(`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202610/14/20261014105726`) has
`monitor_id: 9m6zn3csd5wf` and `monitor_member_agent_name: 10f--mon`, and its follow-up
`10f--2` exists.

The other 7 runs are the Grok runs `toobig-81.test_release_watch.0--plan`,
`bob-cli-7n.4--plan`, `10f--2`, `sase-1kx.5.1--1`, `sase-1kx.5.1--2`, and
`sase-1kx.2--plan`, plus the Codex run `sase-1kx.4--plan`.

Root cause: `src/sase/instructions/readout.py::_classify_artifact_dir` recognizes
handoffs only by scanning the `tool_calls.jsonl` tail for command text such as
`sase monitor start`. For non-Claude providers, the handoff command's own tool call is
never recorded, because the handoff SIGTERMs the runner process group before the call is
flushed. Grok's session (`~/.grok/sessions/<cwd>/f77f5c58-…/chat_history.jsonl`) ends
with the `sase monitor start -p verify -f … -- just check` call, but `tool_calls.jsonl`
ends at the previous `sase final prepare`.

The same flaw inflates the pre-cutover baselines. On athena since 2026-09-10, these
"misses" all have host handoff evidence:

- Grok legacy: 13 of 13.
- Muse legacy: 24 of 24.
- Grok `none`: 19 of 22.
- Muse `none`: 49 of 51.

The baseline JSONs attached to `sase-1ko` (notes #5, #6, #12, #13) were computed with
this classifier.

### Defect 2: Codex rollouts in local-date directories are never found

`src/sase/instructions/run_index.py::find_codex_sessions` builds its
`~/.codex/sessions/YYYY/MM/DD` directory list from the **UTC** dates of
`run.started_at..end`. Codex files a rollout under its **local** start date. Any Codex
run that starts between 00:00Z and local midnight (US evening) is therefore invisible.

`sase doctor -D -C instructions` currently WARNs `instructions.conformance` with
`no matching root session` for `sase-1kx.5.3.3.1--1`, `sase-1kx.5.3.3.1--plan`,
`sase-1kx.5.3.land--plan`, and `sase-1kx.5.3.2--1`. All four were rendered between
2026-10-15T02:11Z and 03:19Z. Each manifest records its thread id, for example
`01a13d94-c6ae-70c0-bb68-e268763dfae7`. That rollout exists at
`~/.codex/sessions/2026/10/14/rollout-2026-10-14T23-22-02-01a13d94-….jsonl`, and its
`session_meta` timestamp is `2026-10-15T03:22:02Z`.

The finder is E1 code. It became a live WARN through E3's conformance check, and it also
hides those sessions from `verify`, coverage, and section diffs.

### Defect 3: sessions the run never launched are scored as the run's

`find_claude_sessions` (and the Grok and Codex finders) collect **every** transcript in
the workspace's provider directory inside the run's time window. Run
`research.4g.linker.w1.w0--plan` (bob-cli, Claude, explicit) shows the problem:

- Its root session `2d2b405f-…` carries the bundle marker.
- Five other sessions (`12f46cff-…`, `5d577ed1-…`, `7034b54b-…`, `efd36f3d-…`,
  `fd63a268-…`) are nested `claude -p` calls. bob-cli's `capture-complete` spawned them
  while the agent tested it. Their prompts are `@body+excercise *` and similar, and
  their `entrypoint` is `sdk-cli`.
- The manifest's `delivery.session_ids` lists only `2d2b405f-…`.

These sessions currently:

- WARN `instructions.coverage` ("uncovered sessions: research.4g.linker.w1.w0--plan/…");
- make Claude's `verify` row read `native-full 2×`, `foreign auto-memory`, and
  `contract 1× [1,2]`, so Claude still looks partly legacy (this is part of what the
  research read as "Claude still legacy");
- print as `◌ no matching manifest or unverifiable session` under `verify -a`.

`coverage_records.session_is_foreign` only rejects sessions that carry _another
invocation's_ marker. Marker-less sessions fall through, and since the landing fix they
match nothing, so they count as uncovered.

All four adapters record their provider session ids through
`ActiveInstructionDelivery.record_session`, so a recorded id is the precise claim.

There is a related latent bug. For Codex, `RootSession.session_id` is the rollout
filename stem (`rollout-2026-10-14T23-22-02-<thread-id>`), but manifests record the bare
`<thread-id>`. So `matching_manifests`' "recorded `session_ids`" branch can never match
a marker-less Codex session, for example a Codex render-failure fallback. That session
is then uncovered, because rule 3 does not time-match records that recorded ids. Claude
(transcript stem) and Grok (session directory name) already record the same string that
discovery uses.

## Changes

### 1. Readout: classify handoffs from host records before tool-call text

In `src/sase/instructions/readout.py`:

- Give `_classify_artifact_dir` an optional `meta` argument. `build_readout` already
  reads `agent_meta.json`, so pass it in at both call sites (provisional and finished).
  Otherwise read it inside.
- Keep "accepted submission" first.
- Then add a private host-evidence step that returns a class or `None`:
  - `None` when `handoff_aborted.json` exists (the runner's aborted-handoff record, see
    `src/sase/agent/handoff_inflight.py`).
  - `None` when no entry in `meta["lifecycle_phases"]` has `phase == "handoff"`. The
    runner marks this phase whenever it adopts a pending handoff marker
    (`src/sase/axe/run_agent_exec.py`, `run_agent_exec_plan.py`,
    `run_agent_exec_pipe.py`, `run_agent_successor.py`).
  - `PLAN_HANDOFF` when `plan_path.json` exists. Only the plan-handoff handler writes it
    (`run_agent_exec_plan.py`).
  - `MONITOR_PREPARED` when `meta` has `monitor_id` or `monitor_member_agent_name` and
    the run has a prepared completion intent. That means either
    `continuation/completion_intents/index.json` has a non-empty `intents` object, or
    `_has_real_prepare(tail) != -1`. Otherwise `MONITOR_PLAIN`.
  - `OTHER_HANDOFF` for any other adopted handoff (questions, gate, pipe, or successor).
- Then keep the existing `tool_calls.jsonl` tail scan unchanged as the fallback for runs
  that predate lifecycle phases, followed by not-required and miss.
- Read each file at most once per run, and keep the helpers private.

### 2. Codex rollout discovery covers local-date directories

In `run_index.find_codex_sessions`, widen the scanned directory range by one day on each
side: from `(run.started_at - 1 day).date()` through `(end + 1 day).date()`.

A local date always lies within ±1 day of the UTC date, so this is exact for any
timezone. `_codex_rollout_matches` still requires a `session_meta` cwd and timestamp
inside `[run.started_at, end]`, so the wider scan cannot add false matches. Keep the
366-day guard. Add a short comment saying Codex files rollouts by local date.

### 3. A session the run's adapter never recorded is foreign

In `src/sase/instructions/coverage_records.py`:

- Add a private helper `_session_recorded(session, record) -> bool`. It is true when
  `session.session_id` is in `record.session_ids`. For `session.provider == "codex"`, it
  is also true when the stem ends with `-<recorded id>`.
  - Use it in `matching_manifests`' session-id branch. This fixes marker-less Codex
    sessions.
  - Use it in the new rule below.
- Extend `session_is_foreign(session, records)`. Keep the existing foreign-marker rule,
  and **also** return `True` for a marker-less session when all of these hold:
  - the run has at least one same-provider, non-helper root record;
  - every such record recorded `session_ids`;
  - none of them records this session.

  Update its docstring. When a record recorded no ids (legacy, shadow, or
  pre-writeback), the rule does not apply, and time-matching stays as it is.

  This is safe because an explicit record whose real session gets misrecorded still
  WARNs `no matching root session` in conformance.

- Every existing caller already uses `session_is_foreign`, so this one change covers
  them all:
  - verify observation drop: `_verify_collection._drop_foreign_observations`;
  - coverage: `coverage_summary.cover_sessions` and `purpose_coverage_rows`;
  - the conformance association in `src/sase/doctor/checks_instructions.py`.

  Confirm each caller passes the run's records, not records from all runs.

- `verify -a`: in `src/sase/main/instructions_handler.py`, skip section diffs for
  foreign sessions. Print one line per foreign session instead, for example
  `<run>/<session>: ⊘ foreign — not recorded by this run; not scored`. Use
  `render_section_diffs` or a sibling in `src/sase/instructions/render.py`, so `-a`
  stays transparent about what was excluded. Keep JSON output stable. If `-j` includes
  per-session diffs, mark foreign sessions with a field instead of dropping them
  silently. Otherwise leave JSON alone.

### 4. Docs

In `docs/agent_providers.md`:

- "Verifying instruction delivery", association paragraph: add the new rule. A
  marker-less session that no same-provider root record lists, when all of them recorded
  session ids, is not the run's session. A nested `claude -p` started by a tool inside
  the workspace is the typical case. verify, coverage, and conformance skip such
  sessions, and `-a` labels them foreign. Also note that Codex rollout stems match
  recorded thread ids, and that Codex rollouts are found in local-date directories.
- "Soak readout": in the class list, say that handoffs are first recognized from host
  records:
  - the lifecycle `handoff` phase;
  - `plan_path.json` for a plan handoff;
  - `monitor_id` with a prepared completion intent, or without one;
  - other adopted handoffs.

  Say that an aborted handoff (`handoff_aborted.json`) is never a handoff. Say that
  tool-call text is the fallback for older runs. Explain why: non-Claude providers do
  not record the handoff call itself.

  Add one sentence noting that readouts computed before this fix overcount misses for
  Grok, Codex, and Muse.

Check `docs/cli.md` and `docs/instruction_bundles.md` for any wording this change makes
false, and fix it.

### 5. Tests

Use the existing fixture helpers. Put new tests in the smaller modules, and keep every
test file under 700 lines (`toobig` thresholds 1000/850/700). `test_verify_cli.py` is
already at 667 lines, so do not grow it.

- `tests/instructions/test_readout.py`:
  - handoff phase plus `monitor_id` with no tool-call text → monitor (plain);
  - the same with `continuation/completion_intents/index.json` intents →
    monitor-prepared;
  - handoff phase plus `plan_path.json` → plan;
  - handoff phase alone → other;
  - `handoff_aborted.json` with a handoff phase and no submission → miss;
  - `final_submission.json` still beats host evidence;
  - the existing tail-text tests stay green.
- `tests/instructions/test_run_discovery.py` (or `test_scoreboard_sessions.py`): a run
  started `2026-10-15T03:19Z` finds a rollout filed under `2026/10/14` whose
  `session_meta` timestamp is `2026-10-15T03:22Z`. A rollout in an adjacent day
  directory with a timestamp outside the window is still excluded.
- Coverage and conformance (`test_scoreboard_coverage.py`,
  `test_scoreboard_conformance.py`, `test_scoreboard_sessions.py`, or a new
  `test_scoreboard_foreign.py`). Build fixtures with the real manifest and
  `delivered_text` writers, as the existing scoreboard tests do.
  - An explicit Claude run has a marked root session listed in `session_ids` plus two
    marker-less unlisted sessions in the same window. Expect:
    - the extra sessions are foreign;
    - they are absent from the provider row and observations;
    - they are not counted uncovered;
    - conformance is OK.
  - A record with empty `session_ids` still time-matches a marker-less session (legacy
    path unchanged).
  - A marker-less Codex session whose rollout stem ends with the recorded thread id
    matches by session id (for example an `inherits_native` record with `session_ids`).
  - An explicit record whose only session is unlisted and marker-less still WARNs
    `no matching root session`.
- `verify -a` prints the foreign line for an unlisted session. Use a small CLI or render
  test.

## Verify

1. Run `just fmt`, then `sase tool run check`. Do **not** run `just check-full`.
2. Live checks from this checkout's venv (`.venv/bin/sase …`). The named runs live in
   athena's corpus, so run these when `hostname` is `athena`. On another host, run only
   the readout and doctor checks against that host's corpus, and say so in your summary.
   - `.venv/bin/sase instructions readout -s 2026-10-14T10:35:00Z`: the Grok explicit
     row drops from `≤7/36` to `≤0/36`, and Codex explicit from `≤1/44` to `≤0/44`.
     Monitor counts rise by the same amounts. Investigate any miss that remains.
   - `.venv/bin/sase instructions verify -a research.4g.linker.w1.w0--plan`:
     - one scored session, `2d2b405f-…`;
     - five `⊘ foreign` lines;
     - Claude row contract `1×`, foreign `—`, and native-full `0` or `◌` (the root
       transcript is partial, over the read cap).
   - `.venv/bin/sase instructions verify -a sase-1kx.5.3.3.1--1` (and the other three
     Codex agents above): one explicit session each, bundle `1×`.
   - `.venv/bin/sase instructions verify -s 2026-10-14T16:00:00Z -p claude`: no
     `native-full 2×` or `auto-memory` caused by nested sessions.
   - `.venv/bin/sase doctor -D -C instructions`: `instructions.coverage` no longer names
     the nested bob-cli sessions, and `instructions.conformance` no longer reports the
     four Codex `no matching root session` runs. Both checks use a 24 h window, so
     either may already have aged out; say which. Any other WARN must be traced to a
     real delivery fault and reported, not suppressed.
3. Record corrected readouts. The flag beads' removal condition depends on "accepted
   readout results". With the fixed classifier, run these and keep the JSON output,
   naming the host:
   - `.venv/bin/sase instructions readout -j` for
     `-s 2026-09-10T00:00:00Z -u 2026-10-05T00:00:00Z`;
   - `-s 2026-10-06T00:00:00Z -u 2026-10-14T10:35:06Z`;
   - `-s 2026-10-14T10:35:00Z`.

   Then append one `sase bead note` to each of `sase-1kv`, `sase-1ks`, `sase-1kt`, and
   `sase-1ku`. Each note states that provider's corrected pre-cutover and post-cutover
   miss rates. It also says that the `sase-1ko` baseline attachments (notes #5, #6, #12,
   #13) predate this fix and overcount misses for providers whose final handoff call is
   not recorded. Attach the three JSON files to `sase-1kt` with `sase bead attach`; Grok
   is the provider whose soak verdict changes. Do not change any flag state, and never
   run `sase flag disable`.

## Non-goals

- The research's E4 re-scope (keep a committed full `AGENTS.md`,
  `sase init memory --check` in CI, header, and shim deletion), the TUI "as launched"
  row, and the export home-metadata leak (tracked as `sase-1l2`). These are not live E3
  defects.
- agy `home ✗` predates E3 (the same result appears in September runs).
- Making non-Claude providers record the in-flight handoff tool call in
  `tool_calls.jsonl`. The readout becomes authoritative from host records instead.
- The soak verdicts themselves. They stay with each flag bead's FlagTriage after 7 days.
