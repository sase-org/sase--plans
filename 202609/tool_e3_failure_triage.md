---
tier: epic
title: "E3: failure triage — every failure labeled, no KNOWN failure hides the rest"
goal: "On a red master, an agent's `sase tool run check` runs past stages whose failures
  are all KNOWN or FLAKY, so its tests still run. It labels every failure item NEW,
  KNOWN, FLAKY, or UNKNOWN with evidence, prints one verdict line, and keeps the exit
  code that fail-fast `just check` would have returned. `sase tool failures` groups the
  machine's red-master signatures, and verify-monitor follow-ups carry the verdict.
  KNOWN precision is proven by a chronological backtest before any agent sees a label.

  "
phases:
  - id: ledger-hygiene
    title: Record runs under the catalog repo's identity and stop nested stage events
    depends_on: []
    size: medium
    description:
      "ledger-hygiene: fix sase-182 (a run's project and fingerprint identity come from
      the catalog's repo, not SASE_PROJECT) and sase-114 plus the nesting guard (a
      run_silent stage's children never append stage events or monitor diagnostics to
      the enclosing run); close both beads."
  - id: core-failure-items
    title: Durable failure items, extractors, and normalization in sase-core
    depends_on: []
    size: large
    description:
      "core-failure-items: add the additive triage tables, the versioned extractor
      registry and normalization, golden fixtures from real athena logs, early
      fingerprint_before persistence, retention (including stage output files), and the
      extract/record/show bindings, in sase-core only."
  - id: core-classification
    title: Pure classification, verdict, and failures aggregation in sase-core
    depends_on:
      - core-failure-items
    size: large
    description:
      "core-classification: implement the witness-based NEW/KNOWN/FLAKY/UNKNOWN rule as
      a deterministic pure function with two tightening knobs, the failure-kind and
      legacy mapping, the verdict, REPEAT detection, owner matching, the store-backed
      stage and settle operations, and failures aggregation, in sase-core only."
  - id: keep-going
    title: Opt-in stage continuation with exit-code parity
    depends_on:
      - ledger-hygiene
    size: medium
    description:
      "keep-going: add the run_silent continuation protocol (SASE_TOOL_CONTINUE
      handshake, continued/stopped/recipe_finished records, --finish), the recipe finish
      line, the executor safety net, and sase tool run -k/-x; opt-in and complete, so no
      flag."
  - id: bindings-and-backtest
    title: Pin the core, gather triage inputs, and pass the precision backtest
    depends_on:
      - ledger-hygiene
      - core-classification
    size: large
    description:
      "bindings-and-backtest: move the core pin and add adapters and validators for
      every new binding, build the bounded input gatherers, write
      tools/tool_triage_backtest, and pass the at-least-95% hand-audited KNOWN precision
      gate on athena before any label is stored."
  - id: record-and-render
    title: Triage every settled run and render it
    depends_on:
      - keep-going
      - bindings-and-backtest
    size: large
    description:
      "record-and-render: capture failed-stage output, run fail-open settle-time triage
      in the shared executor body, persist continuation facts, render the triage block
      and verdict in the footer behind the new tool_failure_triage flag, and add the
      ungated triage section to sase tool show and show -j."
  - id: known-gated-continuation
    title: Continue past all-KNOWN stages by default for agents
    depends_on:
      - record-and-render
    size: medium
    description:
      "known-gated-continuation: add the hidden _triage-stage verb and the bounded
      fail-safe run_silent decision, and make known mode the default for
      agent-attributed runs of run_silent tools behind the flag."
  - id: failures-and-followups
    title: sase tool failures and triage in verify-monitor follow-ups
    depends_on:
      - record-and-render
    size: medium
    description:
      "failures-and-followups: add the sase tool failures subcommand over the Rust
      aggregation, and a flag-gated Failure triage section in verify-monitor follow-up
      prompts, for both reserved runs and wrapped raw just check."
  - id: acceptance-and-governance
    title: Prove the landing criteria, remove the flag, and document
    depends_on:
      - known-gated-continuation
      - failures-and-followups
    size: medium
    description:
      "acceptance-and-governance: add the triage smoke case group, re-run the backtest,
      run the live athena acceptance, remove the tool_failure_triage flag, and ship
      docs, the named memory edits, glossary strands, and the decision record."
proposed_by: bbugyi200.athena.0rq
create_time: 2026-09-24 19:06:57
status: wip
---

- **PROMPT:**
  [prompts/202609/tool_e3_failure_triage.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/tool_e3_failure_triage.md)

# Plan: E3 — failure triage for `sase tool run`

## Outcome and scope

E1 gave SASE a machine-local ToolRun ledger. E1.5 made nearly every agent `check` a
ToolRun, and E2 (`sase-17p`, landed) made hand-offs durable. What the ledger does not do
yet is tell an agent whether a failure is its own. On athena, 86% of sase `check` runs
failed (426 of 497), `lint (symvision)` was the first failing stage in 56% of them, and
because `just check` is fail-fast, `test (scoped)` ran in only 20% of runs. 76% of
agents ended their session on a run whose tests never executed. E3 fixes that,
retargeted per the consolidated revalidation:

> With master red exactly as it is on any given day, an agent's `sase tool run check`
> (1) runs past every stage whose failures are all KNOWN or FLAKY, so the scoped tests
> run behind a master-red symvision stage; (2) labels every failure item NEW, KNOWN,
> FLAKY, or UNKNOWN, with its evidence and, where one exists, a _possible_ owning bead;
> (3) ends with one verdict line, for example
> `verdict: no_new_failures — 26 KNOWN, 1 FLAKY; exit 1 because KNOWN failures remain`;
> and (4) exits with the same code fail-fast `just check` would have returned.

A normal user can verify it without reading internals:

- Run `sase tool run check` from an agent on a red-master tree. The footer shows
  per-stage class counts with `continued`/`stopped` markers, labeled items (NEW and
  UNKNOWN first), and one verdict line. The exit code matches `just check`.
- `sase tool failures` lists the machine's current signature groups with agent counts
  and first and last seen. Linked-repo groups never appear under sase's `check`.
- `sase tool show RUN -j` still shows items, classes, evidence, and verdict after the
  run's logs have been reaped.
- A verify monitor's follow-up prompt starts its failure evidence with the verdict and
  the NEW and UNKNOWN items. The starter wrote no baseline prose.

Design inputs, read through `sase artifact read`:

- `research:202609/sase_tool_e3_e4_landing_criteria/sase_tool_e3_e4_landing_criteria.md`
  — the authority for this epic. §4 is E3's scope and rules, and §4.8 is the landing
  gate. §3.2 lists claims that must not be cited. Read the `__cdx`, `__cld`, `__mus`,
  and `__gem` siblings only when a phase needs their detail.
- `research:202609/sase_tool_epic_roadmap/sase_tool_epic_roadmap.md` (§4 cross-epic
  invariants) and `plan:202609/tool_e2_durable_handoff.md` (decision 6, the wire rule;
  the shared post-begin executor body; `terminal_cause` values).
