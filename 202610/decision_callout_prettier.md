---
tier: tale
title: Keep proposed plans valid through prettier
goal:
  A plan that passes `sase plan validate` can no longer fail host validation after `sase
  plan propose`, and a decision callout parses the same whether or not prettier joined
  its header with the first body line.
size: medium
proposed_by: bbugyi200.apollo.6p
create_time: 2026-10-11 07:02:11
status: wip
---

# Plan: Keep proposed plans valid through prettier

## What failed

The `research.0u.linker.w0` agent (project `gh_bobs-org__bob-cli`, opus at xhigh) spent
30 minutes writing an epic plan. `sase plan validate` passed it with 0 warnings, and the
agent then ran `sase plan propose`. Propose formatted the plan with prettier, archived
it to `~/.sase/plans/202610/daily_stats.md`, and killed the runner. When the host
created the plan gate, it validated the archived copy and raised:

```
PlanApprovalValidationError: plan failed epic validation: .../daily_stats.md:941
[decision-branch-unknown] decision callout names unknown decision `ref_in_done With `ref_in_done`
```

The planner turn had already ended, so nothing could act on "Fix the plan and retry".
The agent was marked FAILED, and the plan had to be recovered by hand.

What the agent wrote (valid):

```markdown
> [!decision] ref_in_done With `ref_in_done = yes` (default), `#ref`/`^ref` tasks count
> in `done`, ...
```

What `format_with_prettier` archived. Prettier 3.6.2 with `--prose-wrap=always` treats a
blockquote as one paragraph and does not recognize callouts, so it joins the header with
the first body line:

```markdown
> [!decision] ref_in_done With `ref_in_done = yes` (default), `#ref`/`^ref` tasks count
```

This reproduces on a minimal tale: a toggle `ref_in_done` whose callout body contains
`=` passes `sase plan validate`, then fails it after one prettier pass.

There are two defects:

1. **Header parsing (root cause).** In sase-core,
   `crates/sase_core/src/plan/decisions/callout.rs` `parse_callout_header` splits at the
   first `=` anywhere on the header line. A `<id> = <key>` callout survives the join;
   docs/sdd.md already shows the joined form
   `> [!decision] grouping = pane Group bindings by pane.` as its example. A bare `<id>`
   toggle callout breaks as soon as its joined prose contains `=`: the id becomes
   `ref_in_done With `ref_in_done`.
2. **Propose archives bytes it never validated.** `handle_plan_propose_command`
   (`src/sase/main/plan_propose_handler.py`) validates the authored text. It then stamps
   managed fields, formats with prettier, archives, and kills the runner, without
   validating the result. The first validation of the formatted bytes is the host's
   `require_plan_approval_validation`, which runs after the agent can no longer fix
   anything.

Fix 1 removes the failure that happened. Fix 2 means any future break caused by
formatting surfaces while the agent is still alive, never as a FAILED agent after
handoff.

## Change 1: sase-core tokenizes the callout header

Work in the linked checkout from `sase repo open sase-core` and read its `AGENTS.md`
first.

In `parse_callout_header`, replace `after.find('=')` with token-based parsing:

- The id is the run of characters after the marker, up to the first whitespace or `=`.
- After the id, skip whitespace. If the next character is `=`, the branch is the first
  whitespace-delimited token after it. Keep returning `Some(String::new())` when nothing
  follows `=`, so missing-branch reporting is unchanged.
- Otherwise there is no branch. The rest of the line is prose and is ignored.
- Every currently accepted form must parse as it does now:
  - `id`
  - `id = key`
  - `id=key`
  - `id =key`
  - `id = no thanks`
  - `id =`, which gives an empty branch
  - the bare marker, which gives an empty id
- Span and fence handling stay as they are.

Update the module doc comment and the `RawCallout::value` doc comment. Prose after the
header tokens is ignored, so a header that a Markdown formatter joined with its first
body line parses the same as the unjoined form.

Tests:

- In the `callout.rs` tests:
  - The exact failing header,
    ``> [!decision] ref_in_done With `ref_in_done = yes` (default), ...``, parses to id
    `ref_in_done` with no branch.
  - A table-driven case: for every header form above, appending prose that contains both
    `=` and a bare `=` yields the same `(id, value)` as the header alone.
  - `> [!decision] grouping = mode where a = b` parses to branch `mode`.
- In `crates/sase_core/src/plan/decisions/tests.rs`, add a grammar-level test: a plan
  whose toggle `ref_in_done` defaults to true, with the joined callout above in its
  body. It validates with no `decision-branch-unknown` diagnostic. Its callout wire is
  `id: ref_in_done`, `key: None`, `branch: yes`.

Nothing else in this repo changes: no wire, binding, version, or changelog edits. Use a
`fix(plan): ...` Conventional Commit subject. Run `sase tool run check`; `check` is
guarded, so do not run raw `just check`. The full run includes the `sase_core_py` and
`plan_validate_parity` tests.

## Change 2: sase propose validates the bytes it archives

In `src/sase/main/plan_propose_handler.py`:

