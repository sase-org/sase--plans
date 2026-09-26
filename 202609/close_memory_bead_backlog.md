---
tier: epic
title: Resolve the open SASE memory bead backlog
goal: Apply the audited memory corrections and close all 15 currently open memory
  task beads with evidence-based reasons, including the two tasks gated by active
  epics.
phases:
- id: reconcile
  title: Reconcile bead scope and close no-edit tasks
  depends_on: []
  size: small
  description: 'reconcile: audit the live bead census, correct stale task descriptions
    by note, close no-edit tasks, and install external dependencies on the later phase.'
- id: reference
  title: Correct existing reference memory and tools guidance
  depends_on:
  - reconcile
  size: medium
  description: 'reference: update existing source files, regenerate memory outputs,
    verify them, and close seven implementation beads.'
- id: decisions
  title: Record machine-link and explicit-handoff decisions
  depends_on:
  - reference
  size: medium
  description: 'decisions: add two accepted strands, mark the old record superseded
    in part, regenerate memory outputs, and close two decision beads.'
- id: post_landing
  title: Document hold and remote dispatch after their epics land
  depends_on:
  - decisions
  size: medium
  description: 'post_landing: after the hold, dispatch, and parity epic dependencies
    close, verify current contracts, publish their guidance, and close two memory
    beads.'
- id: final_audit
  title: Verify the complete memory backlog is closed
  depends_on:
  - post_landing
  size: small
  description: 'final_audit: check regenerated memory and links, verify completion
    evidence and close reasons, and confirm no memory task bead remains non-closed.'
proposed_by: bbugyi200.athena.0qi
create_time: 2026-09-26 07:00:19
status: wip
bead_id: sase-1ae
---

- **PROMPT:** [prompts/202609/close_memory_bead_backlog.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/close_memory_bead_backlog.md)
- **BEAD:** [sase-1ae](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ae/README.md)

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                    | Why                                                                    |
| ------------ | --------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| derives-from | [research:202609/memory_bead_backlog_audit/memory_bead_backlog_audit.md][1] | Uses the consolidated per-bead audit and its corrected memory wording. |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202609/memory_bead_backlog_audit/memory_bead_backlog_audit.md

<!-- sase:links:end -->

# Resolve the open memory bead backlog

## Scope and current evidence

The audited source is
`research:202609/memory_bead_backlog_audit/memory_bead_backlog_audit.md`; read it with
`sase artifact read`, not from the sidecar filesystem. The live
`sase bead list -T memory -s open -s ready -s in_progress -s snoozed -n 0` still
contains the report's 15 beads. `sase-1ac` closed after the report and already updated
the `%queue` multiplier paragraph; preserve that newer wording. The agent-session rename
`sase-17m` is closed; use _session_ instead of the report's older _family_ wording.
`sase-11l.11`, `sase-xe.16`, and `sase-133` are still in progress, so their final
contracts must be checked after landing.

The user explicitly authorized memory edits and closing all open memory beads, including
beads needing no edit and the abandoned message boards. Invoke `/sase_memory_write`
before editing memory. Read relevant reference notes with `/sase_memory_read`; read each
bead with `sase bead read ... -r ...` before changing its state. Do not apply a task's
stored `proposed_change` when the research report corrects it. Avoid adding new core
memory. Do not hand-edit generated root `AGENTS.md` or provider shims. `tools/AGENTS.md`
is the one authored instruction file in scope; `sase memory init` regenerates its
provider copies.

At each editing phase, run `sase memory init --no-commit`, then
`sase memory init --check`, inspect the diff and memory rendering, and run `just check`
through the documented `sase tool run` or verify-monitor path. `just check-full` is not
requested. Close each task only after its scope is verified, using
`sase bead close ID -R done -r "..." -n "..."`; the close reason should state the actual
disposition and the note should name the evidence. If a generated output or source
contract changed since this plan, update the wording to the landed behavior and explain
the deviation in the bead note. No implementation code change is planned.

## Reconcile: task state, dependencies, and no-edit closures

1. Re-run the unbounded active memory-bead census and inspect the 15 task records. If
   another memory task opened after planning, determine whether it is part of this same
   backlog and account for it in the final census; do not silently declare zero while an
   active task remains.
2. Close `sase-sl` as `done` without editing memory. Verify that `lint_and_test.md`
   already states exact pixel equality locally and in CI, introduced by `1c246dc74`, and
   give that as the close reason. Do not recreate `build_and_run.md` or add an unrelated
   renderer escape-hatch note.
