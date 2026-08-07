---
tier: tale
title: Render a bead's Notes section only when the bead has notes
goal:
  Every bead surface renders a `Notes` section only when the bead actually has notes, no surface emits a `_No notes._`
  placeholder, and gate validation still accepts both the new note-less shape and every preview already persisted in the
  old shape.
proposed_by: bbugyi200.athena.un
create_time: 2026-08-07 09:38:32
status: done
---

- **PROMPT:**
  [prompts/202608/empty_bead_notes_section.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/empty_bead_notes_section.md)

# Plan: Render a bead's Notes section only when the bead has notes

## Objective

A task bead with no notes currently renders an empty `## Notes` section whose only content is the placeholder
`_No notes._`. Remove that section entirely when a bead's notes are blank, on every surface that renders one, and keep
the TaskTriage/BeadSnooze gate contract sound while doing it.

## Verified starting state

Measured in this workspace at `c37e68f7a`; re-verify before changing anything.

### The one surface that actually emits the placeholder

`src/sase/bead/_task_gate_preview.py:50,72` is the only producer of `_No notes._` in the repo:

```python
notes_text = notes.strip() or "_No notes._"
...
f"## Description\n\n{description_text}\n\n"
f"## Notes\n\n{notes_text}\n"
f"{evidence_section}"
```

It backs both gate previews:

- `render_task_triage_preview` → `src/sase/bead/_task_gate_spec.py:156` (`task.md` in every TaskTriage bundle). This is
  what the reported screenshot shows.
- `render_bead_snooze_preview` → `src/sase/bead/snooze_gate.py:286`, which prefixes a snooze callout and then delegates
  to the same function, so the BeadSnooze wake gate inherits the same empty section.

### The surfaces that are already conditional

These were checked directly and do **not** emit a placeholder. They are guarded on truthiness rather than on `.strip()`,
so a whitespace-only notes value still produces a visually empty section; that is the only defect left on them.

| Surface                 | Site                                                                   | Current guard                           |
| ----------------------- | ---------------------------------------------------------------------- | --------------------------------------- |
| `sase bead show`        | `src/sase/bead/cli_detail.py:302`                                      | `if issue.notes:`                       |
| Published bead pages    | `src/sase/bead_pages/rendering_identity.py:110-113`                    | `if not value: continue`                |
| ACE bead detail body    | `src/sase/ace/tui/widgets/artifacts/beads_detail.py:154`               | `if issue.notes:`                       |
| ACE agent BEAD lane     | `src/sase/ace/tui/widgets/prompt_panel/_agent_bead_section.py:107,122` | `.strip()` — already correct            |
| Mobile bead detail wire | `src/sase/integrations/_mobile_helper_beads.py:301`                    | `issue.notes or None` — already correct |

`sase bead show sase-10` (a bead with no notes) was run and prints no `NOTES` section today. No bead in
`sase/repos/beads/issues.jsonl` currently holds whitespace-only notes, so the `.strip()` tightening is hardening, not a
live bug fix.

### Why the renderer cannot be changed alone

`src/sase/notification_gates/kind_validation/task_triage.py:182-248` and
`src/sase/notification_gates/kind_validation/bead_snooze.py:180-236` are near-identical copies of one algorithm. The
gate payload does not carry the agent-authored description and notes, so validation recovers them from the persisted
preview:

1. Render a template with `__TASK_TRIAGE_DESCRIPTION__` / `__TASK_TRIAGE_NOTES__` (or the `__BEAD_SNOOZE_*__` pair) in
   place of the two free-form fields.
2. Partition the template on those markers to obtain a prefix, an inter-field separator (`\n\n## Notes\n\n`), and a
   suffix.
3. Slice the persisted preview with the prefix and suffix, `partition` the remaining body on the separator, re-render
   with the recovered halves, and compare byte for byte. **If the separator is absent, `expected_preview` is set to
   `None` and the gate is rejected.**

That last clause is the blocker: once a blank-notes preview stops containing `\n\n## Notes\n\n`, the recovery finds no
separator and every such gate fails validation with `invalid_task_triage_preview`.

