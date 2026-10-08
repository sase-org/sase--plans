---
tier: tale
title: Environment-independent accepted Plan Decision sheets
goal:
  Make accepted Plan Decisions render from the frozen definitions the reviewer saw, and
  finish the handoff repairs for bead read, epic inheritance, the memory guard, quotable
  question rounds, Telegram re-exports, provenance tests, Symvision, and the %auto
  receipt note.
size: medium
proposed_by: bbugyi200.apollo.sase-1hi.10.2
bead: sase-1hi.10.2
create_time: 2026-10-08 06:44:05
status: wip
---

- **PARENT:**
  [202610/plan_decisions_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_decisions_landing_repairs.md)
- **BEAD:**
  [sase-1hi.10.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1hi/sase-1hi.10.2.md)

# Plan: Environment-independent accepted Plan Decision sheets

This tale implements phase bead **sase-1hi.10.2** only. The authoritative repair list is
section 2 of `plan:202610/plan_decisions_landing_repairs.md`. The parent design is
`plan:202610/plan_decisions.md` (sections 3, 4, 6.4, and 6.8). Those decisions are
final. This tale edits no memory notes and does not change sase-core or sase-telegram.

Work only in the sase repo. Do not close the parent epic **sase-1hi.10** or
**sase-1hi**. Do not create beads. Record anything out of scope as
`sase bead note sase-1hi.10.2 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`.

Before editing, read `lint_and_test.md` and `symvision.md` with `sase memory read`.
Verification is `sase tool run check` in this repo. Do not run `just check-full`.

These failures are already known on master. If `sase tool run check` reports them, name
them in the close note and do not fix them here:

- `tests/test_macro_terminology.py::test_macro_string_literals_avoid_xprompt_terms`
  (sase-1hr)
- `tests/ace/tui/widgets/test_identity_header_raw_prompt.py::test_hinted_raw_prompt_moves_to_identity_and_keeps_its_markers`
  (sase-1hy)
- `test_candidates_fast_path_child_cpu_budget[snippet]` under the parallel lane
  (sase-1g3)
- the unused-public Symvision backlog owned by sase-1hp

Add no new Symvision unused-public symbols. A public name that only a later phase of
epic sase-1hi.10 will import gets one `--epic-symbol` row on the `_lint-symvision`
recipe, keyed to that phase's still-open bead (sase-1hi.10.3 cli, sase-1hi.10.4 tui, or
sase-1hi.10.6 telegram). Prefer keeping new helpers private so no new row is needed. The
Justfile comment above `_lint-symvision` says a comment line inside the continued
command swallows later arguments; do not put one there.

## 1. Accepted sheets ignore the reader's environment

`load_stamped_decisions` in `src/sase/sdd/plan_decision_handoff.py` calls
`build_definitions(validation, "")`. That re-resolves memory selectors and re-checks
quotes against the current process's `SASE_ARTIFACTS_DIR` and cwd. Three failures
follow:

- a reviewed "you asked" memory row redisplays as `quote_not_found`;
- a verified default-`true` memory row shows `yes ●` in `sase plan show`, because the
  empty artifacts dir clamps `effective_default` to false while the stamped answer stays
  true;
- a selector that no longer resolves makes `build_definitions` raise, and the `except`
  turns that into `None`, so the sheet and the coder block vanish.

Accepted plans must not do that. Pending plans (no `decided_by`) keep today's
`build_definitions` path so a live review still uses fresh host facts.

### Freeze the definitions at stamp time

Write a sibling JSON file next to the plan. For `foo.md` the sibling is
`foo.plan-decisions.json`. Shape:

```json
{ "schema": 1, "definitions": [] }
```

`definitions` is the frozen list the reviewer resolved: the gate bundle's
`payload.decisions`, or the list `build_definitions` produced during a gateless
approval. Do not put this list in plan frontmatter. Archived validation would reject an
unknown key, and this tale does not change sase-core.

`stamp_durable_plan` (`src/sase/plan_gate_stamp.py`) reads that list from the gate
bundle `request.json` reachable on the approval context (the same bundle
`plan_gate_decisions.py` already opens on the retry path). `stamp_direct_file` takes the
list as a new optional argument. Both direct callers pass it:

- `src/sase/main/plan_direct_approval_run.py` (`_stamp_direct_decisions`)
- `src/sase/bead/cli_work_from_plan.py` (the hand-run `sase bead work` stamp)

`resolve_plan_decisions_for_direct_approval` may grow an additive `definitions` key on
its result so those callers do not resolve twice. Do not change the meaning of `values`.