1. Move the existing pre-check into a local helper that takes the plan text and returns
   a `PlanValidationResult`. The pre-check is `validate_plan(text, target_tier)`, plus
   `validate_host_checks(..., strict_quotes=True)` when decisions exist, with the same
   mapping of host-check exceptions to `decision-host-check-failed`. Use the helper for
   the existing authored-text check; the output must not change.
2. Run the helper on the final text. That text is `formatted`: stamps, parent and bead
   header sections, then `format_with_prettier`. Run it before writing the scratch file,
   before `move_plan_to_sase`, and before any marker write or runner kill. Also compare
   the `(id, key, branch)` sequence of `plan.decision_callouts` between the authored
   result and the formatted result. Line numbers are expected to differ.
3. If the formatted text fails validation, or its callout sequence differs, exit 1 and
   leave everything untouched:
   - Leave the scratch file byte-for-byte as the agent wrote it. Do not write back the
     stamped, inlet-stripped, or formatted text: the artifact-link inlet must survive a
     re-propose.
   - Create no archive and no pending marker, and do not kill the runner. The agent
     stays alive and can fix the plan and propose again.
4. Keep the error short and easy to read through `tail`. Agents often pipe propose into
   `tail -15`, so do not print the schema table here. Print, in order:
   - The headline:
     `Error: formatting this plan for the archive would change its meaning; nothing was proposed.`
   - For each new error diagnostic: `formatted line N [code] message`, then that line of
     the formatted text, indented.
   - For callout drift: the lost or changed `(id, key)` callouts.
   - A final one-line hint:
     `Give each > [!decision] header its own paragraph or rephrase the affected text, then rerun sase plan propose.`
5. When prettier is missing, fails, or times out, `formatted` equals the stamped text
   and the second check simply repeats the authored check. Accept the small extra cost
   of calling sase-core twice.
6. Confirm that `publish_plan_artifact_link_inlet` and `_derive_links_for_archived_plan`
   do not rewrite the archived plan's bytes. If one of them does, validate its output
   with the same helper before the marker write.

Propose fails here instead of archiving the plan unformatted because the approval-time
paths format with prettier again and then re-validate. Those paths are
`sdd/plan_archive.py`, `sdd/_write.py`, `workflows/commit/plan_hooks.py`, and
`sdd/plan_header_refresh.py`. An unformatted archive would only move the failure to
approval time.

Propose validates in authoring mode. The gate uses launch mode on the same bytes, which
is less strict. So when propose exits 0, the host gate's validation passes.

Tests: follow `tests/test_plan_command_handler_metadata.py` and
`tests/plan_command_handler_helpers.py`, and patch
`sase.file_references.format_with_prettier`:

- The formatter renames a callout's id to an unknown one. Propose exits 1. The scratch
  file is unchanged. No plan reaches the archive under the test home, no pending marker
  exists, and the runner kill is not called. Stderr contains `decision-branch-unknown`
  and the offending formatted line.
- The formatter merges two callouts into one blockquote paragraph, so one header
  disappears but the plan still validates. Propose exits 1 and reports the dropped
  callout.
- An identity formatter keeps today's success path. The existing propose tests stay
  green.

These sase tests must not depend on the parser fix in Change 1, because sase CI builds
the core revision pinned in `sase-core-revision.txt`. This change needs no pin move.

## Change 3: docs

- In `docs/sdd.md`, after "A `> [!decision] <id>` block marks the true branch of a
  toggle...", state the header grammar:
  - The id is the first word after the marker.
  - The line has a branch only when `=` directly follows the id.
  - Any other text on the header line is prose.
  - Prettier joins a header with its first body line, so both layouts mean the same.
- In `docs/sdd.md`, next to the paragraph that starts "When `sase plan propose` submits
  a plan for approval": propose formats the plan with prettier and re-validates the
  formatted copy before archiving. If formatting would change validity or the decision
  callouts, propose exits non-zero and proposes nothing.
- Check that the callout sentence in the `src/sase/main/plan_explain.py` explain text is
  still accurate. Change it only if it now contradicts the grammar.

## Verification

- sase-core: `sase tool run check` passes.
- sase: read the `lint_and_test` memory note, then run its required check, which is
  `just check` through `sase tool run`.
- Manual (needs a sase-core build that includes Change 1): a tale whose toggle callout
  was joined by prettier and has `=` in its prose passes `sase plan validate`. Do not
  run `just install` or `just install-dev`. If this host's editable core has to be
  rebuilt to observe the fix, propose that rebuild through `/sase_gate`.

## Out of scope

- The handoff error text "Fix the plan and retry", which is addressed to a planner that
  has already exited. After Change 2, propose exits 0 only when the exact archived bytes
  pass the stricter authoring validation. That leaves the path unreachable in normal
  operation, so rewording it buys little.
- Making prettier recognize callouts, or rewriting callouts to put the header on its own
  line. The parser fix makes the joined form canonical, which is what the docs already
  show.
- The `daily_stats` plan itself, which is being recovered separately.
