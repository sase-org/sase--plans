---
tier: tale
size: medium
title: "Finish landing sase-1id: close the truthful-%auto gaps and close the epic"
goal:
  Every %auto surface keeps the epic's promise. A %auto:plan agent never widens to bare
  %auto after a re-exec. A parked cross-tier gate shows as pending everywhere. Invalid
  gate auto arguments still error, and all surfaces show one error message. Epic
  sase-1id is closed with its plan marked done.
proposed_by: bbugyi200.athena.sase-1id.land
bead: sase-1id
create_time: 2026-10-09 03:28:53
status: wip
---

- **PARENT:**
  [202610/auto_p0_safety_tales.md](https://github.com/sase-org/sase--plans/blob/main/202610/auto_p0_safety_tales.md)
- **BEAD:**
  [sase-1id](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1id/README.md)

# Plan: finish landing epic `sase-1id` (Truthful `%auto`)

## Context

Epic `sase-1id` (plan `plan:202610/auto_p0_safety_tales.md`) has six closed phases:

| Phase           | Commit                                              |
| --------------- | --------------------------------------------------- |
| `grammar`       | sase `0ac86ad40c`, sase-core `e8606a56`             |
| `live_meta`     | `c70ee9af3d`                                        |
| `tier_mismatch` | `771127db29`                                        |
| `epic_workers`  | `e6adb110af`                                        |
| `prompt_bar`    | `bd6c7173dd` (despite its symvision commit subject) |
| `docs_truth`    | `c58ae7491a`                                        |

The three task beads (`sase-1hg`, `sase-15s`, `sase-1hh`) are closed. The `sase-11g`
inherit_mode note is posted.

The land agent already did the following and recorded each on `sase-1id` as a note:

- **Skills deployed.** chezmoi commit `a8d1a2fe` carries the updated `/sase_questions`.
- **Follow-ups triaged.** Every `PROPOSED FOLLOW-UP` from the phase beads is handled.
- **Lint failure routed.** The `_lint-test-waits` failure in
  `tests/ace/tui/test_plan_decision_ace_stale.py` went to `sase-1hi.10.7.6`. Do not fix
  it here.

`sase bead epic-symbols sase-1id` is currently empty. The epic has no `parent_bead`.

The land review found the defects below. Each was **caused by this epic**, so it is epic
work. Fix all of them, then close the epic in the same turn.

Constraints that still apply:

- **Rust core boundary.** Shared behavior lives in `sase_core`. Open the core repo with
  `sase repo open sase-core` and read its `AGENTS.md` (no bare `cargo`; use
  `just test -p sase_core <filter>` and `sase tool run check` there).
- **Both repos in one declaration.** Commit sase-core and sase in this turn's single
  final declaration. The host commits sase-core first and moves `sase-core-revision.txt`
  itself; do not hand-edit the pin.
- **No feature flags.**
- **No memory edits.**
- **Never hand-edit `CHANGELOG.md`.**
- **No `just check-full`.**

## 1. Post-wait re-exec must not widen `%auto:plan` (regression from `live_meta`)

`src/sase/axe/run_agent_runner_refresh.py` `_live_auto_prompt_mode` (~96-118) maps any
`auto_approve_argument` other than `tale`/`epic`, including `"plan"`, to mode `"plan"`.
`set_prompt_auto_mode` (`src/sase/macro/_directive_edit_wait.py` ~48) renders mode
`"plan"` as bare `%auto`.

As a result, a `%auto:plan` agent whose runner re-execs after a blocking wait is
rewritten to bare `%auto`. The re-extraction then drops `auto_approve_argument`, and the
agent auto-approves and launches epic plans that should park. Legacy values (`off`,
`foo`) are widened to bare `%auto` the same way.

- In `_reconcile_prompt_with_live_auto_state`, derive the replacement directive from
  `auto_launch_prefix` (`src/sase/monitor/continuation_delivery.py` ~216). That function
  already re-emits only `plan`/`tale`/`epic` literally and otherwise falls through to
  the action/approve checks. Strip its trailing newline and write it with
  `set_prompt_directive(prompt, {"auto"}, replacement_or_None)`. One rule then serves
  both paths.
- Delete `_live_auto_prompt_mode` if nothing else uses it.
- Add tests beside `test_refresh_reconcile_keeps_live_tale_mode` in
  `tests/test_plan_auto_live_meta.py`:
  - `{'approve': True, 'auto_approve_argument': 'plan'}` keeps a literal `%auto:plan`;
  - bare meta keeps bare `%auto`;
  - toggle-off meta strips the directive;
  - a legacy `{'auto_approve_argument': 'off'}` with no `approve` strips it, and never
    yields bare `%auto`.

## 2. A failed disk read must not strip live auto keys (`live_meta`)

`src/sase/axe/run_agent_markers.py` (~23-38 and ~105) and
`src/sase/axe/run_agent_wait_markers.py` (~268) pass `disk_meta={}` to
`overlay_live_auto_keys` when the disk read fails. That violates the helper's own
contract (`src/sase/axe/agent_meta.py` ~51-61: "a corrupt file must not destroy the only
good copy"). The overlay then strips every auto key and writes the result, which turns
auto off permanently.

- Track whether the read produced a dict, and pass `disk_meta=None` otherwise. The
  overlay then re-reads and, finding nothing, passes through unchanged.
- Also make the in-memory `agent_meta` drop any auto key the overlay removed;
  `agent_meta.update(merged)` alone keeps stale keys.
- Add tests: a corrupt or missing `agent_meta.json` at merge time keeps the in-memory
  auto keys, and a toggle-off on disk removes them from the in-memory dict as well.

## 3. Cross-tier normalization must not swallow invalid arguments (`tier_mismatch`)

`src/sase/notification_gates/service.py` `_normalize_cross_tier_plan_spec` (~105-120)
turns any argument that `plan_auto_covers_tier` rejects into a manual gate, invalid
values included. A hand-built `sase gate create` plan spec with `auto.argument: "foo"`
therefore parks silently, although the epic plan says invalid values stay errors.

- Normalize only valid-but-uncovered arguments: the known set in
  `src/sase/_plan_gate_metadata.py` (`_PLAN_AUTO_VALID_ARGUMENTS`). Expose a small
  public predicate there rather than importing the private set.
- Let invalid ones reach `validate_gate_spec`, which must raise `invalid_auto_argument`.
- Add a `create_gate` test for both cases (for example in
  `tests/test_plan_gates_execution.py`):
  - an `epic_plan` spec with `argument: "tale"` parks as manual with a notification;
  - an `epic_plan` spec with `argument: "foo"` raises before publication.

## 4. The agent scan wire must carry `auto_approve_argument` (sase-core, `tier_mismatch`)

`tier_mismatch` added a trailing `auto_approve_argument` to the Python `AgentMetaWire`
(`src/sase/core/agent_scan_wire_markers.py` ~276-281). The Rust scanner never fills it,
so it is always `None`.

A `%auto:plan` agent writes `approve=True` and `auto_approve_argument="plan"` with no
plan action. With the argument missing, `recorded_auto_covers_plan` counts a parked
`%auto:plan` epic gate as covered. The gate is then hidden from pending in:

- the agent list (`src/sase/integrations/_agent_list_entry_status.py` ~176-190);
- the TUI enrichment (`src/sase/ace/tui/models/_loaders/_meta_enrichment_wire.py` ~146).

In sase-core:

- Add a trailing `#[serde(default)] pub auto_approve_argument: Option<String>` to
  `AgentMetaWire` in `crates/sase_core/src/agent_scan/wire.rs`, next to the existing
  `auto_approve_plan_action` style. Keep it trailing and additive, with no schema bump.
- Fill it in the scanner builder in `crates/sase_core/src/agent_scan/scanner.rs` (~1494,
  where `auto_approve_plan_action` is coerced) with
  `coerce_str(data.get("auto_approve_argument"))`.
- Update any Rust wire/golden/parity fixture that enumerates `AgentMetaWire` fields, and
  add a scanner test that reads the argument from `agent_meta.json`.

In sase:

- Make sure the Python wire decoding maps the new key. Check any Python↔Rust wire-field
  parity test.
- Add an integration test that writes a real `agent_meta.json`
  (`approve: true, auto_approve_argument: "plan"`) with a pending epic-plan gate and
  goes through the real scanner, not a hand-built `AgentMetaWire` as
  `tests/test_enrich_agent_plan_meta.py` (~215) does today. The entry must show as
  pending review.

## 5. One `%auto` error message on every surface (`grammar`)

The epic plan requires every surface to show the exact same message, but the colon
spellings differ:

- **Python** rebuilds colon spellings as `%auto:<value>`: `_resolve_auto_fields` in
  `src/sase/macro/_directive_values.py` (~544) and `_auto_match_error` in
  `src/sase/macro/_directive_scan.py`.
- **Rust** passes the source slice: `typed_units.rs` ~426 and the editor
  `auto_directive_diagnostics`.

So `%a:foo` reports `'%auto:foo'` in Python and the prompt bar but `'%a:foo'` in the
typed planner and the LSP. ``%auto:`foo` `` differs the same way.

- Make the core build colon spellings canonically as `%auto:<unquoted value>`, the form
  Python and the prompt bar already show. Apply this in both Rust callers, or in the
  classifier itself so that callers cannot diverge. Paren spellings keep their literal
  source slice on both sides; they already agree.
- Extend `tests/test_auto_grammar_parity.py` to assert identical message text, not just
  accept/reject, for `%a:foo`, ``%auto:`foo` ``, `%auto:foo`, and `%a(epic=ask)`. Do
  this through `extract_prompt_directives`, `scan_auto_directive`, and the typed planner
  binding.
- Update any Rust test that pinned the old message.

## 6. Stale `%auto` completion test (`grammar`)

`tests/ace/tui/widgets/test_directive_completion_candidates.py` (~130-133)
`test_directive_completion_includes_representative_descriptions` still asserts the
pre-epic description and hint. It fails on master today.

- Update both assertions to the current core text from
  `crates/sase_core/src/editor/directive/metadata.rs` (~784-785) and keep exact
  equality.
- Phase `sase-1i5.9.1.2.1.7` may land the same edit first. If master already has it,
  skip this step.

## 7. Prompt bar polish (`prompt_bar`)

- **Duplicate message.** In `src/sase/ace/tui/widgets/_prompt_input_bar_dispatch.py`, a
  blocked submit with both `%dispatch` and an invalid `%auto` shows the message twice:
  the `Auto error` segment (~451) plus the identical preflight override (~458). Do not
  append the override when its message equals the auto segment's. Alternatively, do not
  set an override for auto errors at all, since the segment already shows them. Add a
  widget assertion.
- **Redundant precheck.** In `src/sase/macro/_directive_scan.py`,
  `scan_auto_directive`'s precheck `"%auto" not in prompt and "%a" not in prompt`
  reduces to `"%a" not in prompt`. Simplify it.

## 8. Missing `live_meta` tests

The `live_meta` phase spec asked for two kinds of test that are absent.

- **Write-back sites.** Add "toggle state preserved" coverage: meta with `approve`
  removed on disk stays off through the full-overwrite write-backs. Cover the sites in
  `src/sase/axe/run_agent_runner_launch.py` (~82, ~240, ~326, ~334) and the
  workspace-rebind path (`run_agent_workspace_identity.py` `_persist_agent_meta`, and
  `refresh_linked_repos_for_workspace` → `write_agent_meta`). Drive the helpers each
  site uses where a full runner is impractical.
- **The `sase-15s` scenario.** Write a bare `%auto` agent meta, then remove the auto
  keys as `sase agent persist-directive` does for the `A` toggle. The next plan gate
  built through the real readers (`build_plan_approval_gate_spec` or the
  `plan_gate_turn/create.py` path) must be manual and not marked handled.

## 9. Wording that still misleads

- **Stale comment.** In `src/sase/main/plan_propose_handler.py`, the comment at ~80-82
  says a pinned auto action "is the target tier", and the docstring step 3 (~57) has the
  same drift. Reword both: a pinned tier now selects coverage, and a cross-tier pin
  parks the plan for review.
- **`docs/macros.md` (~2939).** "defaults to plan mode when bare" reads as if bare
  `%auto` equals `%auto:plan`. Say that bare `%auto` covers both plan tiers.
- **`docs/macros.md` (~3252).** "later submits a plan" should say "an epic plan".
- **Undocumented propose line.** Document the "`%auto:<mode>` does not cover <tier>
  plans; this plan waits for review" line next to the existing
  `auto-approved: every decision takes its default` passages in `docs/cli.md` (~580) and
  `docs/sdd.md` (~540).
- **Child-epic prompts.** Commit `c64a6b3ea7` steers re-planned phases toward a child
  epic, and phase and land workers now run under `%auto:tale`. In
  `src/sase/default_config.yml`, add one short clause in two places stating that such a
  child epic plan waits for human approval before its clan launches:
  - the phase-worker child-epic sentence (~2237-2240);
  - the land prompt's child-epic path (~2163-2170).

  Keep the `{{ bead_id }}` templating intact. Update any rendering test that pins this
  text.

## Verification

1. **Workspace venv.** Run `just install-venv` if this workspace's venv is stale: it
   rebuilds `sase_core_rs` from the linked core checkout, which item 4 needs. This may
   exceed 10 minutes, so give it a long timeout or a `/sase_monitor`.
2. **sase-core.** Run `just test -p sase_core <filter>` for the touched suites, then
   `sase tool run check` from the sase-core checkout.
3. **sase targeted tests.** Run `tests/test_plan_auto_live_meta.py`,
   `tests/test_axe_plan_successor_auto_inherit.py`, `tests/test_auto_grammar_parity.py`,
   `tests/test_directives_flags.py`, `tests/test_macro_auto_scan.py`,
   `tests/ace/tui/widgets/test_prompt_dispatch_context_line.py`,
   `tests/ace/tui/widgets/test_directive_completion_candidates.py`,
   `tests/test_plan_gates_execution.py`, `tests/plan_gate_turn/test_create.py`,
   `tests/test_enrich_agent_plan_meta.py`, `tests/test_plan_command_handler.py`, and the
   new tests.
4. **sase check.** Run `just fmt`, then `sase tool run check`. Judge failures by its
   triage labels. Anything UNKNOWN is yours. The known
   `tests/ace/tui/test_plan_decision_ace_stale.py` `_lint-test-waits` failure is
   `sase-1hi.10.7.6`'s and is not this tale's.
5. **No `just check-full`.**

## Closeout (this tale's final step; it lands epic `sase-1id`)

Do these in the same turn as the code, after verification passes. Do not wait for this
work's own commit, SHA, push, or CI.

1. **Epic symbols.** Run `sase bead epic-symbols sase-1id`. For each listed
   `--epic-symbol` entry, resolve the symbol by wiring it up, privatizing it, adding a
   non-test pragma, or deleting it, per `symvision.md` (read it with
   `/sase_memory_read`). Re-key a Justfile line only to a still-open later bead that
   needs the exemption. The list is empty today; keep it empty.
2. **Close the epic.** Run `sase bead close sase-1id --note "<verification>"`. The note
   summarizes:
   - all six phases verified against the source;
   - task beads `sase-1hg`, `sase-15s`, and `sase-1hh` closed;
   - the `sase-11g` note present;
   - `/sase_questions` deployed in chezmoi commit `a8d1a2fe`;
   - follow-up triage recorded in the epic's land notes (the `_lint-test-waits` issue
     routed to `sase-1hi.10.7.6`, the other proposals declined as already fixed or
     non-specific, and blog posts declined);
   - each of items 1-9 above fixed, with the tests that prove it;
   - the `sase tool run check` result for sase and sase-core.

   If the close is rejected for leftover `--epic-symbol` entries, finish that cleanup
   and close again. Never use `--force` merely to make the close succeed.

3. **Symvision.** Run `just symvision` and confirm the whitelist is clean.
4. **Plan file.** Set `status: done` in the frontmatter of the epic's plan file, the
   PLAN path that `sase bead read sase-1id -r "Need the plan path for closeout"` shows
   (`plan:202610/auto_p0_safety_tales.md` in the plans sidecar). It currently reads
   `status: wip`. Change only that field.
5. **Parent.** `sase-1id` has no `parent_bead`, so the landing is finished.
