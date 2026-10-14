---
tier: tale
size: medium
title: 'E3 landing finish: Codex fallback home discovery, honest conformance, epic
  closeout'
goal: Codex keeps native home discovery whenever it has no active explicit delivery.
  The scoreboard, coverage, and instructions.conformance attribute each session only
  to its own invocation, count Muse deliveries once, see render failures, and come
  back clean on the live corpus apart from genuine fallbacks. Epic sase-1ko is closed
  with its plan marked done.
proposed_by: bbugyi200.apollo.sase-1ko.land
bead: sase-1ko
status: done
---

- **PARENT:**
  [202610/e3_explicit_delivery_cutover.md](https://github.com/sase-org/sase--plans/blob/main/202610/e3_explicit_delivery_cutover.md)
- **BEAD:**
  [sase-1ko](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ko/README.md)

# Plan: finish landing E3 (epic `sase-1ko`)

## Context

Epic `sase-1ko` ("E3 (instruction migration): Explicit instruction delivery cutover",
plan `plan:202610/e3_explicit_delivery_cutover.md`) has all ten phases closed. Its land
agent checked every phase against the code, the commits, and the live corpus. The
primary checkout (the installed `sase`) is at `508eb963a3` and carries every E3 adapter
commit, so real launched agents now run with explicit delivery. Run
`sase bead read sase-1ko -r "<why>"` first. The land agent's triage note records every
follow-up outcome, so do **not** re-triage the phase follow-ups.

Verified complete (do not redo): manifest wire v2 and pin; delivery-plan hook, boundary,
0644 bundles, atomic fallback, deduplicated notifications; `sase instructions show`,
`readout`, and dismissed-run discovery; Claude, Codex, Grok, and Muse adapters with
their four sunset flags (`sase-1kv`, `sase-1ks`, `sase-1kt`, `sase-1ku`); the decision
record `decisions:instructions-per-invocation`; docs. `just symvision` is clean, and
`sase bead epic-symbols sase-1ko` is empty. The test suites `tests/instructions`,
`tests/llm_provider`, and `tests/test_llm_provider_codex_shadow_home.py` pass (1586
passed). `sase-11s` is now closed.

Two defects caused by the epic remain, and the exit criterion
"`instructions.conformance` is OK" fails on the live corpus. Every fault is in E3 code.
Fix them, re-verify live, then close the epic **in the same turn** (step 8).

## 1. Codex: tie native-home suppression to the active delivery, not the flag

`src/sase/llm_provider/_codex_home.py::_create_shadow_codex_home` picks AGENTS
suppression and `memories/` omission from `explicit_instructions_enabled()`, the flag.
`CodexProvider.invoke` (`src/sase/llm_provider/codex.py`) picks its developer text from
`current_instruction_delivery()`. So when the flag is on and there is no active explicit
delivery, the shadow home drops the `~/AGENTS.md` fallback and any SASE-owned
`~/.codex/AGENTS*.md`, and Codex runs with only the single-turn directive and no home
layer. This happens on render failure, `oversize`, invalid bundle, or no
`artifacts_dir`. That breaks design decision 1 ("without an active explicit delivery, an
adapter behaves exactly as it does today") and decision 5 ("native discovery left
intact"). It happened in production at 2026-10-14T13:03:36Z: agents `research.18.image`
and `research.19.image` each wrote a `00-codex.error.json` (render_failed in a
mixed-version process during a primary sync). Claude
(`_claude_instruction_delivery.py`), Grok, and Muse already key everything to the active
root slot.

- In `_codex_home.py`, decide "explicit" with a small helper that is true only when
  `current_instruction_delivery()` is non-None, its `provider == "codex"`, its
  `root.plan.status == "explicit"`, and its `root.text` is non-empty. This is the same
  condition `CodexProvider.invoke` uses for `explicit_text`; reuse that helper from
  `codex.py` too, so the two can never disagree.
- Key all three behaviors to that helper, matching Claude:
  - skipping SASE-owned `AGENTS.md` / `AGENTS.override.md`;
  - skipping `_link_home_agents_fallback`;
  - omitting `memories/` (foreign memory is off only while explicit delivery is active,
    like Claude's `autoMemoryEnabled: false`).
- The sqlite / `history.jsonl` / `log` / `tmp` allowlist stays unconditional.
- Drop the now-unused `explicit_instructions_enabled` import from `_codex_home.py`.
  `codex.py` still uses it for the hookimpl.
- Tests (`tests/test_llm_provider_codex_shadow_home.py`): `_shadow_home_for` currently
  toggles only the flag. Add a way to bind an explicit Codex delivery. Reuse the
  `_active_delivery(...)` pattern from
  `tests/llm_provider/test_explicit_instruction_providers.py` with
  `bind_instruction_delivery` / `reset_instruction_delivery`. Cover:
  - flag on with an active explicit delivery: no SASE `AGENTS.md`, no `memories`;
  - flag on with **no** active delivery: the `~/AGENTS.md` fallback and `memories` are
    linked, exactly as with the flag off;
  - flag on with a delivery whose root fell back (`inherits_native`): same as legacy;
  - an end-to-end `invoke_with_instructions` render failure with the flag on: the shadow
    home seen by the patched `Popen` env has the home fallback;
  - the existing user-file byte-identity and allowlist tests stay green. Update the
    explicit-delivery tests that only flipped the flag, so they also bind the delivery.

## 2. Session association: a session never matches another invocation's manifest

The live scoreboard and doctor misattribute sessions. Examples are from 2026-10-14;
these agents are archived, and `verify -a NAME` still resolves them:

- `sase-1ko.10--plan` (Grok) picks up three harness-probe sessions from the same
  workspace that carry other invocations' markers, and scores them `bundle 0`.
- `research.17.cdx` has no `ended_at`, so its time window is open. It picks up later
  rollouts in its workspace: one carries a marker for invocation `e72b315d…`, and one is
  an image agent's legacy rollout.
- `7i--1` (Codex, 4 explicit attempts) and `7j--code` (Grok, ordinary plus
  conflict_repair) associate correctly when all of a run's records are considered. The
  conformance check matches each record alone, so every session cross-matches every
  record by time.

Changes:

- `src/sase/instructions/coverage_records.py::matching_manifests`:
  1. If the session carries any bundle marker, match **only** records whose
     `invocation_id` is among the markers. When none match, return `[]`; never fall
     through to session id or time.
  2. Otherwise, match records whose `session_ids` contain the session id.
  3. Otherwise, time-match only records that recorded **no** `session_ids` (legacy,
     shadow, or pre-writeback). An explicit record with recorded session ids never
     claims a session it did not record.
- Add a public helper in the same module (for example
  `session_is_foreign(session, records) -> bool`): the session carries markers, and none
  of them name an invocation among the run's root records.
  - Use it in `src/sase/instructions/_verify_collection.py` to drop such root
    observations (and their Claude helper observations) from a run's results, so they no
    longer pollute provider rows.
  - Use it in `src/sase/instructions/coverage_summary.py`, so a foreign session is not
    reported as uncovered.
  - Marker-less sessions that match nothing stay as unmatched observations. That is
    honest coverage, and `instructions.delivery` already skips them.
- `src/sase/doctor/checks_instructions.py::_check_instructions_conformance`: stop
  calling `matching_manifests(session, [record])` per record.
  - Associate each of the run's root sessions once with
    `matching_manifests(session, roots)[0]` and group sessions by record. Skip foreign
    sessions.
  - Then, per explicit record:
    - WARN `no matching root session` only when no session associated;
    - WARN `root marker N×` when an associated session's count for that record's
      invocation is not 1;
    - keep the existing helper checks.
- Docs: `docs/agent_providers.md` "Verifying instruction delivery". In the association
  paragraph ("A session first matches a manifest by its bundle marker's invocation
  id…"), add that a session carrying another invocation's marker never falls through,
  and that explicit records with recorded session ids are not time-matched.

## 3. Muse: count one delivery once and see every attempt's session

- `src/sase/instructions/muse.py::_prompt_prefixes`: deduplicate identical prefix texts
  in order, like `_rules_texts` and the Claude identical-`prompt_snapshot` dedupe in
  `instructions/claude.py`. Muse 1.4.7 stores one submitted prompt in several records
  (`runtime.user_intent.accepted` `model_messages` and `runtime.session`
  `event.prompt`). Today one delivery reads as `bundle 2×` / `contract 2×`, for example
  `research.17.mus` session `a4f8e1ae-…`.
  - This also fixes `RootSession.bundle_markers`, because `coverage_sources` builds them
    from the same prefixes.
- Muse session discovery: `runs.find_muse_session_id(run)` returns only
  `run_metadata.json`'s last `muse_session_id`. That id belongs to the final retry
  attempt, so earlier attempts' sessions are invisible. In `research.17.mus`, manifests
  `00`/`01`/`02` record sessions `ede7cb5f…`, `ed9c98c6…`, and `a4f8e1ae…`, and all
  three logs exist.
  - Add a helper returning the ordered, deduplicated union of that id and every root
    manifest's `delivery.session_ids` for provider `muse`.
  - Iterate it in both `coverage_sessions.root_sessions` and
    `_verify_observers.observe_muse`. Skip ids whose log is missing.

## 4. Conformance sees render failures

A render failure writes `NN-<provider>.error.json` with `"delivery_fallback": "legacy"`
and notifies `render_failed`. `_check_instructions_conformance` never reports it,
because:

- it selects only runs with an explicit record or a non-declared fallback record;
- it reads only `reason` / `delivery_fallback` (`"legacy"`) from the error document.

The flag-removal condition requires zero non-declared fallbacks. Fix both:

- treat an error entry that carries `delivery_fallback` as reason `render_failed`;
- select runs that have such an entry, still under the newest-10-per-provider and 24 h
  bounds.

The two real 2026-10-14T13:03:36Z Codex render failures will then WARN until they age
out of the 24 h window. That is correct behavior; record them as genuine in the addendum
(step 7).

## 5. `instructions.delivery` must not call unverifiable cells drift

In the explicit branch of `_check_instructions_delivery`, a `None` `native_full_count`
(`◌`, a partial read with no native occurrence) is unverifiable, not drift. Today every
explicit Claude or Codex session over 512 KiB WARNs `native-full ◌`. Treat `None` as
non-offending for native-full, matching the legacy branch, which already maps `◌` to 0.
Keep the other columns as they are.

## 6. Tests

Use fixtures written with the real `delivered_text` and manifest writers, as in
`tests/instructions/test_scoreboard_conformance.py`, `test_scoreboard_doctor.py`, and
`test_scoreboard_coverage.py`:

- association: a foreign-marker session matches nothing, is dropped from verify
  observations, and is not counted uncovered;
- an explicit record with recorded session ids does not time-claim an unlisted
  marker-less session;
- conformance over a run with several explicit attempts, each with its own session, is
  OK (the regression for the per-record bug);
- conformance WARNs on a doubled marker and on an `.error.json` render failure;
- Muse: duplicated prompt records count 1×, and retry-attempt sessions from manifest
  `session_ids` are observed;
- doctor: an explicit partial session with `native_full_count=None` does not WARN.

The existing E1/E2/E3 scoreboard fixture tests must stay green.

## 7. Verify, including live

- `just fmt`, then `sase tool run check` (wrapped). Do **not** run `just check-full`.
- From this checkout's venv, re-score the live corpus:
  - `.venv/bin/sase instructions verify --since 2026-10-14T10:35:00Z`;
  - `-a research.17.mus` (expect bundle 1×, contract 1×, three observed sessions);
  - `-a sase-1ko.10--plan` (no foreign sessions);
  - `-a research.17.cdx` (no foreign-marker rollout);
  - `-a 7i--1` and `-a 7j--code`;
  - `verify -p claude -H` (helper bundle ✓, root marker 0, 0 accepted);
  - `.venv/bin/sase doctor -D -C instructions`. `instructions.conformance` should WARN
    only on genuine non-declared fallbacks. It should name only the two render_failed
    Codex runs above while they are inside the 24 h window. Any other WARN is a defect
    in this work, unless you trace it to a real delivery fault and record it.
- These rows are the launch-probe evidence that the `record` phase left pending. They
  come from real launched root agents on installed E3 code: Claude explicit since about
  2026-10-14T13:09Z, plus Codex, Grok, and Muse. They also supply the Muse live observer
  rows that its 429 quota blocked in the harness. Do not submit new LaunchApprovals, and
  never `sase flag disable`.
- Write `e3_acceptance_addendum.md` outside the repo, and attach it with
  `sase bead attach sase-1ko <file> -N e3_acceptance_addendum.md -n "<one-line summary>"`.
  It holds:
  - the sase revision;
  - the per-provider live rows (bundle, delivery, contract, home, project, directive,
    native-full, foreign, helpers);
  - the doctor summary, with the genuine render_failed runs named;
  - the Codex fallback fix evidence;
  - what stays for the post-landing soak: each provider's 7-day window from its first
    explicit manifest, with FlagTriage on the four flag beads. That is watch work, not
    landing work.

## 8. Close the epic (final step, same turn)

1. Run `sase bead epic-symbols sase-1ko`. For every listed `--epic-symbol` entry,
   resolve the symbol under the Symvision epic-whitelist policy: wire it up, privatize
   it, add a non-test pragma, or delete it. Re-key an entry only when a still-open later
   bead needs the exemption. `sase bead close` refuses while any remain. It is empty
   today; re-check, because your change may add public helpers. Give each a real
   non-test consumer.
2. Close the epic. Never use `--force`, and never use it merely to make the close
   succeed: `sase bead close sase-1ko --note "<verification>"`. The note must cover:
   - the land agent's verification: all ten phases checked against code and commits;
     symvision clean; 1586 instruction/provider tests passing; `sase-11s` closed;
     follow-ups triaged into `sase-1l0`, `sase-1l1`, `sase-1l2`, `sase-1l3` with +1s on
     `sase-1hy`, `sase-1ia`, `sase-1ki` (see the epic's LAND TRIAGE note);
   - this work's fixes;
   - the `sase tool run check` ToolRun id;
   - the live verify and doctor results;
   - the attached addendum.
3. Run `just symvision` and confirm it is clean.
4. Set `status: done` in the frontmatter of the epic's plan file: the PLAN path that
   `sase bead read sase-1ko` prints for `plan:202610/e3_explicit_delivery_cutover.md`,
   in the plans sidecar.
5. `sase-1ko` has no `parent_bead`, so nothing further is required.
