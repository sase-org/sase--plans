---
tier: tale
title: The FlagTriage gate and its generalized bead gate reconciler
goal:
  A due flag bead raises exactly one pending trusted FlagTriage gate whose Remove,
  Extend, Keep, and Close options each apply their host effect, and the one bead-scoped
  gate reconciler owns all three gate kinds without ever giving a bead two gates.
size: medium
proposed_by: bbugyi200.athena.sase-nb.6
bead: sase-nb.6
create_time: 2026-08-16 18:00:18
status: done
---

- **PARENT:** [202608/feature_flags.md](feature_flags.md)
- **BEAD:**
  [sase-nb.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-nb/sase-nb.6.md)

# Plan: The FlagTriage gate and its generalized bead gate reconciler

This is phase `gate` (bead `sase-nb.6`) of epic `sase-nb`, "Feature flags whose removal
is a bead, a deadline, and a gate" (`plan:202608/feature_flags.md`). Phases `core`,
`registry`, `bead`, `look`, and `lint` have landed; `cli`, `ui`, `consumer`, and
`memory` are in flight in parallel, so this plan stays strictly inside the seams the
epic assigned to `gate`.

## Why this shape

The epic's whole point is that deletion is the scarce resource, and the gate is the
channel that makes an overdue flag's removal an _answerable question_ rather than
optional hygiene. Three things must be true when this lands:

1. a due flag bead asks the question exactly once (no duplicate notifications, no second
   gate on a bead that already has one),
2. every honest answer has a button, so no owner is ever forced to click the wrong one,
3. every answer applies a real host effect through the same locked commit/push semantics
   the existing bead gates use.

The existing `TaskTriage`/`BeadSnooze` pair already solves (1) and (3) for task beads
under one chop, one lane state, and one lock. This phase generalizes that machinery to a
third kind rather than forking it, because "which gate does this bead have" is a single
question: a second chop owning the third kind could only race the first into giving a
bead two pending gates.

## Grounding

Verified in this workspace at `c8b5e962e`. Line numbers are from that tree.

| Fact                                                             | Evidence                                                                                                                                                           |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| A gate kind is a spec, preview, response, and effects module set | `bead/_task_gate_spec.py`, `_task_gate_preview.py`, `_task_gate_response.py`, `_task_gate_actions.py`, `bead/task_gate.py` facade                                  |
| A second kind was already added by mirroring the first           | `bead/snooze_gate.py` (`BEAD_SNOOZE_KIND`), `notification_gates/kind_validation/bead_snooze.py`                                                                    |
| Adapters are a flat tuple; each kind is one entry                | `notification_gates/adapters.py:267-364`; host effects dispatch in `apply_side_effects` (`adapters.py:78-123`)                                                     |
| Kind validation is dispatched by an `if` chain                   | `notification_gates/validation.py:214-241`, re-exported from `kind_validation/__init__.py`                                                                         |
| Gate validation rebuilds the spec and compares it byte for byte  | `kind_validation/task_triage.py:58-141`; description/notes are recovered by `preview_recovery.preview_matches_renderer`                                            |
| A gate declaring `panel: "beads"` bypasses the Gates tab         | `ace/tui/modals/notification_modal_tags.py:35-48` — `BeadSnooze` stays out of `_GATE_TAB_ACTIONS` for exactly this reason                                          |
| One chop owns every bead-scoped gate, under one lock             | `scripts/sase_chop_bead_task_triage.py:1-9,59`; `_GATE_KINDS = (TASK_TRIAGE_KIND, BEAD_SNOOZE_KIND)`                                                               |
| Its gateable set and kind choice are two small functions         | `gateable_tasks` (`scripts/_bead_task_triage_state.py:206-212`), `expected_gate_kind` (`scripts/_bead_task_triage_gates.py:30-32`)                                 |
| The reconciler already defers beads with an in-flight launch     | `active_task_launch_bead_ids()` (`bead/task_launch.py:146`), consumed at `sase_chop_bead_task_triage.py:280,313,374`                                               |
| Gate options support typed declared inputs, including `enum`     | `notification_gates/model_inputs.py:42-52,344-352`; snooze declares one (`bead/snooze_gate_input.py:37-52`)                                                        |
| The one shared due-ness predicate already exists                 | `flag_removal_due(record, *, today, release)` (`bead/flag_due.py:26-43`), states `live`/`soon`/`due`                                                               |
| The shared countdown label already exists                        | `flag_due_presentation(record, *, today, release).label` (`bead_flag_presentation.py:89-94`), currently whitelisted at `Justfile:326` as an unconsumed epic symbol |
| Flag beads carry key + both thresholds                           | `FlagRecord` (`bead/model.py:179-201`); `flag` present iff `issue_type == FLAG` (`bead/model.py:352-357`)                                                          |
| Extend has a store path already                                  | `proj.update(id, flag=flag_to_dict(record))` — the path `sase bead update --remove-by` uses (`bead/cli_crud.py:326-395`, `bead/flag_codec.py:20`)                  |
| `sase bead work` currently rejects flag beads                    | `cli_work_entry.py:219-284` dispatches only PLAN and TASK; `cli_work_task.py:120-129` re-checks `IssueType.TASK`                                                   |
| The launch helper is already type-agnostic below that check      | `submit_task_launch_task` builds `sase bead work <id> --yes-to-all` (`bead/task_launch.py:32-44,85-120`)                                                           |
| Closing a bead settles its pending gate                          | `bead/close_gate_settle.py:23`, pre-filtered to `IssueType.TASK` at `bead/cli_crud.py:694-705`                                                                     |
| Gate actions drive the red notification indicator                | `_PRIORITY_ACTIONS` (`notifications/priority.py:11-20`) lists `TaskTriage` and `BeadSnooze`                                                                        |
| Adapter capability parity is pinned by a test                    | `tests/test_notification_gates.py:397-419` asserts the exact `generic_form` kind set                                                                               |
| The registry is empty of flags today                             | `feature_flags/registry.py:18-26`; `feature_flag_definitions()` returns a key→definition map                                                                       |

