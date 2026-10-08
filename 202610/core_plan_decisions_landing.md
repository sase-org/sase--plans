---
tier: tale
size: medium
title: Repair remaining Plan Decisions contracts and finish the core epic landing
goal: Preserve all valid decisions, reject incomplete or malformed review state, and
  close sase-1hi.1.1 and its parent phase after verification.
proposed_by: bbugyi200.apollo.sase-1hi.1.1.land
bead: sase-1hi.1.1
status: done
---

- **PARENT:**
  [202610/core_plan_decisions.md](https://github.com/sase-org/sase--plans/blob/main/202610/core_plan_decisions.md)
- **BEAD:**
  [sase-1hi.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1hi/sase-1hi.1.1.md)

# Remaining work and ownership

This tale finishes the interrupted landing of **sase-1hi.1.1**, whose parent is the
phase **sase-1hi.1** inside **sase-1hi**. The four child phases are already closed.
Implement the repairs below, verify them, and finish both normal closes in this same
turn. Do not wait for this tale's commit SHA, push, or CI: the host commits after the
turn ends. The containing epic **sase-1hi** remains with its waiting land agent.

Read these artifacts through `sase artifact read`:

- `plan:202610/core_plan_decisions.md`: the epic contract and closeout requirements.
- `plan:202610/plan_decisions.md`: authoritative Sections 2, 4, and 6.1.
- `research:202610/plan_frontmatter_decisions/plan_frontmatter_decisions.md`: the
  archive population under “Evidence from this host.”

Read `sase bead read sase-1hi.1.1 -r "Need the landing audit and child notes"` and every
child `.1` through `.4` with audited reads. The lander's audit note records the defects
and completed follow-up triage. Open `sase-core` with
`sase repo open sase-core -r "Finish the core Plan Decisions landing"`; use its printed
path and read its `AGENTS.md`. All domain repairs and tests belong there. Read sidecar
artifacts through `sase artifact read`, and open `plans` before changing the original
epic plan's status. No memory files need changing.

The reviewed core commits are `96d5b67b`, `c089cf17`, `df735e42`, and `88d63855`. Core
HEAD and origin/master at audit were `9ea87c11`. All seven decision bindings are
registered; the additive validated wire remains schema version 3, with legacy parity
fixtures preserved. Keep those contracts intact.

## 1. Preserve warning-only decisions and enforce Archived completeness

In `crates/sase_core/src/plan/decisions/grammar.rs`, `validate_decision_frontmatter`
compares total diagnostic length before/after `validate_one_decision` when deciding
whether to append its wire. A valid ask lacking `?` adds only
`decision-ask-not-question`, but the decision disappears while the plan still returns
`ok: true`. Compare newly added error diagnostics instead. A warning must leave the full
decision in author order, including choices, memory, and answer.

Add regression coverage on tale and epic plans, in Authoring, Launch, and Archived as
appropriate, that asserts the actual returned decisions, payload, digest, and sheet;
checking only `ok` and the warning code misses the defect. Pin a Python module
round-trip as well. An actual error must still fail validation.

`apply_archived_completeness` currently only considers parsed `decided_by` and answer
presence, and the absent/empty decisions paths return before it runs. Trigger
completeness from the presence of any system stamp (`decided_by`, `decided_via`, or an
answer), including a `decided_via`-only archive. Require valid `decided_by` and valid
answers for every existing decision together. Cover absent and empty decisions with a
transport-only stamp, a transport-only stamped decision, partial vectors, and fully
stamped and unstamped controls. Preserve existing invalid-provenance errors, auto's
no-transport rule, Launch size normalization, and Archived strict sizes.

## 2. Repair Unicode and fence handling in body checks

In `crates/sase_core/src/plan/decisions/callout.rs`, `memory_note_on_line` checks `src/`
with `&line[relative - 4..relative]`. That offset can split a UTF-8 character. A
standalone Rust probe importing the actual source, without editing the checkout,
confirmed that `Update 🚀 sase/memory/tui.md.` panics at this slice. Use a byte
comparison or safe prefix operation. Add tests through the public validator, not just a
copied helper, for this ordinary Unicode prose, read-only Unicode text, and the existing
`src/` exclusion. The advisory heuristic must return warnings instead of panicking.

`visible_lines` removes fence lines altogether. Consequently an open callout can span
intervening non-blockquote fenced code and absorb a later quote. With body base 8, this
input currently ends its first callout at line 13 rather than 9:

````markdown
> [!decision] toggle before

```
code
```

> after
````

Retain boundary information when parsing spans so a fence/non-blockquote boundary ends
the active callout even when its contents are ignored. Track fence delimiter character
and length: a shorter backtick or tilde run cannot close a longer opening fence.
Preserve original document line numbers and valid adjacent callout behavior. Add focused
tests for both delimiter types, unequal delimiter lengths, CRLF, and callouts
before/after a fenced block. Fenced markers must not execute or count as body
references.

## 3. Validate shared output state and restrict skipped-memory follow-ups

In `crates/sase_core/src/plan/decisions/sheet.rs`, `check_sheet_row` only checks that
memory provenance is nonblank; any other string is accepted by both summary and prompt
block. Reject values outside `asked | not_asked | quote_not_found | inherited` with an
actionable `PlanError`/Python `ValueError`. Apply shared row validation to local and
inherited sheets. Cover malformed inherited memory rows and invalid kinds, value/default
shapes, and identifiers or selectors capable of introducing extra instruction lines. A
memory row must be a boolean toggle with nonempty, one-line selectors. Keep asks and
epic titles safely quoted. Derive/check sheet counts and changed flags consistently
rather than trusting contradictory inherited wire state. Reuse existing grammar/value
rules where practical; avoid a separate permissive interpretation in presentation code.

`block_row` currently tells every auto-off memory row to file skipped work. The approved
rule only does this for an **unrequested** memory change left off without human review.
Use frozen provenance to distinguish `not_asked`/`quote_not_found` from
`asked`/`inherited`; verified requested or inherited rows left off must not manufacture
a follow-up. Human-declined rows remain quiet. Add exact-output tests for all three
audiences and these provenance cases, plus binding error round-trips. Preserve the
existing accepted summary and prompt output for valid cases.

## 4. Make archive evidence auditable and recheck integration

Grammar phase `.1` note #2 reports 25 init plans and 32 memory-path plans, but does not
identify individual artifacts. Its compact regression tests have no specific source
refs, contrary to the epic's acceptance requirement. Discover the archive with
`sase plan search "sase memory init"` and `sase plan search "sase/memory/"` (use JSON
and an appropriate limit/source). Read at least 20 actual archived memory-editing plans
from the report's Aug–Oct 2026 population with
`sase artifact read <ref> "Need heuristic tuning evidence"`, plus read-only and
disclaimer controls. Record the exact artifact refs and observed warnings/quiet controls
in a note on `sase-1hi.1.1`, and associate compact representative regression phrases
with their source refs in tests. Tune only the advisory heuristic when the evidence
warrants it; do not copy whole reports into tests. Use CLI-resolved refs for legacy
local plans rather than inventing sidecar paths.

Recheck drift from the core's first implementation commit through current HEAD,
excluding commits for this epic; also inspect current base history if applicable. At
audit the only post-first-commit non-epic core change was `9ea87c11`, the wait default
flip and directive metadata. `4d5cf650`, since epic creation, was isolated bead
read-model work. Primary commits since creation were `2268f50d79`, `69a4431521`,
`3df340f909`, `5a3f8ae574`, `3a4178b15a`, and `2f4d5f9bb0`. The sibling human-text
gatherer from `3df340f909` produces source/ref/text compatible with the quote matcher.
The others are artifacts, scheduler, wait UI, and test splits; the audit found no
duplicate or conflicting Plan Decisions implementation. Repair any new integration drift
that belongs to this child. The outer gate phase owns Python consumers and moving
`sase-core-revision.txt` when it starts calling the bindings; do not pull that phase
into this tale.

## 5. Verify the repaired contracts

Run focused core and PyO3 tests through `just test -p sase_core <filter>` and
`just test -p sase_core_py <filter>` in the opened core checkout. Exercise all seven
registered bindings, the existing tale and epic/Archived integrations, the new
regressions, and `plan_validate_parity`. Run `just fmt`, then `sase tool run check`
there (the wrapped `just check`). Never run bare cargo.

Read `lint_and_test.md` with `/sase_memory_read` and run `sase tool run check` in the
primary repo if changing any primary tracked file. Do not run `just check-full`. Use
`/sase_monitor` if verification requires a long handoff, with an explicit continuation
to complete step 6; wait for monitor start itself to exit.

All four child `PROPOSED FOLLOW-UP:` notes (.1#1, .2#1, .3#1, .4#1) describe the same
unrelated core editor wait-keyword matrix failures. The lander applied `/sase_new_task`,
searched all task statuses and the last-week CI/all-type sweeps, and found no duplicate.
Source history proves `21c539fd` from `sase-1h7.3` added `for_epic` without updating two
exact keyword expectations. This has already been recorded as a `DISCOVERED ISSUE:` on
the still-open causal epic **sase-1h7**; no task was created and no proposal was
discarded. Carry this outcome into the close note. If those two nodes still fail, verify
the exact unchanged clean-base failure or named KNOWN evidence and record it; do not
weaken assertions or treat it as remaining Plan Decisions work. Any new failure caused
by these repairs must be fixed before closeout.

## 6. Finish this epic's closeout, then close only its parent phase

Do this in the same coding turn after verification. Do not wait for this turn's commit,
SHA, push, or CI result. Ensure this tale's own completion/readiness does not leave an
unclosed descendant blocking the original epic: close its assigned tale bead normally
first if needed after its implementation is complete, then continue the explicit
original-epic closeout here.

1. Re-read `sase-1hi.1.1` and its children with `sase bead read`, confirming every note
   has been addressed and recording the fixed regressions, archive evidence, legacy
   parity, seven-binding checks, and post-start integration review.
2. Run `sase bead epic-symbols sase-1hi.1.1`. Resolve each listed symbol by wiring,
   privatizing, a justified non-test pragma, or deletion per `symvision.md`; re-key an
   entry only to a still-open bead that actually needs the exemption. Audit had no
   entries, but recheck. Retire any exemptions for the tale as well.
3. Close normally with
   `sase bead close sase-1hi.1.1 --note "<specific verification and integration evidence; all four directive proposals routed to sase-1h7>"`.
   If rejected, finish the named symbols or incomplete descendants and retry. Never
   force just to succeed and never force a successful nested landing.
4. Run `just symvision` in the primary repo if available, preserving the result.
   Independently attributed unrelated failures belong to their existing owner and must
   be reported honestly; resolve all exemptions for the closing beads.
5. Open `plans` using `/sase_repo`, audit-read `plan:202610/core_plan_decisions.md`, and
   set **`status: done`** in that original epic plan's frontmatter. Use the resolved
   PLAN path from the bead/printed repo path, never a workspace path copied from another
   agent. Include this sidecar change in the final declaration.
6. Run `sase bead read sase-1hi.1.1 -r "Need the parent link"`. Its parent is the phase
   **sase-1hi.1**. Re-read that phase and the outer plan's Section 6.1 to confirm the
   completed child delivers its entire core scope. Append stable API shapes and combined
   verification evidence to `sase-1hi.1`, including effective defaults, strict caller
   semantics, canonical digest contents, frozen memory/web identities, quote source/ref
   identity, and summary/prompt enum spellings.
7. Run `sase bead epic-symbols sase-1hi.1`, resolve or appropriately re-key every entry,
   and close only this parent phase normally:
   `sase bead close sase-1hi.1 --note "<combined core contracts, binding registration and round-trips, legacy parity, archive evidence, integration and follow-up outcomes verified>"`.
   Confirm the whitelist with `just symvision` again if the close or cleanup changes its
   inputs. Leave **sase-1hi** and `plan:202610/plan_decisions.md` open for their waiting
   land agent. If the parent scope is incomplete or ambiguous, record the precise
   blocker there and report it instead of forcing a close.
8. Finish with `/sase_final`: declare commits for every repository changed by the tale,
   including core and the plans sidecar. The host commits after the turn.
