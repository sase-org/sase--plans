---
tier: tale
title: Finish exact-run reads and land published-state repair
goal:
  Published-state repair validates current run identity, bounds all reads, preserves
  retry progress, and closes sase-1pu.5.1.5 and its ready parent chain.
size: medium
proposed_by: bbugyi200.athena.sase-1pu.5.1.5.land
proposed_with: codex/gpt-6-astra
bead: sase-1pu.5.1.5
status: done
---

- **PARENT:**
  [202611/published_state_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202611/published_state_landing_repairs.md)
- **BEAD:**
  [sase-1pu.5.1.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pu/sase-1pu.5.1.5.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1pu.5.1.5.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1pu.5.1.5.land.md)
- **COMMITS:**
  - [496cfdf](https://github.com/sase-org/sase--plans/commit/496cfdfb636bdc41353cb305e2f6b6f7e7d3b4d7)
    — chore(plans): mark published-state repair plans done

# Finish exact-run reads and land published-state repair

Complete only the remaining work in epic `sase-1pu.5.1.5`, then perform its nested
closeout in this same coding turn. This tale has no land agent. Its coder must finish
the landing; do not wait for this turn's commit SHA, push, or CI before closing.

The linked epic plan is `plan:202611/published_state_landing_repairs.md`. Its direct
parent is plan bead `sase-1pu.5.1`, linked to
`plan:202610/published_run_state_convergence.md`; that plan's direct parent is phase
`sase-1pu.5` in `plan:202610/task_backlog_epic_3.md`. The final inherited decision is
`optional_scope = keep_five_seeds`. Keep the existing scope: no unrelated features,
memory edits, fleet backfill, global install, or historical purge.

One coder can finish this bounded Python adapter/transaction repair using the already
landed Rust contracts. Keep classification, terminal precedence, window/cursor rules,
and scheduler policy in Rust. There is no new core API or containing-SHA dependency in
this tale. Preserve the existing core pin or a newer containing revision.

## Landing evidence and what already works

The lander read the epic and both phases, every note, both implementation commits, the
Rust classifier/discovery and registered bindings, Python consumers, and the parent
landing notes and plans. Reviewed sase HEAD and `origin/master` are `a0b00da79b`. Core
`dc5383f6` is the actual phase `.1` commit and the Python pin; `a0b00da79b` is phase
`.2`. Existing focused repair, preservation, and scheduler suites passed 47/47 in 41.05
seconds. Parent checkpoint wiring is a real production consumer, and
`sase bead epic-symbols` reports no entries for either `sase-1pu.5.1.5` or
`sase-1pu.5.1` at this tip.

The earlier phase check, ToolRun `af92a91bfbe6635cd5c072966a9dc112`, exited 1; all lint
stages passed, and its only test failure was the existing AGY timeout flake. Its triage
verdict is `no_new_failures`, not a passing exit. Do not inherit a full-pass claim. No
`just check-full` is authorized.

Integration since this child epic began was reviewed: sase `288d2afeeb` adds staged
handoff readoption and held-notification text, without changing marker outcomes or
publication entry points. Core `dc119c0b` is the unrelated four-file staged-handoff
rustfmt fix. Neither needs a new feature integration change. Retain the earlier
background commit enqueue, planner publication scope, protection reuse, and scheduler
pre-charge/backoff integration already present. Recheck any newer commits before
closing.

The landed tests miss real defects. A temporary sidecar/index probe at the reviewed tip
reproduced the failures below. Read its saved script with
`sase artifact read file:explicit:ed35fecafbc1e54897ae808e "Reproduce landing gaps"` and
its output with
`sase artifact read file:explicit:80004e2a57ce240ac8f0ef75 "Review landing evidence"`.
The script uses temporary directories and repository test builders, not live backfill.

Reproduced failures:

- Missing dismissed index: `filesystem_observations_for_run` calls the rebuilding
  archive loader; one selected completed run invokes the rebuild once.
- Change current `agent_meta.artifact_agent_id` after indexing: the old published run
  still resolves to `completed`. Change current metadata `name` to another name while
  leaving `done.name` unchanged: dictionary merging erases the conflict and still
  resolves to `completed`.
- Replace the indexed artifact directory with a symlink outside the projects root: the
  reader accepts its completion marker.
- A real indexed dismissed bundle with `agent_name=foo.job`, `cl_name=SomePatch`, and
  the selected suffix is missed: `active / missing_evidence`. The lookup incorrectly
  uses the agent name as the patch-name filter. Conversely, a stale matching summary
  whose current bundle has a different `agent_name` and `raw_suffix` still yields
  `dismissed / updated_from_dismissed`.
- With `state_limit=1`, payload planning reads an unselected sibling's `prompt.md` and
  all eight hood snapshots in the fixture. `_plan_manifest_and_pages` still calls
  `load_validated_publication`, which revalidates every file in changed hoods and loads
  the rest of the publication. The test `_READ_HOOK` observes only the custom marker
  reader, and the seeded sidecar records contain only state files, hiding these reads.
- Advance controlled time beyond the deadline during the last evidence read:
  `plan_repair_window` returns `error=None`, `budget_exhausted=False`, a next cursor,
  and a changed row. There is no post-read deadline check.

These are unfulfilled epic contracts, not unrelated task beads.

## 1. Complete the filesystem evidence adapter

Work primarily in `_publication_state_repair_observations.py`, its narrow core-facing
adapters, and `inventory_sources.py` as needed. Read the ACE memory before touching
anything under `src/sase/ace/`.

Treat cached records and summary rows only as locators. Validate canonical paths against
their owning roots before reading; canonicalize both sides so legitimate symlinked
ancestors work. Validate current metadata's durable artifact ID, project, owner,
canonical name, and the derived source-run identity. Pass contradictory claims from
separate current sources to the existing Rust resolver without overwriting them. Map
real archive keys (`agent_name`, `raw_suffix`, project/source owner fields, import
markers) rather than assuming the completion-marker field names. Do not silently discard
malformed or unreadable identity metadata and then accept an anonymous terminal marker.
Keep supported older markers when their exact directory/current metadata prove identity.
Use Rust's existing multiple-observation and uncertainty handling; do not implement a
second Python precedence/classification policy.

Use a read-only, no-rebuild dismissed-index query through a thin adapter to the existing
core binding. `ace.dismissed_bundle_index.query_summaries` already has that underlying
capability and returns `None` for an unavailable index. Do not call the convenience
`load_dismissed_bundle_summaries` that rebuilds. Select candidates without conflating
`agent_name` with `cl_name`, then validate current selected bundle identity. Keep all
matching observations so conflicting copies remain unresolved and later dismissal can
win under the existing Rust rule.

Bound candidate decoding and diagnostics. The current artifact query requests full
history with no row limits; the word `bounded` in its Python wrapper primarily denotes
lock acquisition. Use finite `active_limit` and `recent_completed_limit`, disable the
unlimited `include_full_history` branch, and request a sentinel row to detect
overflow/incompleteness. Do not rely on `window_limit` alone: its specialized path
excludes hidden rows and leaves the active tier unlimited. Missing, busy, oversized, or
ambiguous lookup must be explicitly unresolved, never trigger a history walk, rebuild,
or transcript read. Preserve preview purity. Restrict actual marker/bundle reads to
selected immutable runs. Share the corrected observation mapping with normal publication
so its state bytes obey the same rules.

## 2. Finish bounded state publication and deadline handling

Keep the existing transaction lock, upstream pull, protection checks, and retry
protocol. Carry already validated selected snapshots from discovery to payload planning
rather than reopening them unnecessarily. Before digesting or parsing a snapshot,
enforce the existing JSON byte bound; `previous_snapshot` currently performs an
unrestricted `read_bytes` before the bounded reader.

Give state-only page planning a narrow path that reuses the validated selected snapshot
metadata and existing manifest/index context without hydrating unselected content. Do
not weaken ordinary publication's `verify_run_files` contract globally. Keep all
unselected file references and bytes, foreign owners, corrupt-sibling entries, and
unaffected browsing pages intact. Preserve previously rendered index rows for unselected
units rather than loading arbitrary extra snapshots to regenerate them. Use the existing
targeted/degraded rendering infrastructure, extending it narrowly as needed; do not add
another persistent corpus index. New state, snapshot, manifest, and page bytes must
still pass the normal publication-protection boundary.

Propagate the work deadline through evidence and payload/page planning. Check before and
after potentially expensive reads and before beginning another publication step,
including the last selected run and no-write paths. Budget expiry and uncertain
publication must report a failure, `pass_complete=False`, and no falsely advanced
cursor. Preserve the starting scheduler checkpoint and its existing failure backoff. Do
not begin a push-ahead operation after exhausting the budget. On partial work, advance
only the position actually verified by the existing core window policy; safe replay of
the starting window is preferable to skipping unverified rows. Recompute observations
and snapshots after non-fast-forward recovery. Keep individually corrupt or unresolved
rows diagnosable without starving healthy later rows.

## 3. Prove the production paths and verify

Turn the reproduced defects into regression tests using temporary roots, a real
artifact/archive index, and temporary bare remotes. Extend the existing focused test
modules or add focused siblings; avoid a large monolithic test file.

- Cover changed IDs and conflicting metadata/marker names after indexing, malformed
  identity metadata, foreign/imported evidence, and artifact/archive symlink escapes.
- Cover real dismissal where patch and agent names differ, stale bundle summaries,
  completion plus later dismissal, contradictory copies, and missing/busy indexes.
  Assert that preview neither creates nor rebuilds an index.
- Seed many same-hood runs with real prompt/chat/content references, plus more than five
  hoods and a foreign/corrupt sibling. Instrument actual filesystem read entry points
  and archive/index calls, not only `_READ_HOOK`. With `state_limit=1`, prove finite
  discovery and candidate work, selected-run evidence reads, no unrelated content
  hydration, and preservation of unrelated files and index rows.
- Expire controlled time during the last evidence read and page generation. Test both
  preview and apply, including no-write and mixed partial outcomes; verify error,
  cursor, pass-complete, and persisted scheduler checkpoint/backoff fields.
- Drive the real CLI and `run_agent_state_refresh` through mixed success, failure,
  stop/dismissal, and live evidence. Verify remote state files, snapshot fields,
  manifest digests, pages, restart/resume, and a second no-op pass. Exercise later
  background/planner publication and concurrent same-owner recovery, not just injected
  inventory objects, so terminal preservation is proved at current entry points.

Run relevant focused tests, `just fix`, and `sase tool run check` (the wrapped
`just check`). Read lint/test memory first. Use `/sase_monitor` if a command requires
handoff, with a continuation explicitly retaining the closeout below. Do not run
`just check-full`. Record actual exit codes and triage evidence; unrelated known flakes
must not hide a new product failure. If changed behavior makes the existing runbook
inaccurate, read docs-writing memory and correct only that description.

## 4. Finish this epic and its ready parent chain in this coding turn

All step-3 follow-up triage from the lander is already recorded on `sase-1pu.5.1.5`.
Include these outcomes in the close note: phase `.1` note 1's staged-handoff rustfmt
proposal was declined as a new task because `dc119c0b` and closed `sase-1qk.1` resolve
it; resolution evidence was added to owner `sase-1pj.6.1`. Phase `.2` note 1's AGY flake
was corroborated with a +1 on existing `sase-19m`, citing the audited ToolRun. No new
task was warranted. The parent lander's older proposal outcomes are recorded in
`sase-1pu.5.1` note 2 and must also be included when closing that parent.

Re-read `sase-1pu.5.1.5` and both child phases with `sase bead read -r`, their notes,
linked plan, the final diff, and newer commits. Confirm the above acceptance and
inherited decisions. Resolve every entry from `sase bead epic-symbols sase-1pu.5.1.5`:
wire up, privatize, delete, or use a legitimate non-test pragma; re-key only when a
still-open later bead really needs the exemption. Then close normally:

```sh
sase bead close sase-1pu.5.1.5 --note "<actual verification, integration, and every follow-up outcome>"
just symvision
```

Open `plans` through `/sase_repo`, read its artifact through `sase artifact read`, and
set `status: done` in `202611/published_state_landing_repairs.md` frontmatter only after
the close succeeds. If symbol cleanup is still refused, finish the cleanup and retry.
Never use `--force` to make a successful nested landing advance.

Next re-read parent `sase-1pu.5.1`, its previous landing notes, all descendants and
every note (original phases `.1` through `.4`, this child epic and both its phases, and
any newly added descendants), and `plan:202610/published_run_state_convergence.md`.
Rerun descendant and linked-plan readiness checks and inspect post-child drift. When
complete, retire all `sase bead epic-symbols sase-1pu.5.1` obligations, run
`sase bead close sase-1pu.5.1 --note "<rechecked acceptance, drift, and prior follow-up outcomes>"`,
run `just symvision`, and set `status: done` in
`202610/published_run_state_convergence.md` in the opened plans repo.

Then inspect phase parent `sase-1pu.5`. Verify completed/failed convergence, live-run
safety, concurrent publication, and the tested bounded owner-local backfill procedure.
Resolve `sase bead epic-symbols sase-1pu.5`, append
`TASK RESOLVED: sase-1cc — <actual convergence and preservation evidence>`, and close
only that phase with `sase bead close sase-1pu.5 --note "<actual verification>"`. Leave
containing epic `sase-1pu`, its plan, and seed `sase-1cc` to its waiting lander. If any
parent is incomplete or ambiguous, record its concrete blocker and stop closing at that
parent. Do not force unfinished phases closed. Submit `/sase_final` for the host-owned
code and plan-status commits after finishing the turn's work.