Write the sibling even when the markdown stamp is already complete and the function
would return early, but only when the sibling is missing. Never overwrite a sibling that
is already present: the frozen list stays the one the reviewer saw.

`archive_plan_file` (`src/sase/sdd/plan_archive.py`) copies the sibling when the source
has one, using the destination plan's stem (`<dest-stem>.plan-decisions.json`).
`archive_approved_plan` commits that path together with `archived.path`. An archive that
keeps an existing destination (`preserve_existing`) does not overwrite an existing
sibling.

### Load accepted plans from the freeze

When the plan has `decided_by` set, `load_stamped_decisions` builds the sheet in this
order and never calls `build_definitions`, the quote matcher, or the memory selector:

1. The sibling, when `schema` is 1 and `definitions` is a non-empty list of objects that
   each have an `id`.
2. Otherwise a neutral sheet synthesized from the authored `decisions:` map plus the
   stamped `answer` fields. Each synthetic definition uses the authored `default` as
   `effective_default` (a default `true` stays true; do not clamp it). Copy `ask`,
   choice keys and labels, `why`, and memory `selectors`. Leave `resolved` empty and
   omit `provenance`, so no provenance chip and no `⚠ quote not found` warning appears.
   The stamped answer is the row value. `★` means "value equals the authored default".
   `●` means the reviewer changed it.

Pass that definition list and the stamped answers to the existing `sheet_binding`. If
`sheet_binding` rejects a synthetic definition, match the definition shape already
accepted in `tests/test_plan_decisions_gate.py`. Do not change sase-core.

Every current caller of `load_stamped_decisions` picks this up without a new loader: the
coder block (`coder_decisions_block`), `sase plan show` (`plan_show_handler.py`,
`plan_show_render.py`), the ACE PLAN lane
(`ace/tui/widgets/prompt_panel/_agent_plan_section.py`), `sase bead read`
(`bead/cli_detail_decisions.py`), and the memory guard. Do not change ACE review-modal
layout, Telegram rendering, or the CLI card in this tale. Those belong to later phases.
The live review modal already holds `payload.decisions` from gate build; leave it alone.

Tests, in the existing handoff test module:

- A stamped memory decision whose frozen definition says `provenance: asked`,
  `effective_default: true`, and a quote still renders `you asked` and `★` when loaded
  with `SASE_ARTIFACTS_DIR` unset, cwd `/tmp`, and the quoted note absent.
- The same plan with the sibling deleted still renders the authored default as `★` and
  the stamped answer as the value, and it does not contain `quote not found`. The sheet
  is not `None`.
- A frozen `resolved` path of `sase/memory/tui.md` is still on the sheet after that
  `/tmp` load. No selector lookup runs (the note does not exist in the temp tree).

## 2. `sase bead read` DECISIONS

In `src/sase/bead/cli_detail_decisions.py`:

- `_resolve_design_file` only treats the design string as a filesystem path. Real
  designs are `plan:` refs. Resolve them with `describe_design_reference`
  (`src/sase/sdd/plan_ref_display.py`), which already calls the plan-ref resolver. Keep
  the legacy relative-path fallback that helper already has. Do not resolve `plan:` refs
  against `Path.cwd()`.
- Phase beads and epic beads both read the epic design. Stop forcing `tier="tale"` for
  `epic_phase`. Load both audiences with `tier="epic"`.
- `render_decisions_content_lines` computes `audience` onto the wire and then ignores
  it. For an accepted sheet, render
  `prompt_block_binding(sheet, decided_by, decided_via, audience)` for the stored
  `epic_phase` or `epic_land` audience. Pending sheets stay on `pending_decisions_text`.
  The phase section and the epic section for the same sheet must be allowed to differ;
  assert they match the two audience blocks, not one shared sheet dump.

In `src/sase/bead/cli_detail_json.py`, `issue_detail_wire_dict` calls
`decisions_wire_for_detail(detail)` with no roots. Thread `plan_roots` and `design_cwd`
through, and pass the `_ShowRenderContext` values from `_render_json_batch` in
`src/sase/bead/cli_show_batch.py`. Defaults stay empty so existing unit callers keep
working. The text renderer already passes these; JSON must do the same.

Replace the absolute-path tale fixtures in `tests/test_plan_decisions_handoff.py` with
three cases that go through the real resolver:

- a design stored as a `plan:` ref under a temporary plans root;
- a phase bead whose DECISIONS body is the `epic_phase` block;
- an epic bead whose DECISIONS body is the `epic_land` block.

Cover the JSON envelope with the same `plan:` ref and non-empty `plan_roots`. A `plan:`
ref with empty roots stays absent (`decisions` is `None`), not an error.

