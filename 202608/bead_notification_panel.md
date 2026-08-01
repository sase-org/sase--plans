---
tier: epic
title: Beads notification panel and gate origin attribution
goal: 'Ready task beads arrive as compact `[bead]` rows in their own `Beads` notification
  tab, any SASE gate can declare the notification tab it lands in and the agent it
  was filed on behalf of, and the filing agent is visible in both the notification
  detail pane and the gate action panel.

  '
phases:
- id: contract
  title: Generic gate presentation panel and origin fields
  depends_on: []
  size: medium
  description: 'contract: add the `presentation.panel` and `presentation.origin_agent`
    gate request fields, their normalization and validation, their projection into
    notification `action_data`, the `GateAdapter.display_title` field, the matching
    `sase gate create` options, and the gate documentation and skill-source updates.

    '
- id: bead-gate
  title: Task triage gate identity, filer, and self-heal
  depends_on:
  - contract
  size: medium
  description: 'bead-gate: rename the task triage sender to `bead`, shorten its note
    to `<bead-id> — <title>`, route it to the `beads` panel, carry the bead''s filer
    through `origin_agent` and the Markdown preview, and cancel-and-recreate pending
    gates that still use the previous presentation contract.

    '
- id: inbox
  title: Panel tabs and filer line in the notification modal
  depends_on:
  - contract
  size: medium
  description: 'inbox: resolve notification modal tabs from the declared panel ahead
    of the synthetic HITL and Errors tabs, order and label panel tabs, and render
    the filing agent on the detail pane''s meta line.

    '
- id: action-panel
  title: Gate action panel title and filer line
  depends_on:
  - contract
  size: small
  description: 'action-panel: title the generic gate review modal from its adapter
    instead of the hardcoded "Custom Gate" string and render a "Filed by" line above
    the Context section when the gate declares an origin agent.

    '
- id: visuals
  title: PNG snapshot coverage and documentation sweep
  depends_on:
  - bead-gate
  - inbox
  - action-panel
  size: small
  description: 'visuals: add PNG snapshot goldens for the Beads tab, the filer meta
    line, and the task triage action panel, then reconcile the notification and bead
    documentation with the shipped behavior.'
proposed_by: bbugyi200.athena.qw
create_time: 2026-08-01 07:03:37
status: done
bead_id: sase-cz
---

