---
tier: tale
title: Unblock flag removals that close their flag bead from a gate
goal:
  A flag removal can close its flag bead from its own checkout or a gate turn inheriting
  it, without turning sibling checks red.
size: medium
decisions:
  gate_checkout:
    ask:
      Should a gate command's `sase bead close` automatically run its guards against the
      checkout its gate turn inherits?
    choices:
      auto:
        Gate runner exports the inherited workspace; gated bead closes guard against it
      message_only:
        No runtime change; refusal only names the scanned checkout and the remedy
    default: auto
    why:
      two gate cycles were lost to a close that can never succeed; authors will hit it
      again
    answer: auto
proposed_by: bbugyi200.athena.177
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 08:03:48
status: wip
---

# Unblock flag removals that close their flag bead from a gate

## Problem

Phase `sase-1pj.8.1.7` (retire the `staged_handoffs` flag) is stuck. The gate it
published twice runs `sase bead close sase-1pk ...`, and the gate fails every time with:

> refusing to close sase-1pk: the feature-flag registry
> (src/sase/feature_flags/registry.py) still defines a flag naming this bead ...
> staged_handoffs (bead=sase-1pk)

This is deterministic and cannot succeed:

1. `src/sase/notification_gates/command_runner.py` (`run_owned_command`) always runs
   gate option and operation commands with `cwd=bundle_path`
   (`~/.sase/interaction_requests/...`). The `/sase_gate` skill documents this.
2. `sase bead close` (`src/sase/bead/cli_crud_lifecycle.py`) calls
   `raise_if_surviving_flag_definition` (`src/sase/bead/epic_symbols.py`) with
   `start=symbol_scan_start_for_operation_context(...)`
   (`src/sase/bead/operation_context.py`). That function uses the invocation checkout
   only when cwd is a same-project checkout. Otherwise it falls back to the routed
   owner's **primary workspace**.
3. The primary checkout tracks master, and master still defines the flag. The removal
   lives only in the phase's uncommitted workspace, and it can only land once the phase
   closes.
4. The phase cannot close: in its own checkout, `tools/check_feature_flags` rule 8
   errors, because the still-open bead has no definition there (the bead predates the
   24-hour creation grace).

The result is a deadlock. The bead cannot close until the removal lands, and the removal
cannot land until the bead closes.

There is a second, deeper gap. Even a close run from the right checkout turns every
sibling checkout's `just check` red. Rule 7 ("closed flag bead whose definition
survives") has no in-flight grace. It stays red from the moment the bead closes until
the removal lands on master. In this epic's flow that window includes a full follow-up
check, about 20 minutes. Rule 8 already has the matching grace for flag _creation_
(`src/sase/feature_flags/orphan.py`, commit 488995dd4d). Removal needs the symmetric
rule, because the bead store moves first and the code lands later. Bead notes show this
has hurt before: `sase-1jx.12.1.4`, `sase-1io.7.6.4.1`, `sase-1j6.10.5`, and
`sase-1j6.10.7` all recorded rule-7 failures on clean bases.

## Goal

A flag removal can close its flag bead in two ways:

- from the checkout that removed the definition, or
- from a gate turn that inherits that checkout.

Sibling checkouts get a warning, not an error, while the removal lands. A close from a
tree that still defines the flag is still refused, and the refusal says which checkout
was scanned and why.

## Changes

### 1. Rule 7 landing grace (symmetric with rule 8)

- In `src/sase/feature_flags/orphan.py`, add a classifier for closed beads whose
  definition survives (for example `classify_surviving_definition`). It takes the bead's
  `closed_at`, its closer or `closed_by` if the bead JSON exposes one, the checkout base
  commit time, `now`, and the same `ORPHAN_BEAD_GRACE`.
  - **Warning** when the bead closed after this checkout's shared-history base
    (`checkout_base_committed_at`), or closed within the grace window. Wording: the
    removal is likely another tree's in-flight change.
  - **Error** otherwise. Also error when `closed_at` is missing or unparseable, to fail
    closed like the creation classifier does.
  - Reuse the existing helpers (`_as_utc`, `_coerce_utc`, `_format_age`). Do not
    duplicate them.
- `tools/check_feature_flags`: add `closed_at` (and the closer, if available) to
  `MarkerBead` and `marker_bead_from_issue_dict`. Route the rule-7 finding through the
  new classifier with the `severity` it returns. Update the module docstring's rule 7
  line to match rule 8's wording.
