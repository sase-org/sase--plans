---
tier: epic
title: A task bead's type is legible on every gate notification surface
goal:
  A typed task bead's `task_type` is visible, correct, and identically styled everywhere
  its triage or wake gate appears — the ACE toast, the notification row, the gate detail
  pane, the gate review modal, the Markdown preview, and the mobile wire — carried as
  presentation frozen into the gate at creation time, so no render path reads the
  task-type registry and no surface can disagree with another.
phases:
  - id: chip
    title: A gate may declare one subject chip
    depends_on: []
    size: medium
    description:
      "chip: add the generic `presentation.chip` field (glyph, label, optional color),
      normalize and protect it like `panel_icon`, project it into notification
      `action_data`, and add the tolerant zero-I/O reader every render surface uses."
  - id: freeze
    title: Frozen task-type presentation
    depends_on:
      - chip
    size: medium
    description:
      "freeze: add the module that resolves one task type into a frozen glyph, human
      name, accent, and ordered required-field facts at gate-creation time, plus the
      strict parser, chip projection, note line, and Markdown fact every later phase
      renders from."
  - id: dense
    title: The toast and the notification row
    depends_on:
      - chip
    size: medium
    description:
      "dense: give TaskTriage and BeadSnooze their own toast branch with the chip, the
      typed detail line, and warning severity, and render the chip glyph on notification
      rows."
  - id: detail
    title: The gate detail pane and the gate review modal
    depends_on:
      - chip
    size: medium
    description:
      "detail: render the declared chip as its own row in the notification modal's gate
      pane and in the custom gate review modal's header, from `action_data` only."
  - id: gates
    title: Task bead gates declare their type
    depends_on:
      - chip
      - freeze
    size: medium
    description:
      "gates: freeze the task-type display block into the TaskTriage and BeadSnooze
      payloads, declare the chip, the typed note line, and the type tag, add the `**Task
      type:**` preview fact, and rebuild the whole presentation in kind validation
      instead of hand-checking its fields."
  - id: refresh
    title: A pending gate refreshes when its type presentation changes
    depends_on:
      - gates
    size: small
    description:
      "refresh: fold the frozen display block into the reconciler's presentation
      fingerprint and bump the presentation format version so pending gates carrying the
      old, typeless presentation are replaced."
  - id: prove
    title: Prove it end to end and document it
    depends_on:
      - dense
      - detail
      - gates
      - refresh
    size: medium
    description:
      "prove: add the one end-to-end test that drives a real typed gate through every
      surface, refresh and extend the PNG snapshots, and document the chip field and the
      typed gate surfaces."
proposed_by: bbugyi200.athena.060
status: done
bead_id: sase-pq
create_time: 2026-09-09 19:51:48
---

- **PROMPT:**
  [prompts/202608/task_type_gate_presentation.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/task_type_gate_presentation.md)