- **PROMPT:** [202608/prompts/bead_notification_panel.md](prompts/bead_notification_panel.md)
- **BEAD:** [sase-cz](https://github.com/sase-org/sase--beads/blob/main/pages/sase-cz/README.md)

# Plan: Beads notification panel and gate origin attribution

## Problem

The ACE notification inbox currently renders every ready task bead as:

```
* ✦ [bead-task-triage] Task ready for triage: sase-cx — Flaky:
```

Three problems compound in that one row:

1. **The prefix eats the message.** `[bead-task-triage] Task ready for triage:` is 41 characters of boilerplate before
   the note is truncated at 50 characters, so a column of fourteen ready beads looks like fourteen identical rows and
   the actual bead titles are cut off mid-word.
2. **They drown the HITL tab.** `src/sase/ace/tui/modals/notification_modal_tags.py` routes every gate action in
   `_HITL_ACTIONS` — including `TaskTriage` — into the synthetic `HITL` tab, ignoring the gate's own tags. Fourteen
   triage rows bury a single waiting plan approval.
3. **The filer is invisible.** The bead store records `Issue.created_by`, but nothing carries it into the gate, so
   neither the inbox nor the gate review modal can tell you which agent filed the follow-up you are being asked to
   launch or close.

A fourth defect shows up once you open one of these rows: task triage gates are `generic_form=True`, so they render
through `CustomGateModal`, whose `_title()` hardcodes the string `Custom Gate`. Opening a task triage gate today shows
`✦ Custom Gate  bead-task-triage  bead-task-triage-sase-cx-…`.

## Design

### One generic mechanism, two new presentation fields

The user asked for a `Beads` tab and for the tab choice to be configurable for any SASE gate. Rather than special-casing
`TaskTriage`, this plan adds two optional, generic fields to the schema-version 3 gate presentation block, usable by
every gate kind including `/sase_gate` custom gates:

| Field                       | Meaning                                                         |
| --------------------------- | --------------------------------------------------------------- |
| `presentation.panel`        | The notification modal tab this gate's notification belongs to. |
| `presentation.origin_agent` | The agent this gate was filed on behalf of.                     |

Task triage then becomes an ordinary consumer: `"panel": "beads"` and `"origin_agent": "<bead.created_by>"`.

### Persistence: no Rust core change is required

`Notification.action_data` is already a free-form `dict[str, str]` that round-trips through the Rust notification store
(`src/sase/core/notification_store_wire.py` copies every key with `str(k): str(v)`), so both fields persist by
projecting them into `action_data` at gate-creation time. Do **not** add fields to the `Notification` dataclass; that
would require a matching wire change in `../sase-core` for a pure presentation concern.

For the same reason the projected keys are new and private: `panel` and `origin_agent`. **Never reuse `agent_name`,
`agent_cl_name`, `agent_timestamp`, or `agent_root_timestamp` for the filer.** Those keys are agent-routing fields
consumed by `src/sase/notifications/agent_matching.py`, and populating them would make a pending triage gate silently
auto-dismiss when the filing agent's row is cleaned up.

### Tab resolution and ordering

`_notification_modal_tab_keys` gains one rule, inserted between the muted check and the HITL check:

1. muted → `__muted__` (mute always wins, unchanged)
2. **declared panel → `[panel]`** (new)
3. HITL action → `hitl`
4. error → `errors`
5. tags → each tag
6. otherwise → `None` (General)

Panel tabs are actionable decision queues, so they sort with the other actionable tabs rather than among alphabetical
tag tabs. Target order:

```
 HITL 1 | Beads 14 | Errors 2 | General 3 | Done 19 | memory 1 | Muted 4
```

That is: `hitl`, then declared panel tabs sorted alphabetically, then `errors`, `General`, `Done`, remaining tag tabs
alphabetically, and `Muted` last.

### Reserved panel values

Synthetic tabs stay trustworthy: `errors`, `general`, `muted`, and any value beginning with `__` are rejected at gate
creation. `hitl` is explicitly **allowed** — declaring it simply pins a gate to the default HITL tab. A panel value that
collides with an existing tag tab merges into one tab, which is deterministic and acceptable.

### Vocabulary

One phrase everywhere for the same fact: **"Filed by"**. SASE's own instructions say agents _file_ task beads, and
"filed by" reads correctly for a generic gate too. Agent names render as `@<presented name>` in `#87D7FF`, matching
`src/sase/ace/tui/modals/revive_agent_rendering.py`, and are normalized for display with
`sase.core.agent_identity_facade.present_agent_name` while the raw value is what gets stored.

### Deliberate non-goals

- **Row-level filer.** The filing agent appears in the detail pane and the gate action panel, not in the left-hand list
  rows. Rows are already truncated at 50 characters of note text; the user's request scopes this to "when a task bead
  notification is selected", and keeping rows scannable is the point of this whole change.
- **Renaming the request id prefix.** `_request_id()` keeps producing `bead-task-triage-<bead>-<digest>-g<n>`. It is
  internal identity, the reconciliation state file maps beads to those exact strings, and churning it buys nothing.
- **`payload.created_by`.** The filer is presentation, not trusted decision input. `validate_task_triage_spec` keeps
  pinning the payload to exactly `{bead_id, project, title}`; the bead store remains the durable record of authorship.
- **Renaming `notification_modal_sent_at.py` / `#notification-sent-at`.** The id is referenced by CSS, tests, and a PNG
  golden; the marginal clarity is not worth the churn. The module's docstring gets updated to say "meta line" instead.

## Phase 1: Generic gate presentation panel and origin fields

### New module

Create `src/sase/notification_gates/presentation.py` holding the normalization and the reserved-value policy, so
`validation.py` and `service.py` share one implementation:

- `GATE_PANEL_ACTION_DATA_KEY = "panel"` and `GATE_ORIGIN_AGENT_ACTION_DATA_KEY = "origin_agent"`.
- `RESERVED_GATE_PANELS = frozenset({"errors", "general", "muted"})`.
- `normalize_gate_panel(value: object) -> str | None` — returns `None` when the value is absent or `None`; otherwise
  requires a `str`, strips it, lowercases it, and requires it to match `^[a-z0-9][a-z0-9_-]*$` with length ≤ 32, to not
  be in `RESERVED_GATE_PANELS`, and to not start with `__`. Raise
  `GateError("invalid_presentation", "presentation.panel", <message>)` on any violation; the message must name the
  offending value and, for a reserved value, list the reserved set.
- `normalize_gate_origin_agent(value: object) -> str | None` — returns `None` when absent or `None`; otherwise requires
  a `str`, strips it, requires non-empty, ≤ 128 characters, and no control characters or newlines. Do **not** validate
  against the local agent registry: a gate may legitimately name an agent this host has never seen.

Both helpers return `None` for absent input so callers can uniformly skip the key.

### Validation

In `src/sase/notification_gates/validation.py::validate_gate_spec`, after the existing `presentation.silent` check:

- Call both normalizers on `presentation.get("panel")` and `presentation.get("origin_agent")`.
- Extend the `protected` `action_data` key set with `"panel"` and `"origin_agent"` so producers cannot bypass
  normalization by writing them directly into `presentation.action_data`. The existing `reserved_action_data` error
  already reports the offending keys.

### Projection

In `src/sase/notification_gates/service.py::_build_notification`, after the existing `action_data.update({...})` block,
add each normalized value to `action_data` only when it is not `None`. An absent field must leave `action_data`
byte-identical to today so existing gate fixtures and goldens do not move.

`_build_envelope` needs no change: it already copies the whole `presentation` mapping into `request.json`, so the
declared fields are part of the hashed request and visible in Gate Debug.

### Adapter display title

Add `display_title: str` to `GateAdapter` in `src/sase/notification_gates/adapters.py` and populate it for every
registered adapter: `plan` → `Plan Approval`, `epic_plan` → `Epic Approval`, `question` → `Question`, `launch` →
`Launch Approval`, `hitl` → `HITL`, `task_triage` → `Task Triage`, `custom` → `Custom Gate`. Nothing consumes it in this
phase; phase `action-panel` does. Giving it a value for every adapter now keeps the table complete and avoids a second
edit to the same tuple later.

### CLI

`sase gate create` gains two options in `src/sase/main/parser_gate.py`, and the option block must stay alphabetically
sorted, so the final order is `--origin-agent`, `--panel`, `--sender`, `--tag`:

```
-o, --origin-agent    Attribute the gate to the agent it was filed on behalf of
-p, --panel           Place the gate's notification in a named notification panel tab
```

Extend the parser epilog with one example that uses `--panel`. In `src/sase/main/gate_handler.py::_handle_gate_create`,
fold the new arguments into the same override branch that already handles `--sender` and `--tag`: when either is
supplied, copy the presentation mapping and set `presentation["panel"]` / `presentation["origin_agent"]`. The CLI values
must go through the same `create_gate` validation path rather than being pre-normalized in the handler, so a bad value
reports the same `Error [invalid_presentation] presentation.panel: …` message a JSON producer would see.

### Documentation

- `docs/notifications.md`: document both fields in the gate request section, and document the panel tab rule and the
  reserved values in the "Tags" section (which should also mention that a declared panel takes precedence over the
  synthetic `HITL` and `Errors` tabs).
- `src/sase/xprompts/skills/sase_gate.md`: document `panel` and `origin_agent` in the "Author The Request" section and
  add `"panel"` to the example request's `presentation` block. This is a source-template edit only — do **not** run
  `sase skill init`; deploying to chezmoi requires a committed, merged tree and is not part of this phase.

### Tests

Extend `tests/test_notification_gates.py` with: a gate declaring both fields projects them into `action_data`; an absent
field leaves `action_data` unchanged; each reserved panel value and each malformed panel value raises
`invalid_presentation` targeting `presentation.panel`; a malformed origin agent raises `invalid_presentation` targeting
`presentation.origin_agent`; and setting `panel` or `origin_agent` inside `presentation.action_data` raises
`reserved_action_data`. Extend `tests/test_notification_gate_cli.py` to cover `--panel` and `--origin-agent` reaching
the created notification, and add an assertion that every registered adapter has a non-empty `display_title`.

## Phase 2: Task triage gate identity, filer, and self-heal

### Row text

Three coordinated edits produce `* ✦ [bead] sase-cx — Flaky: test_agents_slow_tool_calls_fold…`:

1. `src/sase/notification_gates/adapters.py`: the `task_triage` adapter's `sender` becomes `"bead"`.
2. `src/sase/bead/task_gate.py::_build_task_triage_gate_spec`: `presentation["sender"]` becomes `"bead"` and
   `presentation["notes"]` becomes `[f"{bead_id} — {title}"]`.
3. `src/sase/notification_gates/kind_validation.py::validate_task_triage_spec`: `expected_note` and the pinned sender
   update to match.

Keep `presentation["tags"] == ["bead", "task"]` unchanged so `sase notify list --tag bead` keeps working, and keep
`ACTION_BADGES["TaskTriage"] == "[task]"` and the `✦` icon unchanged.

### Panel and filer

`create_task_triage_gate` gains a keyword-only `created_by: str = ""` parameter, threaded into
`_build_task_triage_gate_spec`. The spec then sets:

- `presentation["panel"] = "beads"` — always.
- `presentation["origin_agent"] = created_by.strip()` — **only when non-empty**. A bead attributed to the store owner or
  created before attribution existed must produce a gate with no `origin_agent` key at all, never a placeholder like
  `(unknown)`.

`_render_task_triage_preview` gains the same optional value and, when present, emits a filer line directly under the
heading so the Markdown preview shown in both the inbox detail pane and the gate action panel carries it:

```markdown
# sase-cx — Flaky: test_agents_slow_tool_calls_fold_levels_png_snapshots fails only under full parallel suite

**Filed by:** `@claude_coder`

## Description

…
```

The preview stores the raw `created_by` value verbatim; display normalization belongs to the ACE renderers.

`validate_task_triage_spec` must pin all of this so the trusted contract stays airtight: `presentation.get("panel")`
must equal `"beads"`; `presentation.get("origin_agent")` must be absent or a non-empty string; and the preview resource
content must equal `_render_task_triage_preview(...)` for the payload and the declared origin agent. The payload key set
stays exactly `{bead_id, project, title}`.

### Chop wiring

`src/sase/scripts/sase_chop_bead_task_triage.py` passes `created_by=issue.created_by` in its `create_task_triage_gate`
call. `Issue.created_by` is already returned by `rust_beads.list_issues`, so no new bead read is needed.

### Self-heal for pending gates

Without this, the fourteen gates already pending on the user's machine keep their old sender, old note, and no panel
until each bead is individually triaged — the visible improvement would not arrive for weeks. Add a presentation
contract marker to the reconciliation state:

- Define a module-level `_PRESENTATION_CONTRACT = 2` constant next to `_STATE_SCHEMA_VERSION`, with a comment saying to
  bump it whenever the task triage presentation changes in a way that should refresh pending gates.
- Extend `_ProjectState` with `contracts: dict[str, int]` and round-trip it through `_read_state` / `_write_state`
  alongside `gates` and `generations`, defaulting a missing or malformed entry to `1`.
- In `_reconcile`, when a stored gate is still `pending` **and** its bead is still ready **and** its recorded contract
  is older than `_PRESENTATION_CONTRACT`, cancel the pending gate with reason `task_triage_presentation_upgraded` and
  leave the bead in `ready_tasks` so the existing creation loop raises a replacement with the next generation. Count
  these in the existing `canceled` and `gated` summary counters.
- Record `project_state.contracts[bead_id] = _PRESENTATION_CONTRACT` wherever a gate is created, and drop the entry
  wherever the gate entry is deleted.

`cancel_gate` already settles the old notification through `_settle_gate_notification(..., action="cancelled")`, so the
stale inbox row disappears as the replacement appears. A cancellation that raises `already_answered` must be treated
exactly like today's stale-gate path: swallow it and let normal reconciliation proceed.

### Documentation

- `docs/notifications.md`: update the sender table — and **remove the duplicated `bead-task-triage` row**; the table
  currently lists that sender twice, at lines 122 and 126. The surviving row reads `bead`. Update the "Task Triage
  Notification" section to describe the `beads` panel, the shortened note, and the filer line.
- `docs/beads.md`: update the triage section to match the new notification text and mention that the filing agent
  travels with the gate.

### Tests

Update `tests/test_bead/test_task_gate.py` (sender, note, tags, panel, origin agent present and absent, preview text,
and the mutation table's rejection cases for a forged panel and a forged origin agent),
`tests/test_notification_priority.py` (the `bead-task-triage` sender literal), and
`tests/ace/tui/test_notification_custom_gate.py` (`data.sender == "bead"`). Add coverage to
`tests/test_axe_chop_bead_task_triage.py` for: `created_by` reaching the created gate; a blank `created_by` producing no
`origin_agent`; and a stored gate at contract `1` being cancelled and recreated at the next generation while a gate
already at the current contract is left pending and merely counted as skipped.

## Phase 3: Panel tabs and filer line in the notification modal

### Tab resolution

In `src/sase/ace/tui/modals/notification_modal_tags.py`:

- Add `notification_panel_key(notification) -> str | None`, reading `action_data.get("panel")`, stripping and
  lowercasing it, and returning `None` for an absent, blank, or reserved-looking value. The modal is a renderer: it must
  tolerate a malformed stored value rather than raise, because gate-creation validation is what enforces the contract.
- Insert the panel check into `_notification_modal_tab_keys` between the muted branch and the `_HITL_ACTIONS` branch, as
  specified in the Design section.
- Add `notification_origin_agent(notification) -> str | None` reading `action_data.get("origin_agent")` with the same
  tolerant stripping, so phases `inbox` and `action-panel` share one accessor.

### Tab ordering and labels

- `build_notification_tag_tabs` collects a `panel_keys: set[str]` while counting, then orders keys as `hitl`, sorted
  panel keys, `errors`, `None`, `done`, remaining keys sorted by label casefold, `__muted__`. Keys that are both
  panel-declared and tag-derived sort with the panels.
- `_notification_tab_label` keeps `_SYNTHETIC_TAB_LABELS` as the first lookup, then humanizes: split the key on `-` and
  `_`, capitalize each word's first character, and join with spaces, so `beads` → `Beads` and `code-review` →
  `Code Review`. This replaces the current first-character-only capitalization and applies uniformly to panel and tag
  tabs — one labeling rule per strip. Every tag in use today is a single lowercase word, so no existing label changes.

### Filer on the detail pane

In `src/sase/ace/tui/modals/notification_modal_sent_at.py`, extend `_build_sent_at_text` so that when the notification
declares an origin agent the line becomes:

```
sent yesterday 19:32 · 11h ago · filed by @claude_coder
```

Append `" · "` in `dim`, `"filed by "` in `dim`, and `f"@{present_agent_name(raw)}"` in `#87D7FF`. Import
`present_agent_name` from `sase.core.agent_identity_facade` lazily inside the function, matching how the module already
defers `sase.core.time` imports, and fall back to the raw value if normalization raises. The widget keeps `no_wrap` and
ellipsis overflow, so a long name degrades gracefully in a narrow pane. Update the module docstring to describe a meta
line rather than only a sent-at line; keep the module name, `SENT_AT_ID`, and the `#notification-sent-at` CSS rule.

### Tests

`tests/_notification_modal_helpers.py::_make_notification` gains optional `tags` and `action_data` parameters (both
defaulting to empty) so tab tests can build panel-bearing rows. Add tests covering: a panel-declared `TaskTriage`
notification lands in `Beads` and not in `HITL`; a muted panel notification still lands in `Muted`; a reserved or
malformed stored panel value falls back to the pre-existing routing; tab order matches the Design section's target
strip; `beads` labels as `Beads` and `code-review` as `Code Review`; and the meta line renders and omits the filer
clause correctly. Put tab tests in `tests/test_notification_modal_mark_and_tabs.py` and
`tests/test_notification_modal_sections.py`, and meta-line tests in `tests/test_notification_modal_sent_at.py`.

## Phase 4: Gate action panel title and filer line

In `src/sase/ace/tui/modals/custom_gate_modal.py`:

- Add `title: str` and `origin_agent: str | None` to `CustomGateModalData`.
- `_title()` renders `f"[bold cyan]{icon} {title}[/bold cyan]  [bold]{sender}[/bold]  [dim]{request_id}[/dim]"`, so a
  task triage gate reads `✦ Task Triage  bead  bead-task-triage-sase-cx-…` instead of today's wrong `Custom Gate`.
- `_compose_actions` yields, before the `Context` section title and only when `origin_agent` is set, a `Static` with
  `id="custom-gate-origin"` and `classes="gate-review-origin"` whose `Text` is `"Filed by "` in `dim` plus
  `f"@{present_agent_name(origin_agent)}"` in `bold #87D7FF`.

In `src/sase/ace/tui/actions/agents/_notification_custom_gate.py::_load_custom_gate_modal_data`, populate `title` from
`adapter.display_title` and `origin_agent` from the shared `notification_origin_agent` accessor added in phase `inbox`.
Both reads happen in the existing worker thread, so nothing new touches the event loop.

Add a `.gate-review-origin` rule to `src/sase/ace/tui/styles.tcss` next to the existing `.gate-review-section-title`
rule: `height: auto;` and `margin-bottom: 1;`, with no border, so the line sits directly under the centered header
without disturbing the compact `gate-review-shell--compact` layout.

Update `tests/ace/tui/test_custom_gate_modal.py` for the adapter-derived title and the presence and absence of the
origin line, and `tests/ace/tui/test_notification_custom_gate.py` for `data.title == "Task Triage"` and
`data.origin_agent`.

## Phase 5: PNG snapshot coverage and documentation sweep

Read the PNG snapshot guidance in this repo's `CLAUDE.md` before starting: goldens live in
`tests/ace/tui/visual/snapshots/png/`, failures leave artifacts in `.pytest_cache/sase-visual/`, and
`--sase-update-visual-snapshots` accepts intentional changes.

Add three goldens, following the fixture and determinism patterns in
`tests/ace/tui/visual/test_ace_png_snapshots_notification_sent_at.py` — pin timestamps by monkeypatching
`format_absolute_time` / `format_relative_time`, and pin any displayed path:

1. `notification_beads_tab_120x40.png` — a `NotificationModal` seeded with one plan approval, several panel-declared
   task triage rows, one error, and one `done` row, with the `Beads` tab active. This is the golden that proves the tab
   strip order, the tab label, and the compact `[bead] <id> — <title>` row text.
2. `notification_filed_by_120x40.png` — the same modal with a task triage row selected, proving the
   `sent … · … · filed by @…` meta line.
3. `custom_gate_task_triage_120x40.png` — the task triage gate open in `CustomGateModal`, proving the `Task Triage`
   title, the `Filed by` line, the `task.md` preview with its filer line, and the Launch/Close decision controls.

Then reconcile the prose: re-read `docs/notifications.md` and `docs/beads.md` end to end and fix anything the earlier
phases left stale, including the tab table in the "Tabs and Ordering" section, which must gain the panel row and state
that a declared panel outranks `HITL` and `Errors`.

## Verification

Every phase that changes files under `src/` or `tests/` must finish with a green `just check`. Because SASE workspaces
are ephemeral, run `just install` first if the workspace has not been used recently. Phase `visuals` must additionally
run `just test-visual` and confirm the three new goldens are byte-stable across two consecutive runs before committing
them.

Documentation-only edits inside `sdd/research/` are exempt from `just check`, but no phase here is documentation-only:
each doc change ships alongside code in the same phase.