## 3. Epic inheritance fails closed and resolves `plan:` refs

`epic_decision_context` only accepts `epic_plan_snapshot` and `epic_plan_ref` when
`Path.is_file()` is true. A `plan:` ref never loads, and there is no `phase_bead_id`
route and no walk to a phase planner's coder successor.

Resolve, in order, and return the first stamped epic sheet. If none resolve, return
`None`. Do not guess from cwd.

1. `epic_plan_snapshot` when it is an existing file (the launch-time freeze).
2. `epic_plan_ref` through `resolve_plan_reference_from_roots` against the agent's
   project plan roots (from the agent meta project file / workspace), plus an existing
   absolute file. A `plan:` ref that does not resolve is skipped, not an exception.
3. `phase_bead_id`: load that bead through the bead store API `sase bead read` uses. Do
   not hand-build `sdd/beads` paths. Use the parent epic bead's design plan, resolved
   the same way, loaded as `tier="epic"`.
4. `epic_bead_id`: the same, for a land agent that has the epic bead and no phase bead.
   The design plan is that epic's design.
5. Agent-session route: walk `parent_timestamp` / `plan_chain_parent_timestamp` sibling
   artifacts, bounded the same way `plan_human_text._session_root` is bounded, and
   repeat steps 1–4 on each ancestor meta. This is how a phase planner's coder successor
   inherits when its own meta has only the tale path.

An unstamped epic, a missing bead, a cycle, or an unreadable meta yields `None`. Add
tests for a `plan:` ref, a `phase_bead_id` whose parent design is a `plan:` ref, a coder
successor that only has a parent timestamp pointing at a phase agent with that bead id,
and the existing fail-closed cases.

The guard's `_resolve_archive_ref` has the same cwd-only `resolve_plan_reference` call.
Point it at the same root-based resolver so a finalizer whose cwd is `/tmp` can still
open a `plan:` ref recorded on the agent.

## 4. Guard coverage

`src/sase/finalizers/commit_memory_guard.py` already unions sheet rows whose memory
value is true. After section 1, those rows carry the frozen `resolved` paths. Do not
re-resolve selectors inside the guard.