- `decisions:record-before-admit`, `decisions:guarded-recipes`,
  `decisions:check-full-is-explicit`, `decisions:host-owned-completion`,
  `decisions:rust-core-required`, and `decisions:corpus-before-mechanism`. Also
  `sase/memory/lint_and_test.md`, `sase/memory/sase_flags.md`,
  `sase/memory/cli_rules.md`, `sase/memory/symvision.md`, and
  `sase/memory/sase_beads.md`, all through `/sase_memory_read`.

Planning baseline: sase master `fdc3e3caf`; core pin `6d0d0e6d5c` (the linked sase-core
checkout's `master` is `20ac645`, which descends from the pin and has no ToolRun commits
after it). E2 `sase-17p` is closed. `sase-145` (durable finish diagnostics) was closed
by E2's land agent, so E3 verifies its acceptance but does not rebuild it. `sase-182`
(linked-repo identity) and `sase-114` (fixture stage leak) are `ready` and are absorbed
by phase `ledger-hygiene`. Recheck every fact when implementing. Do not redo landed work
or change unrelated task lifecycles. If an absorbed bead is already closed when its
phase starts, verify its acceptance and skip that part.

Every phase that touches `sase-core` opens it with `/sase_repo`, reads its `AGENTS.md`,
works only in the printed path, and verifies with `sase tool run check` from inside that
checkout (about five minutes; give it a generous explicit timeout). Never run bare
`cargo`. Paths below are repository-relative and never name a numbered checkout. Commits
and release sequencing stay with host-owned finalizers. No implementation change
precedes approval.

## Decisions this plan settles

1. **Adopt the revalidated E3 (§4 of the landing-criteria report), not the roadmap's
   original.** Target `check` (497 runs), not `check-full` (1 run). Add KNOWN-gated
   continuation. KNOWN needs an independent witness, and UNKNOWN is the default. Drop
   `rerun` attempt chains (they conflict with E2's one-shot claim; FLAKY evidence
   arrives passively from same-fingerprint repeats). E4 (receipts and completion policy)
   stays out, and its §7 Q1 decision is not made here.

2. **Facts are stored once; judgments are either stored once or computed on read.**
   - An **item** is an immutable extracted fact, stored once per run, stage, and
     signature: extractor name and version, signature digest, bounded display, locator
     paths, and an occurrence count.
   - A **label** is stored once, when the item is classified. It carries the rule
     version, the knob values, and evidence refs. It is never recomputed, so the footer,
     `show -j`, `failures`, and monitor evidence all show the same label.
   - **`failure_kind` and the verdict** are pure functions of stored facts, computed on
     read by Rust. They therefore work on legacy rows, on runs settled by reconcile, and
     on runs that were never triaged.

3. **All classification logic is a pure Rust function.** Python only gathers bounded
   inputs: ancestry, the flake baseline, selection-health witness rows, and bead
   candidates. The output is deterministic under any reordering of input rows. The rule
   has two tightening knobs, `min_witnesses` (distinct witnessing workspaces, default 1)
   and `touched_requires_clean_witness` (default false). Tightening after the backtest
   is therefore a Python constant change, not a new core release.

4. **Selection-health evidence: yes for witnesses, no for the oracle port.** Full-run
   records under `${SASE_HOME:-~/.sase}/test-selection/<project_key>/` are pytest
   witnesses in v1, through the same pure function (their `changed_files` is the touched
   set). FLAKY's baseline source is the repo's `tests/reproducible_flake_baseline.txt`,
   read at base(R). The reproducible-flake _oracle_ in `tests/` is not ported:
   `check-full`'s `selection-health --fail-on-new-flake` already forces every
   reproducible flake into that baseline with a bead, so the baseline is the oracle's
   curated output. Production code never imports from `tests/` or `tools/`.

5. **Recording is ungated and fail-open. The flag gates only what changes an agent's
   default path.** From phase `record-and-render` on, every settled named-tool run is
   triaged and stored, so the corpus accumulates while later phases land. A triage error
   never changes an exit code or blocks a run. The single beta flag,
   `tool_failure_triage`, gates three things: the footer triage block and verdict line,
   `known` as the agent default continuation mode, and the Failure triage section in
   follow-up prompts. `sase tool show`'s triage section and `sase tool failures` are
   explicit views and ship ungated.

6. **No label exists before the precision gate.** Phase `bindings-and-backtest` must
   pass the at-least-95% hand-audited KNOWN precision backtest (DoD-5) before phase
   `record-and-render` stores a single label. Phase `acceptance-and-governance` re-runs
   it for the landing note.

7. **Continuation is a handshake; exit codes cannot be laundered.**
   - Every recorded run of a `stages: run_silent` named tool gets `SASE_TOOL_CONTINUE` ∈
     {`never`, `always`, `known`} from the new wrapper, plus `SASE_TOOL_PYTHON`. When
     the variable is absent (humans, CI, or an older wrapper in a stale venv),
     `run_silent` is byte-for-byte today's fail-fast script and writes no new records.
   - After any continued failure, every later exit, including `--finish`, uses the
     _first_ continued failure's code.
   - If the child exits 0 while a continued failure exists and no `recipe_finished`
     record does, `sase tool run` records the run `failed` and exits 1 with a
     diagnostic. This is the only deliberate exception to preserving the child's exit
     status, and it applies only to continuation that the wrapper itself enabled.
   - An unwrapped recipe line (`probe_core_floor`, `print_scoped_summary`) that fails
     after a continued failure returns that line's own code, as `just` does today. The
     docs say so.

8. **The decision that happened is authoritative.** Continuation decisions are persisted
   from `run_silent`'s own JSONL records (`continued`/`stopped`, with reason and elapsed
   time), not from what the triage helper proposed. A helper that answers `continue`
   after `run_silent` has already timed out cannot rewrite history.

9. **Agent default is keyed on recorded attribution.** With the flag on, a recorded run
   of a `stages: run_silent` named tool defaults to `known` when its recorded `agent`
   attribution is non-empty, and to `never` otherwise. Recorded attribution is
   `SASE_AGENT_NAME` in the foreground, the starter agent for a monitor reservation, and
   `SASE_TOOL_RUN_AGENT` for an E1.5 wrap. `-k` forces `always` and `-x` forces `never`.
   `-H` with `-k` or `-x` is a usage error, because the launch envelope stays unchanged.

