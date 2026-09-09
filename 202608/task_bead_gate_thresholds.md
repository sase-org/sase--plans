---
tier: epic
status: done
title: Task-bead gate thresholds and stale-backlog cleanup
goal: "A ready task bead earns a TaskTriage gate only once it has at least
  `bead.task_triage.min_plus_ones` +1 reports, every gate already raised below that bar
  is canceled and its notification dismissed, and the sub-threshold remainder is swept
  by a new hourly chop that raises one multi-select BeadStaleCleanup gate as soon as
  `bead.task_triage.stale_cleanup_min_beads` beads have sat below the bar for
  `bead.task_triage.stale_after_days` days.

  "
phases:
  - id: triage
    title: Threshold config and TaskTriage suppression
    depends_on: []
    size: medium
    description: "triage: add the grouped `bead.task_triage` config block with its three
      fields, schema entries, and fail-open accessors, add the shared
      staleness/suppression predicates, and teach the bead_task_triage chop to withhold
      a TaskTriage gate from a sub-threshold ready task bead and to cancel the ones it
      already raised.

      "
  - id: gate
    title: BeadStaleCleanup gate contract
    depends_on: []
    size: medium
    description: "gate: build the trusted BeadStaleCleanup gate kind — constants,
      payload, per-bead select/unselect inputs, spec builder, payload-pure preview,
      command wrapper, response translation, adapter registration, and kind validation.

      "
  - id: actions
    title: BeadStaleCleanup host effects
    depends_on:
      - gate
    size: small
    description: "actions: close the beads a reviewer selected, grouped per project
      through the locked bead-store mutation and commit path, and route the answered
      gate to that effect from the adapter.

      "
  - id: chop
    title: bead_stale_cleanup chop
    depends_on:
      - triage
      - actions
    size: medium
    description: "chop: extract the shared enabled-project inventory, add the hourly
      bead_stale_cleanup chop with its own lane state and reconciliation, register it in
      the housekeeping lumberjack and the console-script table, and drop the epic symbol
      whitelist entries the earlier phases needed.

      "
  - id: polish
    title: Documentation sweep and full verification
    depends_on:
      - chop
    size: small
    description:
      "polish: reconcile the axe, notifications, beads, and configuration guides against
      the landed behavior, state the post-land rollout step that dismisses the
      reviewer's existing sub-threshold notifications, and land a green check-full."
proposed_by: bbugyi200.athena.04x
bead_id: sase-on
create_time: 2026-09-09 19:51:46
---

- **PROMPT:**
  [prompts/202608/task_bead_gate_thresholds.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/task_bead_gate_thresholds.md)