3. Confirm that `sase-xs`, `sase-xt`, and `sase-xu` have no active supervision chain,
   then close them as `canceled` with a reason such as “message-board experiment ended
   in September 2026; these are operational coordination records, not memory updates.”
   Preserve their history; do not create a replacement bead unless a separate, concrete
   feature is requested.
4. Note the audited corrections on `sase-195` (a flag works in `watch_highlighted`, but
   not only in the queued event handler), `sase-148` (avoid stale case counts),
   `sase-sa` (actual path is `glossary/proc-shell.md`), and `sase-16r` (both
   `lint_and_test.md` and `tui_screenshot.md`). Point each note to the consolidated
   research ref.
5. Note on `sase-134` that `ttl=` is optional with a bounded default, and replace
   _family_ with _session_. It already depends on `sase-11l`; add the more precise
   `sase-11l.11` dependency. Note on `sase-ya` that its deliverable is a short
   `type: reference` pointer note, with Focus/Fleet count assertions withheld until
   parity lands. Add dependencies on `sase-xe.16` and `sase-133`.
6. Before completing this phase, add the same external bead dependencies (`sase-11l`,
   `sase-11l.11`, `sase-xe.16`, `sase-133`) to the generated `post_landing` phase bead,
   then inspect its dependency graph. This prevents the phase worker from starting
   early; if an external bead is already closed by then, record that and omit only that
   satisfied dependency. The later phase must not publish unlanded behavior merely to
   close a task.

## Reference: existing notes and tools guidance

Edit each authored source once, using section A of the research report as a wording
guide and checking current code/docs where cited. Preserve the exact-pixel sentence
already present.

- `sase/memory/lint_and_test.md` (`sase-18h`, `sase-16r`): state in recipe comments and
  Two-Speed Verification that `just check` and `just check-full` omit `toobig`; CI gets
  it through `just lint`, `just toobig` runs it on demand, and `toobig_split` owns
  splits. In PNG Snapshot Tests, document update-mode `clean`/`applied`/exit-0
  `partial`; explain that skipped goldens are unchanged and not known current, where to
  find WARNING/manifest `skipped` and `pruning_skipped_reason`, `-n N` translation to
  governed `SASE_PYTEST_WORKERS`, rejection in `PYTEST_ADDOPTS`, bounded
  maintenance-lock wait, and strict nonwriting `--check`.
- `sase/memory/tui_screenshot.md` (`sase-12x`, `sase-16r`): replace the stale “missing
  visual extra” advice with the base `resvg_py` dependency and `sase update`/environment
  reinstall remedy. Add a concise `partial` pointer to `[[lint_and_test.md]]` under
  Golden Maintenance. Correct the bundled font-stack phrase if still stale.
- `sase/memory/xprompts.md` (`sase-st`): explain the `[[ ... ]]` close rule (a `]]`
  closes only before optional whitespace plus `,`, `)`, `}`, `|`, or end of args),
  literal close sequence quoting, structural line-shorthand payload binding, and the
  narrow bare-colon `+` conversion. Leave the newer `%queue` multiplier content intact.
- `sase/memory/tui_perf.md` rule 12 (`sase-195`): retain the working synchronous
  `watch_highlighted` guard pattern, including explicit scroll if needed. Explain why
  checking only the queued `OptionHighlighted` handler fails and when `CommandLinePopup`
  needs pending-echo counts and option identity. Do not repeat the bead's blanket claim
  that `finally` guards never work.
- `sase/memory/glossary/proc-shell.md` (`sase-sa`): define both session-attached
  monitors and stand-alone beta `%proc` shells sharing `shell_kind: "proc"`, their
  distinct ownership/counting, and why an execution-phase gate remains
  `shell_kind: "gate"`. Do not edit `sase/sase.yml` for the glossary.
- `tools/AGENTS.md` (`sase-148`): replace “phase-pending” with `not-run` for live
  supervisor/cold-start cases unless `--live` is passed, and include hand-off in the
  harness contract list. Avoid hard-coded case counts and IDs. Let `sase memory init`
  regenerate the four `tools/` provider copies.

Verify the relevant source contracts and rendered memory, run the checks above, and
close `sase-18h`, `sase-16r`, `sase-12x`, `sase-st`, `sase-195`, `sase-sa`, and
`sase-148` individually with precise reasons. Do not use the task's original erroneous
replacement text for `sase-195`.

## Decisions: two new records and one supersession mark

