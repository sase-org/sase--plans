---
tier: tale
title:
  Close out epic sase-iy by retiring sase-ct on an honest, carved-out exit criterion
goal:
  "Task bead `sase-ct` is closed as a retired umbrella with a reason that forbids future
  `+1` corroboration, phase bead `sase-iy.5` and epic bead `sase-iy` are closed behind
  it, and the flake baseline no longer names `sase-ct` as an owning bead. The two gates
  that blocked the previous two retirement attempts are resolved on the record rather
  than waited on: the flake-baseline gate is green, and the residual `check-full`
  test-cost budget redness is a pre-existing master-wide failure already tracked by task
  bead `sase-j0`, explicitly carved out of this closure by the owner."
size: medium
proposed_by: bbugyi200.athena.xy
create_time: 2026-08-11 08:27:41
status: done
---

- **PROMPT:**
  [prompts/202608/close_out_sase_ct_retirement.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/close_out_sase_ct_retirement.md)
- **AGENTS:**
  - [bbugyi200.athena.xy](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.xy.md)
- **COMMITS:**
  - [560f9b3](https://github.com/sase-org/sase/commit/560f9b3326cb8e20d3b82fccfe614ce5452eb15f)
    — chore(tests): retire stale flake baseline debt

# Plan: Close out epic `sase-iy` and retire the `sase-ct` umbrella

## Problem

Epic `sase-iy` has done its work. Phases `sase-iy.1` through `sase-iy.4` are closed, the
mechanism fixes landed, and the one structural change the whole epic exists for —
teaching `/sase_new_task` to route a retired umbrella's next reporter to a node-specific
bead instead of a `+1` — landed on master as `8501a19ac` and is deployed. What has not
happened is the closure itself. Phase `sase-iy.5` is still `in_progress`, `sase-ct` is
still `ready` with 62 `+1`s and 8 close/reopen cycles, and `sase-iy` is still
`in_progress`.

Two agents have now reached step 4 of the `retire` phase and stopped. Both stopped for
the same reason, and both were right to at the time:

- `sase-iy.5` (2026-08-10T17:56Z) recorded that `just check-full`'s full pytest lane
  passed 28452 / 10 skipped but the command exited 1 on the test-cost budget gate, and
  that `just selection-health --fail-on-new-flake` exited 1 on 8 post-baseline nodes.
- `sase-j7.5` (2026-08-11T01:09Z) recorded the same two failures after shrinking the
  baseline from 7 entries to the retained VCS-selector debt.

The `retire` phase's own instructions say "do not close `sase-ct` on a criterion you did
not meet," so both agents correctly declined. The result is a bead that is functionally
finished and procedurally stuck, and every hour it stays open is another hour in which
an agent can `+1` it for a ninth reopen.

Both blockers have since resolved or been correctly reassigned, and this plan's job is
to say so on the record and finish the closure.

## What is already true, measured at `a3995e1cb`

Re-measure all of this rather than trusting it; these are the observations the plan's
scoping rests on, taken in a clean workspace after `just install`.

**The skill change landed and is live.** `8501a19ac`
(`fix(skills): route retired umbrella duplicates to new tasks`) is an ancestor of
`HEAD`. `src/sase/xprompts/skills/sase_new_task.md` step 4 now checks for a retired
umbrella — "a closed task whose close reason declares it retired and forbids `+1`" —
before corroborating a duplicate, and routes the reporter to step 7 with
`sase bead note <new-task-id> "RELATED: <retired-task-id> — <how it bears on this task>"`.
`tests/main/test_init_skills_sources.py:470`
(`test_sase_new_task_retired_umbrella_routes_to_related_task`) pins that text. The
deployed skill at `~/.claude/skills/sase_new_task/SKILL.md` contains it. This retires
the `retire` phase's single largest risk — "do not close `sase-ct` before the skill
change has landed" — because the window it warns about is already closed.

**The routing rule is already working in the field.** `sase-j6` was filed on
2026-08-10T19:21Z by `sase-j0.w1` as a node-specific bead carrying
`RELATED: sase-ct — ... sase-ct is being closed as a retired umbrella; this node-specific bead is filed directly rather than corroborated onto it, per the retirement's routing rule.`
That is the mechanism this epic built, exercised by an unrelated agent, before the
umbrella even closed.

**The flake-baseline gate is green.** `just selection-health --fail-on-new-flake` exits
0 at `a3995e1cb`:
`no new reproducible flakes (0 current, 5 allowed; records after 2026-08-10T23:36:35Z, at most 5 failures per run)`.
It is not passing vacuously — it printed a long list of nodes it judged from the durable
store.

**The test-cost budget gate is not red right now, and its redness is not this epic's.**
`just test-cost-budget` exits 0 against the most recent recording
(`.../timings/cost/20260811T121247Z-2356016.json`). More importantly, when it _is_ red
it is red for every agent on master, and that is already a filed, `ready`, `large` task
bead: `sase-j0`, "just check-full is red on master: every suite-cost summary budget and
two ACE/Textual cause budgets are exceeded." Epic `sase-ix` already landed against that
same carve-out (`sase-ix.5`: "check-full green except the pre-existing test-cost budget
gate tracked by task sase-j0"). This plan follows that precedent.

**`tests/reproducible_flake_baseline.txt` currently has five entries**, and two of them
name `sase-ct` as their owning bead:

```
# bead: sase-ct; fixed by c0520947d/sase-j7.1, but retained until the
# post-cutoff selection store no longer reports this historical VCS selector debt.
tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py::test_vcs_tag_directory_key_spelling_also_resolves
# (same annotation)
tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py::test_vcs_tag_offers_project_local_xprompts_by_canonical_name
# sase-jb
tests/ace/tui/test_logs_pane.py::test_logs_tab_g_and_shift_g_scroll_detail_extremes
# sase-jf
tests/test_keymaps_registry_loading.py::test_stitches_action_override_wins_over_legacy_commits_alias
# sase-j6
tests/test_bead/test_plus_one_presentation.py::test_post_close_plus_one_badge_marker_search_and_json_agree
```

Those two `sase-ct` annotations are the one genuine in-tree blocker left. The baseline
header says "Adding a node requires a filed SASE bead that names the flake, evidence,
and owner." A bead closed as a retired umbrella that must never be reopened cannot be an
owner. `sase-hk` ("Diagnose the swallowed-exception raiser behind the two
`test_vcs_tag_*` xprompt-selector flake nodes") is `ready` and covers exactly those two
nodes, so a live owner exists.

**`tools/check_test_wait_helpers` exits 0** at `a3995e1cb`.

## The judgment this plan makes, and the owner's authorization for it

The `retire` phase's exit criterion asks for `just selection-health --fail-on-new-flake`
to pass "with no ACE node newly added" to the baseline. Taken literally that is now
unmeetable: `sase-jb`'s node
(`tests/ace/tui/test_logs_pane.py::test_logs_tab_g_and_shift_g_scroll_detail_extremes`)
is an ACE node and it is in the baseline.

Do not treat that as a blocker, and do not paper over it either. `sase-jb`, `sase-j6`,
and `sase-jf` are node-specific task beads, each naming one node, each with its own
evidence and owner. That is precisely the outcome the retirement is engineered to
produce — the umbrella's replacement, not its failure. An umbrella that cannot be closed
until zero ACE nodes are ever flaky again is an umbrella that can never be closed, which
is the exact trap that consumed `sase-h8` and `sase-h8.10`.

The owner has explicitly asked for `sase-iy`, `sase-iy.5`, and `sase-ct` to be closed
out, which authorizes two departures from the letter of the `retire` phase:

1. `check-full` may exit non-zero **only** on `tools/check_test_cost_budgets`,
   attributed to `sase-j0`. Every other gate, including the full pytest lane itself,
   must be green. A single failing test node is still a hard stop.
2. Baseline entries owned by live node-specific beads (`sase-jb`, `sase-j6`, `sase-jf`)
   do not block the closure. They must be named explicitly as outstanding debt in the
   `sase-ct` close note, with their owning beads, rather than counted as "met."

The owner also authorizes closing the epic bead `sase-iy` directly. `sase/memory/`
guidance says a land agent closes the parent epic; the owner asked for it here, so close
it in this plan rather than launching `sase-iy.land`.

## Do this

### 1. Set up and re-measure the starting state

`just install` first — this workspace may be stale. Then record, with output, at the
current `origin/master`:

```bash
git log --oneline -1
git merge-base --is-ancestor 8501a19ac HEAD && echo "skill fix landed"
.venv/bin/python tools/check_test_wait_helpers; echo "wait-helpers exit=$?"
just selection-health --fail-on-new-flake; echo "flake-baseline exit=$?"
grep -c "retired umbrella" ~/.claude/skills/sase_new_task/SKILL.md
```

If the skill fix is _not_ an ancestor of `HEAD`, or the deployed `SKILL.md` no longer
contains the retired-umbrella routing, stop and fix that before anything else — closing
`sase-ct` without live routing is what produced the last eight reopens.

### 2. Make the flake baseline honest and free of `sase-ct`

Nothing in the tree may name `sase-ct` as an owning bead once it is retired. Resolve the
two `test_vcs_tag_*` entries by measurement, not by assumption:

1. Delete both `test_vcs_tag_*` entries and their `# bead: sase-ct` comment blocks from
   `tests/reproducible_flake_baseline.txt`, then run
   `just selection-health --fail-on-new-flake`.
   - **Green** → keep the deletion. The annotation said they were retained only "until
     the post-cutoff selection store no longer reports this historical VCS selector
     debt"; a green gate is that condition being met, and paying the debt is strictly
     better than relabeling it.
   - **Red** → restore both entries and rewrite the annotation to name `sase-hk` as the
     owner instead of `sase-ct`, keeping the `fixed by c0520947d/sase-j7.1` provenance
     and stating that the entry is retained because the post-cutoff store still reports
     it. Record the gate output that forced this branch.
2. Apply the same delete-and-measure test to the `# sase-jf` entry
   (`test_stitches_action_override_wins_over_legacy_commits_alias`). `sase-j8.land`
   recorded it as already fixed on master by `9c46891c5`, which changed the override key
   from `minus` to `f24`. Keep the deletion if the gate stays green; restore it and say
   why if not.
3. Leave `sase-jb` and `sase-j6` alone. Their beads are open and their nodes are live.
4. Add no new entries under any circumstance.

Commit this on its own through `/sase_git_commit`, attributed to `sase-iy.5`, before
touching any bead. Verify `just check` afterward.

### 3. Run the exit criteria on that tree

All four, with counts and durations recorded verbatim for the close notes:

1. `just check-full`. Expected: every lint gate, SASE validation, committed-plans
   validation, and the full pytest cost lane green. The **only** tolerated failure is
   `tools/check_test_cost_budgets`; if it fails, capture which budgets and confirm the
   pytest lane itself reported zero failures. If it passes, say so — that is new
   information for `sase-j0` and worth a note on that bead.
2. `just test-visual`. Expected green. `sase-j7.5` measured 652 passed / 1 skipped after
   refreshing a stale Stitches PNG golden; `sase-iy.5` measured 648 / 1 in 322.17s.
   Compare against those and state plainly whether you matched, beat, or regressed them.
   A golden diff here is a signal you fixed the wrong thing — see "Watch out for."
3. The `residue` node set under contention, zero failures across the repeats:

   ```bash
   just test-contention -- tests/test_agent_group_revival_e2e.py \
     tests/ace/tui/test_commits_pane_filters.py \
     tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py \
     tests/ace/tui/test_plugins_browser_pane_sase_update_dev.py
   ```

   Record the per-node tally, not just "it passed." Both prior attempts reported zero
   node failures across 3 repeats; anything worse than that is a regression to
   investigate, not to average away.

4. `.venv/bin/python tools/check_test_wait_helpers` exits 0, and
   `just selection-health --fail-on-new-flake` exits 0 **non-vacuously** — quote the
   summary line showing the record window and the node list it actually judged.
   `sase-h8` was misled by a vacuous pass; if this one is vacuous, say so in the close
   note rather than counting it.

If a _test node_ fails anywhere in these four, stop. File it through `/sase_new_task` as
its own node-specific bead with `RELATED: sase-ct` — demonstrating the retirement
pattern on a live case is worth more than a clean-looking closure — and report rather
than closing `sase-ct`.

### 4. Close `sase-ct`

Re-read `sase bead show sase-ct` immediately before closing, so the counts in the reason
are the real ones at close time and so you notice any `+1` that arrived while the gates
ran. Then close with resolution `done` and this reason, substituting every `<...>` with
a measured value or a real reference:

> RETIRED UMBRELLA — DO NOT `+1` THIS BEAD.
>
> `sase-ct` tracked a class ("an ACE TUI test failed under the full parallel run and
> passed in isolation") that matches every future ACE timing failure, so every reporter
> corroborated it and every `+1` reopened it: `<N>` `+1`s, `<M>` closures, `<M>`
> reopens, across epics `sase-h8`, `sase-h8.10`, and `sase-iy`. The bead could not stay
> closed because the tracking pattern, not the tests, was the defect.
>
> The live instances are fixed by mechanism: phase `sase-iy.2` fixed the deterministic
> `prompt-catalog:0` convergence hang that held `wait_for_visual_idle` open for its full
> 30s deadline and made the PNG lane red in isolation; `sase-iy.3` widened
> `tools/check_test_wait_helpers` past its `pilot`-receiver and `_wait_until`-name blind
> spots and migrated the attempt-bounded pause loops it then reported, including
> `test_k_on_glossary_term_pushes_glossary_preview_card`; `sase-iy.4` fixed the
> remaining contention-sensitive nodes, measured with `just test-contention` at
> `<before>` → `<after>`.
>
> Verified at `<HEAD>`: `<test-visual tally>`; `just test-contention` on the residue set
> `<tally>`; `tools/check_test_wait_helpers` exit 0;
> `just selection-health --fail-on-new-flake` exit 0 `<summary line>`; `just check-full`
> `<green, or green except tools/check_test_cost_budgets, which is the pre-existing master-wide redness tracked by task sase-j0>`.
>
> Outstanding baseline debt, deliberately not blocking this closure, each owned by its
> own node-specific bead:
> `<list the remaining tests/reproducible_flake_baseline.txt entries with their beads>`.
> That those exist as individual beads instead of `+1`s on this one is the retirement
> working as designed.
>
> If you hit an ACE TUI test that fails under the full parallel run and passes in
> isolation: **do not `+1` this bead, and do not reopen it.** File a new task bead
> through `/sase_new_task` named for the specific failing node ID, and record
> `RELATED: sase-ct` on it with a note explaining how it bears on the new task.
> `/sase_new_task` step 4 enforces this routing as of `8501a19ac`. A node-specific bead
> can be fixed and can stay closed; this umbrella could not.
>
> Re-run the measurement with `just test-contention`, `just test-visual-contention`, and
> `tests/reproducible_flake_baseline.txt`.

Then confirm it took: `sase bead show sase-ct` must report `closed`. If a `+1` reopened
it in the gap, read that `+1`'s observation window — per bead semantics a closed task
only reopens on evidence whose window starts strictly after the close — and if it is
stale-window evidence, re-close and say so; if it is a genuine post-close reproduction,
route it to a new node-specific bead per the rule you just wrote, and re-close.

### 5. Close `sase-iy.5`, then `sase-iy`

Closing never cascades, so order matters.

```bash
sase bead close sase-iy.5 --reason "<gates run, tallies, and that sase-ct is retired and closed>"
sase bead close sase-iy    --reason "<epic outcome: the five phases, and the retirement>"
```

`sase-iy.5` already carries three `PROPOSED FOLLOW-UP:` notes. Dispose of each
explicitly in its close note rather than letting them evaporate:

- the test-cost budget gate → already tracked by `sase-j0`; corroborate or note there
  with what you measured, do not file a duplicate.
- the flake-baseline gate → resolved; the three post-baseline nodes are `sase-jb`,
  `sase-j6`, and `sase-jf`, each already its own bead.
- duplicate contract-manifest tasks → `sase-iu` / `sase-iv`; owner's call, report only.

### 6. Close out the epic's plan and its predecessors

1. Set `status: done` in the frontmatter of `plans:202608/retire_sase_ct_umbrella.md`.
   Open the plans sidecar through `/sase_repo` and use only the path it prints; check
   `sase plan show` first in case the bead close already updated it. Commit that repo
   through `/sase_git_commit`.
2. `sase bead note sase-h8` and `sase bead note sase-h8.10` recording that `sase-ct` was
   retired here, by what criteria, and at what commit — so their land agents are not
   left waiting on a bead that will never reopen. **Do not close them**; they are the
   owner's call, and both are worth flagging in the report because every one of their
   children is closed while the epics themselves remain `in_progress`.

### 7. Report to the owner, without acting

- `sase-iu` and `sase-iv`: byte-identical duplicate contract-manifest tasks from the
  same agent one minute apart. `sase-iv` is `open` and already noted as superseded by
  `sase-iu`. Recommend closing `sase-iv` as superseded.
- `sase-jf`: `sase-j8.land` verified it already fixed on master by `9c46891c5`.
  Recommend closing as done, and note whether you were able to drop its baseline entry
  in step 2.
- `sase-h8` and `sase-h8.10`: `in_progress` with every child closed.
- `sase-j0`: report whether the test-cost budget gate was red or green in your
  `check-full` run.

## Exit criteria

- `sase bead show sase-ct` reports `closed` / `done`, with the retirement reason, and
  stays closed through the end of the run.
- `sase bead show sase-iy.5` and `sase bead show sase-iy` both report `closed`.
- `grep -rn "sase-ct" tests/reproducible_flake_baseline.txt` returns nothing.
- `just check` green on the final tree, and the step-3 measurements recorded on the
  beads.
- `plans:202608/retire_sase_ct_umbrella.md` has `status: done`.

## Watch out for

- **Do not regenerate PNG goldens or widen the diff tolerance** to make
  `just test-visual` green. The historical failures in this class are convergence
  timeouts with `frame_digests=[]`, meaning no frame was ever compared and no golden is
  wrong. A diff under `tests/ace/tui/visual/snapshots/png/` is a signal you fixed the
  wrong thing. The one legitimate exception is a golden genuinely stale from an
  unrelated landed UI change — `sase-j7.5` hit exactly that with the Stitches tab rename
  — and if you hit it, name the commit that made it stale.
- **Do not pragma or suppress a gate to make a criterion pass.** If
  `check_test_wait_helpers` reports a call site, migrate it; if the flake baseline goes
  red, that is information, not an obstacle.
- **`just test-contention` deliberately starves the host**, and other agents run on this
  machine. Keep it scoped to the four residue files; do not soak the full suite.
- **Close in dependency order.** `sase bead close` rejects any bead with an unclosed
  descendant and writes nothing, so `sase-iy.5` must close before `sase-iy`.
- **The close reason is the deliverable, not a formality.** It is the only thing a
  future reporter will read before deciding whether to `+1`. Every `<...>` placeholder
  must be replaced with something a reader can check against the tree; a reason with an
  unsubstituted placeholder is worse than no reason at all.
- **Do not close `sase-jb`, `sase-j6`, `sase-jf`, `sase-hk`, `sase-j0`, `sase-iu`, or
  `sase-iv`.** They are live node-specific beads and the owner's triage queue — leaving
  them open is the point of the retirement, not an oversight.

## Out of scope

- **The Rust `+1` reopen semantics.** `mutation.project.plus_one()` lives in
  `sase-core`; changing whether a `+1` reopens a closed task is a core wire change
  affecting every frontend. This closure relies on the skill-level routing instead. If
  that proves insufficient — if `sase-ct` reopens a ninth time despite `8501a19ac` — the
  core change is a new task bead, not part of this plan.
- **Fixing `sase-jb`, `sase-j6`, or `sase-j0`.** Each is a `ready` task bead awaiting
  the owner's triage.
- **`sase-gk` and `sase-gs`** are snoozed. Leave them snoozed.
- **`sase-e2`** (bead-lock contention) has always been outside this umbrella.