## Decisions a worker must not silently revert

**1. Nothing in a gate's persisted bytes may drift with the clock.** Gate validation
reconstructs the preview and the notification note from the persisted payload and
compares them byte for byte, so a live countdown would make a valid gate fail validation
as it aged. The countdown is rendered from `due_as_of` and `release` values **pinned
into the payload** at creation. Those pinned values are deliberately **not** part of the
presentation fingerprint, so the gate is not canceled and recreated every single day.

**2. Due-ness is derived through `flag_removal_due`, never recomputed.** The reconciler,
the preview, and the note all go through `sase.bead.flag_due.flag_removal_due` and
`sase.bead_flag_presentation.flag_due_presentation`. No new comparison of dates or
releases is written anywhere in this phase.

**3. Due-ness is never persisted on the bead.** The chop does not flip a flag bead to
`ready`. Flag beads use `open` → `in_progress` → `closed` and never take `ready` or
`snoozed`, so the core's "Only task issues can have ready status" / "…snoozed status"
rules stay literally true. Gateable flag beads are `open` **and** `due`.

**4. Generalize the existing chop; do not fork it.** `sase_chop_bead_task_triage` keeps
its name, its script name, its state file, and its lane lock. Only its gateable set, its
kind choice, its `_GATE_KINDS` tuple, and its docstring change.

**5. Remove and Keep launch through `submit_task_launch_task`.** That is the one leased,
deduplicating launch path, and reusing it is also what makes the reconciler's existing
`active_task_launch_bead_ids()` deferral cover flag launches for free.

**6. Stay inside this phase's seams.** `notification_modal_constants.py`,
`notification_modal_tags.py`, `gate_debug_modal.py`, the ACE Beads pane, and
`docs/beads.md` / `docs/notifications.md` / `docs/configuration.md` / `docs/cli.md`
belong to the in-flight `ui` and `memory` phases. Do not edit them.

## Deviation from the epic plan, stated explicitly

The epic plan lists "the flag's call sites" as part of the gate preview. **This plan
omits them**, for a concrete reason: the preview is byte-compared by gate validation, so
anything it renders must come from the persisted payload, and a call-site list is
derived from a mutable source tree that the chop would have to AST-scan on every
reconciliation tick for every enabled project. Everything else the epic asked for (key,
both thresholds, the `look` countdown, the definition's kind and description, the bead's
notes) is rendered from the payload. Call sites remain the `cli` phase's
`sase flag show <key>` surface, which is the right place to investigate a flag.