- **BEAD:**
  [sase-pq](https://github.com/sase-org/sase--beads/blob/main/pages/sase-pq/README.md)

# Plan: A task bead's type is legible on every gate notification surface

`task_type` reaches a task bead's gates today by exactly one route: the preview
Markdown's typed body block, rendered below **Notes**, several screens down a scrollable
document. Every other gate surface is type-blind. The toast has no `TaskTriage` branch
at all and falls through to the generic "print `notes[0]`" branch. The notification row
shows the bead's `✦` gate icon and nothing about what kind of work it is. The gate
detail pane and the review modal show a headline and a note. A reviewer deciding
**Launch** / **Close** / **Snooze** — where **Launch** is the default branch — cannot
see whether they are looking at a confirmed CI failure, a flake that wants more
evidence, or a feature proposal, without opening the gate and scrolling.

This epic makes the type a first-class part of what a bead gate _is_, at every viewing
distance, without letting any render path read the task-type registry.

## Why this shape

Three constraints dictate the cut, and each is load-bearing.

**Render paths do no I/O and cannot fail.** `_toasts.py` says so in its own module
docstring, and it is why plan toasts read `plan_tier` and the phase/wave counts out of
`Notification.action_data` rather than recomputing them. `task_type_presentation()`
cannot be called there: it consults the live registry (plugin discovery, merged config)
and then the committed snapshot file. So the type's _resolved_ presentation has to be
computed once, at gate creation, and travel with the gate. That is the `freeze` phase,
and it is why it comes before `gates`.

**A gate's persisted bytes are rebuilt and compared.** Kind validation reconstructs the
notification note and the preview byte for byte from the payload. Anything a task gate
renders must therefore be a pure function of persisted payload fields — which the _live_
resolution of a glyph and an accent colour is not, the moment a plugin is installed,
upgraded, or removed. Freezing the resolution into the payload is what makes the note,
the chip, and the preview line reconstructible at all. It also buys the honest
degradation this feature needs: a gate filed when a plugin was installed keeps naming
its type correctly after the plugin is gone.

**The surfaces that need the chip are generic.** The toast formatter, the notification
row builder, the gate detail pane, and `CustomGateModal` serve every gate kind. Teaching
four generic renderers about `task_type` specifically would put a bead-shaped bulge in
the gate contract. The repo has already solved this exact problem three times —
`presentation.panel`, `presentation.panel_icon`, `presentation.origin_agent`, and
`presentation.title` are all sender-declared presentation fields, normalized in one
place, projected into `action_data`, protected against forgery, and rendered
generically. `presentation.chip` is the fourth instance of that established pattern, and
task bead gates become one caller of it. That is the `chip` phase, and it is why it has
no dependencies and everything else has it.

```text
chip ──┬── freeze ── gates ── refresh ──┐
       ├── dense ─────────────────────  ┤
       └── detail ────────────────────  ┴── prove
```

`dense` and `detail` render a chip that only `gates` populates. They still land
independently: both read `action_data` and both are tested against directly constructed
notifications, exactly as the existing PNG snapshot fixtures already build them.

## Grounding

Verified in this workspace at `2c6050e24`. Line numbers are from that tree.

| Fact                                                                 | Evidence                                                                                                |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `TaskTriage` has no toast branch and falls through to the default    | `_format_notification_toast` ends at the "Tmux, None, or unknown actions" branch (`_toasts.py:245-246`) |
| Plan toasts already read frozen fields from `action_data`            | `_resolve_plan_tier`, `decode_plan_counts(n.action_data)` (`_toasts.py:96-146`)                         |
| The toast module is declared zero-I/O and infallible                 | module docstring (`_toasts.py:1-14`)                                                                    |
| `task_type_presentation()` reads the registry, then a file           | `get_task_type_registry()`, `task_type_snapshot_entry()` (`task_type_presentation.py:80-98`)            |
| Notification rows show only the action icon and `notes[0]`           | `_create_styled_label` (`notification_modal_options.py:85-100`)                                         |
| The gate detail pane renders status, meta, notes, tags, decision     | `_render_gate_pane` (`notification_modal_gate.py:69-83`), `_context_group` (`:204-222`)                 |
| The review modal header carries icon, headline, sender, id           | `CustomGateModal._title` (`custom_gate_modal.py:265-274`)                                               |
| Presentation fields are declared, normalized, and projected          | `panel`, `panel_icon`, `origin_agent`, `gate_title` (`presentation.py`, `service.py:344-357`)           |
| Reserved `action_data` keys are protected from producers             | `protected` set (`validation.py:147-165`)                                                               |
| Unknown `presentation` keys are ignored, so the field is additive    | `validate_gate_spec` reads named keys only (`validation.py:92-200`)                                     |
| Both bead gates already carry `task_type` in their payload           | `_task_gate_spec.py:135-136`, `snooze_gate.py:198-199`                                                  |
| The payload contract is a closed field set with an optional one      | `_TASK_TRIAGE_PAYLOAD_FIELDS` / `_LEGACY_...` (`task_triage_payload.py:15-31`)                          |
| Presentation validation hand-checks each field instead of rebuilding | `_validate_task_triage_presentation` (`task_triage.py:141-183`), `bead_snooze.py:144-170`               |
| Option validation already rebuilds and compares whole                | `_validate_task_triage_options` docstring (`task_triage.py:57-64`)                                      |
| The preview renders type only as a body block, below Notes           | `_task_type_body`, `type_section` (`_task_gate_preview.py:93-94,128-141`)                               |
| The preview's metadata block is where identity facts already go      | `**Size:**`, `**References:**` (`_task_gate_preview.py:104-108`)                                        |
| Bead pages already render `**Task type:** <glyph> <slug>`            | `_primary_facts` (`bead_pages/rendering_identity.py:264-268`)                                           |
| Dense bead rows already render the bare glyph in the accent          | `_bead_text` (`ace/tui/widgets/artifacts/beads_rendering.py:394-396`)                                   |
| The reconciler fingerprint omits `task_type` entirely                | `presentation_fingerprint` payload (`scripts/_bead_task_triage_gates.py:69-113`)                        |
| `task_type` and `task_type_fields` are immutable after creation      | `_validate_issue_update` (`bead/_project_mutations.py:510-515`)                                         |
| The notification tab styler already has a task-type rung             | `_IconRung.TASK_TYPE`, `_task_type_tab_glyph_and_color` (`notification_tab_style.py:120,205-217`)       |
| A declared `panel` outranks tags, so a type tag never splits tabs    | `_notification_modal_tab_key` (`notification_modal_tags.py:105-129`)                                    |
| Both bead gate kinds render through the generic review modal         | `generic_form=True` (`adapters.py:387-406`)                                                             |
| Kind validation runs only at gate creation                           | sole caller is `create_gate` (`service.py:68`); `load_and_verify_bundle` checks hashes only             |
| PNG snapshots already cover both surfaces                            | `custom_gate_task_triage_120x40.png`, `notification_beads_tab_120x40.png`                               |

## Decisions a phase worker must not silently revert

**1. No render path resolves a task type.** After `freeze`, the only call to
`get_task_type_registry()` on this feature's paths is inside gate creation. A toast, a
row, a pane, or a modal that imports `sase.task_type_presentation` has taken the wrong
route; it reads `action_data`. The one exception already in the tree —
`notification_tab_style._task_type_tab_glyph_and_color` — is a tab styler, not a gate
surface, and is out of scope.

**2. The chip is generic; only its producer is bead-shaped.** `presentation.chip` names
a glyph, a label, and an optional colour. It never names a task type, a bead, or a slug
vocabulary. If a phase worker finds themselves writing `task_type` inside
`src/sase/notification_gates/` or inside a generic ACE gate renderer, the layering has
been inverted.

**3. Stored presentation is sanitized before it is styled.** The glyph and label reach
Rich markup in the toast and Rich styles everywhere else. Every reader validates the
colour against `#RRGGBB` and escapes `[` in the glyph and label before interpolation. A
malformed stored colour degrades to an uncoloured chip; it never raises and never
reaches a style string. This is why the reader is a normalizing function and not a
`dict.get`.

**4. Dense surfaces omit an untyped bead's chip entirely; detail surfaces say
`untyped`.** Absence of a chip on a toast or a row means "no type", which is what a
reader already infers from a bare bead row. The `·` untyped glyph is not rendered on
gate surfaces. An _unknown_ type — stored slug, no registered spec — is never silently
untyped: it freezes as the `?` glyph in the neutral grey, and the label stays the slug.

**5. The typed facts line is a note, not a new section.** The compact
`<name> · <Label>: <value>` line ships as `presentation.notes[1]`, so the gate detail
pane, the review modal, and the mobile bridge each render it through machinery they
already have. No phase adds a "Task type" section to a generic surface. `notes[0]` keeps
its current shape byte for byte, because the row and the toast still print it.

**6. The decision surface itself does not change.** Option ids, labels, icons, branches,
primary branch, feedback modes, and result schemas are the trusted contract and are
compared byte for byte at creation. Nothing in this epic makes an option type-aware.
Type-derived triage behaviour — a different default branch for a `ci` bead, a type's
`triage.min_plus_ones` shown on the gate — is deliberately out of scope.

**7. Sase memory files are untouched.** `sase/memory/*.md`, `AGENTS.md`, and the
generated provider shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`) are not
edited by any phase of this epic; no user permission for that exists. `docs/*.md` and
`src/sase/xprompts/skills/*.md` are ordinary files and are in scope. `CHANGELOG.md` is
release-please generated and is never hand-edited.

## Seam ownership

`dense`, `detail`, and `gates` run in parallel after `chip`. Ownership is assigned
rather than negotiated:

- `chip` owns `src/sase/notification_gates/presentation.py`, and the chip lines it adds
  to `service.py::_build_notification` and `validation.py::validate_gate_spec`.
- `freeze` owns the new `src/sase/task_type_gate_presentation.py` and the shared chip
  formatter it factors out of `src/sase/task_type_presentation.py`.
- `dense` owns `ace/tui/actions/agents/_toasts.py` and
  `ace/tui/modals/notification_modal_options.py`.
- `detail` owns `ace/tui/modals/notification_modal_gate.py`,
  `ace/tui/modals/custom_gate_modal.py`, and
  `ace/tui/actions/agents/_notification_custom_gate.py`.
- `gates` owns `bead/_task_gate_spec.py`, `bead/_task_gate_preview.py`,
  `bead/snooze_gate.py`, and the three `notification_gates/kind_validation/` modules for
  those kinds.
- `refresh` owns `scripts/_bead_task_triage_gates.py` and
  `scripts/sase_chop_bead_task_triage.py`.
- `prove` owns the PNG snapshot modules, `docs/notifications.md`, `docs/beads.md`, and
  `src/sase/xprompts/skills/sase_gate.md`.

## Phases

### chip — A gate may declare one subject chip

Add the presentation field the rest of the epic renders, following `panel_icon`'s
established shape exactly.

_The field._ `presentation.chip` is an optional object with `glyph` (required), `label`
(required), and `color` (optional). In `src/sase/notification_gates/presentation.py`,
add a frozen `GateChip` record and `normalize_gate_chip(value) -> GateChip | None`. The
glyph goes through the existing `validate_icon` so it is one grapheme; the label is
stripped, non-empty, single-line, free of control characters, and capped at 32
characters; the colour goes through `validate_color` so only `#RRGGBB` is stored. Every
rejection raises `GateError("invalid_presentation", "presentation.chip", ...)` with a
message naming the offending sub-field, matching how `_invalid_panel` and
`_invalid_title` read today.

_Projection._ Add `GATE_CHIP_GLYPH_ACTION_DATA_KEY`, `GATE_CHIP_LABEL_ACTION_DATA_KEY`,
and `GATE_CHIP_COLOR_ACTION_DATA_KEY` (`gate_chip_glyph`, `gate_chip_label`,
`gate_chip_color`). `_build_notification` normalizes the declared chip and writes the
keys it has; a chip with no colour writes two keys, not an empty third.
`validate_gate_spec` calls `normalize_gate_chip` alongside the other normalizers and
adds all three keys to its `protected` set so a producer cannot forge them through
`presentation.action_data`.

_The reader._ Add `gate_chip_from_action_data(action_data) -> GateChip | None`, the
zero-I/O tolerant counterpart every render surface calls. It never raises: a missing
glyph or label yields `None`, and a stored colour that is not `#RRGGBB` yields a chip
with `color=None` rather than dropping the chip. Model the tolerance on
`notification_modal_tags.notification_origin_agent`, and put the reader beside the
normalizer so the two definitions of a legal chip cannot drift.

_Tests._ In `tests/test_notification_gate_presentation.py`, cover each normalizer
rejection and the accepted forms. In `tests/test_notification_gates.py`, cover that a
declared chip lands in the created notification's `action_data`, that a colourless chip
writes no colour key, and that a producer writing `gate_chip_glyph` through
`presentation.action_data` fails with `reserved_action_data`. Cover the reader's
tolerance table directly, including stored junk in every position.

Exit condition: a gate spec declaring a chip creates a notification whose `action_data`
carries it, a forged chip key is rejected, and `gate_chip_from_action_data` returns a
usable chip or `None` for every input in the tolerance table without raising.

### freeze — Frozen task-type presentation

Resolve a task type into persistable presentation once, at gate-creation time.

_The record._ Add `src/sase/task_type_gate_presentation.py` with a frozen
`TaskTypeGateDisplay` carrying `glyph`, `name` (the human label, e.g. `Flaky test`),
`accent_color`, and `facts`: an ordered tuple of `(label, value)` pairs. Two entry
points, and the module docstring must say which is which:
`resolve_task_type_gate_display(task_type, task_type_fields)` is the creation-time
resolver that reads the registry, and `parse_task_type_gate_display(mapping)` is the
strict, zero-I/O parser validation and readers use.

_Resolution._ The resolver returns `None` for an untyped bead. For a typed one it reuses
`task_type_presentation()` so a gate's glyph and accent are the same ones the beads
pane, `sase bead show`, and bead pages already render — this module resolves and
freezes, it does not invent a second palette. `name` is the spec's `label`, falling back
to the slug for an unresolved type. `facts` are the type's **required** fields in spec
order, taking each field's declared `label` and the bead's stored value: values are
stripped, newlines collapsed to spaces, truncated to 80 cells with `…`, and a missing or
empty value drops the pair. Cap at three pairs. For an unresolved type there are no
declared labels, so the raw field names stand in as labels, exactly as
`task_types.body._degraded_unknown_block` already does.

_Projections._ Four pure functions, each taking the record:
`task_type_gate_display_payload(display)` → the JSON mapping stored in a gate payload;
`task_type_gate_chip(display, slug)` → the `presentation.chip` mapping, whose `label` is
the **slug** (`flake`), not the human name, because every other dense bead surface
prints the slug; `task_type_gate_note(display)` → the compact
`Flaky test · Test node ID: tests/x.py::test_y · Evidence: 3/50 under -n 8` line;
`task_type_gate_markdown_fact(display, slug)` → the ``**Task type:** ≈ `flake` `` line
for a Markdown metadata block, matching bead pages' `_primary_facts`.

_Strict parsing._ `parse_task_type_gate_display` accepts exactly
`{glyph, name, accent_color, facts}`, rejects anything else, and enforces the same
bounds the resolver produces — one grapheme, `#RRGGBB`, non-empty single-line name, at
most three two-element string pairs. It raises a `ValueError` the `gates` phase
translates into its kind-specific `GateError`; this module imports nothing from
`sase.notification_gates`.

_Shared formatting._ Both the frozen chip and the live `task_type_chip` must look
identical. Factor the shared formatter out of `src/sase/task_type_presentation.py` so
there is exactly one definition of how a task-type glyph and label are laid out and
styled, and have both call it. Do not copy the format string.

_Tests._ New `tests/test_task_type_gate_presentation.py`: resolution for each builtin
type, the untyped `None`, the unresolved-slug degradation, fact ordering, truncation,
the missing-value drop, the three-pair cap, each projection's exact output, the strict
parser's accept and reject table, and a round trip (`resolve → payload → parse` is the
identity).

Exit condition: `resolve_task_type_gate_display("flake", {...})` round-trips through its
payload projection, and no function in this module reads the registry except the
resolver.

### dense — The toast and the notification row

Make the type visible at a glance on the two surfaces that get about two seconds.

_The toast._ Give `TaskTriage` and `BeadSnooze` their own branch in
`_format_notification_toast`. The message is the chip followed by the existing
`notes[0]`, and when a second note is present it becomes a dimmed second line, exactly
as `_epic_detail_line` already does for epic plans:

```text
≈ flake  sase-cx \[+2] — Flaky: test_x fails only under the parallel suite · 2026-08-01
Flaky test · Test node ID: tests/x.py::test_y · Evidence: 3/50 under -n 8
```

The chip renders as `[bold {color}]{glyph} {label}[/]`, foreground-only. Do not use a
filled `on {color}` chip here or anywhere else in this epic: the neutral grey an
unresolved type freezes with is unreadable behind black text, and the foreground form is
what `beads_rendering`, `wait_modal_beads`, `bead_editor_modal`, and `sase bead show`
already use. A notification with no chip renders exactly what it renders today. Severity
becomes `warning` for both kinds — these are decision gates awaiting the user, like
`HITL` and `LaunchApproval` — which also moves them into the `warnings` bucket of the 4+
grouped batch path. Run `_markup_safe` over the glyph, the label, and the second line,
and truncate the second line to the module's existing `_MAX_NOTE_LEN`.

_The row._ In `_create_styled_label`, append `{glyph} ` styled `bold {color}`
immediately after the action icon, mirroring `_bead_text`'s type-glyph cell. Label and
facts do not appear on the row: it is already dense, and the tag chips the `gates` phase
adds carry the slug. A row whose notification declares no chip is byte-identical to
today's.

_Tests._ Extend `tests/test_notification_toasts.py` with a typed `TaskTriage`, a typed
`BeadSnooze`, an untyped bead gate, a chip whose colour is stored junk, a chip whose
glyph is `[`, and the severity change's effect on grouped batches. Add row-rendering
assertions where the existing notification-row tests live.

Exit condition: a typed bead gate toasts with its chip and its typed detail line at
`warning` severity, an untyped one toasts exactly as it does today, and no input in the
tolerance table raises on the render path.

### detail — The gate detail pane and the gate review modal

Give the two surfaces a reviewer actually decides from a stable place for the chip.

_The pane._ In `_render_gate_pane`, insert a chip row immediately after `_meta_row` and
before the blank line that precedes the context group, so the chip sits with the gate's
identity rather than inside its prose. Render the glyph and label as `bold {color}`.
When the notification declares no chip, the row is omitted entirely — not rendered
blank. The typed facts need no work here: they arrive as `notes[1]` and `_context_group`
already joins every note.

_The modal._ Add an optional `chip: GateChip | None` field to `CustomGateModalData`,
populated in `_load_custom_gate_modal_data` from `notification.action_data` through the
`chip` phase's reader. Render it in `_title` between the headline and the gate kind's
display title, escaped through `rich.markup.escape` like every other segment there. The
modal's `_notes()` already renders `notes[1]`, so the facts arrive with no change. The
field is optional with a `None` default so the existing fixtures that construct
`CustomGateModalData` keep working unchanged.

_Tests._ Cover the pane with and without a chip and the modal title with and without
one, in the existing notification-modal and custom-gate test modules. Assert the chip
row is absent — not empty — for a chipless gate.

Exit condition: a chip-declaring gate shows its chip in both surfaces, a chipless gate
renders exactly as it does today, and neither surface imports anything from
`sase.task_types`.

### gates — Task bead gates declare their type

Freeze the type into both bead gate kinds and make validation rebuild what it checks.

_Payload._ Add an optional `task_type_display` object to the shared task-bead payload,
written by `build_task_triage_gate_spec` and `_build_bead_snooze_gate_spec` from
`resolve_task_type_gate_display(task_type, task_type_fields)`. It is present exactly
when `task_type` is non-empty. In `task_triage_payload.py`, teach
`parse_task_bead_payload` an explicit _optional_ field set instead of the current pair
of hard-coded field-set literals — `closed_at` is already optional in the same way, and
enumerating four combinations does not scale — then validate the block through
`parse_task_type_gate_display`, translating its `ValueError` into
`GateError(code, "payload.task_type_display", ...)`. A block present without a
`task_type` is rejected, mirroring the existing `task_type_fields` rule.

_Presentation._ Both spec builders gain three things, each derived from the frozen block
alone: `presentation.chip` from `task_type_gate_chip`; a second `presentation.notes`
entry from `task_type_gate_note`; and the type slug appended to `presentation.tags`,
giving `["bead", "task", "flake"]`. The tag is display and query metadata only — a
declared `panel` outranks tags in tab classification, so it cannot split the `Beads` tab
— and it lights up the task-type rung `notification_tab_style` already carries.
`notes[0]` is unchanged, byte for byte. An untyped bead's presentation is unchanged in
all three respects.

Every remaining notes consumer inherits the typed line with no work of its own:
`sase notify show` prints each note, `sase notify list` joins them, the notification
catalog carries them, and the mobile bridge row ships them. That is the whole reason the
facts are a note and not a section — five more surfaces, one persisted string.

_Preview._ In `render_task_triage_preview`, add `task_type_gate_markdown_fact(...)` to
the metadata block beside `**Size:**` and `**References:**`. It must go there and
nowhere else: the block sits above `## Description`, outside the marker-delimited region
kind validation uses to recover the agent-authored description and notes, and the
renderer's docstring already explains why anything between those markers must stay a
pure function of the type fields. The typed body block below **Notes** is unchanged.

_Validation._ Replace the hand-written field comparisons in
`_validate_task_triage_presentation` and `_validate_bead_snooze_presentation` with a
rebuild-and-compare, the way `_validate_task_triage_options` already treats options.
Extract the presentation dict construction from each spec builder into a shared helper
those builders and the validators both call, thread `origin_agent` through it as the
input it already is, and compare the whole mapping. This is what keeps the chip, the
tag, and the second note from drifting between what a gate is created with and what is
accepted — and it is why the field must be frozen rather than resolved: the rebuild has
to produce the same bytes on a machine where the plugin is not installed.

_Tests._ Extend `tests/test_bead/test_task_gate.py`, `test_task_gate_snooze.py`,
`test_task_gate_preview.py`, and `test_task_gate_validation.py`: a typed gate's payload,
chip, notes, tags, and preview fact; an untyped gate unchanged; an unresolved slug
degrading honestly; a forged chip, a forged tag, a forged second note, and a mutated
`task_type_display` each failing validation with the kind-specific code; and the
preview's description/notes recovery still working with the new metadata line present
and with notes blank.

Exit condition: creating a typed `TaskTriage` and a typed `BeadSnooze` succeeds, every
tampered variant fails validation, and an untyped bead's gate is byte-identical to the
one this tree produces today.

### refresh — A pending gate refreshes when its type presentation changes

Make the reconciler notice when a gate's frozen type presentation has gone stale.

`presentation_fingerprint` in `scripts/_bead_task_triage_gates.py` hashes every
persisted field that changes a gate, and today it hashes no part of the type. Fold the
resolved display block into it — the block, not the raw `task_type`, because `task_type`
and `task_type_fields` are immutable after creation while the block is not: installing,
upgrading, or removing a plugin, or editing `bead.task_types` project config, changes a
pending gate's frozen glyph, name, accent, or fact labels. Hashing the block is also
what lets a gate filed before a plugin existed heal itself into a properly typed gate on
the next tick. Add it under a `task_type_display` key so no existing untyped bead's
fingerprint changes shape, resolve it through the registry the chop already loads at
`sase_chop_bead_task_triage.py:313` rather than a second lookup, and bump
`_PRESENTATION_FORMAT_VERSION` from 3 to 4 so every gate still advertising the typeless
presentation is replaced on the first tick after upgrade.

_Tests._ Extend `tests/test_axe_chop_bead_task_triage_presentation.py`: a
type-resolution change replaces a pending gate; an untyped bead's fingerprint is
unaffected by the new key's absence; and the version bump replaces gates built at
version 3.

Exit condition: changing a task type's registered glyph replaces that bead's pending
gate on the next reconciliation, and beads with no type see no churn.

### prove — Prove it end to end and document it

Show the whole path works, then write down the contract.

_One end-to-end test._ Add a test that creates a real typed `TaskTriage` gate through
`create_gate`, reads the notification the service appended, and drives that one
notification through every consumer: `format_batch_toasts`, the notification row
builder, the gate detail pane, `_load_custom_gate_modal_data`, and the mobile bridge row
projection. Assert the same glyph, label, and colour appear on all of them, and that the
preview file on disk carries the `**Task type:**` fact. This is the test that fails if a
later change reintroduces a surface that resolves the type itself, and it belongs in
`tests/` beside the existing gate-lifecycle tests rather than in any one surface's
module.

_Snapshots._ Refresh `custom_gate_task_triage_120x40.png` and
`notification_beads_tab_120x40.png`, which both change. Add one new snapshot of a
notification list holding several _differently typed_ bead gates, so the palette is
checked as a set and not one chip at a time; build it from the existing
`_task_notification` fixture in
`tests/ace/tui/visual/test_ace_png_snapshots_notification_beads.py` extended with chip
`action_data`. Accept the intentional changes with
`just test-visual --sase-update-visual-snapshots` and inspect
`.pytest_cache/sase-visual/` before accepting anything unexpected.

_Docs._ In `docs/notifications.md`: document `presentation.chip` in the gate contract
section beside `panel_icon` and `origin_agent`, including its projected `action_data`
keys and their protected status, and add the chip and the typed note line to the **Task
Triage Notification** and **Snoozed Task Notification** sections, whose current text
states the note is exactly `<bead-id> — <title>`. In `docs/beads.md`, extend the
`TaskTriage` gate preview description around line 582 with the `**Task type:**` fact. In
`src/sase/xprompts/skills/sase_gate.md`, document `chip` for gate authors — read
`sase/memory/generated_skills.md` with `/sase_memory_read` first, and follow whatever
regeneration step that note requires for a skill source edit. Do not edit any
`sase/memory/*.md` note, `AGENTS.md`, a generated provider shim, or `CHANGELOG.md`.

Exit condition: `just check-full` is green through `/sase_monitor`, the end-to-end test
asserts one consistent chip across five surfaces, and the notifications documentation
describes `presentation.chip` as a first-class gate presentation field.