`_is_generated_root_path` treats every `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `QWEN.md`,
`OPENCODE.md`, and `AGENTS.md.tmpl` at any depth as generated. Hand-written nested files
such as `src/sase/ace/AGENTS.md` are then flagged, or treated as covered, for the wrong
reason. Count a path as generated only when it is a repo-root basename in that set. The
memory README stays generated through the existing `_canonical_memory_path` branch. A
nested instruction file is `other` and is not checked.

Generated root files and the memory README are covered only when the same repo also
changes a covered memory note. `_collect_committed_paths` flattens every marker into one
list, so a covered note in repo A currently excuses `AGENTS.md` in repo B. Group by the
marker's `cwd`. Keep the flat helper as the one-repo case so existing tests stay valid.

Tests:

- From cwd `/tmp`, an accepted grant whose frozen record path is `sase/memory/tui.md`
  covers a commit of `sase/memory/tui.md`.
- `src/sase/ace/AGENTS.md` alone produces no `memory_change_uncovered` diagnostic. Root
  `AGENTS.md` alone still does.
- Root `AGENTS.md` plus a covered `sase/memory/tui.md` in the same repo is clean. The
  same `AGENTS.md` in a second repo, with the covered note only in the first repo, is
  uncovered.

## 5. Every question round is quotable

`human_authored_texts` (`src/sase/sdd/plan_human_text.py`) reads only
`question_response_path` on each plan-turn link, which is the latest round. Earlier
`/sase_questions` rounds are invisible. Question members already chain with
`question_prev_artifacts_dir` (written in `question_gate_turn/create.py`). The planner
meta never points at that chain.

At the creation site in `handle_questions_marker`
(`src/sase/axe/run_agent_exec_questions.py`), record `question_gate_artifacts_dir` as
the new member's artifacts dir next to `question_response_path`. Add that key to
`record_workflow_metadata`'s retained fields in `run_agent_exec_plan.py` so the writer
keeps it.

`_question_texts` then:

- when `question_gate_artifacts_dir` is a live artifacts dir, walk
  `question_prev_artifacts_dir` from that head, oldest first, with the same 200-link cap
  the question-round walker uses;
- for each member, read `gate_bundle_path` / `response.json` and keep the current
  human-caller rule (`caller == human`, source not `auto_resolution`);
- collect only `custom_feedback` and the response `feedback` note. Selected option
  labels never count;
- when the new field is absent, keep today's single `question_response_path` behavior so
  older runs do not start throwing.

Test in `tests/sdd/test_plan_human_text.py`: a three-round chain on one planner. Only
round 1's `custom_feedback` contains the quote. Rounds 2 and 3 are human responses whose
only extra text is a selected label, or empty free text. `human_authored_texts` returns
the round-1 quote and does not return the selected label.

## 6. Telegram's import surface

sase-telegram must consume decisions only through `sase.sdd.plan_decisions`. Do not edit
the sase-telegram checkout. Re-export these names from `sase.sdd.plan_decisions` and add
them to `__all__`:

- `load_stamped_decisions` (the accepted-sheet loader from section 1)
- `summary_binding` (already defined in that module; keep the name)
- `effective_response_input` (from `sase.notification_gates.model_results`)

The names stay exactly those, for phase sase-1hi.10.6. Importing them into
`plan_decisions` is the non-test use that keeps the source symbols alive. Do not add a
second implementation.

## 7. Provenance test pins

These tests pin an exact payload and now see `prompt_origin` / `prompt_source_surface`
meta rows, or `SASE_PROMPT_ORIGIN` / `SASE_PROMPT_SOURCE_SURFACE` env rows:

- `tests/axe/test_agent_meta_atomic.py::test_generic_and_specialized_agent_meta_writers_use_atomic_publication`
- `tests/test_multi_prompt_launcher_macro_groups.py::test_launch_agents_from_cwd_segment_extra_env_shares_macro_group_counter`
- `tests/test_multi_prompt_launcher_macro_groups.py::test_launch_agents_from_cwd_force_reuse_marker_applies_to_first_swarm_slot_only`

Keep each test about its original claim. Ignore the provenance keys in the equality, or
assert them separately as the only added keys. The force-reuse marker must still be on
the first swarm slot only. Do not weaken that.

## 8. Symvision

- `HumanText` in `src/sase/sdd/plan_human_text.py` is only constructed in that file.
  Rename it to `_HumanText`, drop it from `__all__`, and keep `human_authored_texts`
  public.
- `prompt_origin_for_launch` in `src/sase/agent/launch_provenance.py` is only called by
  `with_launch_provenance` in the same file. Rename it to `_prompt_origin_for_launch`
  and update `tests/agent/test_launch_provenance.py` to the private name. Test imports
  of a private name are allowed; a public name used only by tests is not.
- `read_launch_provenance` is the runner's reader and has no non-test caller.
  `run_agent_runner_bootstrap.py` inlines the same normalize calls. Call
  `read_launch_provenance` there for keys that are present in the environment. Keep the
  current rule that a refreshed re-exec retains existing meta values and that a missing
  key stays `unknown` via `setdefault`. Do not let a missing env key overwrite a
  refreshed origin.

## 9. Document the `%auto` receipt

The receipt choice lives in the docstring of `_post_plan_auto_receipt_best_effort` in
`src/sase/notification_gates/adapters.py` and in `post_auto_approval_receipt`. Document
it in `docs/notifications.md`, next to Silent Notifications. Do not change the flags.

State the implemented choice:

- posted only when a plan or epic gate with at least one decision auto-resolves
  (`source` `auto_resolution`);
- `action` is null, so it is not a gate and opens nothing;
- `silent` is true and `muted` is left false;
- tag `plan_decisions_receipt`;
- dedup key `plan-decisions-receipt-<request id>`;
- plans with no decisions post nothing.

Describe visibility with the Silent Notifications rules already in that doc (no unread
bump, no toast, no bell, no modal, no Telegram delivery; still visible to
`sase notify list`). Do not retune `silent`, `muted`, or `action` in this tale. If that
visibility contradicts parent plan section 6.4.5 ("ACE inbox without a toast or an
unread bump"), record a `PROPOSED FOLLOW-UP:` on sase-1hi.10.2 and leave the flags as
they are.

## Verification and close

Run `sase tool run check` in the sase repo. Fix failures this tale caused. Known master
failures listed above do not keep the bead open.

Before closing, run `sase bead epic-symbols sase-1hi.10.2`. The close refuses while this
phase still has `--epic-symbol` rows. Resolve each symbol, or re-key the row to a
still-open bead (the parent epic or a later phase of sase-1hi.10). Drop a row whose
symbol is now used or private.

Close only this bead:

```bash
sase bead close sase-1hi.10.2 --note "<items fixed, the test that covers each, sase tool run check result, named KNOWN failures, and epic-symbols empty or re-keyed>"
```

Do not set bead status by hand. Do not close an ancestor.