The implementing agent MUST record this on the phase bead before closing it:

```bash
sase bead note sase-nb.6 'PROPOSED FOLLOW-UP: FlagTriage preview omits call sites — byte-compared previews cannot derive them from a mutable tree; consider carrying a payload-captured call-site list once `sase flag show` owns the scan.'
```

## The contract

### Kind, adapter, and registration

- Kind `flag_triage`, action `FlagTriage`, pending action kind `flag_triage`, sender
  `bead`, continuation mode `flag_triage`, `request.json`/`response.json`,
  `legacy_directory_key="bundle_path"`, `auto_policy="forbidden"`, `neutral_only=True`,
  `generic_form=True`.
- Register it in `_ADAPTERS` (`notification_gates/adapters.py`) between `bead_snooze`
  and `custom`, add a `flag_triage` branch to `apply_side_effects`, add
  `"flag_triage": ("remove",)` to the `expected_primary` table in `validation.py`, and
  dispatch `validate_flag_triage_spec(spec)` there.
- Add `"FlagTriage"` to `_PRIORITY_ACTIONS` (`notifications/priority.py`). A due-flag
  gate that does not drive the priority indicator is not the "unignorable nag" the epic
  relies on.

### The five-module set

Mirror the TaskTriage layout exactly. Every helper is a pure function of its arguments,
because gate validation rebuilds the spec from the persisted payload and compares.

- `src/sase/bead/_flag_gate_spec.py` — constants, `build_flag_triage_gate_spec`,
  `flag_triage_option_spec`, `flag_triage_result_schema`,
  `flag_triage_gate_command_script`, `execute_flag_triage_gate_command`.
- `src/sase/bead/_flag_gate_preview.py` — `render_flag_triage_preview` and
  `flag_triage_presentation_note`.
- `src/sase/bead/_flag_gate_response.py` — `FlagTriageResponse` and
  `translate_flag_triage_response`.
- `src/sase/bead/_flag_gate_actions.py` — the four host effects.
- `src/sase/bead/flag_gate.py` — the facade every consumer imports from, and the module
  path the persisted command wrapper names. It also defines `create_flag_triage_gate`.
- `src/sase/notification_gates/kind_validation/flag_triage.py` and
  `flag_triage_payload.py`, exported from `kind_validation/__init__.py`.

Reuse `bounded_gate_title` from `sase.bead.task_gate` rather than restating the 120-char
title rule.

### The payload

```jsonc
{
  "bead_id": "sase-nb.20",
  "project": "sase",
  "title": "Remove the prettier_enabled flag",
  "created_at": "2026-08-01T00:00:00Z",
  "size": "small", // or null
  "refs": [],
  "flag": {
    "key": "prettier_enabled",
    "remove_by_date": "2026-08-01",
    "remove_by_release": "0.16.0",
  },
  "due_state": "due", // pinned; always "due" for a raised gate
  "due_as_of": "2026-08-16", // pinned reconciliation date
  "release": "0.16.0", // pinned installed release
  "definition": { "kind": "sunset", "description": "…" }, // or null when unregistered
}
```

`flag_triage_payload.py` validates this shape exactly (unknown or missing fields are a
`invalid_flag_triage_payload` `GateError`), reusing `is_valid_sase_project_name` for
`project` and `PhaseSize` values for `size` the way `task_triage_payload.py` does, and
returning a frozen `FlagTriagePayload`. Reconstruct the `FlagRecord` through
`sase.bead.flag_codec.flag_from_dict` and call `record.validate()`, mapping `ValueError`
to a `GateError`.

The `definition` block is looked up from `feature_flag_definitions()` **by the chop**
(see below), not by the spec builder, so `build_flag_triage_gate_spec` stays pure.

### Presentation

`sender: "bead"`, `icon: "⚑"`, `title: bounded_gate_title(bead_id, title)`,
`tags: ["bead", "flag"]`, `panel: "beads"`, `panel_icon: "⚑"`, `files: ["flag.md"]`,
`preview: "flag.md"`, plus `origin_agent` when `created_by` is non-empty. Declaring
`panel` is what keeps this gate out of the Gates tab and off `HITL_ACTIONS`, so **no
`sase-core` change is needed by this phase**.

