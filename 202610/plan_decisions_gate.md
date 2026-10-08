---
tier: tale
title: Compile plan decisions in the review gate
goal:
  The plan review compiles, resolves, freezes, and stamps Plan Decisions behind the
  plan_decisions beta flag.
size: medium
proposed_by: bbugyi200.apollo.sase-1hi.3
bead: sase-1hi.3
create_time: 2026-10-07 22:42:46
status: wip
---

- **PARENT:**
  [202610/plan_decisions.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_decisions.md)
- **BEAD:**
  [sase-1hi.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1hi/sase-1hi.3.md)

# Plan

Implement phase `sase-1hi.3` only: compile, resolve, freeze, and stamp Plan Decisions in
the plan gate. The design of record is `plan:202610/plan_decisions.md`, section 6.3,
plus reliability contracts 3, 4, 5, 6, 7, 9, and 10 in section 3 and the quote-check
timing in section 4. Where this plan and the design disagree, the design wins. Read
`sase_flags` and `lint_and_test` with `sase memory read` before editing. Do not edit
`sase-core`. Grammar, resolution, the quote matcher, and the digest already live there.

## Out of scope

Leave these to later phases of epic `sase-1hi`: the coder prompt block and epic
inheritance (`handoff`), `sase plan approve -D` (`cli`), the Decisions panel and visual
goldens (`tui`), Telegram keyboards (`telegram`), the finalizer memory guard (`guard`),
and skill text, flag removal, and docs beyond `docs/notifications.md` (`policy`).

Do not close epic `sase-1hi` or any ancestor. Do not close the `plan_decisions` flag
bead. Do not create task beads. Record extra work as `PROPOSED FOLLOW-UP:` on
`sase-1hi.3`. Close only `sase-1hi.3`, after `sase bead epic-symbols sase-1hi.3` reports
no leftovers (re-key any `--epic-symbol` line to a still-open bead first). A check
failure that reproduces on the clean base tree is a follow-up note, not a reason to
leave the bead open.

## Pin and Python wire

`sase-core-revision.txt` already records `9ea87c1181128ff87d2a90e74b782ffa369e30c3`.
That commit contains the decision work (`96d5b67b` grammar and Archived mode, `c089cf17`
payload/digest/resolve, `df735e42` quote matcher, `88d63855` sheet, summary, and prompt
block). Open the linked checkout with `sase repo open sase-core` and confirm the pin is
an ancestor of that checkout's `HEAD` and that `88d63855` is an ancestor of the pin. Do
not move the pin backward.

Run `just ratchet-core-revision --check`. Exit 0 means leave the pin. Exit 2 means
remote `HEAD` moved; bump with `just ratchet-core-revision` only when that `HEAD` still
contains `88d63855`.

The editable `sase_core_rs` loaded by `sase core health` can be older than the pin and
lack the decision bindings. Before wrapping them, import `sase_core_rs` and require
these callables: `plan_decisions_payload`, `plan_decisions_digest`,
`plan_decisions_resolve`, `plan_decision_quote_match`, `plan_decision_sheet`,
`plan_decision_summary`, `plan_decisions_prompt_block`. If any are missing, rebuild from
the linked checkout the way `docs/rust_backend.md` describes (`just check` refreshes a
stale stamp; `just rust-dev-install-uv-tool` targets the uv tool install). Do not
develop against a checkout older than `88d63855`.

Add those seven names to `REQUIRED_BINDINGS` in `tools/validate_sase_core_rs` and to the
test helper that mirrors that tuple.

In `sdd/plan_validate.py`:

- Accept mode `archived` on `validate_plan` / `validate_plan_file` and pass it through
  to `plan_validate`. Do not bump `PLAN_WIRE_SCHEMA_VERSION` (still 3).
- Extend `_ValidatedPlan` with `decisions`, `decision_callouts`, `decided_by`, and
  `decided_via`. Keys omitted by `skip_serializing_if` mean an empty tuple or `None`, so
  decision-free plans stay valid.
- Rehydrate the wire. A decision is
  `{id, kind, ask, why, choices: [{key, label}], default, memory, requested, answer}`. A
  callout is `{id, key, branch, start_line, end_line}`. Ignore unknown extra keys.

`tests/test_plan_validate.py` pins schema field order against the old core. Once the
pinned extension loads, tale order is `tier`, `title`, `goal`, `size`, `model`,
`decisions`, `decisions.<id>.ask`, `decisions.<id>.choices`, `decisions.<id>.default`,
`decisions.<id>.why`, `decisions.<id>.memory`, `decisions.<id>.requested`,
`decisions.<id>.answer`, `links`, `create_time`, `status`, `decided_by`, `decided_via`,
`bead`, `proposed_by`, `parent`, `bead_id`. Epic order is the same shape with `size`
absent and the phase, `patch`, `bug_id`, and `parent_bead` rows where
`crates/sase_core/src/plan/validate.rs`
`schema_is_ordered_and_contains_exact_phase_guidance` already asserts them. Update every
Python assertion that pins that list.