Validation runs from `create_gate` (`src/sase/notification_gates/service.py:67`) on the incoming spec, which includes
the untrusted spec an agent can hand to `sase gate create` (`src/sase/main/gate_handler.py:54`). Its purpose is to prove
the human-facing preview is a possible output of the trusted renderer, so it must keep rejecting injected structure.

### Backward-compatibility result (measured, not assumed)

`~/.sase/interaction_requests/task_triage/` holds 297 bundles, 238 of which carry `_No notes._`. A prototype of the
proposed renderer plus the proposed two-candidate recovery was run against **all 297 persisted previews** and against
the current renderer as a control:

```
persisted previews checked: 297; validate under proposed scheme: 278
current-renderer validation: {True: 278, False: 19}
```

Identical accept/reject sets. The 19 rejects are an artifact of the prototype's payload reconstruction (it hardcodes
`CloseRecord.resolution=None`), not of either scheme; they are the same 19 `sase-ct` generations under both. The reason
legacy bundles keep validating is that `_No notes._` is recovered as if it were ordinary notes text, and re-rendering
non-blank notes still emits the section — so the old bytes reproduce exactly. **No bundle migration is required.**

A synthetic round-trip battery over `{empty, present, whitespace-only, multi-line}` notes ×
`{empty, present, separator-embedding}` descriptions × `{no evidence, evidence, size+created_at}` also passed, including
the case where a description itself contains `\n\n## Notes\n\n  indented` — which the _current_ single-candidate
recovery rejects. The new fallback candidate fixes that pre-existing false negative as a side effect.

### How pending gates refresh

`src/sase/scripts/sase_chop_bead_task_triage.py` reconciles gates every five minutes. `_presentation_fingerprint(issue)`
(line 218) hashes the bead fields that feed the presentation; when it differs from the value in
`~/.sase/axe/lumberjacks/checks/bead_task_triage.json`, the chop cancels the pending gate and creates a replacement at
the next generation (lines 480-513, 533-556). It hashes renderer _inputs_, not renderer _output_, so a format change
alone does not trigger it. The state file currently tracks **19 live gates** (8 + 11 across two project entries); this
is the entire blast radius of a deliberate refresh. `docs/notifications.md:249-251` already documents this path: "If a
gate becomes terminal, disappears, or uses an obsolete presentation contract while still expected, the next five-minute
scan creates a replacement with a new generation-specific request ID."

## Implementation

### 1. Omit the section in the gate preview renderer

In `src/sase/bead/_task_gate_preview.py`, change `render_task_triage_preview` so blank notes drop the whole section. Pin
these exact bytes (verified against a prototype):

- notes present: `…## Description\n\n{description}\n\n## Notes\n\n{notes}\n{evidence}` — unchanged from today.
- notes blank: `…## Description\n\n{description}\n{evidence}`.

Concretely, keep `description_text` as-is, replace the placeholder fallback with `notes_text = notes.strip()`, build
`notes_section = f"\n\n## Notes\n\n{notes_text}" if notes_text else ""`, and end the description line with a single `\n`
that follows `notes_section`. Because `_task_triage_evidence_preview` already returns a string beginning with `\n`, the
blank line before `## +1 Evidence` is preserved in both shapes. Update the module docstring/function docstring to say
the notes section is conditional, and keep the existing rule that every rendered field stays a pure function of
persisted values.

`render_bead_snooze_preview` needs no change — it delegates, so it inherits the new shape.

### 2. Teach both gate validators the note-less shape

Both validators must accept a preview that reproduces under **either** recovery, still comparing byte for byte:

- candidate A (today's behaviour): the body contains the separator → `description, notes = body.partition(sep)`.
- candidate B (new): treat the whole body as the description and the notes as `""`.

Try A first when the separator is present, then B; accept on the first candidate whose re-render equals the preview
content; otherwise raise the existing `GateError` with its existing code and message. Keep the prefix/suffix guard and
the `marker`/`marker_two` sanity checks exactly as they are — they are what stops an injected preview from passing.

The two implementations are duplicates today and will drift if edited twice. Extract the shared algorithm into a new
`src/sase/notification_gates/kind_validation/preview_recovery.py` alongside the existing shared `resources.py`,
exporting one public helper — for example:

```python
def preview_matches_renderer(
    *,
    render: Callable[[str, str], str],
    description_marker: str,
    notes_marker: str,
    preview_content: str | None,
) -> bool:
```

`bead_snooze.py` already has exactly this `render(description, notes)` closure shape; give `task_triage.py` the same
closure so both call sites reduce to building the closure and raising on `False`. Keeping the helper public avoids a
Symvision private-symbol-across-modules failure — re-read `sase/memory/symvision.md` through `/sase_memory_read` if that
gate complains.

### 3. Tighten the already-conditional surfaces to `.strip()`

Guard on stripped content so whitespace-only notes are treated as no notes, matching `_agent_bead_section.py` and
`normalize_bead_notes`:

- `src/sase/bead/cli_detail.py:302` → `if issue.notes.strip():`
- `src/sase/bead_pages/rendering_identity.py:110-113` → skip when `value.strip()` is empty (applies to `Description`
  too; that is a guard change only and cannot introduce a placeholder).
- `src/sase/ace/tui/widgets/artifacts/beads_detail.py:154` → `if issue.notes.strip():`

Do **not** touch `bead_body_markdown`'s `_No description._` fallback or `render_task_triage_preview`'s
`_No description._`. Descriptions are a required field in practice and the request is scoped to notes; changing them
would alter the description half of the same preview contract for no requested benefit.

### 4. Refresh the pending gates that still advertise the old preview

Without this, the 19 live gates keep showing `_No notes._` until their bead changes or they are triaged — including the
one in the report. Add a module-level constant to `src/sase/scripts/sase_chop_bead_task_triage.py` and fold it into the
hashed payload in `_presentation_fingerprint`:

```python
# Bumped whenever a gate preview or notification-note renderer changes shape, so
# the reconciler replaces pending gates still advertising the superseded one.
_PRESENTATION_FORMAT_VERSION = 2
```

Extend `_presentation_fingerprint`'s docstring to explain the constant, so a future renderer change has an obvious place
to hook. Accept the one-time cost and state it in the commit body: on the first chop tick after this lands, all
currently tracked gates are cancelled with reason `task_triage_presentation_changed` and recreated one generation
higher. `BeadSnooze` replacements are born snoozed to the bead's wake time (`presentation.snooze_until`), so deferred
beads do not resurface early; a manually muted `TaskTriage` notification will come back unread, which is the accepted
trade for fixing the user's live inbox.

### 5. Documentation

- `docs/notifications.md:234-236` — the Task Triage Notification paragraph says the **Filed by** line sits "above the
  task's description and notes". Note that the notes section is present only when the bead has notes.
- `docs/notifications.md:722-723` — "The gate preview is generated from the bead's title, description, and notes." Same
  clarification.
- `docs/beads.md:1087` — the `sase bead show` wrapping paragraph names `DESCRIPTION`, `NOTES`, and `+1 EVIDENCE`
  sections; make it explicit that `NOTES` appears only when the bead has notes.

No `CHANGELOG.md` edit: it is generated by release-please from conventional commits. Use a `fix(bead):` commit whose
subject names the behaviour (for example, `fix(bead): render a bead's Notes section only when it has notes`) and whose
body records the one-time pending-gate refresh.

## Tests

Add to `tests/test_bead/test_task_gate.py`:

- Blank notes render no `## Notes` heading and no `_No notes._`, and the preview ends `## Description\n\n{desc}\n`.
- Whitespace-only notes (`"   \n  "`) behave identically to `""`.
- Non-blank notes still render the section, byte-identical to today.
- Blank notes plus `+1` evidence keep exactly one blank line before `## +1 Evidence`.

Add to `tests/test_bead/test_snooze_gate.py`: the same blank-notes assertion through `render_bead_snooze_preview`, with
the snooze callout still first.

Gate validation is already covered in those same two modules, not in `tests/test_notification_gates.py`:
`test_task_triage_kind_validation_rejects_forged_contracts` (`tests/test_bead/test_task_gate.py:350-414`) and its
BeadSnooze twin drive `create_gate` through a `pytest.mark.parametrize` list of `(mutation, expected_code)` pairs. Reuse
that shape. Add:

- A freshly built blank-notes TaskTriage spec validates (positive case, so it does not belong in the parametrize list).
- A freshly built blank-notes BeadSnooze spec validates.
- **Legacy regression:** a spec whose `task.md` resource content is replaced with the old
  `…## Description\n\nD\n\n## Notes\n\n_No notes._\n` text still validates, proving the 238 persisted bundles are
  unaffected.
- **Injection regressions**, as new entries in the existing parametrize list, both expecting
  `invalid_task_triage_preview`: a blank-notes spec whose preview gains an appended heading after the evidence section,
  and one whose preview `**Size:**` line contradicts the payload. These are what prove the new fallback candidate did
  not weaken the contract — add them and watch them fail against a fallback-only implementation before restoring the
  prefix/suffix guard.
- A description containing `\n\n## Notes\n\n` with blank notes validates (the pre-existing false negative the fallback
  fixes).

Also check `tests/test_axe_chop_bead_task_triage_presentation.py` for fingerprint assertions that pin the hashed field
set, and update them for `_PRESENTATION_FORMAT_VERSION`.

Existing suites to re-run rather than rewrite: `tests/test_bead/test_bead_page_rendering.py` (asserts `## Notes` for a
bead that has notes, so it should stay green), `tests/test_bead_time_surface_coverage.py`, `tests/test_bead/` show
tests, and `tests/ace/tui/visual/test_ace_png_snapshots_notification_beads.py` — its task-triage fixture uses
`notes="Discovered while landing sase-cw."`, so the goldens should not move. If a PNG snapshot does shift, treat it as a
signal to re-read the change, not as something to accept blindly with `--sase-update-visual-snapshots`.

## Verification

1. `just install` first — this is an ephemeral `sase_<N>` workspace.
2. `just check` while iterating.
3. `just check-full` and `just test-visual` before landing: this touches gate validation, three UI surfaces, and the
   reconciler, which is well past what the scoped test lane is meant to backstop.
4. Re-run the persisted-bundle parity check as an acceptance gate: iterate every bundle under
   `~/.sase/interaction_requests/task_triage/` and `~/.sase/interaction_requests/bead_snooze/`, run each `task.md` /
   preview through the shipped validator's recovery path, and confirm the accept set is identical to the pre-change
   accept set (expect 278/297 on the task-triage side in this environment). This is a scratch script outside the repo;
   delete it afterwards and record the counts in the commit body.
5. `sase bead show <bead-with-notes>` and `sase bead show sase-10` (no notes) — one section, one omission, no
   placeholder.
6. After the change is live, confirm on the next chop tick that the replacement gate for a blank-notes ready task
   renders no `## Notes` section in its `task.md`, and that the ACE Beads-panel notification preview matches.

## Non-goals

- The `_No description._` placeholder stays.
- No migration or rewrite of already-persisted gate bundles; they are proven to keep validating, and the reconciler
  replaces the live ones.
- No change to the bead store, the `notes` field, `sase bead note`, or the bead editor modal.
- Replacing `_presentation_fingerprint`'s input hash with a hash of the actually-rendered preview and notification note
  would make every future format change self-refreshing without a constant to remember. That is a real improvement but a
  separate change; if it still looks worthwhile after this lands, file it with `/sase_new_task` rather than folding it
  in here.

## Risks

- **Weakening gate validation.** The added fallback accepts a preview whose body contains no notes separator, so for
  that shape the contract reduces to "prefix and suffix match, and the body re-renders exactly". That is the same
  strength the description half already has — the body is free-form text the renderer emits verbatim — but write the
  injection regression test first and confirm it fails against the fallback-only implementation before adding the
  prefix/suffix guard back.
- **Missing a fourth surface.** The audit above covers every `## Notes` / `NOTES` producer found in `src/`. Re-run
  `grep -rn "Notes" --include=*.py src/sase/` after implementing and confirm the only remaining hits are the unrelated
  `Notes` panels in `src/sase/memory/cli_list.py:138` and `src/sase/skills/cli_list.py:176`, the bead editor label, and
  the `_agent_bead_section.py` field-label tuple.
- **Notification churn.** Step 4 replaces 19 live gates once. If that is judged too disruptive at review time, drop step
  4 alone: the rest of the plan still lands and the stale previews drain as beads change or get triaged.