Follow the existing decisions web frontmatter and roster conventions, using inline
`[[...]]` links rather than invented `type`, `deciders`, or `links` keys. Preserve the
accepted body of an existing decision record.

1. Create `sase/memory/decisions/machine-link-writes-off-primary.md` for `sase-yd`.
   Record the invariant, three rejected alternatives, clone/sync cost, reopen condition,
   implementation references, and `plan:202609/machine_link_mutations_off_primary.md`
   from the bead. Link `[[sase_artifacts.md]]`. Add a concise entry to the decisions
   descriptor roster, regenerate, verify
   `sase memory read decisions:machine-link-writes-off-primary`, then close `sase-yd`.
2. Create `sase/memory/decisions/explicit-handoff-fails-closed.md` for `sase-18a`, using
   decision 3 of `plan:202609/tool_e2_durable_handoff.md` as source. Explicit
   `sase tool run -H` fails closed when its ToolRun reservation cannot commit;
   foreground and monitor-start recording stay fail-open, and later hand-off recording
   failures are incomplete evidence rather than replay triggers. Give the durable
   printed-handle reason, caller retry/foreground cost, and an evidence-based reopen
   condition (a separate durable handle would reopen the choice). Link
   `[[decisions/record-before-admit]]` and `[[decisions/guarded-recipes]]`.
3. In `decisions/record-before-admit.md`, retain `metadata.status: superseded-in-part`,
   make `metadata.superseded_by` include both `decisions/guarded-recipes` and the new
   record, and add a brief back-link paragraph marking only the former blanket fail-open
   clause as superseded for explicit `-H`. Do not alter the accepted claim body or
   hand-write the status line rendered by `sase memory read`. Update the decisions
   descriptor roster, regenerate, verify both records render and link, then close
   `sase-18a`.

## Post-landing: hold and dispatch memory

Start this phase only once its external bead dependencies have closed. Re-read the
landed epic outcomes and current `docs/xprompt.md` / `docs/remote_dispatch.md`; the
September 26 research report is a starting audit, not a substitute for these later
contracts. Check the newer `%queue` multiplier paragraph from `sase-1ac` and session
terminology. Avoid duplicating whole runbooks in memory.

For `sase-134`, add a `%hold` directive row and compact paragraph to
`sase/memory/xprompts.md`: pre-admission effect on _other_ queued agents and
undispatched procs, running-work immunity,
name/`@tribe`/`tribe=`/`hood=`/`pending`/`future` selectors, project/host scope,
optional TTL (default 2 h, cap 12 h), release on explicit release/armer session or shell
end/TTL, fail-open broken store, bare `%hold` error, no `%h` alias, no
`%repeat`/`%dispatch` combination, and `sase agent hold create|run|list|show|release`.
Add the beta `%proc` row and explain `%queue` capacity checking at dispatch, omitted
proc weight 0, and no runner capacity held after dispatch. Verify every detail against
the landed implementation. The original hold plan records pull rather than push and
fail-open rather than host freezing; add the concise decisions strand requested by the
bead, with the accepted rationale and a cost/reopen condition grounded in the landed
contract. Regenerate and verify.

For `sase-ya`, create a short `type: reference` `sase/memory/dispatch.md` pointing to
`docs/remote_dispatch.md`. Record exactly one `%dispatch:<alias>` selector with `local`
reserved; v1 incompatibility with `%wait`, `%queue`, `%clan`, and active `%hold`; clean
published `HEAD` requirement; quarantined enrollment recovery via `sase machine repair`;
and the Launch Target picker/Machines tab's lack of network probes. Add a one-line
`%dispatch` directive-table entry in `xprompts.md` linking `[[dispatch.md]]`. Check
current operation-key recovery and the landed `sase-133` Focus/Fleet facts before adding
any count claim; omit claims that parity evidence cannot support and state that scope
correction in the close reason. Use _session_ terminology. Regenerate, render, verify,
and close `sase-134` and `sase-ya` with what actually landed.

## Final audit and landing

Run `sase memory init --check`, inspect the final generated diff and authored links, and
run `just check`. Re-read the two new decisions and any hold decision through audited
`sase memory read`; verify the no-edit `sase-sl` close and each edited task's close
note. Run the unbounded non-closed memory-task census again and require zero matching
beads. If one remains, resolve it or keep the plan open with a precise blocker and an
explicit continuation; do not mark the epic done while the user's requested backlog
remains. Check that no new always-loaded core text, stale generated provider copy, or
unsourced remote count assertion was introduced.