Switch `sdd/committed_plan_validation.py` (`inspect_committed_plan` /
`validate_plan_for_commit`) from the default Authoring mode to `archived`. Archived is
as strict as Authoring and also allows `answer`, `decided_by`, and `decided_via`.
Unstamped plans must still pass.

## Beta flag

Create the flag only with `sase flag new`. Paste the registry entry it prints into
`src/sase/feature_flags/registry.py`. Do not hand-write the bead and do not reuse epic
`sase-1hi` as the removal bead.

```bash
sase flag new plan_decisions -k beta \
  --when-enabled "Plan validate, propose, and the plan gate compile, resolve, freeze, and stamp Plan Decisions." \
  --when-disabled "Validate and propose reject a plan that contains decisions: with decisions-disabled, the schema table and --explain hide decision rows, and the plan gate adds no decision properties." \
  --remove-when "Epic sase-1hi's policy phase deletes the Off branch and makes Plan Decisions unconditional."
```

Read the flag with `current_flags().enabled(FeatureFlag.plan_decisions)`, the same way
other flags are read.

**Off.** `sase plan validate` and `sase plan propose` add diagnostic
`decisions-disabled` when frontmatter contains a `decisions` key, including an empty
map. Filter the schema table and `--explain` so rows named `decisions`, rows whose names
start with `decisions.`, and `decided_by` / `decided_via` are hidden. Rust still returns
those rows. Gate build adds no `decision_*` properties, no `payload.decisions`, and no
decisions note. A plan that still carries `decisions:` and reaches gate build while the
flag is off fails with the same code instead of compiling a decision gate.

**On.** Everything below. Decision-free plans stay byte-for-byte on the gate wire they
have today.

## Adapter

Add `sase/sdd/plan_decisions.py` as the only Python entry point. Telegram and every
other surface import this module, not `sase_core_rs`, for decisions. It wraps the seven
bindings and owns two host steps.

**Memory scope.** Resolve each memory decision's selectors through
`memory.selector.resolve_memory_selector_batch`. Freeze a web selector's current strand
list into the host record. Build one `PlanDecisionMemoryRecordWire` per resolved note:
`selector`, `kind` (`note` | `web` | `strand`), `scope` (`project` | `home`), `path`,
`type` (`core` | `reference` | `web` | `strand`), `exists`, and `strands` when a web was
frozen. Two decisions whose resolved note sets intersect raise
`decision-memory-overlap`. An unresolvable selector is an error at validate and propose.
At gate build it is a `GateError` (do not publish a gate that failed to resolve).

**Quote verification.** Call `sase.sdd.plan_human_text.human_authored_texts` on the
planner's artifacts dir, then `plan_decision_quote_match`. Host facts per decision id
are `{requested_verified, provenance, resolved}`:

- verified quote: `requested_verified` true, provenance `asked`
- memory decision with no quote, or a default of false: provenance `not_asked`,
  `requested_verified` false
- quote present but unmatched, or a missing quote source: provenance `quote_not_found`,
  `requested_verified` false

Do not emit `inherited` in this phase. Rust clamps an unverified memory default of true
to effective default false. Pass facts for memory decisions always. Omitting a
non-memory id is fine.

**Validate and propose.** Inside an agent (`SASE_AGENT` or `SASE_ARTIFACTS_DIR`),
`sase plan validate` and `sase plan propose` run selector resolution, overlap, and
`decision-requested-unverified`. On a mismatch, print the matcher's closest human
sentence so the planner can fix the quote in the same turn. Outside an agent, validate
does not fail quotes; it says quote verification runs at propose. `propose` always runs
in an agent.

When the plan has at least one decision, `propose` prints `Plan Decisions: N (🧠 M)`
before the handoff marker (`N` decisions, `M` memory decisions). When that plan will be
auto-approved (`get_auto_plan_approval_action()` is not `None`), both validate and
propose also print `auto-approved: every decision takes its default`. Decision-free
output stays unchanged.

## Gate build

In `plan_gate._build_plan_gate_spec`, when the flag is on and the validated plan has
decisions:

- `payload.decisions` is the frozen definition vector from
  `plan_decisions_payload(validated, host_facts)`, built from fresh host facts.
  Verification fails closed here: an unverified quote forces provenance
  `quote_not_found` and does not block gate creation.
- Raw `input_schema` properties named `decision_<id>` go on tale `approve`, `commit`,
  and `feedback`, and on epic `approve` and `feedback`. A toggle compiles to
  `{"type": "boolean"}`. A choice compiles to `{"enum": [keys]}` in author order. They
  are never required. Keep `additionalProperties: false`.