The single presentation note is a pure function of pinned payload fields, e.g.

```
sase-nb.20 [⚑ prettier_enabled] — Remove the prettier_enabled flag · DUE ⧗ +0d
```

built from `flag_due_presentation(record, today=<pinned>, release=<pinned>).label`.
Consuming `flag_due_presentation` means the now-stale
`--epic-symbol "sase-nb(flag_due_presentation)"` line must be deleted from the symvision
invocation in the `Justfile`; leave the other `sase-nb(...)` entries alone, they belong
to phases still in flight.

### The four options

Order is `remove, extend, keep, close`; `query = "remove OR extend OR keep OR close"`;
`primary_branch = ("remove",)`.

| id       | icon | label  | declared inputs                                  | feedback | result schema                                 |
| -------- | ---- | ------ | ------------------------------------------------ | -------- | --------------------------------------------- |
| `remove` | 🚀   | Remove | `winner`: required enum `enabled` \| `disabled`  | optional | `{action, winner}`                            |
| `extend` | ⏳   | Extend | `until`: required line; `release`: required line | required | `{action, remove_by_date, remove_by_release}` |
| `keep`   | ⚑    | Keep   | none                                             | required | `{action}`                                    |
| `close`  | ✕    | Close  | none                                             | required | `{action}`                                    |

Why each exists (do not drop one to "simplify"): `remove`'s `winner` enum is what makes
Remove actionable rather than a shrug — the launched worker is told which branch
survives and which is deleted. `extend` is the only path that defers, and it costs a
written reason **and** a new dated threshold, so perpetual silent extension is
impossible. `keep` makes "this was never temporary" a supported, recorded answer instead
of a permanently red lint. `close` abandons the removal work and deliberately leaves
`tools/check_feature_flags`' closed-bead-with-surviving-definition error to catch the
orphan if the flag itself survives.

`execute_flag_triage_gate_command(option_id)` follows `execute_task_triage_gate_command`
exactly: `@gate_command_entrypoint`, JSON object on stdin, unknown option → exit 2, and
for options with no declared inputs a non-empty input is an error. It **echoes back what
it validated**, so the host effect acts on the value the reviewer actually chose rather
than re-reading free text:

- `remove`: require `winner in {"enabled", "disabled"}`, print
  `{"action": "remove", "winner": …}`.
- `extend`: resolve `until` through `sase.bead.snooze_time.parse_snooze_request` (so
  `90d`, `2026-12-01`, and a full ISO timestamp all work), take `.until`'s date part as
  `remove_by_date`, and validate the `release` line by constructing a `FlagRecord` and
  calling `.validate()` so the release-string rule cannot drift from the store's. Print
  `{"action": "extend", "remove_by_date": …, "remove_by_release": …}`. A
  `SnoozeTimeError` or `ValueError` prints to stderr and returns 2, which leaves the
  gate pending — a mistyped threshold costs a retry, not the gate.

Put the `extend` input declaration and its resolver in a small
`src/sase/bead/flag_gate_input.py`, mirroring `snooze_gate_input.py`, so the declaration
consumed at creation and the one rebuilt by validation are literally the same function.

### The preview (`flag.md`)

Payload-derived material first, then `## Description`, then the conditional `## Notes`
tail — the same ordering `render_task_triage_preview` uses, because
`preview_matches_renderer` recovers description and notes by slicing between them. Omit
the `## Notes` heading entirely when notes are blank.

```markdown
# sase-nb.20 — Remove the prettier_enabled flag

> [!WARNING] **⚑ `prettier_enabled` is due for removal**
>
> **Remove by:** 2026-08-01 · v0.16.0 **Status:** DUE ⧗ +0d (as of 2026-08-16, release
> v0.16.0)

**Filed by:** `@claude_coder`

**Created:** …

**Size:** `small`

**Kind:** `sunset`

## What this flag does

Routes prettier formatting; the disabled branch is the retired `SASE_DISABLE_PRETTIER`
path.

## Description

…

## Notes

…
```