- `src/sase/feature_flags/integrity.py` (`sase doctor flags.registry`,
  `closed_survives`) must use the same classifier, so the doctor and the lint never
  disagree. Add `closed_at` to whatever bead projection it consumes.
- Leave rules 6, 8, and 9 unchanged.

### 2. Gate commands can see the checkout their gate turn inherits (decision `gate_checkout`)

> [!decision] gate_checkout = auto
>
> - In `run_owned_command` (or its two callers, `notification_gates/attempts.py` and
>   `notification_gates/operations.py`), when the bundle belongs to a gate turn whose
>   workspace policy is `inherit`, add `SASE_GATE_WORKSPACE_DIR=<creator workspace_dir>`
>   to the command env, next to the existing `SASE_GATE_COMMAND=1`. The gate turn
>   already records this: `gate_turn/start_claim.py` reads the creator's meta
>   `workspace_dir`, and `gate_turn/models.py` carries `workspace_policy`. Find the
>   existing lookup from bundle/request id to gate-turn record and reuse it; do not add
>   a new store. Never export it for `release` policy or non-turn gates. The cwd stays
>   the bundle directory, so existing commands keep working.
> - In `symbol_scan_start_for_operation_context`, when the invocation cwd is not a
>   same-project checkout, `SASE_GATE_COMMAND=1`, and `SASE_GATE_WORKSPACE_DIR` resolves
>   to a same-project checkout of the routed project, use that checkout. Reuse the
>   `_same_project_invocation_checkout` logic with that directory as the candidate cwd.
>   A different project or a non-checkout value falls back to the primary workspace, as
>   today.
> - Add one sentence to the `/sase_gate` skill source under `src/sase/macros/skills/`
>   (read `generated_skills.md` memory first), next to the existing "run with the gate
>   bundle as their working directory" paragraph. Commands of an `inherit` gate turn
>   receive `SASE_GATE_WORKSPACE_DIR`, and `sase bead close` uses it automatically.
>   Regenerate or deploy as that memory says.

> [!decision] gate_checkout = message_only Skip the runner, env, and skill changes. Only
> change 3 lands.

### 3. Actionable refusal message

`raise_if_surviving_flag_definition` (and its TUI caller in
`src/sase/ace/tui/actions/_artifacts_beads_mutations.py`, which shares the guard) should
state which checkout's registry it scanned. When it fell back to the primary workspace
because the invocation was not a checkout, it should say so and add the remedy: run the
close from the checkout that removed the definition. With `auto`, also mention the gate
turn's inherited workspace. Update the rule-7 rationale in the message: with change 1,
siblings warn during the landing grace, so the refusal now guards against closing a bead
nobody has removed. Plumb a "scanned from fallback vs. local checkout" signal from
`symbol_scan_start_for_operation_context` instead of re-deriving it in the guard.

## Tests

- `tests/feature_flags/test_orphan.py`: the new classifier covers warning when closed
  after the checkout base, warning within the grace, error when older than both, and
  error when `closed_at` is missing.
- `tests/test_check_feature_flags_tool_beads.py` (with
  `_check_feature_flags_tool_helpers.py`): a closed bead with a surviving definition is
  a warning inside the grace and an error outside it; the warning does not fail the tool
  run.
- Doctor integrity tests for `closed_survives`, matching the severities above.
- `tests/test_bead/test_epic_symbols.py`: the refusal names the scanned checkout and the
  fallback reason.
- With `auto`:
  - Operation-context tests: `SASE_GATE_COMMAND=1` plus a same-project
    `SASE_GATE_WORKSPACE_DIR` selects that checkout, and a foreign or missing directory
    falls back to the primary workspace.
  - A gate command test (near `tests/test_gate_command_exec_portability.py`) shows the
    env var is present for an `inherit` gate turn and absent otherwise.
  - An end-to-end-style test closes a flag bead from a gate command whose inherited
    checkout removed the definition, while the primary checkout still defines it, and
    the close succeeds.

Follow `lint_and_test.md` memory for verification (`just check` through
`sase tool run`).

## Out of scope

- Do not touch beads `sase-1pk`, `sase-1pj.8.1.7`, or their gates. The user unblocks
  that phase separately.
- No change to rule 8's semantics or to the gate bundle cwd contract.
- No memory edits. `sase_flags.md` already says to close the flag bead in the removing
  change, which these changes make workable.