- **BEAD:**
  [sase-on](https://github.com/sase-org/sase--beads/blob/main/pages/sase-on/README.md)

# Plan: Task-bead gate thresholds and stale-backlog cleanup

## Why

Every `ready` task bead raises exactly one `TaskTriage` gate, and a gate is a priority
notification. That was fine when `ready` meant "a human decided this is worth doing". It
no longer does. `/sase_new_task` tells every agent that a genuinely new task becomes an
`open` draft and then `sase bead update <id> -s ready`, so the `ready` set is really the
set of _things some agent noticed_, and the notification inbox has become the union of
every agent's discovered follow-up work. The reviewer receives far too many task-bead
gate notifications.

The bead model already carries the signal that separates noise from work: `+1` evidence.
Per `sase/memory/sase_beads.md` a `+1` is independent corroboration from a reporter
other than the creator, and the first valid `+1` on an `open` task promotes it to
`ready` on its own. So "has at least one `+1`" cleanly names the beads a second agent
independently cared about, and today's noise is precisely the hand-promoted, zero-`+1`
remainder.

Suppressing gates alone would only move the problem: the sub-threshold beads stay
`ready` forever and nothing ever prompts a decision about them. This epic therefore
ships both halves — a threshold that quiets the inbox, and a periodic sweep that offers
the accumulated remainder for closure in one batched gate.

## Scope

**In scope.** Three grouped configuration fields; a suppression rule inside the existing
`bead_task_triage` chop plus cancellation of the gates it already raised; a new trusted
`BeadStaleCleanup` gate kind whose single option closes a reviewer-selected subset of
stale beads; a new hourly `bead_stale_cleanup` chop that raises that gate; docs and
tests.

**Out of scope.**

- Any change to bead state itself. A suppressed bead stays `ready` and stays visible to
  `sase bead list`, `sase bead ready`, the ACE Beads panel, and bead pages. Only the
  _gate_ is withheld.
- `BeadSnooze` and `FlagTriage` gates. A snoozed bead's wake gate is a deferral the
  reviewer explicitly asked for, and a flag bead's removal deadline has nothing to do
  with `+1` corroboration. Neither is thresholded.
- `open` task beads. They are drafts and have never been gate-eligible; the stale sweep
  does not adopt them.
- A new generic multi-select control for gate inputs. See "Selecting beads inside the
  gate" below for why the existing enum control is used instead.
- Any Rust core change. See "Rust core backend boundary" below.
- Any edit to `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction
  shims. No phase may touch them; that requires the user's explicit permission in the
  conversation that asks for it.

## Configuration

One grouped block under the existing `bead` key, in `src/sase/default_config.yml`:

```yaml
bead:
  # Volume controls for task-bead triage gates. A ready task bead earns a
  # TaskTriage gate only once independent reporters have corroborated it, and
  # the remainder that never clears that bar is swept periodically instead of
  # accumulating forever.
  task_triage:
    # +1 reports a ready task bead needs before bead_task_triage raises its
    # TaskTriage gate. 0 restores the pre-threshold behavior of gating every
    # ready task bead.
    min_plus_ones: 1
    # Days after creation at which a still-sub-threshold ready task bead is
    # considered stale and eligible for the cleanup gate.
    stale_after_days: 7
    # Stale beads required before bead_stale_cleanup raises its gate. Below
    # this count the chop does nothing.
    stale_cleanup_min_beads: 10
```

Schema (`src/sase/config/sase.schema.json`, under `properties.bead.properties`): a
`task_triage` object with `additionalProperties: false` and three integers —
`min_plus_ones` (`default: 1`, `minimum: 0`), `stale_after_days` (`default: 7`,
`minimum: 1`), `stale_cleanup_min_beads` (`default: 10`, `minimum: 1`).

`min_plus_ones: 0` is deliberately valid: it is the documented escape hatch back to
today's behavior, and it makes the suppression rule expressible as one comparison with
no special case.

## Rust core backend boundary

`sase/memory/rust_core_backend_boundary.md` asks that shared backend behavior live in
`../sase-core`. Everything this epic adds stays in Python, and the reason is that it
adds no new domain state:

1. The predicates read only fields the Rust bead store already returns on `Issue` —
   `status`, `issue_type`, `created_at`, and `plus_one_evidence`. There is no new wire
   type, no new stored field, and no store-format change.
2. The directly analogous artifact is `src/sase/bead/flag_due.py`: a pure Python
   predicate over already-Rust-read `Issue` records that decides whether the same chop
   owes the same reviewer a gate. The new predicates are its sibling and belong beside
   it.
3. The consumers — two axe chops and one gate adapter — are Python-only surfaces. A Rust
   predicate would be a leaf with no Rust caller.

If bead triage policy later migrates to `sase-core`, `flag_due.py` and
`task_triage_policy.py` should migrate together as one unit.

## Feature flags

None. `sase/memory/sase_flags.md` reserves flags for user-reaching behavior that is not
ready — a disabled beta, an early landed path, a deprecation whose old branch must stay
reachable — and explicitly says not to flag anything users are meant to choose forever,
because that is a config field. The three fields here _are_ the permanent user-facing
control, `min_plus_ones: 0` reaches the pre-epic behavior exactly, and the epic lands as
one combined tree, so no phase ever exposes a half-built path.

## Architecture decisions

### The suppression lives in the chop, not in the store read

`_bead_task_triage_state.gateable_beads()` answers "which beads are of a gateable status
and type"; it is patched wholesale by the existing test doubles
(`tests/_axe_chop_bead_task_triage_helpers.patch_project` substitutes
`lambda _path, **_kwargs: list(ready)`). Filtering inside it would silently disable the
new rule in every existing test and would leave `_reconcile` unable to distinguish "bead
is gone" from "bead is below the bar" when it cancels a gate.

So `gateable_beads()` is unchanged, and `_reconcile` applies the predicate itself, where
it still holds the `Issue`:

```python
gateable = _gateable_beads(beads_dir, today=today, release=release)
suppressed = {
    issue.id for issue in gateable if task_gate_suppressed(issue, min_plus_ones=min_plus_ones)
}
live_tasks = {issue.id: issue for issue in gateable if issue.id not in suppressed}
```

`suppressed` then does two jobs: sub-threshold beads never reach the gate-creation loop,
and a tracked pending gate whose bead is in `suppressed` is canceled with the distinct
reason `task_bead_below_plus_one_threshold` rather than the generic
`task_bead_no_longer_ready`.

### Existing notifications are dismissed by the mechanism that already exists

There is no one-shot sweep script in this epic, and that is deliberate. `cancel_gate()`
already calls `_settle_gate_notification()`, which marks the pending action handled and
calls `mark_dismissed()` on the row (`src/sase/notification_gates/executor.py`). The
`bead_task_triage` chop already cancels the pending gate of any bead that stops being
gateable. Making a sub-threshold bead stop being gateable therefore _is_ the dismissal:
on the first `checks`-lane tick after this lands, every sub-threshold `TaskTriage` gate
is canceled and every corresponding notification disappears from the inbox.

That tick happens automatically within five minutes of landing;
`sase axe chop run bead_task_triage` forces it immediately. A bespoke sweep would
duplicate the cancellation path with worse coverage — it would not know which gates the
chop still tracks in its lane state.

### Selecting beads inside the gate

The reviewer must be able to select and unselect each bead. The gate input contract
already has everything needed for that, and using it avoids touching shared gate
infrastructure:

- A `repeatable` `enum` field compiles to an array schema, but ACE renders every `enum`
  field through `_EnumField` (`src/sase/ace/tui/widgets/typed_input_form.py`), a button
  that cycles one value. There is no multi-select widget, so a repeatable enum would
  render as a single-valued control. Building one is a cross-cutting change to ACE, the
  CLI's `--option-input`, and the mobile bridge, for a gate that fires a few times a
  month.
- A `bool` field renders as a free-text input where the reviewer types `true`/`false`.
  Worse than what follows.

**Decision:** one non-required `enum` field per stale bead,
`choices: ["close", "keep"]`, `default: "close"`. ACE renders each as a cycling button
labeled with its bead, so unselecting a bead is one keypress on that bead's control; the
default action — close everything offered — needs no input at all. `TypedInputForm`
hides non-required fields behind its `N optional inputs` fold, which is the right shape
here: accept the batch, or expand and curate it.

Because `values()` omits untouched fields, a field absent from the submitted input means
"unchanged", so the option command treats a missing field as its declared default
`close`. That rule is a property of the command and is unit tested directly.

Field ids must match `^[a-z][a-z0-9_]*$` and bead ids contain `-` and `.`, so the id is
positional — `bead_1`, `bead_2`, … indexing `payload.beads` in order — and the bead id,
project, and age live in the field's `label` and `help`. The response echoes the
selected indexes and translation maps them back through the payload, so a forged
response cannot name a bead the gate never offered.

### One gate for all projects, with a bounded roster

`stale_cleanup_min_beads` counts stale beads across every enabled project, and one gate
carries them all — the reviewer is clearing one backlog, not one backlog per project.
The payload therefore carries `(project, bead_id)` pairs and the host effect groups its
closes by project.

The roster is capped at **50 beads**, oldest first, tie-broken by `(project, bead_id)`.
`MAX_OBJECT_PROPERTIES` is 128, so the cap is well inside the hard bound; it exists so a
300-bead backlog produces a usable modal rather than a wall of controls. The preview
names how many stale beads were left out and the chop logs the same count, per the "no
silent caps" rule — a truncated roster must never read as "this is all of them". The
next hourly tick offers the next 50.

### Gate identity and reconciliation

The chop owns its gate the way `bead_task_triage` owns its per-bead gates: lane state
under `runtime.context.state_dir` holds the pending `request_id`, a monotonic
`generation`, and a `fingerprint` over the offered roster plus the three thresholds. An
unchanged roster leaves the pending gate alone; a changed roster cancels and recreates
it; a roster that falls below `stale_cleanup_min_beads` cancels it and creates nothing.

`stale_as_of` is carried in the payload so the preview can render ages as a pure
function of the payload, and is excluded from the fingerprint — including it would
cancel and recreate the gate every single day, which is exactly the mistake
`flag_triage` avoids with `due_as_of`.

`src/sase/bead/close_gate_settle.py` cancels the per-bead gates of a bead closed through
the CLI. It is not extended to this gate: `find_pending_bead_gates()` keys on a payload
with one `bead_id`, and this payload has many. The chop's own fingerprint reconciliation
is the backstop, and it is exact — a bead closed by hand drops out of the roster on the
next tick, which cancels and recreates the gate.

## Symvision and the epic symbol whitelist

`just check` runs symvision per phase branch, test references never keep a public symbol
alive, and several symbols are introduced one phase before their consumer lands. Each
phase below names the symbols it must whitelist via
`--epic-symbol "<epic-bead-id>(<symbol>)"` in the `_lint-symvision` recipe of the
`Justfile`, using the id of the epic bead that parents the phase bead. The `chop` phase
removes every entry this epic added; `polish` verifies none remain.

---

# Phase `triage`: Threshold config and TaskTriage suppression

## Deliverables

**`src/sase/default_config.yml`** — the `bead.task_triage` block exactly as given under
"Configuration", placed after `big_epic_phase_threshold` and before `push_after_commit`.

**`src/sase/config/sase.schema.json`** — the `task_triage` object described under
"Configuration", with a `description` on the object and on each of the three fields.

**`src/sase/bead/config.py`** — three accessors beside `get_big_epic_phase_threshold`,
sharing one private `_task_triage_config() -> dict` reader and following that function's
fail-open contract exactly (any exception, a non-dict merged config, a non-dict section,
a `bool`, a non-`int`, or an out-of-range value all fall back to the shipped default):

- `DEFAULT_TASK_TRIAGE_MIN_PLUS_ONES = 1`, `get_task_triage_min_plus_ones() -> int`
  (floor 0)
- `DEFAULT_TASK_TRIAGE_STALE_AFTER_DAYS = 7`,
  `get_task_triage_stale_after_days() -> int` (floor 1)
- `DEFAULT_TASK_TRIAGE_STALE_CLEANUP_MIN_BEADS = 10`,
  `get_task_triage_stale_cleanup_min_beads() -> int` (floor 1)

**`src/sase/bead/task_triage_policy.py`** (new) — the two pure predicates both chops
share, beside `flag_due.py` and in its style (explicit arguments, no clock or config
reads inside):

```python
def task_gate_suppressed(issue: Issue, *, min_plus_ones: int) -> bool:
    """Whether a ready task bead is withheld from triage for want of +1 reports."""

def stale_task_bead(issue: Issue, *, min_plus_ones: int, stale_after_days: int, now: datetime) -> bool:
    """Whether a suppressed ready task bead has sat below the bar long enough to sweep."""
```

`task_gate_suppressed` is `True` only for `issue_type == TASK` and `status == READY`
with `issue.plus_one_count < min_plus_ones`; every snoozed bead, every flag bead, and
every other status is `False`, so the `BeadSnooze` and `FlagTriage` paths are
structurally untouched.

`stale_task_bead` is `task_gate_suppressed(...)` **and** a parsed `issue.created_at` at
or before `now - timedelta(days=stale_after_days)`. Parse with
`datetime.fromisoformat(value.replace("Z", "+00:00"))` — the convention already used in
`sase/bead/model.py` and `snooze_time.py` — and return `False` for an empty, unparsable,
or naive `created_at` rather than raising, so one malformed record can never break a
chop pass or sweep a bead whose age is unknown.

**`src/sase/scripts/sase_chop_bead_task_triage.py`** — read
`get_task_triage_min_plus_ones()` once per `_reconcile` call, compute `suppressed` per
project as shown under "The suppression lives in the chop", withhold those beads from
`live_tasks`, and pass `reason="task_bead_below_plus_one_threshold"` when cancelling a
tracked pending gate whose bead is suppressed. Add a `suppressed` counter to
`_summary()` and include it in the `reason=None` liveness test alongside
`gated`/`canceled`.

Two ordering details matter. The suppression is applied where `live_tasks` is built, so
it runs before the already-tracked-gate loop pops from it — that is what makes a
previously gated bead take the cancel branch. And the in-flight-launch check keeps
precedence: a bead with a detached launch in flight is still deferred, never canceled,
because cancelling a gate whose launch is mid-flight would race the launch.

**`src/sase/default_config.yml`** — update the `bead_task_triage` chop `description`
body to state the `+1` bar.

**Docs.** `docs/configuration.md` (the `bead` section: yaml sample plus three table
rows), `docs/axe.md` (the `bead_task_triage` narrative around lines 241–300),
`docs/beads.md` (the triage lifecycle around lines 315 and 578), `docs/notifications.md`
(the `TaskTriage` section around line 312). Each states the bar, that `min_plus_ones: 0`
restores the old behavior, that suppression withholds the gate without changing bead
state, and that an already-raised gate is canceled and its notification dismissed when
its bead falls below the bar.

## Symvision

`get_task_triage_stale_after_days`, `get_task_triage_stale_cleanup_min_beads`, and
`stale_task_bead` have no non-test consumer until the `chop` phase. Add exactly those
three `--epic-symbol` entries to `_lint-symvision`.

## Tests

`tests/test_bead/test_task_triage_policy.py` (new):

- `task_gate_suppressed` is `True` for a ready task with fewer than `min_plus_ones`
  evidence entries and `False` at or above it; `False` for `min_plus_ones=0` whatever
  the count; `False` for a snoozed task, an open task, and a due flag bead.
- `stale_task_bead` is `True` only when suppressed and
  `created_at <= now - stale_after_days`; boundary cases at exactly `stale_after_days`
  and one second short of it; `False` for empty, unparsable, and naive `created_at`;
  `False` for a bead that has cleared the `+1` bar however old it is.

`tests/test_bead/test_config.py` (extend): each accessor returns a configured value,
falls back for `None`, `0`/`-1` below its floor, `True`, and `"7"`, and does not
propagate an exception from `load_merged_config`.

`tests/test_config_schema.py` (extend): the schema accepts each field at its floor and
above; rejects a negative `min_plus_ones`, a zero `stale_after_days`, a zero
`stale_cleanup_min_beads`, a `bool`, a string, and an unknown key under `task_triage`;
and the three schema `default`s equal the three `default_config.yml` values and the
three `DEFAULT_*` constants.

`tests/test_axe_chop_bead_task_triage.py` (extend, with `min_plus_ones` monkeypatched at
the chop module):

- A ready task with zero `+1` gets no gate and is counted in `suppressed`.
- A ready task with one `+1` still gets its gate.
- A tracked pending gate whose bead has fallen below the bar is canceled with reason
  `task_bead_below_plus_one_threshold`, and its lane-state entry is dropped.
- A suppressed bead with an in-flight detached launch is deferred, not canceled.
- Snoozed beads still receive `BeadSnooze` gates and due flag beads still receive
  `FlagTriage` gates at `min_plus_ones: 5`.
- `min_plus_ones: 0` reproduces the pre-epic gating exactly.

---

# Phase `gate`: BeadStaleCleanup gate contract

Mirror the `flag_triage` module layout — the facade is the front door every consumer
imports from, and the implementation lives in focused siblings.

## Constants and identity

`src/sase/bead/_stale_cleanup_gate_spec.py`:

| Constant                               | Value                                  |
| -------------------------------------- | -------------------------------------- |
| `BEAD_STALE_CLEANUP_KIND`              | `"bead_stale_cleanup"`                 |
| `BEAD_STALE_CLEANUP_CONTINUATION_MODE` | `"bead_stale_cleanup"`                 |
| `BEAD_STALE_CLEANUP_QUERY`             | `"close"`                              |
| `BEAD_STALE_CLEANUP_CLOSE_OPTION_ID`   | `"close"`                              |
| `BEAD_STALE_CLEANUP_OPTION_IDS`        | `("close",)`                           |
| `BEAD_STALE_CLEANUP_PRIMARY_BRANCH`    | `("close",)`                           |
| `BEAD_STALE_CLEANUP_PREVIEW_PATH`      | `"stale.md"`                           |
| `BEAD_STALE_CLEANUP_COMMAND_PATHS`     | `{"close": "commands/close"}`          |
| `BEAD_STALE_CLEANUP_MAX_BEADS`         | `50`                                   |
| `BEAD_STALE_CLEANUP_CLOSE_REASON`      | `"Stale task bead swept from triage."` |

Option label `Close selected`, icon `🧹`, feedback `optional` — a note, when the
reviewer writes one, becomes the close reason; otherwise
`BEAD_STALE_CLEANUP_CLOSE_REASON` is used.

## Payload

```jsonc
{
  "beads": [
    {
      "project": "gh_sase-org__sase",
      "bead_id": "sase-a1",
      "title": "Follow up on cache invalidation",
      "created_at": "2026-08-01T09:14:02-04:00",
      "plus_one_count": 0,
      "size": "small", // or null
    },
  ],
  "omitted_count": 12, // stale beads beyond BEAD_STALE_CLEANUP_MAX_BEADS
  "min_plus_ones": 1,
  "stale_after_days": 7,
  "stale_cleanup_min_beads": 10,
  "stale_as_of": "2026-08-17T11:00:00-04:00",
}
```

`beads` is non-empty, at most `BEAD_STALE_CLEANUP_MAX_BEADS` entries, and free of
duplicate `(project, bead_id)` pairs; `project` and `bead_id` are non-empty strings.
`src/sase/notification_gates/kind_validation/bead_stale_cleanup_payload.py` parses it
into a frozen `BeadStaleCleanupPayload` and rejects anything else, exactly as
`flag_triage_payload.py` does.

## Inputs, command, and result

`bead_stale_cleanup_selection_inputs(beads)` returns one declaration per bead, a pure
function of the payload roster:

```python
{
    "id": f"bead_{index}",                    # index is 1-based, matching payload order
    "label": f"{bead.bead_id} — {truncated_title}",
    "type": "enum",
    "required": False,
    "default": "close",
    "choices": [{"value": "close", "label": "Close"}, {"value": "keep", "label": "Keep"}],
    "help": f"{project_display} · +{bead.plus_one_count} · created {date} ({age} days ago)",
}
```

Use `sase.project_display_names.project_display_name_for` for `project_display` —
user-facing text must never show a ProjectSpec key. Derive `age` from `stale_as_of` so
the declaration stays payload-pure. Truncate the title with the existing
`bounded_gate_title` helper's convention so one long title cannot dominate the form.

The command wrapper follows `task_triage_gate_command_script`: a `#!{sys.executable}`
script importing `execute_bead_stale_cleanup_gate_command` from the facade
`sase.bead.stale_cleanup_gate`, persisted into the bundle and revalidated byte for byte.
Decorate the entrypoint with `@gate_command_entrypoint`.

The command reads its stdin object, resolves every `bead_<n>` field (**absent means
`close`**, per the decision above), and emits:

```json
{ "action": "close", "close_bead_indexes": [1, 2, 5] }
```

Indexes are 1-based, sorted, deduplicated, and bounded by the roster length the command
was built for. Selecting nothing is a command failure — print
`select at least one bead to close, or dismiss this gate` to stderr and return `2`,
which leaves the gate pending so the reviewer can retry rather than losing the gate to a
mis-click. The result schema requires `action` (`const: "close"`) and
`close_bead_indexes` (`type: array`, `items: {"type": "integer", "minimum": 1}`,
`minItems: 1`, `uniqueItems: true`), `additionalProperties: false`.

## Preview and presentation

`src/sase/bead/_stale_cleanup_gate_preview.py` renders `stale.md` as a **pure function
of the payload** — a heading, one sentence naming the three thresholds, a table of
`bead id · project · age · +1 · size · title`, and a footer naming `omitted_count` when
it is non-zero. Nothing agent-authored reaches it (bead titles are carried in the
payload), so validation re-renders and compares directly; no `preview_matches_renderer`
marker recovery is needed.

`bead_stale_cleanup_presentation_note(payload)` returns the one-line notification note,
e.g. `12 stale task beads · no +1 after 7 days`. Presentation is `sender: "bead"`,
`icon: "🧹"`, `tags: ["bead", "task", "stale"]`, `panel: "beads"`, `panel_icon: "◈"`,
`files`/`preview` naming `stale.md`. No `origin_agent` — a chop, not an agent, produced
it.

## Registration

- **`src/sase/notification_gates/adapters.py`** — a `GateAdapter` with
  `kind="bead_stale_cleanup"`, `display_title="Stale Task Cleanup"`,
  `action="BeadStaleCleanup"`, `pending_action_kind="bead_stale_cleanup"`,
  `sender="bead"`, `request_filename="request.json"`,
  `response_filename="response.json"`, `legacy_directory_key="bundle_path"`,
  `auto_policy="forbidden"`, `neutral_only=True`, `default_feedback="optional"`,
  `generic_form=True`. `apply_side_effects` is wired in the `actions` phase.
- **`src/sase/notification_gates/kind_validation/bead_stale_cleanup.py`** and its export
  from that package's `__init__.py`; dispatch from `validate_gate_spec` in
  `validation.py` beside the `flag_triage` branch. Validate structure (continuation
  mode, query, single `close` branch, no groups or operations), rebuild the one option
  from `bead_stale_cleanup_option_spec(payload)` and compare it whole, check the
  resource set/roles/executability and the command content byte for byte, compare the
  presentation field by field, and re-render the preview and compare.
- **`src/sase/notifications/priority.py`** — add `"BeadStaleCleanup"` to
  `_PRIORITY_ACTIONS`. It is a human decision gate like every other bead gate, and it
  fires rarely by construction.
- **`src/sase/notification_gates/debug.py`** — add `"BeadStaleCleanup": "🧹"` to
  `_icon_for_action`.

`create_bead_stale_cleanup_gate(...)` on the facade builds the spec and calls
`sase.notification_gates.service.create_gate`, matching `create_flag_triage_gate`.

## Symvision

`create_bead_stale_cleanup_gate` has no non-test consumer until the `chop` phase, and
`translate_bead_stale_cleanup_response` none until `actions`. Add those two
`--epic-symbol` entries.

## Tests

`tests/test_bead/stale_cleanup_gate_test_helpers.py` (new, modeled on
`flag_gate_test_helpers.py`) building payload rosters and creating gates into a
redirected gate root.

`tests/test_bead/test_stale_cleanup_gate.py`: the spec round-trips through `create_gate`
and validates; one input field per bead in payload order with the positional ids,
`close` default, and both choices; the roster cap and `omitted_count` render in the
preview; the presentation block matches.

`tests/test_bead/test_stale_cleanup_gate_preview.py`: the preview is byte-identical
across two renders of the same payload; ages derive from `stale_as_of`, not the wall
clock; project names render as display names, never ProjectSpec keys; the omitted footer
appears only when `omitted_count > 0`.

`tests/test_bead/test_stale_cleanup_gate_validation.py`: a tampered option, a tampered
command script, a tampered preview, an added resource, a mutated presentation field, a
duplicate `(project, bead_id)`, an empty roster, and a roster longer than the cap are
each rejected with the expected `GateError` code.

Command tests: absent fields default to `close`; explicit `keep` removes exactly that
index; all-`keep` exits `2` with the reviewer-facing message; a non-object stdin, an
unknown field id, and an out-of-range value each exit `2`.

---

# Phase `actions`: BeadStaleCleanup host effects

## Deliverables

**`src/sase/bead/_stale_cleanup_gate_response.py`** — `BeadStaleCleanupResponse`
(selected `(project, bead_id)` pairs, optional `feedback`, `source`) and
`translate_bead_stale_cleanup_response(bundle_path, response)`, which reads the
persisted request envelope, re-parses its payload, and maps `close_bead_indexes` back
through the roster. An index outside the roster, a duplicate, or an empty list raises
`GateError` — the response can only ever name beads the gate itself offered.

**`src/sase/bead/_stale_cleanup_gate_actions.py`** —
`close_bead_stale_cleanup(decision)`. Re-check the action the way `close_task_triage`
does, group the selected beads by project, and for each project resolve the checkout
with the same helper `close_task_triage` uses (`resolve_task_launch_cwd_for_project`,
wrapped so a `FileNotFoundError`/`ValueError` becomes a `GateError` naming the project)
and run one locked mutation:

```python
with bead_store_mutation(auto_commit_bead_store, cwd=cwd) as mutation:
    mutation.project.close(bead_ids, reason=reason, resolution="canceled")
    ...
    mutation.commit(close_mutation_commit_message(...))
```

`resolution="canceled"` matches the TaskTriage close path and
`sase/memory/sase_beads.md`: these beads are being abandoned, not completed. `reason` is
the reviewer's note when present, else `BEAD_STALE_CLEANUP_CLOSE_REASON`.

Projects are processed in sorted order so a partial failure is deterministic. A project
that raises leaves the already committed projects closed and propagates a `GateError`;
the executor records it through `record_execution_error` and the gate's next tick
re-offers whatever is still stale. Per `sase/memory/sase_beads.md` closing never
cascades, so a bead with an unclosed descendant is refused by the store — surface that
error rather than forcing it, since a task bead with children is not backlog noise.

**`src/sase/notification_gates/adapters.py`** — an `apply_side_effects` branch for
`bead_stale_cleanup` that translates and calls `close_bead_stale_cleanup`, placed beside
the `flag_triage` branch.

## Symvision

Remove the `translate_bead_stale_cleanup_response` epic entry the `gate` phase added;
the adapter is now its consumer. `close_bead_stale_cleanup` is consumed in the same
phase.

## Tests

`tests/test_bead/test_stale_cleanup_gate_actions.py`:

- Answering `close` with the default selection closes every offered bead, one commit per
  project, all with resolution `canceled`.
- Unselecting beads closes exactly the remainder.
- The reviewer's note becomes the close reason; with no note the default reason is used.
- A response naming an index outside the roster, a duplicate index, or an empty list
  raises `GateError` and closes nothing.
- A response for a project whose checkout cannot be resolved raises `GateError` naming
  that project, and the other project's closes still committed (sorted-order
  determinism).