When `definition` is `null`, replace the `**Kind:**` line and the
`## What this flag does` section with a single callout stating that no registry
definition names this key and that `tools/check_feature_flags` treats that as an error.
Never mention a memory-note path in this text: the gate ships to every install and
`sase/memory/sase_flags.md` is local to this repo (see the owner's design-correction
notes on the epic bead).

Escape backticks in interpolated values the way `_task_gate_preview._markdown_code`
does.

### Response translation and host effects

`translate_flag_triage_response(bundle_path, response)` mirrors
`translate_task_triage_response`: re-read the bead identity out of the **persisted
request** (never the response), require exactly one selected option id, require the
matching option result's `action` to equal the selection, require feedback for `extend`,
`keep`, and `close`, and require `winner` / `remove_by_*` to be present and well-typed
for their actions. It returns a frozen `FlagTriageResponse` carrying `bead_id`,
`project`, `title`, `key`, `action`, `feedback`, `source`, `winner`, `remove_by_date`,
`remove_by_release`.

Each effect re-checks the action it implements and resolves the project checkout through
`resolve_task_launch_cwd_for_project`, mapping failures to a
`GateError("invalid_flag_project", "payload.project", …)`. All mutations go through
`bead_store_mutation(auto_commit_bead_store, cwd=…)` with a
`require_mutation_commit_message` / `close_mutation_commit_message` message, exactly
like `_task_gate_actions.py`. Attribute mutations with `bead_gate_actor`.

