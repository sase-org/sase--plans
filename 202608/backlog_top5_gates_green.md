---
tier: epic
status: done
title: Task backlog top five — turn the mandatory verification gates green
goal: "`just check-full`'s last gate (`just selection-health --fail-on-new-flake`) stops
  reporting nodes that were already fixed, and stops reporting the two live
  full-parallel-lane flakes it currently names; the recurring generated-memory drift
  that flips `just check` red at the SASE validation step can no longer reappear
  undetected; and monitor reconciliation's locked settle path stops issuing an
  archive-scaled artifact-index query per candidate. Closes task beads sase-o0, sase-nd,
  sase-nz, sase-n0, and sase-ne.

  "
phases:
  - id: flake_retire
    title: Retire already-fixed nodes from the flake-baseline gate (sase-o0)
    depends_on: []
    size: medium
    description: "flake_retire: declare `# fixed-at:` retirement in
      tests/reproducible_flake_baseline.txt for every currently-reported node whose fix
      commit can be named, starting with the proven
      test_override_pills_keep_narrow_top_bar_in_bounds case, and audit the three
      remaining unexplained nodes without declaring any node that cannot be tied to a
      named fix commit.

      "
  - id: memory_check
    title: Reconcile the two generated-memory drift checkers (sase-n0)
    depends_on: []
    size: medium
    description: "memory_check: make `sase validate`'s memory step and `sase init memory
      --check` resolve the same template source for the same generated note so they
      cannot give opposite answers on one tree, and add a regression gate; leave a `TASK
      NEEDS APPROVAL` note rather than editing or regenerating any memory file content.

      "
  - id: reconcile_marker
    title: Bound the monitor settle path's artifact-index reads (sase-ne)
    depends_on: []
    size: medium
    description: "reconcile_marker: switch monitor reconciliation's locked settle and
      re-read from get_monitor() to the path-direct read_monitor_marker() helper (or an
      equivalently bounded lookup), preserve settlement semantics, and pin the bound
      with a test asserting the reconcile pass does not scale its artifact-index queries
      with candidate count.

      "
  - id: monitor_idle_bound
    title: Deflake the supervisor idle-timeout no-hang bound (sase-nd)
    depends_on:
      - flake_retire
    size: large
    description: "monitor_idle_bound: root-cause and fix
      tests/monitor/test_monitor_supervise.py::test_run_supervisor_idle_timeout_fires_after_output_stalls
      exceeding its 5.0s `_NO_HANG_TIMEOUT` under a contended full parallel lane,
      without weakening the idle-timeout contract the test asserts.

      "
  - id: approval_anchor
    title: Deflake headless epic approval against an inflight launch (sase-nz)
    depends_on:
      - flake_retire
    size: large
    description:
      "approval_anchor: root-cause and fix
      tests/test_plan_approval_actions.py::test_headless_epic_approval_submits_while_inflight_launch_holds_anchor
      failing intermittently under the full parallel lane, without weakening the
      inflight-launch-holds-anchor assertion."
proposed_by: bbugyi200.athena.sase-ns.6.land--1
parent_bead: sase-ns.6
bead_id: sase-ns.6.6
create_time: 2026-09-09 19:50:02
---

- **PROMPT:**
  [prompts/202608/backlog_top5_gates_green.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/backlog_top5_gates_green.md)
- **BEAD:**
  [sase-ns.6.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ns/sase-ns.6.6.md)

# Task Backlog Top Five — Turn The Mandatory Verification Gates Green

## Why This Epic Exists

A backlog triage pass on 2026-08-17 reviewed all ready SASE task beads and selected the
five with the most impact. The organizing fact is that **the last gate of
`just check-full` is currently red on clean master**, and every agent in this repo must
pass that gate before landing.

At the triage HEAD, `just selection-health --fail-on-new-flake` reported seven
reproducible flakes exceeding `tests/reproducible_flake_baseline.txt`:

```
tests/ace/tui/test_top_bar_order.py::test_override_pills_keep_narrow_top_bar_in_bounds
tests/fakey/test_usage_limit_e2e.py::test_usage_limit_failure_disables_only_fakey_and_preserves_error
tests/main/test_var_integration.py::test_var_cli_end_to_end_refreshes_index_and_round_trips_machine_outputs
tests/monitor/test_monitor_supervise.py::test_run_supervisor_idle_timeout_fires_after_output_stalls
tests/test_bead/test_cli_golden.py::test_bead_cli_golden_contract[stats]
tests/test_plan_approval_actions.py::test_headless_epic_approval_submits_while_inflight_launch_holds_anchor
tests/test_query_profile.py::test_provider_query_schema_derives_fields_from_the_notes_fixture
```

This epic addresses six of the seven: `flake_retire` covers the first, third, fifth and
seventh; `monitor_idle_bound` covers the fourth; `approval_anchor` covers the sixth. The
`test_usage_limit_e2e` node is deliberately out of scope — it belongs to the in-progress
epic sase-n4 and must be left to that epic.

Two further beads round out the five: `memory_check` (sase-n0) removes a recurring red
on `just check`'s SASE validation step that has now hit at least six independent agent
shells, and `reconcile_marker` (sase-ne) removes a measured 1.51s-per-candidate
archive-scaled query from monitor reconciliation's locked settle path.

## Rules Every Phase Must Follow

**Bead protocol.** Each phase owns exactly one task bead.

1. Before starting work, mark the bead in progress:
   `sase bead update <bead-id> -s in_progress`
2. When the work is done (or when you have concluded you cannot finish it), append a
   note describing what you actually did — or, if you could not finish, an honest
   justification naming what blocked you:
   `sase bead note <bead-id> "<what you did / why you could not>"`
3. If and only if the work is genuinely complete and verified, close the bead:
   `sase bead close <bead-id> --note "<what you verified>"` Do not close a bead you did
   not finish. Leave it ready and say so in your note.

**Approval protocol.** Do not ask the user a question directly. When a phase hits a
decision this plan marks as owner-owned, append a note that begins with the literal
token `TASK NEEDS APPROVAL` to that phase's task bead, describing the decision, the
options, and your recommendation. Then finish everything in the phase that does not
depend on that decision. Each phase below states explicitly whether it has an approval
trigger; phases without one are objective improvements and must not raise an approval
note just to be cautious.

**Follow-ups.** Phase workers must not create task beads. Record discovered follow-up
work as a `PROPOSED FOLLOW-UP: <summary — detail>` note on your own phase bead. The land
agent routes those.

**Verification.** Run `just install` first — workspaces are ephemeral and may hold stale
dependencies. Then run `just check` before replying. `just check-full` routinely outruns
a single agent turn, so run it **only** through `/sase_monitor` with a `--next` action,
never inline. Phases that change flake behavior must additionally run
`just selection-health --fail-on-new-flake` and report its exact node list.

**Boundary.** Per the repo's Rust core backend boundary rule, shared backend and domain
behavior belongs in `../sase-core`. None of these five phases is expected to cross that
boundary; if one turns out to, open the sibling repo through `/sase_repo` and say so in
your note rather than reimplementing core logic locally.

---

## Phase `flake_retire` — Retire Already-Fixed Nodes (sase-o0)

**Bead:** sase-o0 · size medium · _Declare fixed-at retirement for already-fixed nodes
still red in the flake-baseline gate_

### Background

`tests/reproducible_flake_baseline.txt` supports a per-node directive:

```
# fixed-at: <UTC timestamp> <node id>
```

It retires only that node's failure evidence recorded at or before the instant its fix
landed. It does not touch any other node's evidence, does not move the gate's bar for
new flakes, and a failure of the same node recorded _after_ the timestamp is ordinary
live evidence again. The file header documents the mechanism and its known limitation.

The directive was introduced by closed bead sase-nv and applied to the nine
config/config-cache nodes fixed by `3a22ff04f`, plus the config-center node fixed by
`d9b2984a7`. Nodes whose fixes landed _before_ the syntax existed were never declared,
so the gate stays red on historical evidence for tests that now pass.

### The Proven Case — Do This One First

`tests/ace/tui/test_top_bar_order.py::test_override_pills_keep_narrow_top_bar_in_bounds`
was fixed by commit `981106799` _"docs: migrate docs and tests off retired model-alias
names"_ (2026-08-16T04:53:04Z, epic sase-mf phase sase-mf.4). Task bead **sase-mp is
already closed as done** on that fix — do not reopen it; only its historical evidence
needs retiring.

The durable store holds 12 failure records for the node. The last two are at
2026-08-16T05:28:43Z and 05:37:11Z, both from HEAD `6f7052fc9`, which is **not** an
ancestor of `981106799` — i.e. stale pre-fix trees, exactly the "Known limitation" the
baseline header documents. A probe verified that adding

```
# fixed-at: 2026-08-16T04:53:04Z tests/ace/tui/test_top_bar_order.py::test_override_pills_keep_narrow_top_bar_in_bounds
```

retires 8 more failures and drops the node out of the gate's report (7 → 6 exceeding
nodes). The node also passes in isolation under `-p no:randomly`.

Follow the file's existing convention: a preceding comment naming the bead and the
commit that fixed the node, then the `# fixed-at:` line.

