---
tier: tale
title: Finish sase-1s5 by landing the exact-run dismissal lookup and closing the epic
goal:
  F11's exact run-identity dismissal lookup lands in sase-core and sase, verified on
  athena's real data; F10's apollo after-numbers are recorded; sase-1s5.3 and epic
  sase-1s5 are closed in the same turn.
size: medium
decisions:
  apollo_rollout:
    ask:
      Stage a human-approved apollo reinstall gate whose follow-up measures F11
      convergence in production?
    choices:
      gate:
        Gate the apollo reinstall; its follow-up records apollo's F11 after-numbers on
        sase-1s5.3
      manual:
        No gate; the reply gives the reinstall command and no agent records apollo F11
        numbers
    default: gate
    why:
      F11's goal is apollo convergence; the gate's guard now runs after the commits are
      pushed
    answer: gate
proposed_by: bbugyi200.athena.1a8
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 05:55:29
status: wip
---

# Finish sase-1s5: land the exact-run dismissal lookup, record apollo's F10 results, and close the epic

## Context

Epic `sase-1s5` (plan `plan:202611/health_watch_fixes.md`) has three phases. You are
finishing it in place of its land agent. `sase-1s5.land` was dismissed, so nothing else
will close the epic, and the user asked for it to be finished and closed. This tale is
the landing: it ends with the epic closed in the same turn as the final code.

State found on 2026-11-09:

- **`sase-1s5.1` (closed).** Landed in sase `ed534b0a9f`.
- **`sase-1s5.2` (closed).** Landed in sase-core `bee250a3` (batched dismissal-lineage
  walk), released in sase-core v0.44.1. Its apollo reinstall gate
  (`custom-196bbcc8-c58f-4df3-ae8a-8137e67e2897`) failed because it was approved before
  its commit was on origin, and the user cancelled it. However, apollo was reinstalled
  by hand at about 2026-11-09 10:27 UTC, with sase-core `b1688e01` and sase
  `9dc824fbb7`, and its gateway restarted then. Between 10:27 and 10:42 UTC there were
  13 presentation builds of 497–814 ms (mean 606 ms), all logged at `INFO`, with
  `timed_out_waiters=0` and no `504 Gateway Timeout`. So F10 is fixed in production.
  Only its after-numbers and the `sase-1j1.6` note are still unrecorded.
- **`sase-1s5.3` (in progress, F11).** Its 2026-11-08 run implemented the change.
  sase-core `just check` passed, and sase's check passed except for one failure that
  already exists on master. The run then ended its turn by **carrying** the uncommitted
  edits into a gate turn instead of committing them. Its apollo gate
  (`custom-d211de80-1933-4568-8265-ac6fa2ba3eea`) failed its own source-presence guard
  because the code was never pushed, and the user cancelled the gate. **Nothing from
  this phase is on either master.** Reimplement it from the spec below. That run's sase
  diff is kept, for reference only, in
  `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202611/08/20261108070206/main_diff.txt`.
  It applies cleanly to sase `9dc824fbb7`.
- **F11 still reproduces on both machines.** apollo's
  `~/.sase/axe/logs/lumberjack-checks.log` holds 1,397
  `dismissed index lookup is incomplete` lines, and athena's holds 8,070. athena's
  dismissed index (`~/.sase/dismissed_bundles/index.sqlite`) has 62,907 summary rows.
- **Query-plan probe.** I ran the real `query_summaries` SQL shape on a scratch copy of
  athena's index: the `LEFT JOIN`, the `COALESCE` visibility filter, the
  `project_file LIKE`, the `ORDER BY`, and `LIMIT 65`. Without the new indexes the plan
  is `SCAN s` and takes 18.7 ms. With both new indexes it is `MULTI-INDEX OR` over
  `idx_dismissed_bundle_agent_name` and `idx_dismissed_bundle_source_run_id` and takes
  0.23 ms. The result is the same either way.