- Beads spanning two projects produce two independent mutations.

---

# Phase `chop`: bead_stale_cleanup chop

## Shared project inventory

`enabled_project_stores`, `coerce_project_inventory`, `ProjectInventory`, and
`project_display_name` currently live in `src/sase/scripts/_bead_task_triage_state.py`
and are needed verbatim by the second chop. **Move** them to
`src/sase/scripts/_bead_gate_projects.py` and import them from both chops — do not
re-export them from the old module, which would leave a symbol with no direct consumer.

`enabled_project_stores` hardcodes a `[bead_task_triage]` log prefix; give it a
`chop: str` keyword so each chop labels its own warnings.
`sase_chop_bead_task_triage.py` keeps its module-level `_enabled_project_stores` binding
so the existing test doubles that patch it are unaffected.

## The chop

**`src/sase/scripts/sase_chop_bead_stale_cleanup.py`** —
`@builtin_chop("bead_stale_cleanup")`, with
`_STATE_FILENAME = "bead_stale_cleanup.json"` and
`_LOCK_FILENAME = "bead_stale_cleanup.lock"` under `runtime.context.state_dir`, held
with `file_lock` for the whole pass exactly as `bead_task_triage` does.

Pass outline:

1. `runtime.context.dry_run` returns the summary with `reason="dry_run"` before any gate
   work.