### The Remaining Audit

Three nodes are reported but their root cause is not yet determined:

| Node                                                                                                          | Failure records | Last record                              |
| ------------------------------------------------------------------------------------------------------------- | --------------- | ---------------------------------------- |
| `tests/main/test_var_integration.py::test_var_cli_end_to_end_refreshes_index_and_round_trips_machine_outputs` | 15              | 2026-08-16T19:55:21Z at HEAD `0ec2018f1` |
| `tests/test_bead/test_cli_golden.py::test_bead_cli_golden_contract[stats]`                                    | 17              | 2026-08-16T23:08:29Z at HEAD `e50d8a953` |
| `tests/test_query_profile.py::test_provider_query_schema_derives_fields_from_the_notes_fixture`               | 10              | 2026-08-16T15:12:02Z at HEAD `95d66f59c` |

All three pass in isolation today, and none failed across the 117 full-run records after
2026-08-16T06:00Z — but "passes in isolation" does not distinguish a fixed node from a
live full-lane flake. For each node, do exactly one of:

- **Identify the fix commit** (git-log the node's file and the code it exercises across
  the window between its earliest and latest failure record), confirm every failure
  record predates that commit, and declare the `# fixed-at:` entry naming the commit.
- **Leave it reported** and record on your phase bead, as a `PROPOSED FOLLOW-UP:` note,
  that the node is unexplained and needs a flake bead.