- **No schema bump is needed.** Every writable open of the archive store runs
  `create_schema`, which executes `INDEX_DDLS` (`CREATE INDEX IF NOT EXISTS`). Existing
  stores therefore gain the indexes the next time they are opened for writing (the next
  dismissal, upsert, or rebuild). Read-only queries before that are still correct, just
  slower (about 19 ms). Do **not** bump `ARCHIVE_STORE_SCHEMA_VERSION`: a version
  mismatch drops and rebuilds every index table.
- **Version skew is safe.** `DismissedBundleIndexQueryWire` has no
  `deny_unknown_fields`, so an older installed core ignores the two new keys. It then
  runs today's unfiltered query, which overflows, and the new Python code reports that
  as `unresolved`, never `live`.

Before working in sase-core, open it with `sase repo open sase-core` and read its
`AGENTS.md`. Then run `just rust-install` from the sase checkout, so `.venv` builds
`sase_core_rs` from the linked sase-core checkout. The sase tests below need the new
wire fields.

## Step 1: exact run-identity filters in sase-core

All edits are in `crates/sase_core/src/agent_archive/`.

1. **`wire.rs`, `DismissedBundleIndexQueryWire`.** Add these two fields right after
   `suffixes`:

   ```rust
   #[serde(default)]
   pub agent_names: Option<Vec<String>>,
   #[serde(default)]
   pub source_run_ids: Option<Vec<String>>,
   ```

2. **`store.rs`, `INDEX_DDLS`.** Add these entries:
   - `CREATE INDEX IF NOT EXISTS idx_dismissed_bundle_agent_name ON dismissed_bundle_summaries(agent_name)`
   - `CREATE INDEX IF NOT EXISTS idx_dismissed_bundle_source_run_id ON dismissed_bundle_summaries(source_run_id)`

3. **`store.rs`, `query_summaries`.**
   - **Short-circuit.** Next to the empty-`suffixes` check, add this rule: when either
     identity field is `Some` and both fields are `None` or empty, return
     `Ok(Some(Vec::new()))` before opening the store.
   - **Identity clause.** Otherwise, add one clause
     `(s.agent_name IN (?,..) OR s.source_run_id IN (?,..))`. Include only the non-empty
     lists, sort each list for deterministic parameters, and AND the clause with every
     existing filter.
   - **SQL builder.** Move the SQL and parameter building into a private helper, for
     example
     `fn build_summary_query(request: &DismissedBundleIndexQueryWire) -> (String, Vec<rusqlite::types::Value>)`.
     `query_summaries` calls it after the short-circuits. The helper exists so a test
     can run `EXPLAIN QUERY PLAN` on the real statement.