2. Read the three thresholds once. Read the project inventory; an inventory failure
   returns `reason="project_inventory_unavailable"` without touching the pending gate,
   so a transient failure can never cancel a healthy gate.
3. Per project,
   `rust_beads.list_issues(beads_dir, statuses=[Status.READY], issue_types=[IssueType.TASK])`
   and keep the beads `stale_task_bead(...)` accepts, with
   `now = core_time.local_now()`. A project that raises is skipped and recorded; if any
   project was skipped the pass may create or leave a gate but must not cancel one,
   because the true roster is unknown.
4. If the total is below `stale_cleanup_min_beads`: cancel the tracked pending gate if
   there is one (`reason="stale_backlog_below_threshold"`), clear the state, and return
   `reason="below_stale_threshold"`.
5. Otherwise sort oldest-first with the `(project, bead_id)` tie-break, take the first
   `BEAD_STALE_CLEANUP_MAX_BEADS`, and record `omitted_count`. `log.info` the omitted
   count when non-zero.
6. Fingerprint the offered roster plus the three thresholds (`sha256` over canonical
   JSON; **not** `stale_as_of`). Compare with lane state: a pending gate with an equal
   fingerprint is left alone (`skipped`); a terminal or missing gate, or a changed
   fingerprint, cancels what is pending and creates a new gate with `generation + 1`.