- Approve and commit result schemas gain a required `decisions` object whose properties
  are those same ids, fully resolved (boolean or the choice enum).
  `execute_plan_gate_command` copies the normalized `decision_*` stdin values into that
  object. Reject and feedback result schemas stay as they are.
- `presentation.notes` gains a second line `N decisions · 🧠 M` (`1 decision` when `N`
  is 1). The first line stays `Tale ready for review:` / `Epic ready for review:`.

In `ace/tui/modals/plan_approval_gate_data.py`, treat every property whose name starts
with `decision_` as host-collected, beside the existing `HOST_COLLECTED_PROPERTIES`
names. This is a prefix rule, not a list of ids. Without it, Enter opens the raw YAML
panel while the flag is on and before the `tui` phase lands. Do not build the Decisions
panel here.

## Kind validation

Add `_validate_plan_decisions` in `notification_gates/kind_validation/plan.py` and call
it from `validate_plan_spec`.

- When `payload.decisions` is absent, decision properties are absent and result schemas
  have no `decisions` object. Existing sealed checks stay exact.
- When it is present, it round-trips through core (`plan_decisions_digest` of the
  payload equals a fresh digest of the same vector).
- Each of approve, commit (tales), and feedback has exactly the `decision_*` properties
  compiled from that payload, with the same types, and none are required.
- Approve and commit result schemas require `decisions` and match the compiled value
  types.
- Query, option ids, groups, command scripts, and the edit operation stay sealed exactly
  as they are today.

## Normalization

Add
`GateAdapter.normalize_option_inputs(envelope, selected_option_ids, option_inputs, *, source, caller)`.
The base implementation returns the inputs unchanged. The plan adapter:

- Collects `decision_*` values across the selected options. Any disagreement on the same
  id is `decision_conflict`.
- Calls `plan_decisions_resolve` with caller `auto` when `source == "auto_resolution"`,
  otherwise the `caller` argument (`human` or `agent` from `gate_response_caller()`).
  `auto` takes defaults only. An `agent` memory value of true is
  `memory_decision_requires_human` unless the effective default is already true.
- Writes the identical resolved vector back onto every selected option that declares
  decision properties. Omitted ids become `effective_default`. Explicit defaults and
  omissions must digest to the same `input_identity`.

Call the hook once at the top of `execute_gate_selection`, before
`accept_gate_decision`. Pass that output into both the receipt path and the later
execution path. `accept_gate_decision` currently resolves inputs again for
`value_digest`; it must digest the normalized vector, not the raw omitted one. Resolver
errors raise `GateError` before `accept_gate_decision` and before any command. Contract
3, 4, and 9.

`service._resolve_auto_gate` already calls `execute_gate_selection` with
`source="auto_resolution"`. That source selects caller `auto` inside the hook. Do not
trust `response["caller"]` for `%auto`: the executor still records
`gate_response_caller()` on the response.

## Revision binding

`execute_gate_selection` gains optional `expected_review_revision`. The envelope's
`review_revision` starts at 1 and increments in `notification_gates/edits.py` and
`operations.py`. A provided value that differs from
`int(envelope.get("review_revision", 1))` raises `stale_review` before any side effect.
An absent value stays unchecked so mobile and older clients keep working. Contract 5.

`sase gate answer` reads optional payload field `review_revision` and passes it through.
Do not require the field on the CLI flags. ACE and Telegram will submit it in their own
phases.

## Edit freeze

The plan adapter's `validate_edited_resource` still calls
`require_plan_approval_validation`. When `payload.decisions` is present, also:

- Refuse the edited plan when its frontmatter contains `answer`, `decided_by`, or
  `decided_via`.
- Rebuild definitions for the edited plan with `plan_decisions_payload`, using the host
  facts already frozen inside `payload.decisions` (`requested_verified`, `provenance`,
  `resolved` copied back out by id). Do not re-walk memory or re-check quotes during an
  edit.
- Compare `plan_decisions_digest` of that rebuild with the digest of
  `payload.decisions`.

On mismatch or a forbidden answer field, raise with this exact text:
`Decisions are fixed for this review. Change answers in the Decisions panel, or send feedback to change the questions.`
Prose edits outside `decisions:` still validate and advance the revision. Contract 6.

## Feedback carry

`assemble_feedback_replan_prompt` takes the resolved decision rows for a feedback
submission. When any value is changed from its effective default, append:

```markdown
### Reviewer's provisional decisions

- grouping = mode (was pane)
- tui_note = yes (not authorization; default it on only by quoting human feedback text)
```

List only changed values, in author order. A memory decision the reviewer switched on is
included and explicitly called **not** authorization. Unchanged defaults are omitted.
Both callers must pass the rows: `axe/run_agent_exec_plan.py` and
`plan_gate_turn/followup.py`. Read the resolved values from the feedback option's
normalized inputs, not from a second resolution.

