---
tier: epic
title: Restore the parsed release declaration and reconcile current release gates
parent_bead: sase-1lh.7.5.4
goal:
  The pending release targets 0.18.0, current release gates pass, and ci_watch publishes
  an installable sase 0.18.0 so the interrupted landing can finish.
phases:
  - id: release-integration
    title: Reconcile concurrent gate changes and issue a parseable release declaration
    description:
      "release-integration: reconcile the current gate contracts and readiness races,
      then declare 0.18.0 using a parser-verified nested commit."
    size: medium
    depends_on: []
  - id: release-watch
    title: Verify the committed declaration and observe guarded publication
    description:
      "release-watch: verify the actual committed declaration, regenerate the target
      once when needed, and observe publication with an honest bounded verdict."
    size: small
    depends_on:
      - release-integration
proposed_by: bbugyi200.athena.sase-1lh.7.5.4.land
create_time: 2026-10-17 00:23:05
status: wip
---

- **PROMPT:**
  [prompts/202610/restore_release_declaration.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/restore_release_declaration.md)
- **PARENT:**
  [202610/finish_deadline_release_target.md](https://github.com/sase-org/sase--plans/blob/main/202610/finish_deadline_release_target.md)

# Finish the remaining release integration

This child resumes **sase-1lh.7.5.4**, whose plan is
`plan:202610/finish_deadline_release_target.md`. Its directly parented plan ancestors
are **sase-1lh.7.5** (`plan:202610/finish_recapture_release.md`), **sase-1lh.7**
(`plan:202610/sase_1lh_landing_repair.md`), and **sase-1lh**
(`plan:202610/v0_18_0_green_ci_release.md`). Read their saved landing notes and the
relevant plans through audited commands. Accepted decisions remain **dispatch=yes**,
**visual_flakes=recapture**, **stale_epics=supersede**. No memory change is proposed.

The previous deadline repair is complete. Do not repeat it. At master
**28ced2a125f61548244f0086dccd7888c7bcbd02**, 151 timeout, check-recapture,
base/verify/salvage/apply, and workflow tests pass. This includes the early-pipe-EOF
deadline, TERM-ignoring descendants after leader exit, bounded cleanup, and immutable
check-mode golden/index bytes. The later 5ea5fa5452 helper split retains the shared
attempt driver and carries the deadline to the same execution helper. The lander
reviewed both child beads and every note, the implementation commit, later changes, and
fetched origin/master, which matched this audit baseline. There are no `--epic-symbol`
entries for sase-1lh.7.5.4.

This is an epic because a repair coder cannot observe its own host-created commit,
release-please interpretation, or publication before its turn ends. One bounded coding
phase is followed by one no-edit observer. The `parent_bead` handoff resumes the
interrupted landing; there is no closeout phase.

## Verified release failure

Phase **sase-1lh.7.5.4.2** closed canceled, correctly. Publish **39000384000**, at
5ea5fa5452, succeeded in regeneration and metadata synchronization but skipped build,
both install-smoke jobs, and publish. PR **#299** remained OPEN, title **chore(master):
release 0.17.2**, head **e18c3584ee5bfae8475c0eff0fd46963cdd3370b**. PyPI still served
sase **0.17.1** and sase-core-rs **0.40.1**; sase 0.18.0 was absent.

Repair commit **69e0cd75400465a1433717bad5e8e623379814c8** contains the literal
`Release-As: 0.18.0`. Publish's release-please **17.6.0** fetched that commit, but its
parser returns no `RELEASE AS` note from the actual message. SASE appends a terminal
`SASE_...=` tag block and Markdown reference definitions after the declaration, causing
the earlier line to be interpreted as body text. Thus grepping the commit for the line
is insufficient. This is the concrete reason the previous recovery did not work.

The lander reproduced the actual parser result and verified this supported alternative:

```text
BEGIN_NESTED_COMMIT
chore: release 0.18.0

Release-As: 0.18.0
END_NESTED_COMMIT
```

Keep this block in the commit body. release-please's `splitMessages` isolates it before
parsing, yielding `notes: [{title: "RELEASE AS", text: "0.18.0"}]` even with linked
BEAD/AGENT and MODEL provenance appended. The lander verified it after applying
`update_trailing_commit_tags` twice through the real Rust-backed facade; all SASE tags
still parse correctly. Do not move the declaration after SASE's footer: that would break
SASE's terminal-footer grammar. Do not edit the finalizer or Rust footer model for this
one-time release, disable provenance, change normal pre-1.0 version policy, or leave a
persistent `release-as` configuration override.

To inspect upstream source, use `/sase_repo` to open `gh:googleapis/release-please`. The
reviewed checkout was 45e6e20d824a94b51478ea0b51c01ab6352eb25b; tag v17.6.0 contains the
relevant `src/commit.ts` split/parser and `src/strategies/base.ts` version selection.
The published npm package `release-please@17.6.0` reproduces the workflow parser.
Install it only in ignored temporary scratch for the offline probe, without scripts; it
is not a new project dependency. Retrieve the landing's retained parser and CI evidence
from the parent bead's artifact links. The parser reproduction is
`file:explicit:a70448d0ed695d81976faa24`; the Master Gate failure log is
`file:explicit:5bc92da2e11b343f55822d95`. The 15-failure integration reproduction is
`file:explicit:7a86bb9a4fdfa3b2768be6c1`; the 151 passing maintenance tests are
`file:explicit:6009028d74dfa6f77e159199`.

## Rules

- Fetch and fast-forward before work; inspect concurrent fixes before editing. Do not
  overwrite another epic's fixes or repeat resolved work.
- No manual commits, branches, release-branch pushes, merges, or package-version edits.
  Host completion creates the repair commit. Only the no-edit watch phase dispatches
  workflows, within the bounded inherited policy. ci_watch owns the merge.
- Preserve the current ci_watch gates, exact version matching, core compatibility
  checks, exact pixel equality, update-mode salvage, and shared retry driver.
- Use `just fix` and `sase tool run check` for file changes. Never run
  `just check-full`. A check-full-only failure is test-infrastructure follow-up, not
  product work.
- Read lint/test and relevant ACE, TUI, tools, and docs-writing memories before edits in
  those areas. Shared backend changes belong in sase-core, accessed through
  `/sase_repo`, with binding/pin updates if necessary. Expected repairs here are tests,
  readiness predicates, and a local typing correction, not a backend redesign.
- No skips, xfails, broader timeouts, sleeps, weakened assertions, or copied CI goldens.
  Tests must wait for the behavior they assert. Phase workers record discoveries as
  `PROPOSED FOLLOW-UP:` notes rather than creating task beads.

## Phase release-integration

### Reconcile the concurrent provenance and configuration contracts

The lander reproduced **15 failures / 32 passes in 13.25s** across the affected modules.
They match Master Gate **38999581980** and **39000343130**. They belong to active epic
**sase-1m3**, phases .1/.2, which now has a `DISCOVERED ISSUE:` note. The newer Master
Gate **39001404469** was still running with lint already red. Check for intervening
fixes first; adopt only remaining changes needed for this release.

1. Commit **7545e85abf** intentionally stopped PR URLs/non-SHAs from becoming commit
   identities. Keep that rule. Reconcile stale output fixtures in
   `tests/test_done_agent_loader.py` (seven failing nodes),
   `tests/test_axe_run_agent_exec_finalize_attachments.py::test_finalize_loop_legacy_commit_metadata_scans_workspace_head`,
   and
   `tests/test_bead/test_bead_page_publication.py::test_sidecar_publication_anchors_associations_and_commit_links_to_primary`.
   Use valid fixture SHAs and current explicit metadata where appropriate, and prove
   legacy recovery still finds a real commit rather than merely accepting stale
   expectations. Inspect the exact traces before deciding which fixtures or adapters
   need correction.
2. Fix mypy's incompatible object-to-string assignments in
   `src/sase/agent/identity.py::resolve_turn_model_label` (original CI lines 183-184,
   now approximately 200-201). Preserve model/provider validation and unknown
   attribution. Use properly typed candidate values, not casts that bypass validation.
3. Commit **7c6b32457b** changed linked config resolution to
   `load_merged_config_for_local_path`. Keep target-project isolation and `declared_by`
   provenance. Reconcile tests still patching the old cwd-based loader: two legacy SDD
   cases in `tests/main/test_repo_path.py`, four configured-sibling cases in
   `tests/main/test_workspace_handler_project_resolution.py`, and
   `tests/main/test_workspace_handler_list_path.py::TestList::test_json_list_includes_registered_workspace`.
   Preserve the actual legacy layout and explicit project-state contracts; use the
   target loader or realistic fixture files rather than restoring cwd contamination.

Run all affected modules plus the new resolver, provenance, footer, and runtime-tag
tests. Exercise representative cases under Python 3.12 as well as the local interpreter
when their behavior differs. Record the adopted fixes on sase-1m3 so its own lander does
not repeat them.

### Resolve observed release-gate readiness failures

Check current CI before touching these; do not treat one local pass as broad resolution.
The audit routed all existing ownership before this plan:

- **sase-1a7**, also noted on **sase-1i5**: Full CI **38996026102**, at 3e950ebdbd,
  failed only
  `tests/ace/tui/widgets/decks/test_deck_spread_pilot.py::test_files_ctrl_j_scrolls_page_anchor_to_top`
  in Python 3.12. Its shared `wait_for_anchor_scroll` exhausted 5 seconds. Python
  3.13/3.14, visual, perf, lint, and isolation jobs passed. This exact test/helper is
  unchanged at the audit baseline and passes serially in 2.45 seconds. Phase sase-1i5.7
  fixed the shared wait but explicitly left this sibling unverified.
- **sase-1ls**: Master Gate 39000343130 test(5) again lost Archive selection to
  `message:status` in `test_mounted_arrival_then_clear_name_keeps_selection`; the
  unchanged source passes locally. Existing task corroborated.
- **sase-j7 note #78**, now corroborated: the same Master Gate test(4) failed
  `tests/ace/tui/test_commits_pane_collection.py::test_sidecar_filter_and_compatibility_toggle_share_collection_scope`
  because selection changed from `cccc...` to `aaaa...` after `d`. Its source passes
  locally. After `j`, the current wait merely checks selection is non-None, which need
  not prove the intended row has been selected. Inspect timeline/collection delivery
  before choosing the correction. Global-state leakage is not proven.

Within this bounded integration phase, fix remaining semantic readiness defects that
still block release using these exact nodes and small repeat runs; preserve intended
selection, layout, and filtering behavior. Do not broaden into the old flake epic or
retire an entire task based on one pass. If a fix requires a newly discovered product
redesign, stop with evidence and plan that actual remaining scope instead of weakening
the gate. No full screenshot recapture is needed for test-only lifecycle/readiness
changes; if rendered behavior changes, follow the visual memory's required proof.

### Make the release declaration executable evidence

Include the nested-commit block above in the primary repository's `/sase_final` commit
message for these integration repairs. First compose the candidate message with linked
BEAD/AGENT/MODEL tags through the actual runtime-tag facade, twice, and parse it with
the release-please version used by Publish. Require both preserved SASE provenance and
the exact `RELEASE AS=0.18.0` parsed note. Record the exact intended block and probe
result in the phase note. The following phase checks the actual host-created message. Do
not wait for this turn's SHA, push, or CI.

If 0.18.0 has independently published before implementation, do not redeclare an already
published version. Finish remaining verified integration repairs and let the lander
reassess release acceptance against the actual released contents.

## Release-watch preparation

This phase makes no file edits. Verify the sibling repair commit on origin/master,
including its complete host-stamped message. Parse the actual message with the same
release-please version as Publish and require the exact `RELEASE AS` note, not a grep
match. If that fails, close this phase canceled with
`release not confirmed: RELEASE_DECLARATION_UNPARSEABLE` and retain the parser output.
Never dispatch the known-ineffective declaration again.

If PR #299 still targets 0.17.2, observe an in-flight Publish first. Otherwise this
phase may dispatch **one** normal regeneration on a commit containing the verified
declaration, unless such a regeneration already completed:

```sh
gh workflow run publish.yml --repo sase-org/sase --ref master -f publish_existing=false
```

Record the run ID. When waiting is necessary, use `/sase_monitor` with a bounded
20-minute preparation deadline and a follow-up that resumes this same no-edit phase. Do
not duplicate a dispatch on resume. Require a successful regeneration and an open/merged
PR carrying 0.18.0, or verified 0.18.0 on PyPI, before starting the ordinary watch.
Skipped build/publish jobs only prove regeneration. If the version remains wrong, close
canceled with `release not confirmed: RELEASE_TARGET_NOT_RESTORED`, the observed title,
run URL, and parsed-commit evidence. Do not weaken version matching.

The inherited ordinary watch and its close guarantees follow verbatim below.

## Phase release-watch

This phase makes no file edits and always closes its bead with an honest verdict.

1. **Idempotent start.** Read this phase bead. If it is already closed, because this is
   a relaunch after recovery, run `/sase_final` and stop.
2. Fetch and fast-forward. Confirm that every sibling phase's commit is on
   `origin/master`, and use the newest `origin/master` SHA as FIXED_SHA. Pick CORE as
   the newest sase-core-rs on PyPI that passes `tools/check_sase_core_rs_bindings`
   against FIXED_SHA (0.40.1 at planning time). Inspect current Full CI, Master Gate,
   and PR #299, and do not duplicate an in-flight workflow.
3. **Preflight inline.** Append `--no-dispatch` to both commands when
   `dispatch = false`.

   ```sh
   PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --once --dry-run --close-bead <phase-id>
   ```

   It must exit 0 and print a fully parsed state line. On failure, close this phase with
   `-R canceled -r "release not confirmed: WATCHER_BROKEN (preflight)"` and the output,
   then run `/sase_final`. Do not repair code in this phase.

4. **Start one `/sase_monitor`.** Use timeout `14h`, statuses `WATCHING RELEASE` /
   `RELEASE WATCHED`, and `--tail-lines 200`, around:

   ```sh
   PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 timeout -k 10m 750m tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --deadline 12h --close-bead <phase-id>
   ```

   The `--next` follow-up instruction must say all of the following:
   - Make no edits.
   - Read `sase monitor show <id>`, the verdict file, and the phase bead.
   - If the bead is still open, close it. Use `--note @<verdict file>` for `RELEASED`.
     Otherwise use `-R canceled -r "release not confirmed: <VERDICT>"`, or
     `UNKNOWN (<monitor state>, exit <code>)` when no verdict exists.
   - Never reopen the bead, restart the watch, merge, dispatch, or create beads.
   - Finish with `/sase_final`.

   If `sase monitor start` exits nonzero twice, close canceled with
   `release not confirmed: WATCHER_NOT_STARTED` and both errors, then run `/sase_final`.
   Run the start command in the foreground until it exits.

### Why this bead always closes

Each layer has its own deadline, and each deadline finishes inside the next layer out:
the watcher stops at 12 h, the outer `timeout` at 12.5 h (plus a 10-minute SIGTERM
grace), and the monitor at 14 h.

| Path                                                 | Who closes the bead                                               | What releases the land agent              |
| ---------------------------------------------------- | ----------------------------------------------------------------- | ----------------------------------------- |
| Any verdict, including the watcher's 12 h `DEADLINE` | Watcher, in-process retries                                       | Monitor follow-up runs `/sase_final`      |
| Outer `timeout` at 12.5 h                            | Watcher's signal path, a detached closer, inside the 10 min grace | Follow-up (monitor `failed`)              |
| Monitor timeout at 14 h (SIGTERM, then SIGKILL)      | Detached closer, or the follow-up if the close never ran          | Follow-up (monitor `timeout`)             |
| Watcher crash or hang                                | Outer `timeout` path above, or the follow-up                      | Follow-up                                 |
| Preflight or monitor-start failure                   | The phase agent, inline                                           | The phase agent's own `/sase_final`       |
| A human stops the monitor                            | Detached closer (`STOPPED`)                                       | Needs a human: `sase bead work <epic-id>` |
| Reboot (monitor `lost`) or a SIGKILL from outside    | Nobody; the bead stays open                                       | Needs a human: `sase bead work <epic-id>` |

The last two rows can only come from a human or the host, and SASE surfaces both.
`wait_checks` raises "Wait dependency can never self-resolve" for the parked land agent.
The recovery is safe because of step 1:

- `sase bead work <epic-id>` relaunches only the land agent when this bead is closed.
- It relaunches this phase when the bead is open, and the phase then watches again with
  a fresh preflight.

## Landing and parent continuation

The child lander verifies code, every phase note, concurrent changes, and all six
release facts: ci_watch merged PR #299; GitHub release v0.18.0 exists; Publish build and
publish succeeded; PyPI lists 0.18.0; an unpinned fresh isolated install resolves 0.18.0
and reports the resolved sase-core-rs version; and the relevant Master Gate and Full CI
are green. A canceled observer is not success. Read the successful Full CI visual
manifest and retain every unstable node for the original parent's triage. Use
`/sase_monitor` for external waiting; never end an ordinary turn promising to resume.
Repair genuine remaining release work through the tier-aware plan loop.

After this child is complete, its `parent_bead` returns the lander to
**sase-1lh.7.5.4**. These are landing duties, not another phase:

1. Re-read sase-1lh.7.5.4, every descendant and note, and
   `plan:202610/finish_deadline_release_target.md`. Recheck the actual deadline repair,
   all saved follow-up dispositions, named-phase/linked-plan readiness, release
   evidence, and post-child drift. Run `sase bead epic-symbols sase-1lh.7.5.4`; resolve
   each entry or re-key it only to a genuinely dependent still-open later bead. Close
   normally with
   `sase bead close sase-1lh.7.5.4 --note "<verified scope, integration, release evidence, and every follow-up outcome>"`,
   run `just symvision`, then set `status: done` in
   **202610/finish_deadline_release_target.md** in the plans repo opened through
   `/sase_repo`. Never use `--force` to advance a successful nested landing.
2. Re-audit direct parent plan **sase-1lh.7.5**, its previous landing note, all
   descendants/notes, `plan:202610/finish_recapture_release.md`, and post-child drift.
   Rerun descendant and linked-plan readiness checks. If complete, retire every
   `sase bead epic-symbols sase-1lh.7.5` entry, close normally with concrete evidence,
   run `just symvision`, and mark **202610/finish_recapture_release.md** done.
3. Repeat the full readiness review for **sase-1lh.7** and
   `plan:202610/sase_1lh_landing_repair.md`. Clear its epic symbols, close normally, run
   symvision, and mark **202610/sase_1lh_landing_repair.md** done only if complete.
4. Repeat for **sase-1lh** and `plan:202610/v0_18_0_green_ci_release.md`. Complete its
   post-release duties: triage every unstable node from the Full CI relied on by
   ci_watch; retain existing dispositions for cwd warnings and Telemetry; verify Install
   smoke healed. Honor the approved stale_epics=supersede branch for **sase-1kx** only
   after that plan's task-closeout audit of **sase-16p, sase-1gh, sase-19d, sase-1kg,
   sase-1ii**. Supersede **sase-1jz** only when its own goal is proven. Those
   deliberately authorized supersessions are distinct from normal successful parent
   closure. Clear sase-1lh epic symbols, close normally, symvision, and mark
   **202610/v0_18_0_green_ci_release.md** done only when all scope is complete.
5. Stop at the first incomplete or ambiguous parent. Add a note naming the concrete
   blocker and report it; do not force closure or mark its plan done.

If a normal close rejects a named phase that was previously canceled, verify its
required outcome against this child's completed work, record the replacement evidence,
and reopen/complete that phase normally only when its scope has actually been fulfilled.
Do not force a canceled phase to count as a successful nested landing. If its outcome is
still missing, keep the ancestor open and record that blocker.