7. `request_id` is `f"bead-stale-cleanup-{digest}-g{generation}"`, where `digest` is the
   first 12 hex characters of the roster digest. Producer is
   `{"chop": "bead_stale_cleanup"}`.

Summary fields: `gated`, `canceled`, `skipped`, `stale`, `offered`, `omitted`,
`projects`, plus `reason`. Every filesystem or gate call is wrapped the way
`bead_task_triage` wraps its own — a failure warns and retries on the next tick, never
raises out of the chop.

**`src/sase/default_config.yml`** — register the chop in the `housekeeping` lumberjack
(hourly) with `timeout: "2m"` and a description in the established two-part form: a
one-line summary, a blank line, then a body naming the thresholds, the roster cap, the
single-gate-at-a-time invariant, and that it cancels its gate once the backlog drops
below the bar. Hourly is the right lane: the input changes on a scale of days, and the
`checks` lane is for latency-sensitive work.

**`pyproject.toml`** —
`sase_chop_bead_stale_cleanup = "sase.scripts.sase_chop_bead_stale_cleanup:main"` in
`[project.scripts]`, keeping the block's sorted order.

**`Justfile`** — delete every `--epic-symbol` entry this epic added; all of them now
have real consumers.

**Docs.** `docs/axe.md`: a row in the chop table and a `housekeeping` subsection
describing the chop. `docs/notifications.md`: `bead_stale_cleanup` / `BeadStaleCleanup`
in the gate-kind and action tables, the priority list, and the Beads panel note.
`docs/beads.md`: the stale-cleanup half of the triage lifecycle.