10. **Pin discipline that a host-owned commit flow can actually follow.** A core phase
    cannot pin its own commit: finalizers create commits after the turn, and E2 shipped
    its pin behind its own bindings (`sase-17p` note #1). So core phases change only
    `sase-core`. The first sase phase that calls a new binding first moves
    `sase-core-revision.txt` to a pushed `sase-core` commit that contains it. In that
    same change it adds the adapters, the `tools/validate_sase_core_rs` /
    `tools/check_sase_core_rs_bindings` / `tools/smoke_sase_core_rs_tool_runs` entries,
    and the real-binding round-trip tests. This is how DoD-12's pin clause is met: sase
    never calls or requires a binding that the pinned core lacks. Do not touch the
    published `sase-core-rs` window in `pyproject.toml`.

11. **Wire discipline (E2 decision 6), extended to new tables.**
    - `schema_version` stays 1. Add only new tables or nullable columns, and no new
      values in existing enums. Class, kind, verdict, and decision strings live in new
      tables and new wires.
    - New tables reference `runs` with `ON DELETE CASCADE`, so an older core's retention
      can still delete runs. A compat test proves it.
    - Read-only opens never create tables, so every new read path probes `sqlite_master`
      and treats a missing table as empty.
    - No `ToolDefinitionWire` or catalog change. The sase `check` definition digest must
      not move.
    - Every triage JSON object carries its own `schema_version: 1`.

12. **Monitor results are not changed.** `MonitorResultWire` has no extension slot, and
    changing the continuation contract is outside E3. The follow-up prompt reads the
    stored triage through `tool_run_triage_show` at launch time. The worker settles and
    triages its ToolRun before the monitor proc exits, so the triage exists by then.

13. **No automatic writes.** E3 never creates, updates, or `+1`s a bead. Owner
    suggestions are computed once, at settle, and stored with the run as "possible
    owner" (open `ci`/`flake`/`bug` task beads) or "possibly fixed by `<id>` — your HEAD
    may predate the fix" (closed within the lookback).

14. **Legacy identity rows stay as history.** The 51 pre-fix linked-repo rows labeled
    `gh_sase-org__sase` are not rewritten. Live evidence never sees them, because stored
    items all postdate the identity fix. The backtest assigns a pre-fix row to sase's
    catalog only if its recorded head resolves to a commit in the sase repository.

15. **Settled parameters** (the report's §4.9 "planner must settle" list):
    - the helper mechanism is the hidden `sase tool _triage-stage` subprocess, spawned
      by the stdlib helper with a hard 10 s timeout;
    - the lookback `L` is 7 days, and ancestry is at most 2,000 first-parent commits;
    - selection-health records are witnesses (decision 4);
    - footer caps are at most 10 NEW and 10 UNKNOWN items, and at most 3 examples each
      for KNOWN and FLAKY, beside full counts;
    - the legacy-row mapping is the table under "Classification contract".

16. **Memory changes are named here, so plan approval authorizes them.** They are the
    DoD-13 edits from the landing-criteria report, listed under "Memory and governance".
    Each is made through `/sase_memory_write` in phase `acceptance-and-governance`. The
    decision record is written at landing, after the backtest has settled the knobs,
    rather than now.

## Binding contracts

### Identity

- **Tool identity** `T` = (catalog repo project identity, tool name). After phase
  `ledger-hygiene`, `runs.project` is that identity.
- **Evidence runs** are settled, non-ad-hoc runs of the same `T` with the same
  `extra_args_digest` and a complete `fingerprint_before`, settled within `L`. Evidence
  gathering is bounded to the newest 500 such runs.
- **Stage key** is the top-level stage description; `stage_id` stays an execution-local
  address. A run of a `stages: none` tool, or a failed run with no failed stage, uses
  the reserved stage key `*` for items extracted from its output of record. Renaming a
  stage description breaks history continuity by design, and the docs say so.
- **Signature** `s(i)` = SHA-256 over (extractor name, extractor version, normalized
  key). It is stage-independent within `T`. Signatures from different extractor versions
  are never compared.

### Durable additions (sase-core, wire schema stays 1)

| Addition                     | Holds                                                                                                                                                                                                                                                                                                                                                            |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tool_triage_items`          | item id (deterministic digest of run, stage key, extractor, version, signature), run id, stage id?, stage key, extractor, extractor version, signature, display (≤ 512 chars, redacted), locator paths (repo-relative), occurrences, created ts; nullable label columns: class, touched, rule version, knobs, evidence JSON, possible owners JSON, classified ts |
| `tool_triage_stages`         | run id, stage id, stage key, extraction status (`parsed`, `generic`, `output_missing`, `output_truncated`), output path?; nullable decision columns: mode, decision (`continue`/`stop`), reason, elapsed ms, decided ts                                                                                                                                          |
| `tool_triage_runs`           | run id, continuation mode, recipe-finished ts?, first continued exit code?, continuation extra ms?, repeat-of run id?, triaged ts, diagnostics                                                                                                                                                                                                                   |
| observe `fingerprint_before` | optional field on the observe request, persisted at spawn so a mid-run stage triage can read base(R) and dirty paths; finish may resend an equal value                                                                                                                                                                                                           |
| retention                    | triage rows follow the detail cut (`detail_days`, 60) and are deleted before their run at the summary cut; files under a settled run's `stage_output/` directory (next to its `events_path`) follow `log_days` and the aggregate log cap                                                                                                                         |

Item and label writes are idempotent: stored rows win on replay, and a second classifier
never overwrites a label.

### Classification contract (pure Rust; §4.3 of the report, verbatim in effect)

**Run-level `failure_kind`** (computed on read):

| Kind             | When                                                                                         | Items?                                                                                                        |
| ---------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `control`        | `terminal_cause` ∈ {`stop_requested`, `interrupt`, `timeout`}                                | no                                                                                                            |
| `infrastructure` | `terminal_cause` ∈ {`launch_failed`, `owner_lost`, `wrapper_lost`, `signal`}                 | no                                                                                                            |
| `environment`    | exited nonzero before any stage completed, and a recognized `_setup` marker is in the output | one environment item with a remedy hint (`just install` / `sase update`), class UNKNOWN, reason `environment` |
| `verification`   | exited nonzero with a failed stage, or with unparseable failure output                       | yes                                                                                                           |
| `none`           | exit 0                                                                                       | no                                                                                                            |

**Legacy mapping** for rows with `terminal_cause` NULL. Anything ambiguous is
`infrastructure`, never `verification`:

| Row facts                                   | Kind (reason)                        |
| ------------------------------------------- | ------------------------------------ |
| `succeeded`                                 | `none`                               |
| `failed`, exit code present and not 126/127 | `verification`                       |
| `failed`, exit 126/127, or no exit code     | `infrastructure` (`legacy_unmapped`) |
| `interrupted`                               | `control` (`legacy_interrupted`)     |
| `signaled`                                  | `infrastructure` (`legacy_unmapped`) |
| `lost`                                      | `infrastructure` (`legacy_lost`)     |
| `created` / `running`                       | not settled; no kind                 |

**Definitions:**

- `touched(i, R)`: `P(i)` intersects `R`'s dirty paths in the catalog repo.
- `base(R)`: `fingerprint_before.repos[catalog repo].head`.
- **Witness** `W` for `s` relative to `R` must satisfy all of: `W ≠ R`; a different
  workspace or a clean tree; `s ∈ items(W)`; not `touched(s, W)`; `base(W)` is in `R`'s
  ancestry (ancestor or equal); settled within `L`. A selection-health full-run record
  witnesses pytest-extractor signatures under the same conditions, using `changed_files`
  as its touched set (a null `changed_files` together with `tree_dirty: true` is never a
  witness).
- **Clearing run** `C` must satisfy all of: `base(C)` lies on the ancestry between the
  newest witness's base and `base(R)`; `C` completed the item's stage; `s ∉ items(C)`;
  and not `touched(s, C)`.

**Item classes** (for `verification` runs; the first match wins):

1. **FLAKY** if either holds:
   - `s` is an active entry in the repo's flake baseline at `base(R)`; or
   - two runs with the same complete fingerprint digest disagree on `s`. This clause
     applies only to test-stage extractors (pytest, cargo test), because lint verdicts
     read unfingerprinted state (`sase-vr`, bead state).
2. **KNOWN** if at least `min_witnesses` distinct-workspace witnesses exist and no
   clearing run is newer than the newest witness. When `touched_requires_clean_witness`
   is set, a touched item also needs a clean-tree witness.
3. **NEW** if there is no FLAKY or KNOWN evidence, and either `touched(s, R)` holds or a
   _pass witness_ exists. A pass witness is a run at `base(R)` or at an ancestor within
   `L` that completed the stage without `s` and did not touch `P(s)`.
4. **UNKNOWN** otherwise. Generic-extractor items are always UNKNOWN (reason
   `extractor_generic`). **UNKNOWN is yours.**

**Never KNOWN:** a bead or title match; ad-hoc runs, as subjects or as witnesses; a
witness with a different `extra_args_digest`; evidence from another machine; an item in
a file the run's own diff added.

**Verdict** (computed on read, evaluated in order):

| Verdict           | Condition                                                                                                                                                                                                          |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `pass`            | exit 0                                                                                                                                                                                                             |
| `undetermined`    | `failure_kind` ∈ {`control`, `infrastructure`, `environment`}                                                                                                                                                      |
| `new_failures`    | at least one NEW item                                                                                                                                                                                              |
| `undetermined`    | any UNKNOWN item; a failed stage with no parsed item; no `recipe_finished` (some stage never ran); a `stages: none` tool (v1 cannot know that every step ran); or the run was never triaged (reason `not_triaged`) |
| `no_new_failures` | every item is KNOWN or FLAKY, and the recipe reached `recipe_finished`                                                                                                                                             |

**Evidence refs on every label:**

- witness run ids (and selection-record ids);
- distinct agent and workspace counts;
- first-seen time;
- the clearing run id, if any;
- the baseline line or flake source;
- `touched`;
- for UNKNOWN, the ordered rejection reasons (`no_witness`, `witness_cleared`,
  `untouched_no_pass_witness`, `extractor_generic`, `environment`,
  `insufficient_witnesses`, `touched_needs_clean_witness`).

**REPEAT (advisory only):** when `R`'s complete fingerprint digest equals a prior failed
evidence run's digest with the same signature set, the triage records `repeat_of`, and
the footer prints `REPEAT of <run>`. It is never a refusal.

### Extractors and normalization (pure Rust, auto-detected by output shape)

| Extractor                                                  | Signature key                                                        | Locator paths |
| ---------------------------------------------------------- | -------------------------------------------------------------------- | ------------- |
| symvision                                                  | category + symbol + path                                             | path          |
| mypy                                                       | path + error code + message with quoted names masked; no line number | path          |
| ruff, fmt (python, markdown), keep-sorted                  | rule or check + path                                                 | path          |
| pytest (`FAILED` / `ERROR` lines)                          | node id without parametrization                                      | test file     |
| toobig                                                     | path                                                                 | path          |
| cargo test                                                 | crate::test path                                                     | none          |
| environment (`_setup` markers)                             | marker kind                                                          | none          |
| generic fallback (only when no specific extractor matched) | stage key + hash of the normalized last ~40 lines                    | none          |

Normalization strips ANSI escapes, timestamps, durations, PIDs, and hex ids. It also
strips workspace roots (`…/sase_<N>/`, `…/sase/repos/linked/<repo>/`, and the catalog
project root passed in the request), cargo target dirs, and `/tmp` paths. It removes
line and column numbers wherever the item kind is not positional. Repeated identical
signatures in one stage collapse into one item with an occurrence count. Display text is
at most 512 characters, redacted, and never contains an absolute checkout path or an
environment value.

### Rust bindings (in the domain that already binds `tool_run`)

| Binding                    | Kind                                                                                                                                                                                                                     | Phase                                                   |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------- |
| `tool_run_triage_extract`  | pure: stage output → items + extraction status                                                                                                                                                                           | `core-failure-items`                                    |
| `tool_run_triage_record`   | store: record items, stage facts, run facts (continuation, decisions); idempotent                                                                                                                                        | `core-failure-items`                                    |
| `tool_run_triage_classify` | pure: subject + evidence + inputs + knobs → labels, evidence refs, owners, `repeat_of`                                                                                                                                   | `core-classification`                                   |
| `tool_run_triage_verdict`  | pure: run facts + labeled items + finish facts → `failure_kind` + verdict                                                                                                                                                | `core-classification`                                   |
| `tool_run_triage_stage`    | store: extract, record, gather evidence, classify, and persist one stage's items                                                                                                                                         | `core-classification`                                   |
| `tool_run_triage_settle`   | store: extract the missing stages or the run output, classify unlabeled items, attach owners, record run facts; returns the triage object                                                                                | `core-classification`                                   |
| `tool_run_triage_show`     | store: the triage object for `{run_id}` (stored facts only in `core-failure-items`); `core-classification` adds the computed kind and verdict and the `{owner_kind, owner_id}` selector (the newest run with that owner) | `core-failure-items`, extended in `core-classification` |
| `tool_run_failures`        | store: signature groups with filters                                                                                                                                                                                     | `core-classification`                                   |

Request wires are `deny_unknown_fields`, result wires are lenient, and every wire
carries `schema_version: 1`. Expected refusals are typed result values, not exceptions.

### Continuation protocol (`tools/run_silent`, stdlib helper `tools/_run_silent_record.py`)

- **Nesting guard** (phase `ledger-hygiene`, extended by `keep-going`). `run_silent`
  runs its stage command without `SASE_TOOL_RUN_EVENTS`, `SASE_MONITOR_DIAGNOSTICS_DIR`,
  `SASE_TOOL_CONTINUE`, and `SASE_TOOL_PYTHON`, so a nested `run_silent` records nothing
  into the enclosing run's events or monitor diagnostics. `SASE_TOOL_RUN_ID` stays, so a
  nested `sase tool run` keeps its parent link and process-group ownership.
  `tests/conftest.py` scrubs the same variables plus `SASE_TOOL_RUN_ID` at session
  start. That covers `stages: none` tools such as `test`, whose pytest children are not
  under a `run_silent` stage. Before adding the scrub, confirm that no test-runner
  infrastructure (`tools/run_pytest`, the selection-health plugin, suite-gate leases)
  reads these variables. Tests that need them set them explicitly.
- **Records** (helper schema stays 1; new kinds only):
  - `continued` and `stopped` carry stage id, mode, exit code, reason (`mode_always`,
    `all_known_or_flaky`, `new_item`, `unknown_item`, `no_items`, `helper_timeout`,
    `helper_error`, or `mode_never`), elapsed ms, and a timestamp;
  - `recipe_finished` carries the first continued exit code, or null.

  `_jsonl_to_core_events` forwards only `started`/`finished` to core. Today it maps
  every non-`started` kind to `stage_finished`, which must be fixed. The ingestor
  collects the new kinds as continuation facts.

- **Failed stage output** (phase `record-and-render`). With `SASE_TOOL_RUN_EVENTS` set,
  the helper writes a bounded head-plus-tail copy (256 KiB, reusing `_bounded_output`)
  to `<events dir>/stage_output/<stage_id>.log` and adds the relative `output_path` to
  the `finished` record.
- **Decisions.** In `always` mode, a failed stage records `continued` and exits 0. In
  `known` mode, the helper spawns `"$SASE_TOOL_PYTHON" -m sase tool _triage-stage …`
  with a hard 10 s timeout. Only an explicit `continue` continues. Timeout, crash,
  unparseable output, or `stop` all stop. Any other value, or a missing
  `SASE_TOOL_RUN_EVENTS`, is fail-fast.
- **`tools/run_silent --finish`**, the last line of the `check` and `check-full`
  recipes:
  - without the handshake variable: silent, exit 0;
  - with it: append `recipe_finished`;
  - if any stage continued: print one line
    `✗ N stage(s) failed; continued past them (first exit C)` and exit `C`.

## Memory and governance

Each edit below goes through `/sase_memory_write` in phase `acceptance-and-governance`,
then `sase memory init`. Every one of them comes from DoD-13 of the landing-criteria
report:

- `sase/memory/lint_and_test.md`: a short "Reading `sase tool run` triage" paragraph. It
  says:
  - UNKNOWN is yours;
  - never call a failure "unrelated" without a KNOWN or FLAKY label or equivalent
    evidence;
  - never `+1` a bead for an item the footer already links as its possible owner;
  - agent runs continue past all-KNOWN/FLAKY stages by default, `-x` restores fail-fast,
    and `-k` always continues.
- `sase/memory/symvision.md`: one line saying master-red symvision items are labeled
  KNOWN in `sase tool run check`, and NEW or UNKNOWN symvision items are yours.
- New glossary strands **Failure Signature** and **Triage Verdict**, linked to
  `glossary:tool-run`.
- A new decisions strand: _"Triage annotates; it never changes an exit code, and KNOWN
  requires an independent witness"_. It covers the claim, the alternatives rejected
  (bead-mapping KNOWN, a strict merge-base reference, `rerun` chains, and
  always-continue), the costs (+time on continued runs, UNKNOWN-heavy thin ledgers), and
  the reopen conditions in "Relationship to open work".

No skill source changes. `docs/tool.md` and `docs/monitors.md` are ordinary docs.

## Acceptance cannot depend on a green master

`check` is red most days. Every criterion is observable behavior on fixture ledgers,
fixture recipes, and the live red master: a label, an evidence ref, a verdict, an exit
code, a stage row, a decision. None is "`just check` passes". Phases still run
`sase tool run check` as `lint_and_test.md` requires (inside `sase-core` for core
phases). Pre-existing master-red items are not blockers. They are E3's KNOWN fixtures.

## Relationship to open work (out of scope)

- **E4** (receipts, prepared-completion policy, §7 Q1): E3 changes no landing or
  completion policy. Prepared completion still requires exit 0.
- **E5** owns any TUI Failures view. **E6** must forecast from stage durations or
  condition on the recorded continuation mode.
- **`sase-180`** (toobig splits redden symvision; symvision's internal stage masking) is
  complementary: E3 contains the damage, and `sase-180` removes a cause.
- **`sase-vr`** (lint toolchain versions not fingerprinted) is why the same-fingerprint
  FLAKY clause excludes lint stages.
- **`sase-17e`/`sase-17g`** (routing, detach/join) are orthogonal.
- **`sase-17m`** (agent-session rename): any new monitor-side field uses the flat
  `monitor_*` convention and the shared accessors. E3 adds none.
- **`sase-j0`, `sase-th`, `sase-10w`** (red lanes) are fixtures, not blockers.
- **Reopen conditions**, recorded as `PROPOSED FOLLOW-UP:` notes at landing:
  - `rerun` reopens when the ledger shows more than about 2 h/week of manual re-runs of
    FLAKY-labeled items. It would then be a new run with `retry_of_run_id`, never
    attempt N and never `parent_run_id`.
  - CI evidence reopens when UNKNOWN stays above about 40% of items on a machine.

## 1. ledger-hygiene

**Identity (`sase-182`).** A named tool's run must be recorded under the project
identity of the repo that owns the resolved catalog, not under `SASE_PROJECT`. An ad-hoc
run takes its identity from its cwd's repo. This covers:

- `runs.project`;
- `fingerprint_before/after.project_identity`;
- `fingerprint.repos[0].identity`.

Replace the three near-identical wrappers (`observe.py::_project_identity`,
`query.py::_project_identity`, and `executor_recording.py::_current_project_identity`)
with one helper keyed on the catalog project root. Keep `SASE_PROJECT` only as agent
context. Catalog resolution must not change. `sase tool runs` without `-a` then lists
the cwd catalog repo's runs. The identity string for a linked repo must equal the key
that repo's other SASE stores use when it is a registered project (for example its
test-selection project key). Otherwise it is a stable identity derived from the repo.
Leave the historical rows unrewritten (decision 14) and note that in `docs/tool.md`.

Tests:

- from an agent-like environment (`SASE_PROJECT` = the host), a named run inside a
  second fixture git repo with its own catalog records that repo's identity in all three
  places;
- an ad-hoc run records its cwd's identity;
- `sase tool runs` scoping follows cwd.

Close `sase-182` with a note citing the tests.

**Nesting guard and `sase-114`.** Implement the nesting guard from "Continuation
protocol" for the variables that exist today (`SASE_TOOL_RUN_EVENTS`,
`SASE_MONITOR_DIAGNOSTICS_DIR`), using a subshell `unset` so builtins and external
commands behave alike. Add the `tests/conftest.py` session scrub. Fix the known leakers
explicitly:

- `tests/monitor/test_continuation_baseline.py` (it builds `{**os.environ, …}`);
- both paths in `tests/monitor/test_monitor_diagnostics.py`, including the
  `run_supervisor` path that copies `os.environ`.

Regression tests:

- an outer `run_silent`-staged ToolRun whose stage runs a nested failing `run_silent`
  records exactly its own top-level stages;
- the outer monitor diagnostics directory stays untouched (`sase-114`'s acceptance);
- a `stages: none` run whose child is pytest records no stage rows;
- `run_silent` output with none of the variables set is byte-identical to today's, for a
  passing and a failing stage (golden).

Close `sase-114` with a note citing the tests.

## 2. core-failure-items

Work only in `sase-core`, in a new `crates/sase_core/src/tool_run/triage/` module (a
facade `mod.rs` plus focused files; each file at most 1,500 lines; follow `AGENTS.md`
conventions). Implement:

- **Wires:** item, extract request/result, record request/result, and show
  request/result.
- **Extractors and normalization:** every row of "Extractors and normalization", each
  versioned, with auto-detection by output shape. Specific extractors run together, and
  the generic fallback runs only when none matched.
- **Golden fixtures:** from real, redacted athena stage outputs under the triage
  fixtures directory, one per extractor at least. Take them from retained
  `~/.sase/tools/logs/<run>/stdout.log` files, split at the `✗ <stage>` markers, and
  strip absolute paths and anything secret-shaped before committing. Add:
  - two-workspace fixtures showing that the same failure seen from different workspace
    roots yields one digest;
  - collision fixtures showing that different node ids, symbols, error codes, or paths
    yield different digests;
  - a display bound and redaction test;
  - a test that cross-version comparison is refused.
- **Store:** the three tables and the observe `fingerprint_before` field from "Durable
  additions".
  - Create them in `SCHEMA_SQL` with `IF NOT EXISTS` and `ON DELETE CASCADE`.
  - Probe `sqlite_master` on every new read path.
  - Make `tool_run_triage_record` idempotent (stored rows win).
  - Make `tool_run_triage_show` return stored facts only; kind and verdict arrive in the
    next phase.
- **Retention:** triage rows at the detail cut and summary cut. File candidates for
  `stage_output/` files, which retention finds from the run's `events_path` parent,
  under `log_days` and the aggregate log cap.
- **Compat tests** (extend `store/tests/compat.rs`):
  - an old-shape store without the tables is read as empty and gains them on write;
  - the pre-change query shape still loads a store with triage rows;
  - the old retention SQL deletes runs that have triage rows (the cascade holds);
  - retention reports and deletes triage rows and stage output files.
- **Bindings:** `tool_run_triage_extract`, `tool_run_triage_record`, and
  `tool_run_triage_show` in the telemetry binding domain, with the binding round-trip
  test extended.

Touch nothing in the sase repo (decision 10). Record the pushed-commit expectation for
the pin in a phase-bead note.

## 3. core-classification

Work only in `sase-core`, in the same module. Implement:

- `tool_run_triage_classify`, exactly the "Classification contract", as a pure function
  with the two knobs as request parameters.
- `tool_run_triage_verdict` (kind with the legacy mapping, and the verdict).
- REPEAT detection.
- Owner matching: each item's locator token matched against the supplied candidates'
  `node_id`/`location` fields and titles. At most two matches, open ones as "possible
  owner" and closed ones as "possibly fixed by".

Then the store-backed operations:

- `tool_run_triage_stage`: extract, record, gather the evidence runs with their items
  and completed stage keys from the store, classify, and persist the labels. Idempotent.
- `tool_run_triage_settle`: the settle contract from the table.
- `tool_run_triage_show`: extended with the computed kind and verdict, and with owner
  lookup.
- `tool_run_failures`: signature groups keyed on (`T`, stage key, extractor version,
  signature).
  - Each group reports runs, distinct agents, distinct workspaces, first and last seen,
    the newest occurrence's class and possible owners, and the last run id.
  - Filters: project (default: the caller-supplied identity), all projects, tool, class,
    days (default 7), limit.
  - Groups sort by agents descending, then by last seen.

Tests:

- DoD-3: one fixture per `terminal_cause` producing its kind and no labels; a `_setup`
  missing-binding output producing `environment` with a remedy; every legacy mapping
  row.
- DoD-4 fixture ledger cases (a)–(j), exactly as §4.8 lists them.
- Knob cases: `min_witnesses = 2` rejects a single-workspace witness, and the clean-tree
  knob rejects a touched item with only dirty-tree witnesses.
- A property test showing that shuffling every input list yields byte-identical output.
- Verdict table cases, including `no_new_failures` refused without `recipe_finished`,
  with any UNKNOWN item, and for `stages: none`.
- `failures` aggregation over a fixture store, including cross-project isolation.

Add golden request/result fixtures for classify, stage, settle, show, and failures. Add
the bindings and extend the round-trip test. Touch nothing in the sase repo.

## 4. keep-going

Implement the continuation protocol's `always` and `never` modes and `--finish` (see
"Continuation protocol" and decision 7). In detail:

- The wrapper exports the handshake variables for recorded runs of `stages: run_silent`
  named tools.
- `-k/--keep-going` and `-x/--fail-fast` go on `sase tool run`: mutually exclusive,
  before `TOOL`, and valid only for named `stages: run_silent` tools. Anything else
  exits 2, and so does combining either with `-H`. In this phase the default mode is
  `never`.
- `run_silent` and the helper gain the `continued`/`stopped`/`recipe_finished` records,
  first-code exits, `--finish`, and the extended nesting-guard list.
- `_jsonl_to_core_events` gets its kind fix. The ingestor collects continuation facts
  and keeps them in memory for the footer and for phase `record-and-render`, which
  persists them.
- The `check` and `check-full` recipes end with `@tools/run_silent --finish`. Update
  `tests/test_justfile_lint.py`.
- The executor gets the safety net. If the core refuses `failed` with the child's exit
  0, keep the wire rules, record the diagnostic `continuation_unfinished`, and document
  the chosen recording.
- `sase monitor`'s `_parse_monitor_tool_words` declines `-k`/`-x`, as it does for
  `-q`/`-v`/`-T`/`-H`, so the E1.5 argv carries them.
- Extend the conftest scrub list.

Tests (DoD-6):

- under `-k`, a fixture recipe `[fail, pass, fail, pass]` records all 4 stages and exits
  with stage 1's code;
- a property test over stage-outcome combinations × a missing `--finish` × a stop after
  a continuation × an unknown mode value shows that none yields exit 0 when any stage
  failed;
- `-x` restores fail-fast;
- human output with no handshake variable is byte-identical to today's (golden), and
  `--finish` is silent;
- an old-wrapper simulation (no handshake variable) writes no new records;
- the usage errors;
- the monitor word parser declines `-k`/`-x`.

No flag: `-k` is opt-in and complete on its own.

## 5. bindings-and-backtest

**Pin and adapters (decision 10).** Move `sase-core-revision.txt` to a pushed
`sase-core` commit containing phases 2–3. Verify that it is reachable from the linked
checkout's `origin/master` and that a pinned build exposes every new binding. Then, in
the same change:

- add thin adapters to `src/sase/core/tool_run.py`;
- add the new names to `tools/validate_sase_core_rs`,
  `tools/check_sase_core_rs_bindings`, and `tools/smoke_sase_core_rs_tool_runs`,
  deliberately;
- add real-binding round trips to `tests/core/test_tool_run_store.py`;
- whitelist adapters that later phases consume with
  `--epic-symbol '<epic bead id>(<symbol>)'` in the Justfile symvision recipe (see
  `sase/memory/symvision.md`).

**Input gatherers** in a new `src/sase/tool/triage_inputs.py`. Each is bounded and
fail-open, with a diagnostic. A missing input can only produce fewer KNOWN or FLAKY
labels, never more.

- ancestry: `git rev-list --first-parent --max-count=2000 <base>` in the catalog repo
  root;
- the flake baseline at `base(R)`
  (`git show <base>:tests/reproducible_flake_baseline.txt`), parsed with
  `tools/selection_health`'s rules (active node ids; `fixed-at` retires an entry) and
  reimplemented, not imported;
- selection-health full-run records within `L`: schema 2, `kind: full-run`, honoring
  `SASE_TEST_SELECTION_HEALTH_DIR`/`_PROJECT_KEY`, keyed by `T`'s project;
- bead candidates: `ci`/`flake`/`bug` task beads, open and closed within `L`, through
  the bead store's `list_issues`.

Put the tightening knobs here as module constants.

**`tools/tool_triage_backtest`**, a read-only script with a pytest twin over fixture
ledgers and logs:

- It selects settled `check` runs of the sase catalog: post-fix rows by `T`, pre-fix
  rows by head resolvability (decision 14).
- It reconstructs per-stage output from retained logs by `✓/✗ <description>` markers
  (compact `stdout.log`, or the owner log for owner-bound runs).
- It drops fixture-contaminated stage rows (descriptions outside the recipe's top-level
  list, or repeated).
- It extracts with the pure binding and replays chronologically with the pure
  classifier, using only prior runs as evidence.
- It writes a JSON report and a Markdown audit worksheet. They contain:
  - the label distribution and the UNKNOWN reason histogram;
  - a seeded random sample of at least 50 KNOWN labels, each with its witnesses, the
    run's diff paths, and the item;
  - the KNOWN count on files the run's diff added;
  - every KNOWN-but-touched item;
  - the runs excluded for missing logs.

Run it on athena while the post-E1 logs are still retained. Hand-audit the sample. An
item counts as pre-existing when its cause exists at `base(R)` unchanged by `R`'s diff,
checked through the evidence and `git`; write one justification line per sample. The
gate is DoD-5: at least 95% pre-existing, zero KNOWN on added files, and every
KNOWN-but-touched item dispositioned. If it fails, tighten the knobs and re-run. Never
relax the audit. If the knobs cannot reach the gate, change the rule in `sase-core`
under a new rule version, and phase `record-and-render` moves the pin first. Record the
distribution, the audit summary, the final knob values, and the report location in this
phase's bead note.

## 6. record-and-render

- **Failed-stage output capture** in the helper (see "Continuation protocol"). Test that
  core retention reaps it.
- **Settle-time triage in the shared post-begin executor body**, so foreground runs and
  E2's adopted worker behave identically. It runs after `tool_run_finish` and only for
  named tools. It calls `tool_run_triage_settle` with:
  - each failed stage's output file, for stages without items;
  - the bounded tail (256 KiB) of the output of record for `stages: none` tools and for
    failed runs with no failed stage;
  - the ingestor's continuation facts and decisions;
  - the gatherers' inputs;
  - the knobs.

  Budget: at most 5 s in total, and each gatherer has its own timeout. On error or
  overrun, store a diagnostic, print `triage unavailable: <reason>` (flag on), and keep
  the exit code. Recording is ungated (decision 5). Ad-hoc runs get no triage rows.

- **Flag:** create `tool_failure_triage` with `sase flag new tool_failure_triage`, using
  these sentences:
  - enabled: "Agent-visible failure triage: the `sase tool run` footer shows labeled
    items and a verdict line, agent runs of `stages: run_silent` tools continue past
    all-KNOWN/FLAKY stages, and verify-monitor follow-ups include the triage section";
  - disabled: "The footer, the default continuation mode (`never`), and follow-up
    prompts are unchanged; triage is still recorded and visible through `sase tool show`
    and `sase tool failures`";
  - remove when: "The E3 epic lands with the precision backtest passing and the triage
    smoke group green".

  Paste the printed registry entry.

- **Footer** (flag on). In compact mode, after the existing tail:
  - one line per failed stage, with class counts and a `continued`/`stopped` marker;
  - items within the caps from decision 15, NEW and UNKNOWN first, each with a short
    evidence summary (witness agents and first seen, or rejection reasons) and any
    possible owner;
  - the `REPEAT of <run>` line;
  - the existing `sase tool show RUN -l` pointer;
  - the verdict line last, for example
    `verdict: no_new_failures — 26 KNOWN, 1 FLAKY; exit 1 because KNOWN failures remain`.

  Non-compact runs print only the per-stage lines and the verdict line. Succeeded runs
  print no verdict line. `-T` keeps working. With the flag off, the footer stays
  byte-identical (golden).

- **`sase tool show RUN`** gains a TRIAGE section: kind, verdict, per-stage extraction
  status and decision, and an items table with CLASS, STAGE, ITEM, EVIDENCE, and OWNER.
  `show -j` gains the versioned `triage` object. Both read the store only, so they work
  after log reaping.

Tests:

- foreground and adopted-worker runs of a fixture `run_silent` catalog in an isolated
  `SASE_HOME` with seeded evidence runs produce the expected items and labels;
- DoD-2: after reaping, `show -j` still shows items, classes, evidence, and verdict, and
  `sase-145`'s spawn, stage-ingest, and log-write diagnostics still render;
- the footer, `show -j`, and the stored rows agree on ids and labels (DoD-8);
- fail-open under an unwritable store and under a slow gatherer, with the exit code
  unchanged;
- DoD-11: every bead mutation entry point is patched to raise, and triage still
  succeeds;
- linked-repo runs never witness sase items;
- both flag states.

Remove the `--epic-symbol` entries this phase's callers consume.

## 7. known-gated-continuation

Add the hidden `sase tool _triage-stage` verb (`help=argparse.SUPPRESS`, kept out of the
metavar list like `_adopt`). Its internal arguments are the run id, the stage id and
description, and the output path. It:

- reads the run's persisted `fingerprint_before`;
- gathers ancestry, baseline, and witnesses (no bead candidates; owners come at settle);
- calls `tool_run_triage_stage`;
- prints one JSON line `{decision, reason, counts}`.

It answers `continue` only when the stage has at least one item and every item is KNOWN
or FLAKY. Wire the helper's `known` path to spawn it with the hard 10 s timeout and to
record the authoritative `continued`/`stopped` record (decision 8).

Make `known` the default mode per decision 9 when the flag is on. The
monitor-reservation worker and E1.5 wraps inherit it through recorded attribution.

Tests (DoD-7):

- in the agent default mode, a stage whose items are all KNOWN or FLAKY continues;
- any NEW or UNKNOWN item stops;
- a stage with only a generic item stops;
- a simulated helper exceeding 10 s stops, and so does a crash;
- each decision appears in `show -j` with its evidence and elapsed time;
- a human run defaults to `never`;
- a monitor-reserved run with a starter agent defaults to `known`;
- both flag states.

Record `continuation extra ms` per run (the time spent after the first continued
failure) so the landing note can report the real cost.

## 8. failures-and-followups

**`sase tool failures`** over `tool_run_failures`:

- Options, in help order: `-a/--all`, `-c/--class {new,known,flaky,unknown}`,
  `-d/--days N` (default 7), `-j/--json`, `-n/--limit N`, `-t/--tool TOOL`.
- A rich table with columns CLASS, TOOL/STAGE, SIGNATURE (display), RUNS, AGENTS, FIRST,
  LAST, and OWNER.
- `-j` is versioned. An empty result prints `no recorded failures` and exits 0.
- The default scope is the cwd catalog repo's project.
- Subcommands become `failures, list, run, runs, show, stop, wait`, with help text per
  `sase/memory/cli_rules.md`.
- Add completion kinds and hints for the new value slots, regenerate the completion spec
  (`just sync-completion-spec`), and update `tests/main/test_parser_tool.py`.

**Verify-monitor follow-ups** (flag on). In `compose_followup_prompt`, add a
`## Failure triage` section before `## Selected diagnostics`. Resolve the run from
`monitor_tool_run_id`, or else by owner (`monitor`, monitor id) for a wrapped raw
`just check` whose reservation fell back. The section holds:

- the verdict line;
- NEW items, then UNKNOWN items, within the caps, with evidence summaries and possible
  owners;
- KNOWN and FLAKY counts;
- the pointer `sase tool show RUN -j`.

Omit the section when the run has no triage. Change no monitor result wire (decision
12).

Tests:

- DoD-10 fixtures for a monitor-reserved run and for a wrapped raw `just check`, each
  delivering the section with no starter prose;
- flag-off prompts unchanged (golden);
- DoD-9 in a fixture: `failures` counts equal a direct store query, linked-repo groups
  stay out of sase's `check`, and an empty result exits 0;
- the JSON shapes.

Remove the consumed `--epic-symbol` entries. None may remain after this phase.

## 9. acceptance-and-governance

- **Smoke harness.** Add a triage case group to `tools/smoke_sase_tool_runs` and its
  pytest twin, following the harness rules: isolated `SASE_HOME`, fixture commands only,
  live cases labeled `not-run` without `--live`. Cover:
  - an all-KNOWN stage continuing to the tests;
  - a NEW item stopping;
  - exit-code parity with a fail-fast run;
  - the safety net;
  - `show -j` after reaping;
  - `failures` grouping;
  - no bead writes.

  Map each case to its DoD. Note the new group in `tools/AGENTS.md` (it corroborates
  `sase-148`).

- **Backtest re-run (DoD-5).** Re-run `tools/tool_triage_backtest` on athena with the
  final knobs, re-audit a fresh sample of at least 50, and publish the results in the
  bead note.
- **Live athena acceptance, before the flag is removed.** Run
  `sase -f tool_failure_triage tool run check` on the day's master. Record the run id,
  the footer, the verdict, and whether continuation reached `test (scoped)` behind
  master-red lint stages. If master happens to be green, say so and rely on the
  fixtures. Record `sase tool failures -j` beside the matching direct ledger query
  (DoD-9). Confirm with `sase tool list -j` that the sase `check` definition digest is
  unchanged (DoD-12). Do not start a monitor from this turn. DoD-10's live observation
  is a post-landing owner check.
- **Remove `tool_failure_triage`:**
  - delete every Off branch and make the On branches unconditional;
  - drop the registry entry and the both-state scaffolding;
  - close the flag bead in the same change.
- **Docs.** Add a `## Failure triage` section to `docs/tool.md` covering:
  - kinds, classes, rules, knobs, and verdicts;
  - continuation modes and the exit-code contract, including the unwrapped-line caveat;
  - `failures`;
  - identity (linked repos, and the pre-fix rows left as history);
  - stage-description renames breaking continuity;
  - the fact that thin ledgers are mostly UNKNOWN.

  Note the follow-up triage section in `docs/monitors.md`.

- **Memory.** Make the edits in "Memory and governance" through `/sase_memory_write`,
  then run `sase memory init`.
- **Bead notes.** Record:
  - the DoD-0 to DoD-14 checklist status with evidence;
  - the post-landing measurement baseline: the share of failed `check` runs reaching
    `test (scoped)` (11.5% before E3), the share of agents whose final run executed
    tests (24%), and extra continuation time per day;
  - `PROPOSED FOLLOW-UP:` notes for the `rerun` and CI-evidence reopen triggers, for the
    E4 re-scope decision (§7 Q1 of the report), and for anything else discovered. They
    are notes, not new beads.

## Landing criteria (DoD, from §4.8 of the report)

| DoD | Criterion                                                                                                                                                     | Proved in                                                                  |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| 0   | `sase-182` and `sase-114` closed; `sase-145`'s acceptance re-verified (closed by E2)                                                                          | `ledger-hygiene`, `record-and-render`                                      |
| 1   | Signatures: goldens from real athena logs, cross-workspace equality, collisions, display bound, version refusal                                               | `core-failure-items`                                                       |
| 2   | Durable facts survive log reaping in `show -j`, alongside `sase-145` diagnostics                                                                              | `record-and-render`                                                        |
| 3   | Run kinds and legacy mapping, with no labels for control or infrastructure                                                                                    | `core-classification`                                                      |
| 4   | Classification fixtures (a)–(j)                                                                                                                               | `core-classification`                                                      |
| 5   | Precision backtest: ≥ 95% of ≥ 50 audited KNOWN labels pre-existing, zero on added files, KNOWN-but-touched dispositioned                                     | `bindings-and-backtest`, re-run in `acceptance-and-governance`             |
| 6   | Continuation safety: golden human output, `[fail, pass, fail, pass]`, the no-exit-0 property, `-x`                                                            | `keep-going`                                                               |
| 7   | Gated continuation, fail-safe stop, decisions in `show -j`                                                                                                    | `known-gated-continuation`                                                 |
| 8   | Honest surfaces: exit parity, four verdicts, and agreement across footer, `show -j`, `failures -j`, and monitor evidence; every JSON carries a schema version | `record-and-render`, `failures-and-followups`                              |
| 9   | `failures` on real data, matching a direct query, with linked repos isolated and empty results exiting 0                                                      | `failures-and-followups`, live in `acceptance-and-governance`              |
| 10  | Follow-ups carry the verdict and the NEW/UNKNOWN items for both reserved and wrapped runs                                                                     | `failures-and-followups`                                                   |
| 11  | No automatic bead writes                                                                                                                                      | `record-and-render`, smoke group                                           |
| 12  | Wire schema 1, older core still reads and reaps, no catalog or digest change, pin never behind a called binding                                               | `core-failure-items`, `bindings-and-backtest`, `acceptance-and-governance` |
| 13  | Governance: flag removed and its bead closed; docs; memory; glossary; decision record                                                                         | `acceptance-and-governance`                                                |
| 14  | Green-master independence                                                                                                                                     | every phase                                                                |