## Stamping

Stamp the durable plan, never the bundle copy of `plan.md`.

In `prepare_plan_terminal_response`, after `_sync_reviewed_plan_to_durable_best_effort`
and before the archive commit, write answers into the durable file through
`sdd/frontmatter.py`. `set_frontmatter_fields` already uses `sort_keys=False`. Update
the existing `decisions` map in place so author order survives: each decision gains
`answer:`, and the document gains top-level `decided_by` and `decided_via`. Toggle
answers are YAML booleans. Choice answers are canonical keys.

Map the response:

| source                 | decided_by    | decided_via |
| ---------------------- | ------------- | ----------- |
| `auto_resolution`      | `auto`        | absent      |
| `tui`, `plan_response` | from `caller` | `tui`       |
| `cli`                  | from `caller` | `cli`       |
| `mobile`               | from `caller` | `mobile`    |
| `telegram`             | from `caller` | `telegram`  |

`caller` `human` maps to `decided_by` `reviewer`. `caller` `agent` maps to `agent`. Any
other source fails the stamp. Do not invent a surface.

This function already runs for approve, commit, and epic before `prepare_epic_launch`,
so tales (approve-only, commit-only, and both) and epics stamp on the live-gate routes.
`prepare_epic_launch` must see the stamped durable file.

Stamping is idempotent. Matching answers are a no-op. Different answers are an error. If
a retry finds a durable plan with no answers, re-stamp from the resolved vector on
`response.json`. Contract 7.

## Direct routes

Expose `resolve_plan_decisions_for_direct_approval(plan, overrides, caller)` on the
adapter. It runs the same host facts (fail closed) and resolver. This phase always
passes an empty `overrides` map. The `cli` phase will pass `-D` through that argument
later.

Use it, then stamp, from:

- `execute_direct_approval` in `main/plan_direct_approval_run.py`, before
  `_archive_plan`. That command already refuses `SASE_AGENT`, so the caller is `human`.
  No live gate means take effective defaults.
- The hand-run `sase bead work` path that approves an epic plan file without a live gate
  (`bead/cli_work_from_plan.py` / `bead/epic_from_plan.py`, or wherever that path
  archives the file). Caller is `agent`. Do not double-stamp a path that already went
  through `prepare_plan_terminal_response`.

An agent caller still cannot force a memory decision on. Effective defaults apply,
including the unverified-quote clamp.

## Docs and tests

Add a "Plan Decisions" subsection under `docs/notifications.md` "Command-backed
interaction gates" covering `payload.decisions`, `decision_*` raw properties,
normalization before the receipt, `stale_review`, and the edit freeze. State that
omitted and explicit defaults share one `input_identity`, and that an absent
`review_revision` stays unchecked.

Tests, with the flag forced both ways where the behavior differs:

- Flag off: `decisions-disabled`; schema/`--explain` hides the new rows; a decision-free
  gate spec is unchanged.
- Flag on: schema rows visible; `Plan Decisions: N (🧠 M)` and the `%auto` sentence;
  closest-sentence text on `decision-requested-unverified`.
- Kind-validation drift: a `decision_*` property missing or typed wrong, and a result
  schema missing `decisions`, fail. Query, groups, and command bytes still match.
- Omitted versus explicit defaults produce the same `input_identity`.
  `decision_conflict` rejects disagreeing approve/commit values before a receipt.
- `sase gate answer` from an agent process that submits memory `true` gets
  `memory_decision_requires_human` and writes no receipt. `%auto` (`caller` `auto`)
  keeps a verified memory default of true and clamps an unverified one to false.
- Edit freeze: an `ask` change and an injected `answer` use the exact refusal sentence;
  a prose-only edit advances `review_revision`.
- `stale_review` on a mismatched `expected_review_revision`; a missing revision still
  accepts.
- Stamping on tale approve, tale commit, epic approve, and the direct no-gate route. The
  bundle `plan.md` has no `answer`. Re-stamp of the same vector is a no-op; a different
  vector errors. A retry with an unstamped durable file re-stamps from `response.json`.
- Committed-plan validation in Archived mode accepts a stamped plan and still rejects an
  Authoring error.
- Feedback prompt contains `### Reviewer's provisional decisions` only for changed
  values, and a memory value switched on says it is not authorization.

Run `sase tool run check` in this repo. Do not run `just check-full`. Do not run
`sase tool run check` in `sase-core` unless a pin bump actually changes that repo, which
this tale should not.

## Done

`sase bead epic-symbols sase-1hi.3` is clean. Close only that bead with a note that
names the flag-on and flag-off checks you actually ran. The flag bead stays open for the
policy phase.