4. **Tests in `store/tests.rs`:**
   - Add more than 64 unrelated hidden bundles plus targeted ones. Then check that:
     - filtering by agent name returns only that agent's bundles;
     - filtering by source run id does the same;
     - the two filters OR together;
     - both AND with `project_name` (another project's bundle is excluded) and with
       `visibility` (a `visible` bundle is excluded);
     - `agent_names: Some(vec![])` alone returns nothing;
     - an empty name list together with a run id returns the run-id matches.
   - A serde test: a request dict without the new keys deserializes both fields to
     `None`. The existing query tests cover "omitted fields behave as today", so keep
     them green unchanged.
   - An `EXPLAIN QUERY PLAN` test over the SQL that `build_summary_query` returns, with
     both identity filters plus `project_name`, `visibility: hidden`, and `limit`. Run
     it against a store opened through the normal writable path. Assert that the plan
     names both new indexes and has no `SCAN s` row. The probe above shows SQLite
     choosing `MULTI-INDEX OR` for exactly this shape.
   - If the planner ever refuses the indexes in the test, restructure the clause as
     `s.rowid IN (SELECT rowid … WHERE agent_name IN (…) UNION SELECT rowid … WHERE source_run_id IN (…))`.
     Never weaken the assertion.

5. **Verify.** In the sase-core checkout, run `just fmt`, then `sase tool run check`.
   - Commit subject, for the host commit:
     `feat(agent-archive): filter dismissed-bundle summaries by exact run identity`.
   - This is not a breaking change, because older readers ignore the new keys.
   - Do not touch `crates/sase_core_py/src/agent_custody/mod.rs`. It already
     deserializes the whole request dict into the wire type.

## Step 2: use the filters in sase, and never drop a failed lookup

1. **`src/sase/ace/dismissed_bundle_index/_api.py`, `query_summaries`.** Add keyword
   arguments `agent_names: set[str] | None = None` and
   `source_run_ids: set[str] | None = None`. Put them in the request dict as sorted
   lists, or `None` when not given.

2. **`src/sase/agents_sync/_publication_state_repair_observations_runs.py`,
   `_locate_dismissed_bundles_for_run`.**
   - Call
     `query_summaries(root, project_name=target.project_key, agent_names={local_name} if local_name else None, source_run_ids={source_run_id}, limit=_DISMISSED_CANDIDATE_LIMIT + 1)`.
   - Keep `_summary_is_candidate` as the post-filter, and keep the overflow guard. It
     now reports only genuine overflow.
   - In the `except (OSError, RuntimeError, ValueError, ImportError) as exc:` branch,
     return `(), f"dismissed index lookup failed: {exc}"` instead of `(), None`.
   - `project_name` is matched in core as `s.project_file LIKE '%/<project_key>/%'`.
     That matches both local and `/fleet/<key>/` project files. The post-filter's
     `project_name == target.project_key` check stays authoritative.

3. **Same module, `filesystem_observations_for_run`.**
   - When `bundle_error` is set, always append the `AGENT_RUN_STATE_EVIDENCE_DISMISSED`
     / `AGENT_RUN_STATE_OBSERVATION_UNREADABLE` observation. Do this even when artifact
     observations exist, and drop the early return.
   - At the end, return `tuple(observations), bundle_error`, so the diagnostic line and
     the `unresolved` counter stay truthful.
   - The resolver then answers `unreadable_evidence`, never `live_evidence`.

4. **Tests in `tests/agents_sync/test_publication_state_repair/`:**
   - **`test_production_dismissal.py`.**
     - Give `_write_dismissed_bundle` an `index: bool = True` parameter, so a test can
       write many bundles and rebuild the index once, and allow `agent_name=None`.
     - Test 1: 70 unrelated bundles plus the target's bundle. A published `active` run
       resolves to `dismissed`, with `report.changed == 1`.
     - Test 2: the target's bundle has no `agent_name` but the matching `source_run_id`,
       while another bundle has the same agent name and a different run id. The target
       still resolves to `dismissed`.
   - **`test_production_evidence.py`.** An artifact dir with no outcome, and
     `sase.ace.dismissed_bundle_index.query_summaries` monkeypatched to return 65 rows.
     Assert `report.live == 0`, `report.unresolved == 1`, `report.changed == 0`, and a
     `dismissed index lookup is incomplete` diagnostic.

5. **Verify.**
   - Run the two modules above plus the rest of
     `tests/agents_sync/test_publication_state_repair/`.
   - Then run `just fmt` and `sase tool run check`.
   - `tests/test_core_rust.py::test_require_rust_extension_raises_when_missing` already
     fails on clean master `9dc824fbb7`, raising `ImportError: No module named 'sase'`
     from its own patched `import_module`. If the scoped lane selects it, treat it as
     pre-existing evidence for step 5. Every other failure is yours.

## Step 3: real-data check of F11 on athena (report-only, writes nothing)

1. Pick an athena project whose recent `agent_state_refresh` diagnostics include
   `dismissed index lookup is incomplete`. `gh_sase-org__sase` is the likely candidate.
2. **Before.** Run the installed CLI: `sase agent sync --repair-states -p <project>`.
   Without `--apply` it only reports. Record examined, live, unresolved, and changed,
   plus the count of `incomplete` diagnostics.
3. **After.** Run the same command through this checkout's code:
   `.venv/bin/sase agent sync --repair-states -p <project>`, from the sase checkout
   root, after `just rust-install`.
4. **Pass bar:**
   - no `dismissed index lookup is incomplete` diagnostics;
   - every run with a matching dismissed bundle is reported as changing to `dismissed`;
   - `live` is no more than the number of genuinely running agents for that project.
5. If the bar fails, find out why and fix it before going on. Do not close anything
   while F11 still fails on real data.
6. Record a `sase bead note sase-1s5.3 "F11 AFTER (athena report-only): …"` note with
   the before and after numbers.

## Step 4: record F10's apollo results

All reads in this step go over `ssh apollo`.

1. **Gateway builds.** From `~/.sase/service/procs/gateway/output.log`, summarize the
   presentation-scope `fleet snapshot build finished` lines since the 2026-11-09 ~10:27
   UTC gateway restart (strip ANSI first): count, min, max, and mean `duration_ms`; any
   `WARN` level; the sum of `timed_out_waiters`; and any `504 Gateway Timeout` since the
   restart.
2. **Fleet catalog timing.** Time
   `sase_core_rs.assemble_fleet_catalog({"sase_home": str(Path.home() / ".sase")})` with
   apollo's installed tool Python, `~/.local/share/uv/tools/sase/bin/python`. It must
   finish in under 2 s.
3. **Pass bar:**
   - builds stay under 4 s and are logged at `INFO`;
   - `timed_out_waiters=0` across at least 10 minutes;
   - no 504 since the restart;
   - the catalog finishes in under 2 s.
4. **Record.**
   - Add
     `sase bead note sase-1s5.2 "F10 AFTER (apollo, reinstalled 2026-11-09 ~10:27 UTC): …"`.
   - Add a note on `sase-1j1.6` that its long-running-build criterion can now be
     re-measured, with these numbers. Do not close `sase-1j1.6`.
5. If F10 fails the bar, run `sase bead open sase-1s5.2` (this also reopens the epic),
   note the evidence, skip step 6's closes, and report.

## Step 5: land triage and integration

- **The one `PROPOSED FOLLOW-UP` (`sase-1s5.3` note #2,
  `test_require_rust_extension_raises_when_missing`).** It is not caused by this epic.
  `sase-1rq.2.1.4.1.5.2` (closed) and `sase-th` (in progress) mention the test. Run
  `/sase_new_task`, identify `sase-1s5.3` as the proposer, and cite the clean-master
  reproduction. Record the outcome (corroborated, attached, or created) for the epic's
  close note.
- **Integration.** At planning time no commit since the epic started had touched the
  changed paths. Both diffs applied cleanly to sase `9dc824fbb7` and sase-core
  `b1688e01`. The other `query_summaries` callers
  (`src/sase/ace/dismissed_agents_bundles.py`,
  `src/sase/ace/tui/widgets/decks/record/relations.py`, and
  `src/sase/ace/tui/widgets/decks/record/facts.py`) look up by suffix and need no
  identity filters. Re-run `git log` over those paths quickly and handle anything new.
- **Epic symbols.** `sase bead epic-symbols sase-1s5` reported no entries at planning
  time. Re-run it, and resolve or re-key any entry that has appeared.

## Step 6: close out the epic in this turn

1. **Close the phase.** Run
   `sase bead close sase-1s5.3 --note "<what landed, tests, the athena report-only numbers>"`.
2. **Close the epic.** Run `sase bead close sase-1s5 --note "<…>"`. Normally only the
   land agent closes an epic. Here the land agent was dismissed and the user explicitly
   asked for the close, so this tale acts as the lander. The note covers:
   - phases 1–3 verified against their commits;
   - F10's apollo after-numbers;
   - F11's athena report-only after-numbers;
   - the follow-up triage outcome;
   - the integration check;
   - and, under `apollo_rollout = gate`, that the apollo F11 measurement follows in the
     gate turn.
3. **Symvision.** Run `just symvision`.
4. **Mark the plan done.** Open the plans repo with `sase repo open plans`. Set
   `status: done` in the frontmatter of `202611/health_watch_fixes.md`.

> [!decision] apollo_rollout = gate Before `/sase_final`, stage one custom gate with
> `/sase_gate`, titled "Reinstall apollo for sase-1s5 F11". Its approve command runs
> from athena:
>
> ```sh
> ssh -o BatchMode=yes -o ConnectTimeout=20 apollo /bin/bash <<'REMOTE'
> set -eu
> export PATH="/home/bryan/.local/bin:/home/bryan/.cargo/bin:$PATH"
> CORE=/home/bryan/projects/github/sase-org/sase-core
> SASE=/home/bryan/projects/github/sase-org/sase
> git -C "$CORE" fetch origin
> git -C "$SASE" fetch origin
> git -C "$CORE" grep -q source_run_ids origin/master -- crates/sase_core/src/agent_archive/wire.rs \
>   || { echo "sase-core origin/master lacks the sase-1s5 identity-filter commit" >&2; exit 2; }
> git -C "$SASE" grep -q agent_names origin/master -- src/sase/ace/dismissed_bundle_index/_api.py \
>   || { echo "sase origin/master lacks the sase-1s5 dismissal-lookup commit" >&2; exit 2; }
> git -C "$CORE" checkout master && git -C "$CORE" pull --ff-only origin master
> git -C "$SASE" checkout master && git -C "$SASE" pull --ff-only origin master
> cd "$SASE"
> SASE_GLOBAL_INSTALL_BYPASS='Human approved sase-1s5 F11 rollout' just install-dev >&2
> sase service proc restart gateway >&2
> REMOTE
> printf '{"status":"installed"}\n'
> ```
>
> - **No scheduler restart.** `agent_state_refresh` runs as the
>   `sase_job_agent_state_refresh` `script:` subprocess on each tick, so it picks up the
>   new install without one.
> - **Gateway restart.** It keeps the running gateway on the installed core, and
>   re-observes F10's first request after a restart.
> - **Approve follow-up prompt.**
>   1. Sleep about 15 minutes through `/sase_monitor`, covering at least three
>      five-minute ticks.
>   2. On apollo, after the install time, measure: no new
>      `dismissed index lookup is incomplete` lines; at least one `agent_state_refresh`
>      tick with `changed>0`; `live` no more than the genuinely running agents;
>      `bob-cli-hg.14--mon` published or reported as `dismissed`, using
>      `sase agent sync --repair-states -p gh_bobs-org__bob-cli`; and gateway builds
>      still at `INFO` with no 504.
>   3. Record `sase bead note sase-1s5.3 "F11 AFTER (apollo): …"`.
>   4. If F11 does not converge, run `sase bead open sase-1s5.3` (this reopens the
>      epic), note the evidence, and report.
> - **Reject follow-up prompt.** Note on `sase-1s5.3` that the rollout was declined and
>   that apollo converges at the next human reinstall.
> - **Do not include athena.** Reinstalling athena under its running agents is the
>   user's call.

> [!decision] apollo_rollout = manual Stage no gate. The final reply gives the user the
> exact apollo reinstall (fast-forward both checkouts, `just install-dev`,
> `sase service proc restart gateway`) and the F11 pass bar from the gate variant, and
> says that apollo's F11 after-numbers are not recorded by an agent.

## Finishing the turn

- Finish through `/sase_final`. **Commit** the sase repo, the sase-core sibling, and the
  plans repo. **Never choose `carry`**: carrying is what stranded phase 3.
- The host commits the pinned sase-core sibling first and writes its pushed SHA into
  `sase-core-revision.txt` before the sase commit. Do not hand-edit the pin.
- Stage the gate, if any, before declaring. Do not mutate files after the declaration.
- Commit subject for the sase change:
  `fix(agents-sync): find dismissed bundles by exact run identity and never read a failed lookup as live`.
  Mention `sase-1s5.3` in the commit body.

## Verification summary

- sase-core: the new store tests pass, and `sase tool run check` passes.
- sase: the state-repair tests pass, and `sase tool run check` passes. The only allowed
  failure is the pre-existing one, with evidence.
- athena report-only F11 check passes its bar, with numbers noted on `sase-1s5.3`.
- apollo F10 after-numbers are noted on `sase-1s5.2` and `sase-1j1.6`.
- `sase-1s5.3` and `sase-1s5` are closed, `just symvision` is clean, and the plan file
  says `status: done`.