- `remove_flag_triage(decision) -> Proc`: append an attributed note recording the
  winning branch (`"Flag removal triaged: the <winner> branch wins."` plus the
  reviewer's note when present), commit with the `note` message, then
  `submit_task_launch_task(bead_id, project=…, feedback=<removal brief>, origin=task_launch_origin_from_gate_source(source))`.
  The brief tells the worker this is a feature-flag removal bead, names the key, names
  the winning branch, and says to delete the losing branch and the registry entry in the
  same change that closes the bead. Write the returned proc id back into `response.json`
  from `apply_side_effects` the way the `task_triage` launch branch does.
- `extend_flag_triage(decision)`: build the new `FlagRecord` (same key, new thresholds),
  `proj.update(bead_id, flag=flag_to_dict(record))`, append an attributed note recording
  the old thresholds, the new thresholds, and the required reason, and commit with the
  `update` message. The bead stays `open`, so the reconciler's next tick finds it no
  longer due and cancels the pending gate as stale.
- `keep_flag_triage(decision) -> Proc`: append an attributed note recording the
  permanence rationale, commit, then launch a promotion worker through the same
  `submit_task_launch_task` path with a brief that says to promote the definition to
  `kind: "ops"` with that rationale or convert it to an ordinary config field, then
  close the bead.
- `close_flag_triage(decision)`:
  `proj.close([bead_id], reason=decision.feedback, resolution="canceled")` with the
  standard close-commit message.

### Validation (`kind_validation/flag_triage.py`)

Follow `task_triage.py` section for section: structure (continuation mode, query,
branches, no groups/operations), payload parse, options rebuilt whole through
`GateOption.from_mapping(flag_triage_option_spec(id), index)` and compared, resources
(exactly the four command scripts plus `flag.md`, each command byte-compared against
`flag_triage_gate_command_script`), presentation compared field by field against the
rebuilt note, and the preview compared through `preview_matches_renderer` with
`__FLAG_TRIAGE_DESCRIPTION__` / `__FLAG_TRIAGE_NOTES__` markers.

## The reconciler

All changes are in `scripts/_bead_task_triage_state.py`,
`scripts/_bead_task_triage_gates.py`, and `scripts/sase_chop_bead_task_triage.py`.

1. **Gateable set.** Rename `gateable_tasks` → `gateable_beads` and widen it to one
   store pass over `statuses=[OPEN, READY, SNOOZED]`, `issue_types=[TASK, FLAG]`,
   filtered in Python so the widened status list cannot leak `open` task beads:
   - `TASK`: keep `READY` and `SNOOZED` (today's behavior, unchanged);
   - `FLAG`: keep `OPEN` with a non-`None` `flag` record whose
     `flag_removal_due(record, today=today, release=release)` is `"due"`. `today` and
     `release` are parameters, defaulted by the chop to `core_time.local_now().date()`
     and `sase.__version__` — the same pair `bead/cli_detail.py:433` and
     `tools/check_feature_flags:549` already use. Keep them as explicit arguments so
     tests never freeze the clock globally.
2. **Kind choice.** `expected_gate_kind(issue)` returns `FLAG_TRIAGE_KIND` when
   `issue.issue_type is IssueType.FLAG`, else today's snoozed/ready answer.
3. **Request ids.** Add `FLAG_TRIAGE_KIND: "bead-flag-triage"` to `REQUEST_ID_PREFIXES`.
4. **Fingerprint.** `presentation_fingerprint` gains an optional `flag_due_state`
   argument and adds a `"flag"` block — key, both thresholds, and that due state —
   **only when `issue.flag` is not None**. Adding the key unconditionally would change
   every task gate's fingerprint and pointlessly replace every pending task gate once on
   upgrade. Do **not** put `due_as_of` or `release` in the fingerprint: they are
   presentation pinning, and including them would cancel and recreate every pending flag
   gate daily.
5. **Creation.** `create_gate` in `_bead_task_triage_gates.py` gains a
   `flag_gate_factory` parameter and a flag branch that raises `ValueError` when
   `issue.flag is None`, computes the pinned `today`/`release` and the due state, looks
   the key up in `feature_flag_definitions()` to build the `definition` block (or
   `None`), and calls the factory. The task/snooze branches are untouched.
6. **Kind tuple and versions.** `_GATE_KINDS` gains `FLAG_TRIAGE_KIND`. Do **not** bump
   `_PRESENTATION_FORMAT_VERSION` or `_GATE_CONTRACT_VERSION`: no existing gate's shape
   changes, and bumping would needlessly replace every pending task and snooze gate.
7. **Docstrings and description.** Update the chop module docstring to say it owns every
   bead-scoped gate (task triage, snooze wake, flag triage), and update the
   `bead_task_triage` chop `description` in `src/sase/default_config.yml` (around
   line 737) to name the third kind and the flag rule (a live flag bead gets a
   `FlagTriage` gate once both its thresholds have passed).

The rest of the loop needs no change: the stale-gate cancel, the wrong-kind replace, the
untracked/inactive sweeps, the in-flight-launch deferral, and the generation counter are
all kind-agnostic already.

## Closing the loop outside the chop

- `bead/close_gate_settle.py`: add `FLAG_TRIAGE_KIND` to `_CLOSE_SETTLE_GATE_KINDS` and
  generalize the module and function docstrings from "task bead" to "bead". Keep the
  public function name `settle_closed_task_bead_gates` — renaming it churns two call
  sites and a docs sentence for no behavior change.
- `bead/cli_crud.py:694-705`: widen the pre-filter from
  `issue.issue_type is IssueType.TASK` to `in {IssueType.TASK, IssueType.FLAG}` so
  closing a flag bead settles its pending gate immediately instead of waiting for the
  next tick. Update that helper's docstring.
- `bead/cli_work_entry.py:219-284` and `bead/cli_work_task.py:120-124`: accept
  `IssueType.FLAG` on the standalone-bead launch path, so `sase bead work <flag-id>` —
  the command `submit_task_launch_task` builds — works. `cli_work_task`'s status guard
  (`OPEN`, `READY`, `IN_PROGRESS`) already covers a flag bead's lifecycle and must not
  be widened. Update the two rejection messages and the `--launch-feedback` guard
  message to name "epic plan, task, or flag beads"; update the assertions in
  `tests/test_bead/test_cli_work_epic_validation.py:51,89` to match.

Do **not** reword the `bd/work_task` xprompt in `default_config.yml`: it is shared by
every task launch, and the launch brief this phase passes as `--launch-feedback` already
tells the worker it is looking at a feature-flag removal bead.

## Documentation

Update `docs/axe.md` only — its `bead_task_triage` row (line ~241) and the "one pending
gate per bead" section (lines ~245-295) describe the reconciler this phase generalizes,
and no other phase claims that file. `docs/beads.md`, `docs/notifications.md`,
`docs/configuration.md`, and `docs/cli.md` belong to the epic's `memory` phase; leave
them alone.

## Tests

Mirror the existing bead-gate test layout. New files:

- `tests/test_bead/flag_gate_test_helpers.py` — a `flag_triage_spec(**overrides)`
  builder over `build_flag_triage_gate_spec`, matching `task_gate_test_helpers.py`.
- `tests/test_bead/test_flag_gate.py` — spec shape (kind, query, primary branch,
  presentation, payload), the four option specs including the declared `winner` enum and
  the `until`/`release` lines, and `execute_flag_triage_gate_command` for each option:
  happy path, unknown option, non-object stdin, missing/invalid `winner`, unparseable
  `until`, and a malformed `release`.
- `tests/test_bead/test_flag_gate_preview.py` — the registered and
  unregistered-definition renderings, blank-notes omission, backtick escaping, and a
  golden assertion that the countdown text comes from the pinned `due_as_of`/`release`.
- `tests/test_bead/test_flag_gate_validation.py` — one accept case plus a reject case
  per section: wrong continuation mode, wrong query, forged option (tampered `inputs`),
  tampered command script, tampered preview, tampered presentation note, and a payload
  with an unknown field.
- `tests/test_bead/test_flag_gate_actions.py` — each host effect against a temporary
  bead store: Remove appends the winner note and submits exactly one launch carrying the
  winning branch; Extend rewrites both thresholds, leaves the bead `open`, and records
  the reason; Keep records the rationale and launches; Close closes as `canceled` with
  the required reason; and each helper raises `GateError` when handed the wrong action.
- `tests/test_axe_chop_bead_flag_triage.py` — a due flag bead raises exactly one pending
  `FlagTriage` gate; a second tick is idempotent (`skipped`, not `gated`); a `live` or
  `soon` flag bead raises nothing; an `in_progress` flag bead raises nothing; a bead
  whose thresholds moved gets its pending gate replaced once; a bead with an in-flight
  `sase bead work` launch is deferred; and a task bead that already holds a `TaskTriage`
  gate never gains a second gate when a flag bead is reconciled in the same pass.

Updates to existing tests:

- `tests/_axe_chop_bead_task_triage_helpers.py` — rename the `_gateable_tasks` patch to
  `_gateable_beads`, and add `make_due_flag(...)` / `make_live_flag(...)` builders. The
  four `tests/test_axe_chop_bead_task_triage*.py` files pick the rename up through the
  helper.
- `tests/test_notification_gates.py:397-419` — add `"flag_triage"` to the `generic_form`
  kind set.
- `tests/test_bead/test_cli_work_epic_validation.py:51,89` — the reworded rejection
  message, plus a new case asserting a flag bead is now accepted rather than rejected.

## Verification

```bash
just install
just check
```

`just check` runs every whole-repo lint gate plus the diff-scoped test lane. Pay
attention to two gates in particular:

- **symvision** — every new public def needs a real non-test consumer. Delete the stale
  `--epic-symbol "sase-nb(flag_due_presentation)"` line from the `Justfile` once the
  preview consumes it. If a symbol this phase adds is genuinely only consumed by a later
  epic phase, add a `--epic-symbol "sase-nb(<Symbol>)"` entry rather than a pragma.
- **`_lint-flags`** (`tools/check_feature_flags`) — must stay green; this phase adds no
  registry entries and resolves no flag at import time.

If `just check` reports an escalated or unusual selection, or takes long enough to
threaten the turn, hand `just check-full` to `/sase_monitor` with a `--next` action
instead of running it inline.

Manual end-to-end confirmation of the epic's loop (force a real bead's `remove_by` into
the past, run the reconciler, answer with Extend then Remove) belongs to the epic's land
agent, per the epic plan's Verification section. Prove the same behavior here with the
chop tests rather than by mutating a live bead store.

## Exit condition

A flag bead whose date and release thresholds have both passed raises exactly one
pending `FlagTriage` gate; Extend rewrites the bead's thresholds and the next tick
cancels the gate; Remove records the winning branch and submits one `sase bead work`
launch carrying it; Keep and Close apply their effects; the reconciler is idempotent
across ticks; a bead already holding a task gate never gains a second one; and
`just check` is green.