For `test_bead_cli_golden_contract[stats]` specifically: bead sase-na.3 notes the golden
fixture was missing a `Flags` line, which the in-progress epic sase-nb may have since
supplied. Check whether a sase-nb commit updated that golden; if it did, that is your
fix commit.

The durable evidence lives in `~/.sase/test-selection/gh_sase-org__sase/*full-run.json`
(each record has `failures`, a HEAD, and a `changed_files` list).
`tools/select_tests --explain` and `just selection-health` are the supported readers.

### Scope Guardrail — Read This Before Editing The Baseline

The baseline file's header states its entries are "debt to remove, not suppressions to
grow." Therefore:

- **Do not** bump the file-wide `# effective-after:` timestamp as a shortcut.
- **Do not** declare a `# fixed-at:` for any node you cannot tie to a named fix commit.
  That would turn the directive into the blanket suppression the header forbids.
- **Do not** add plain node entries (suppressions) for any of these four nodes.

### Done Means

- `just selection-health --fail-on-new-flake` reports strictly fewer exceeding nodes
  than the seven listed above, with every removal justified by a named fix commit.
- Every node you did not declare is either fixed by `monitor_idle_bound` /
  `approval_anchor` (leave those alone — they are other phases' work), owned by epic
  sase-n4 (`test_usage_limit_e2e`, leave alone), or recorded as a `PROPOSED FOLLOW-UP:`
  on your phase bead.
- `just check` is green.

### Approval Trigger

None. Declaring a proven fix is an objective improvement. The guardrail above is the
control, not an approval gate.

---

## Phase `memory_check` — Reconcile The Two Memory Drift Checkers (sase-n0)

**Bead:** sase-n0 · size medium · _`sase validate` and `sase init memory --check`
disagree about `sase/memory/sase_sizes.md`, so running the mandatory memory init flips
`just check` red_

### Background

Two entry points check generated-memory drift, and they resolve **different template
sources for the same generated note**, so they can give opposite answers for the same
file at the same commit. The reported reproduction:

```
$ sase validate
SASE validation
  ok     init memory --check      <-- GREEN
  ...

$ sase init memory --check --diff
Needs attention:
  run  init memory  update 2 memory files and provider shims
       ~ update  sase/memory/sase_sizes.md  +5 -4  generated SASE size memory
       ~ update  sase/memory/README.md      +4 -4  memory README
```

An agent that obeys the mandatory memory-init workflow therefore produces a red
`just check`; an agent that does not leaves the drift latent for the next agent.

**This is a recurring, cross-agent cost.** Bead sase-i7 was closed as superseded by
sase-n0 during the 2026-08-17 triage after collecting five independent reproductions of
the same drift across six agent shells between 2026-08-09 and 2026-08-17. Two shapes
were reported:

- the five chezmoi home provider shims
  `~/.local/share/chezmoi/home/{AGENTS,CLAUDE,GEMINI,QWEN,OPENCODE}.md`, each +2/−2,
  heading style only;
- `~/.local/share/chezmoi/home/sase/memory/README.md` at +2/−3 and +3/−2, differing on
  which memory notes the README lists.

The second shape is the one that recurred most recently and is the same file sase-n0's
own reopening +1 names. **The home/global (chezmoi) template source is in scope**, not
just the project-scoped note. Open the chezmoi repo through `/sase_repo` — never by
guessing a path.

### Important: The Symptom Is Currently Latent

At the triage HEAD, on a clean tree after `just install`, **both checkers were green**:
`sase validate` reported `ok init memory --check`, and `sase init memory --check --diff`
reported `SASE is initialized. No init subcommands need to run.`

Do not conclude from that the bug is fixed — the bead was already closed once on exactly
that reasoning and was reopened by a fresh reproduction. The reported defect is a
_disagreement between two code paths_, and one green run of each does not prove they
agree in general. Verify the claim **structurally**: read what `sase validate`'s memory
step passes as its template source and compare it to what `sase init memory --check`
resolves. If they genuinely resolve the same source today, say so with the code
evidence, and the phase's remaining deliverable is the regression gate.

### Scope

1. Determine whether the two entry points still resolve different template sources for a
   project-scoped generated note. Read the code; do not infer from a green run.
2. If they do, reconcile them onto a single resolution so they cannot disagree.
3. Add a regression gate: a generated project note whose generator output differs from
   the committed file must not be able to pass `sase validate`. This is the durable
   deliverable and is required whether or not step 2 finds a live divergence.
4. Determine which alias vocabulary is authoritative. The known disagreement was between
   `@<size>` built-in aliases and `@<size>_worker` aliases. Grepping the repo finds no
   `_worker` alias anywhere (`src/sase/default_config.yml`, `src/sase/xprompts/`), which
   suggests the `_worker` wording comes from the home/global template rather than the
   project template — but **do not assume the generator is right**. Establish which
   names actually exist and record your finding.

### Approval Trigger — This Phase Has One

`CLAUDE.md` is absolute: never add, edit, or remove entries in `sase/memory/*.md`,
`AGENTS.md`, or the generated provider shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`,
`QWEN.md`) without explicit user permission in the current conversation. **A plan file
does not grant that permission** — this plan explicitly does not, and this paragraph is
not permission.

So:

- Do all of the code-side work (steps 1–4 above): the resolution reconciliation, the
  regression gate, and the authoritative-alias determination.
- If making the tree consistent requires editing generated memory content or running
  `sase memory init` / `sase init memory`, **stop at that boundary** and append a
  `TASK NEEDS APPROVAL` note to sase-n0 stating: exactly which files would change and by
  how much, which alias vocabulary you determined is authoritative and why, and your
  recommendation for which side should be corrected.
- Do not close sase-n0 if the approval-blocked portion is material to the fix. Note what
  remains and leave it ready.

### Done Means

- The structural question is answered with code evidence, one way or the other.
- A regression gate exists that fails `sase validate` on generator/committed divergence
  for a project-scoped generated note, with a test.
- Any memory-content change that the fix implies is described in a `TASK NEEDS APPROVAL`
  note rather than performed.
- `just check` is green.

---

## Phase `reconcile_marker` — Bound The Settle Path's Index Reads (sase-ne)

**Bead:** sase-ne · size medium · _Monitor reconciliation's locked settle path runs a
full-history artifact-index query per candidate via `get_monitor()`_

### What Is Wrong

`reconcile_dead_supervisor()` (`src/sase/monitor/reconcile.py:69`) takes the lane lock
and then calls `get_monitor(record.lane, record.monitor_id)` to re-read the record under
the lock; `_reconcile_dead_supervisor_locked()` re-reads through the same callable.

`get_monitor()` (`src/sase/monitor/store.py:246`) walks `_monitor_records()` →
`_project_records()`, which issues an **unbounded** artifact-index query
(`include_full_history=True`, `active_limit=None`, `recent_completed_limit=None`,
`include_hidden=True`) over `~/.sase/agent_artifact_index.sqlite`, then linear-scans the
result for the one `artifact_dir` it already knows.

That is O(archive) work per settled candidate. On the author's real state that query
measured **1.51 s per call against a 115 MB index** (epic plan
`plan:202608/tui_startup_monitor_reconcile.md`, Defect 3), and the index only grows.

### Why It Survived Epic sase-n7

sase-n7 bounded the query that _selects_ reconciliation candidates
(`_reconciliation_monitor_records()`, commit `9fe82045d`) and moved the whole pass off
the synchronous TUI disk load (`39bdd6772`). So this call is no longer on the ACE
startup critical path. It is still archive-scaled work **inside** the settle path, it
multiplies by the number of dead-supervisor candidates — exactly the burst case after a
reboot — and the reconcile pass holds the lane lock across it.

### The Fix Already Has A Home

`read_monitor_marker(project_name, artifacts_dir)` (`src/sase/monitor/store.py:257`)
exists for precisely this shape and documents it: _"get_monitor() runs a full-history,
unlimited, hidden-inclusive index query ... to find one record it already knows the path
to."_ The reconcile path holds the member's `artifacts_dir`, so the locked settle and
re-read should use the marker helper instead of `get_monitor()`.

`reconcile_dead_supervisor()` takes `get_monitor` as an injected `GetMonitor` callable,
so the change is at the **call sites** in `src/sase/monitor/store.py`
(`reconcile_dead_supervisors`, `active_monitor_for_lane`,
`monitor_blocking_start_for_lane`) plus any test doubles — not a signature change.

Confirm the marker read preserves the semantics the locked re-read depends on: it must
still observe a concurrent settle and bail when the record has already left `running`.
If it cannot, say so explicitly in your bead note and keep `get_monitor()` with a
_bounded_ query instead — do not trade correctness for the bound.

### Done Means

- The locked settle and re-read use a path-direct or equivalently bounded lookup.
- Settlement semantics are unchanged: a monitor whose supervisor died still settles, and
  pre-reboot monitors still reconcile to `lost`.
- A new test pins the bound by asserting the reconcile pass does not scale its
  artifact-index queries with candidate count. Mirror the existing
  `tests/monitor/test_monitor_store_reconcile.py::test_reconcile_dead_supervisors_uses_bounded_active_monitor_index_query`
  and `::test_reconcile_dead_supervisors_reads_proc_store_once`.
- `just check` is green.

### Approval Trigger

None. This is an objective, measured performance fix with a documented target helper.

---

## Phase `monitor_idle_bound` — Deflake The Supervisor Idle-Timeout Bound (sase-nd)

**Bead:** sase-nd · size large · _Flaky
`test_run_supervisor_idle_timeout_fires_after_output_stalls` exceeds 5s no-hang bound_

`depends_on: [flake_retire]` — so that `tests/reproducible_flake_baseline.txt` has a
single writer at a time and your `# fixed-at:` entry lands on top of that phase's.

### The Failure

```
tests/monitor/test_monitor_supervise.py::test_run_supervisor_idle_timeout_fires_after_output_stalls

  assert 5.825556540999969 < 5.0
  at `assert elapsed < _NO_HANG_TIMEOUT`
```

Observed under an escalated full `just check` (6 xdist workers, 31,161 items). The child
is `sh -c 'echo started; sleep 30'` with `idle_timeout_seconds=0.2` and
`timeout_seconds=30.0`.

**The idle-timeout contract held.** `exit_status == 1` passed — the supervisor _did_
fire the idle timeout. What failed is the 5.0s wall-clock bound around a 0.2s idle
timeout: the whole `run_supervisor()` call took 5.83s under a contended lane.

### Evidence

The reporting session touched neither monitor supervision nor that test file (it only
added reserved `lease(...)` RUNNING-field labels), and its isolated rerun passed. The
bead carries two independent +1s: a 13-worker full `just check` recurrence from an
unrelated test-file split whose immediate isolated rerun passed, and a confirmation from
epic sase-ns.6's land agent that `just selection-health --fail-on-new-flake` still names
the node above the baseline after that epic retired ten other nodes. The node's failure
records come from distinct workspaces with disjoint changed-file sets, which is the
evidence shape the flake gate requires.

### Related, Not The Same

Closed bead sase-lk fixed three _other_ nodes in the same file
(`term_ignoring_chatty_child`, `times_out_after_partial_line`,
`completes_when_grandchild_holds_stdout`) via a `BoundedLogPipe.close()` join-timeout
change; those nodes' pipes did not EOF promptly. Same file and same 5.0s
`_NO_HANG_TIMEOUT` constant, different node and different failure mode. Read that fix
before starting — it may or may not be the same mechanism reaching this node, and
establishing which is part of the work.

### Scope

Reproduce under a **contended full parallel lane**. A narrow file-scoped
`just test-contention` run is not sufficient evidence for this class — closed bead
sase-mv records file-level contention being green while the full lane was red. Then
determine which of these is true, and fix accordingly:

- the supervisor really does take ~5.8s of wall clock to notice a 0.2s idle stall under
  load (a real responsiveness defect — fix the supervisor);
- or the bound is measuring scheduler starvation rather than supervisor behavior (make
  the bound contention-tolerant in a principled way — e.g. bound the supervisor's own
  post-stall work rather than total wall clock — without removing the no-hang property).

### Approval Trigger — This Phase Has One

If the **only** viable remediation you find is to relax or delete the no-hang bound
outright — i.e. the test stops asserting that the supervisor does not hang — do **not**
do it unilaterally. Append a `TASK NEEDS APPROVAL` note to sase-nd explaining what you
measured, why no principled bound survives a contended lane, and what you would weaken.
A principled reformulation of the bound that still fails on a genuine hang is an
objective fix and needs no approval.

### Done Means

- The node passes under a contended full parallel lane, demonstrated with durable
  full-run evidence, not just an isolated rerun.
- After the fix commit lands, add the `# fixed-at:` entry naming that commit to
  `tests/reproducible_flake_baseline.txt` in a follow-up commit (this is the convention
  the file already uses — the fix commit hash must exist before the entry can name it),
  so the node's pre-fix historical evidence stops holding the gate red.
- `just selection-health --fail-on-new-flake` no longer names the node.
- `just check` is green, and a monitored `just check-full` is green or its remaining
  failures are all accounted for by other phases of this epic.

---

## Phase `approval_anchor` — Deflake Headless Epic Approval (sase-nz)

**Bead:** sase-nz · size large · _Flaky
`test_headless_epic_approval_submits_while_inflight_launch_holds_anchor` under the full
parallel lane_

`depends_on: [flake_retire]` — same single-writer reason as `monitor_idle_bound`.

### The Failure

`tests/test_plan_approval_actions.py::test_headless_epic_approval_submits_while_inflight_launch_holds_anchor`
fails intermittently under the full parallel test lane and passes on a focused rerun.

### Evidence — This One Is Solidly Reproducible

From the durable selection store `~/.sase/test-selection/gh_sase-org__sase/`, read
2026-08-17: **12 full-run failure records** spanning 2026-08-06T13:35:09Z to
2026-08-17T01:37:59Z, recorded from six different workspaces at twelve different HEADs
with pairwise-disjoint changed-file sets, including runs whose only other failure was
this node. That is exactly the evidence shape the flake gate requires (two eligible full
runs with disjoint change sets plus an interleaved independent pass), so this is
reproducible flakiness rather than a single-run blip. Two of the twelve are from the 24
hours before triage, so **it is currently live**.

The node has never been in `tests/reproducible_flake_baseline.txt`; it is one of the
nodes keeping `just check-full`'s last gate red. It was introduced 2026-08-02 by
`186fd2010`.

### Scope

Reproduce under a **contended full parallel lane** — not a file-scoped
`just test-contention` run, for the same sase-mv reason given in the previous phase.
Then root-cause the shared state or timing dependency and fix it **without weakening the
test's inflight-launch-holds-anchor assertion**.

The node name points at a headless epic approval racing an in-flight launch that holds
the workspace anchor, so the leading shared-state candidates are:

- the approval lock / lease path,
- the agent-name registry,
- the workspace claim registry.

In-progress epic **sase-j7** owns process-global state leaking between tests; if the
root cause turns out to be a leaked global, that epic's leak detector from phase
sase-j7.2 is the right instrument — use it, and record the connection on your phase bead
as a `PROPOSED FOLLOW-UP:` note so the land agent can route it. This was filed as a
narrow task rather than routed to sase-j7 because the node predates that epic and its
failure mode has not been shown to be a global-state leak.

Retired umbrella bead sase-ct forbids `+1`/reopen and directs narrow node-specific
records like this one; do not attempt to reopen it.

### Done Means

- The node passes under a contended full parallel lane, demonstrated with durable
  full-run evidence.
- The inflight-launch-holds-anchor assertion is intact — verify by reading the assertion
  before and after and stating in your bead note that it is unchanged, or naming exactly
  what changed and why it is not a weakening.
- After the fix commit lands, add the `# fixed-at:` entry naming that commit to
  `tests/reproducible_flake_baseline.txt` in a follow-up commit.
- `just selection-health --fail-on-new-flake` no longer names the node.
- `just check` is green.

### Approval Trigger — This Phase Has One

If the fix requires changing **production** approval or launch semantics (the anchor
protocol, the approval lock's scope, when a launch claims a workspace) rather than test
isolation or a test-visible seam, append a `TASK NEEDS APPROVAL` note to sase-nz
describing the semantic change and why test-level isolation is insufficient. Fixing test
isolation, or a genuine race in the code under test that violates its own documented
contract, needs no approval.

---

## Land Agent — Required Handoff

This epic is one round of an ongoing backlog-triage loop. After verifying, integrating,
and closing this epic, the land agent must continue the loop:

1. Confirm the five task beads (sase-o0, sase-n0, sase-ne, sase-nd, sase-nz) are each
   either closed or left ready with an honest note explaining what blocked them.
2. Check whether ready task beads remain for the `sase` project (`sase bead ready`). At
   the time this plan was written, **21 ready task beads remained** after the triage
   closed sase-nn (fixed by `3a22ff04f`) and sase-i7 (superseded by sase-n0), and this
   epic took five more.
3. **If ready task beads remain**, start a pseudo monitor via `/sase_monitor` running
   `sleep 1`, with a `--next` action instructing the next agent to follow this exact
   prompt, with these exact steps:

   > (1) Review all of my current open (not in-progress) sase task beads for the "sase"
   > sase project. (2) Close any task beads that are no longer relevant with a good
   > reason. (3) Select the 5 task beads that would have the most impact if worked to
   > completion. (4) Use your /sase_plan skill to fix the issues / make the improvements
   > that correspond with these 5 task beads. Make sure the plan file you propose tells
   > the agent(s) to: (a) if you think any of these 5 beads need approval from the user
   > before working (be lenient here and do not ask for approval for objective
   > improvements), do not ask directly, but instead leave a `TASK NEEDS APPROVAL` note
   > on the bead; (b) mark the bead(s) you intend to work as in-progress by changing
   > their status with the `sase bead update` command; (c) leave a brief note on the
   > task bead(s) explaining the work that was done to fix the reported issue / make the
   > requested improvement or, if the agent was unable to complete the work, justifying
   > why they were unable to do so; (d) close each of the 5 task beads that it was able
   > to finish; (e) if there are more task beads associated with the "sase" project, the
   > agent should then start a pseudo monitor using the `sleep 1` command with a next
   > action that instructs the next agent to follow this exact prompt (with these exact
   > same steps); (f) if there are no more task beads to work, the agent should move on
   > to the next numbered step in this prompt. (5) Review all `TASK NEEDS APPROVAL`
   > notes left by prior agent shells and consolidate them into a single report for the
   > user with suggested next actions. (6) Terminate.

   The `--next` action must also carry forward a short status line naming which beads
   this round closed and how many ready task beads remain.

4. **If no ready task beads remain**, skip the monitor and instead perform step (5) of
   that prompt directly: sweep every `TASK NEEDS APPROVAL` note left by prior agent
   shells (`sase bead search "TASK NEEDS APPROVAL"`) and consolidate them into a single
   report for the user with suggested next actions.

### Known Open `TASK NEEDS APPROVAL` State

At the time this plan was written, a full `sase bead search "TASK NEEDS APPROVAL"`
returned matches only inside epic descriptions and plan text — **no task bead actually
carried an approval request**, so nothing was owed to the project owner. Re-sweep rather
than assuming that still holds; this epic's `memory_check`, `monitor_idle_bound`, and
`approval_anchor` phases each have a live approval trigger that may have fired.

## Out Of Scope

- `tests/fakey/test_usage_limit_e2e.py::test_usage_limit_failure_disables_only_fakey_and_preserves_error`
  — the seventh exceeding node, owned by in-progress epic sase-n4.
- The ACE PNG visual lane. Sixteen goldens fail deterministically in the serial visual
  lane, tracked by beads sase-ny (7 stale goldens) and sase-dc (4 retry-E2E nodes, two
  of which fail on retry _state_ rather than pixels). Those were considered and
  deliberately deprioritized for this round: the PNG suite is excluded from both
  `just test` and `just test-cost`, so `just check` and `just check-full` stay green and
  no landing agent is blocked by them.
- Reducing the artifact index itself (bead sase-kh: 4,706 of 6,775 rows hidden, 103 MB,
  growing ~46 rows/day). `reconcile_marker` removes one archive-scaled _reader_; the
  retention policy behind it is a separate owner-owned decision.
- Migrating any of this into `../sase-core`.