## Tests

`tests/test_axe_chop_bead_stale_cleanup.py` (new, with helpers modeled on
`_axe_chop_bead_task_triage_helpers.py`):

- Below `stale_cleanup_min_beads` no gate is created; at exactly the threshold one is.
- Beads younger than `stale_after_days`, beads at or above the `+1` bar, snoozed beads,
  `open` beads, and flag beads are never counted.
- A roster larger than the cap offers the 50 oldest and reports the rest in `omitted`.
- A second pass with an unchanged roster leaves the pending gate alone and creates
  nothing.
- A changed roster cancels the pending gate and creates one with the next generation.
- A backlog that drops below the threshold cancels the pending gate and clears the
  state.
- A project whose store read fails is skipped, is reported, and does not cancel a
  healthy pending gate.
- An inventory failure returns `project_inventory_unavailable` and touches nothing.
- `dry_run` creates and cancels nothing.
- Stale beads from two projects appear in one gate, ordered oldest-first with the
  documented tie-break.

`tests/test_axe_lumberjack_config.py` (extend): the new chop is present in
`housekeeping` with its script name and timeout. Add the console-script assertion to
whichever existing test covers `[project.scripts]` chop entries.

---

# Phase `polish`: Documentation sweep and full verification

## Deliverables

Read the four guides end to end as a set — `docs/configuration.md`, `docs/axe.md`,
`docs/notifications.md`, `docs/beads.md` — and reconcile them against the landed code
rather than against this plan. Specifically confirm:

- The `bead.task_triage` fields, their defaults, their floors, and the
  `min_plus_ones: 0` escape hatch are stated once, authoritatively, in
  `docs/configuration.md`, and referenced rather than restated elsewhere.
- The `bead_task_triage` narrative says a ready task bead needs the `+1` bar, that
  suppression withholds the gate without changing bead state, and that a gate already
  raised below the bar is canceled and its notification dismissed.
- `bead_stale_cleanup` appears in the chop table, the housekeeping lane description, the
  gate-kind table, the action table, and the priority-notification list, and nowhere
  claims the roster is unbounded.
- No document shows a ProjectSpec key where a project name belongs.

Confirm the `Justfile` carries no `--epic-symbol` entry added by this epic.

## Rollout

State in `docs/notifications.md`, in the `bead_task_triage` section, the one fact a
reader needs after upgrading: gates already raised for beads below the new bar are
canceled and their notifications dismissed automatically on the first `checks`-lane tick
after the upgrade, and `sase axe chop run bead_task_triage` forces that tick
immediately. Note the same for `sase axe chop run bead_stale_cleanup`, which is how a
reviewer can see the cleanup gate without waiting for the hour.

## Verification

Run `just check-full` through `/sase_monitor` — it routinely outruns one agent turn, so
it must not be run inline — and hand it a `--next` action so the follow-up agent acts on
the result. Land the phase only on a green run. Also run `just docs-check`, since this
epic edits four guides.
