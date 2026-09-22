---
tier: epic
title: Context-aware Enter on the Agents tab (retire the ,n leader key)
goal: "Pressing Enter on an Agents-tab row opens that agent node's pending gate (every
  gate kind, including sudo, launch, HITL, and custom gates), jumps to its Patch, or —
  when both apply — opens a polished one-keypress chooser. The `,n` leader key is
  retired cleanly, and the footer, help, palette, onboarding, and docs all describe the
  new Enter.

  "
phases:
  - id: resolve
    title: Gate resolution and shared notification dispatch
    depends_on: []
    size: medium
    description: "resolve: extract the NotificationModal action routing into a shared
      dispatcher, then build the in-memory AgentEnterTarget resolver (pending gate rows,
      identity-matched gate notifications, legacy question/HITL/remote-attention
      sources, and the node's Patch) and the target executor, all without new UI-thread
      disk reads.

      "
  - id: chooser
    title: AgentActionChooserModal
    depends_on: []
    size: medium
    description: "chooser: add the self-contained single-keypress chooser modal: generic
      choice view model, deterministic g/1-9/p key assignment, AgentView-family styling,
      unit tests, and one PNG visual golden.

      "
  - id: wire
    title: Enter keymap swap and ,n retirement
    depends_on:
      - resolve
      - chooser
    size: medium
    description:
      "wire: add the act_on_agent app action bound to enter (jump_to_agent_patch stays
      as an unbound action), connect the resolver to the chooser, retire
      leader.jump_to_notification as a relocated key, and update the footer, help,
      palette, onboarding, docs, and tests."
proposed_by: bbugyi200.athena.0ph
create_time: 2026-09-22 13:38:25
status: wip
---

- **PROMPT:**
  [prompts/202609/agents_enter_act_on_agent.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agents_enter_act_on_agent.md)

# Plan: Context-aware `<enter>` on the Agents tab

## 1. Problem

Today the Agents tab has two keys for "act on this agent":

- `<enter>` (`ace.keymaps.app.jump_to_agent_patch`) jumps to the Patch the selected row
  resolved (`action_jump_to_agent_patch` in
  `src/sase/ace/tui/actions/agents/_patch_navigation.py`).
- `,n` (`leader_mode.keys.jump_to_notification`) calls `_jump_to_agent_notification` in
  `src/sase/ace/tui/actions/agents/_notification_modal_flow.py`. It works only for
  PlanApproval, EpicApproval, and UserQuestion notifications. It has three separate ways
  to fail:
  1. It requires `agent_is_asking(row.status)`, which covers only
     `{PLAN, TALE, EPIC, QUESTION, WAITING INPUT}`. Gate member rows show `SUDO`,
     `LAUNCH`, `HITL`, or `GATE` (the shell's `pending_status`), so those rows are
     rejected at the first check.
  2. Only those three notification actions are dispatched.
  3. It matches notifications with `agent_matches_notification_identity`
     (`agent_timestamp` / `agent_root_timestamp` / `agent_name`). Sudo, launch, HITL,
     and custom gate notifications carry none of those keys. Their gate-shell stamp adds
     only `raw_suffix`, `artifacts_dir`, and `family_root_suffix`.

  So sudo gates, and every other non-plan/question gate, can never open through `,n`.
  `,n` also silently unhides and jumps to the _first_ asking agent anywhere when the
  selected row isn't asking. That is surprising, and `,J` (`jump_to_next_stopped_agent`)
  already covers it.

The loaded rows already hold everything we need. A gate shell is a family member `Agent`
row with `is_gate` (role `gate` plus `gate_id`), `gate_kind`, `gate_state`,
`gate_notification_id`, `gate_bundle_path`, `gate_label`, `gate_reason`,
`gate_start_status`, `gate_accent`, and `gate_creator_agent`. The agent that created a
gate also carries that `gate_id` as a back-reference, without being `is_gate`. The
NotificationModal's `_on_dismiss` already knows how to open every gate kind.

## 2. UX specification (the contract every phase implements)

### 2.1 What Enter does

The selected row decides the **scope**. The resolver collects the **targets** in that
scope:

- **Gate targets.** A _pending_, answerable gate: `row.is_gate`,
  `gate_state == "pending"`, `stop_time is None`, and not `gate_execution_active`.
  Settled and `settling` gates are never targets.
- **Patch target.** Only when a real Patch name resolves.

| Selected row                                  | Gate targets in scope                                                                                                                                                                           | Patch target                                                                |
| --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| Family container (agent node)                 | Every pending gate row in `concrete_family_shell_rows(container)`, plus gate notifications identity-matched to the container or any member agent shell                                          | The container's Patch, else the newest member agent shell that resolves one |
| Standalone agent row (no family)              | Gate notifications identity-matched to the row. Legacy fallbacks: `status == "QUESTION"` → question-marker target; `status == "WAITING INPUT"` → workflow-HITL target                           | The row's Patch                                                             |
| Gate member row                               | That gate only, if pending. If it is settled: toast `This gate already settled (<stop status or state>)`                                                                                        | none                                                                        |
| Agent-shell member row                        | Pending gate rows it created (gate row `gate_id` equals the member's back-reference `gate_id`, or `gate_creator_agent == member.agent_name`), plus notifications identity-matched to the member | The member's Patch                                                          |
| Workflow step row                             | Workflow-HITL target when `status == "WAITING INPUT"`; identity-matched notifications                                                                                                           | Parent workflow Patch (existing `_resolve_agent_cl_name` logic)             |
| Remote (fleet) row                            | A remote-attention target when `has_pending_remote_attention(row)`                                                                                                                              | none (local-only, same as today)                                            |
| Clan container                                | none. Toast `Select an agent inside this clan`                                                                                                                                                  | none                                                                        |
| Monitor/proc shell rows; focused panel/banner | none. A focused tribe panel already yields no selected agent. A focused group banner (`_current_group_key is not None`) is a silent no-op                                                       | none                                                                        |

Dispatch rule:

- **0 targets:** warning toast `No pending gate or Patch for this agent`, or the scoped
  message from the table.
- **1 target:** run it directly. Patch-only agents behave exactly like today's Enter.
  Gate-only agents behave like `,n`, but for every gate kind.
- **2+ targets:** open `AgentActionChooserModal`.

Ordering: gates first (they block work), newest first. The Patch comes last. The first
row is the **primary** action, and the chooser opens with it highlighted, so
`<enter><enter>` always opens the most urgent gate.

Gate row labels come from `gate_kind`, or from the notification action for
notification-only targets:

| Kind / source                                             | Label                                                                       |
| --------------------------------------------------------- | --------------------------------------------------------------------------- |
| plan                                                      | `Review plan`; `Review tale plan` when the pending status is `TALE`         |
| epic_plan                                                 | `Review epic plan`                                                          |
| question / question marker                                | `Answer question`                                                           |
| sudo                                                      | `Review sudo request`                                                       |
| launch                                                    | `Approve agent launch`                                                      |
| hitl / workflow HITL                                      | `Respond to checkpoint`                                                     |
| task_triage, bead_snooze, flag_triage, bead_stale_cleanup | `Triage task`, `Review snoozed bead`, `Triage flag`, `Clean up stale beads` |
| plugins_required                                          | `Install required plugins`                                                  |
| custom / unknown                                          | `gate_label` if set, else `Open gate`                                       |
| remote attention                                          | `Answer remote request`                                                     |

Keep the label mapping in one table-driven helper (phase `resolve`). The footer uses the
lowercase label.

### 2.2 The chooser (`AgentActionChooserModal`)

Visual target: a 68-column, AgentView-family modal (`max-width: 94%`):

```
╔══════════════════════════════════════════════════════════════════╗
║                        Act on foo.bar                            ║
║          ⏎ again → Review tale plan · or press a key             ║
║                                                                  ║
║  GATE                                                            ║
║ ▌› [g]  ⋔  Review tale plan                         TALE · 12m   ║
║ ▌         sase_plan_enter_keymap.md                              ║
║                                                                  ║
║  PATCH                                                           ║
║    [p]  ⎇  Go to Patch                              Mailed       ║
║            foo enter keymap · PR #123                            ║
║──────────────────────────────────────────────────────────────────║
║        Up/Down or j/k move - Enter select - Esc cancel           ║
╚══════════════════════════════════════════════════════════════════╝
```

- **Structure.** Two sections, `GATE` (plural `GATES` when there are several) and
  `PATCH`. Each row has two lines:
  - line 1: pointer, keycap, glyph, bold label, and a right-aligned badge;
  - line 2: an indented dim subtitle, truncated with `…`.

  Render line 1 as a `rich.table.Table.grid(expand=True)` with a ratio-1 label column
  and a no-wrap right-justified badge column, so the badge stays flush right at any
  width.

- **Gate styling.**
  - Glyph: `GATE_GLYPH` (`⋔`, from `sase.gate_shell.state`), styled with the gate's
    accent/status style (`gate_status_style` from `sase.gate_shell.status`).
  - Badge: the pending status (`TALE`, `EPIC`, `QUESTION`, `SUDO`, `LAUNCH`, `HITL`,
    `GATE`, …) plus a compact age (`12m`), in the same style.
- **Patch styling.**
  - Glyph: `⎇` (the Patches artifact glyph) in the Patch-status color
    (`get_status_color` in `src/sase/ace/display_helpers.py`).
  - Badge: the Patch status, when known from memory.
  - Subtitle: the humanized Patch name, plus the PR label when known.
- **Keys** are assigned by the modal from the section order. The rule is deterministic
  and unit-tested:
  - Exactly one gate row gets `g`. With several gate rows they get `1`–`9` in display
    order, and `g` stays a hidden alias for the first gate.
  - The Patch row gets `p`.
  - `enter` selects the highlighted row, which starts on the first row.
  - `j`/`k`/`up`/`down` move the highlight. `escape` and `q` cancel and return `None`.
  - Mouse click selects a row.
  - Every other printable key is swallowed (`prevent_default` + `stop`) so it can't leak
    to the Agents tab.
  - Dismiss exactly once (`_dismiss_once`, as in `AgentViewModal`).
- **Palette.** Reuse the AgentViewModal visual language: pointer `bold #87D7FF`; keycap
  `bold black on #87D7FF` when focused and `bold #87D7FF` otherwise; label
  `bold #F8F8F2`; subtitle `dim`. The container is `border: double $primary`,
  `background: $surface`, `padding: 1 2`. The focused row is `background: $primary 20%`
  with `border-left: thick $primary`. Section headers and the footer rule match
  `.agent-view-section` and `#agent-view-footer`.
- **Guidance line** names the primary action:
  `⏎ again → <primary label> · or press a key`. This teaches the Enter-Enter shortcut
  without extra prose.

### 2.3 Surrounding surfaces

- **Footer** (conditional hint, `<enter>` sorted first per `src/sase/ace/CLAUDE.md`).
  Replace today's `go to PR` hint with:
  - 1 target: the lowercase target label (`review tale plan`, `answer question`,
    `review sudo request`, `go to PR`, …);
  - 2+ targets: `choose action`;
  - 0 targets: no hint.
- **Leader footer.** Drop the `notification` hint and its `has_notification` plumbing.
- **Help modal** (`modals/help_modal/agents_bindings.py`). Add
  `(d(a.act_on_agent), "Review gate / go to Patch")` (26 chars, within the 32-char cap)
  and remove the `,n` row.
- **Command palette.**
  - New `app.act_on_agent`: "Act on agent (gate or Patch)", Agents only, aliases such as
    `review gate`, `open gate`, `go to patch`, `notification`. It is available when an
    agent is selected and the resolver returns at least one target.
  - `app.jump_to_agent_patch` stays as the direct "Jump to agent's Patch" command.
  - `leader.jump_to_notification` is removed.
- **Onboarding** (`widgets/agent_onboarding.py` ~268–281). Replace "jump to the PR it
  produced" / "or jump to its PR" with copy that says Enter reviews a pending gate or
  jumps to the agent's PR.

## 3. Cross-cutting decisions

- **Action identity.**
  - Add a new app action `act_on_agent`, bound to `enter`
    (`Binding("enter", "act_on_agent", "Act on agent", show=False)`).
  - Keep `jump_to_agent_patch` as a real action, its palette entry, and its
    `jump_to_agent_changespec` legacy alias, with its default key set to `"unbound"`
    (mirror how `answer_remote_attention: "unbound"` is modeled in `AppKeymaps`,
    `default_config.yml`, and the default-binding tables). A user who wants a direct
    Patch key can bind it.
  - No `LEGACY_APP_KEY_ALIASES` rename is needed, because the old id keeps its meaning.
    The live user config has no override for either key.
- **Retiring `,n`.**
  - Remove `jump_to_notification` from the leader defaults. Add a
    `_RELOCATED_LEADER_KEYS` entry in `src/sase/ace/tui/keymaps/registry.py`:
    `Ignoring stale leader_mode.keys.jump_to_notification; open an agent's pending gate with ace.keymaps.app.act_on_agent (Enter) on the Agents tab`.
    A stale override then warns and can never come back as a live command.
  - Leader `n` becomes unbound. Add no press-time hint, matching earlier leader
    retirements.
  - Drop the "auto-unhide and jump to the first asking agent" behavior (use `,J`).
- **No feature flag.** The user explicitly asked to retire `,n` outright, and no old
  branch stays reachable. The new Enter ships as finished, unconditional behavior. This
  follows the precedent of earlier leader-key retirements in `_RETIRED_LEADER_KEYS`.
- **Rust core boundary.** The resolver is a pure projection over already-loaded TUI
  `Agent` rows (which come from the core scan wire) and the in-memory notification
  snapshot. It adds no domain rules, and the chooser, labels, and key assignment are
  presentation. Everything therefore stays in this repo. If a future frontend needs
  "agent node → pending gates", promote the projection to `sase_core` then.
- **Performance** (see the TUI perf rules).
  - Resolution is in-memory only: no stat/glob/JSON reads, and it walks only the
    selected node's subtree. It is safe for the footer and palette context paths.
  - The notification side reads `self._notification_snapshot_cache.notifications`. This
    is the polled, unbounded, non-dismissed snapshot, so the old "complete unread
    dataset" guarantee of `,n` holds.
  - Prefilter it once per snapshot object into a small gate-notification index: by id,
    by `action_data.bundle_path`, by `action_data.raw_suffix`, plus the list of
    gate-action notifications for identity matching. Rebuild the index when the cached
    snapshot's identity changes.
  - Any fallback disk read (notification detail for a dismissed or not-yet-cached gate
    notification; the first snapshot when the cache is still `None`) runs off the pump
    via `spawn_pump_free_task` + `asyncio.to_thread`, calling the module-level
    `_read_notification_detail_for_tui` / snapshot reader rather than the
    attribute-mutating mixin methods.
  - Re-capture the selection by identity (`_agent_by_identity`) and re-resolve on the UI
    thread before acting (rule 4).
  - Never call the synchronous `_refresh_notification_count()` from the new flow. Use
    `_schedule_notification_snapshot_refresh()` after a dispatch.
  - Existing handlers keep their current internals. Don't add new synchronous reads.

## 4. Phase `resolve`: gate resolution and shared notification dispatch

1. **Shared dispatcher.** Create
   `src/sase/ace/tui/actions/agents/_notification_dispatch.py` with
   `open_notification_action(app, notification) -> bool`.
   - Move the per-action `if/elif` chain out of `_show_notification_modal._on_dismiss`
     verbatim: JumpToPatch, JumpToMentorReview, JumpToAgent, Tmux, HITL,
     Plan/EpicApproval, UserQuestion, LaunchApproval, remote attention, SudoRequest
     _before_ the generic-form branch, the generic-form custom gates, ViewErrorReport,
     gate-execution-failed, ViewReport, OpenLaunchControl, and the unsupported-action
     toast.
   - Keep the pre-dispatch `_read_notification_pending_actions_from_provider()` call for
     `PRIVILEGED_GATE_ACTIONS`, so behavior is identical.
   - Return whether a handler ran.
   - `_on_dismiss` keeps its mark-read rule, count refresh, and detail re-read, then
     calls the dispatcher.
   - Add a table-driven test proving every action routes to the same handler as before,
     including SudoRequest winning over generic-form.
2. **Target model and resolver.** Create
   `src/sase/ace/tui/actions/agents/_agent_enter_targets.py` (pure; importable without
   an app).
   - `@dataclass(frozen=True, slots=True) class AgentEnterTarget` with these fields:
     - `kind: Literal["gate", "patch"]`
     - `source: Literal["gate_row", "notification", "question_marker", "workflow_hitl", "remote_attention", "patch"]`
     - `key: str` (stable identity: `gate:<gate_id|notification_id|bundle_path>`,
       `patch:<name>`, `hitl:<identity>`, …)
     - `label: str`, `detail: str | None`, `badge: str | None`,
       `badge_style: str | None`, `age_seconds: float | None`
     - payload: `notification_id`, `bundle_path`, `gate_id`, `row_identity`,
       `patch_name`, `project_file`
   - `resolve_agent_enter_targets(agent, *, gate_notifications: GateNotificationIndex, patch_name_for: Callable[[Agent], str | None], patch_lookup: Callable[[str], PatchSummary | None]) -> AgentEnterResolution`.
     The result holds the ordered `targets` tuple, `scope_title` (the node's presented
     name), and an optional `empty_message` for the scoped toasts in §2.1.
   - Implement the scope table exactly. Reuse `concrete_family_shell_rows`,
     `family_roster_container`, `is_family_container_row`, `is_clan_container`,
     `is_gate`, `is_monitor`, `is_proc_shell`, `has_pending_remote_attention`, and
     `agent_matches_notification_identity`. Never treat a container as a gate: it
     mirrors `gate_state` but is not `is_gate`.
   - Dedupe gate targets by gate id, notification id, and bundle path; a row-backed
     target wins over a notification-only one.
   - Drop notification-only targets whose bundle path belongs to a _settled_ gate row in
     the roster.
   - "Gate action" means `adapter_for_action(n.action) is not None`.
   - Add the question-marker / workflow-HITL legacy targets only when no gate target
     already covers that row.
   - Patch names: `patch_name_for` is `_resolve_agent_cl_name`. Add `"~"` (the
     running-marker fallback in `_running_loaders.py`) to that method's existing `None`
     / `""` / `"unknown"` no-Patch sentinels, so the direct `jump_to_agent_patch` can no
     longer navigate to a Patch named `~` either. The container → newest-member fallback
     lives in the resolver, not in `_resolve_agent_cl_name`, so other callers keep their
     single-row semantics.
   - `PatchSummary` (status, PR label) comes only from the in-memory `app.patches` via
     `_find_patch_index_by_name`. A miss just omits the badge; the Patch target is still
     offered because `navigate_to_patch_tab` widens the query.
   - Labels, subtitles, and badges follow §2.1–2.2. The subtitle comes from row fields
     or the in-memory notification (plan file basename from `files[0]`, title/message),
     never from bundle reads.
   - `build_gate_notification_index(snapshot)` caches per snapshot object.
3. **Executor mixin.** Create `src/sase/ace/tui/actions/agents/_agent_enter_action.py`
   with `AgentEnterActionMixin`, composed into `AgentsMixinCore` next to
   `AgentPatchNavigationMixin`.
   - `_agent_enter_resolution(agent)` wires the resolver to the app: the cached snapshot
     index, `_resolve_agent_cl_name`, and the in-memory patch lookup.
   - `_run_agent_enter_target(target, *, agent_identity)` handles each kind:
     - **patch:** `navigate_to_patch_tab`.
     - **gate_row / notification:** look up the notification in the snapshot index by
       id, then by bundle path. On a hit, call `open_notification_action` and then
       `_schedule_notification_snapshot_refresh()`. On a miss with a `notification_id`,
       spawn a pump-free task that reads the detail off-thread; back on the UI thread,
       re-resolve the agent by identity and confirm the target key is still present
       before dispatching. Toast `Gate <short id> is no longer pending` or
       `Couldn't load gate <short id>; try: sase gate show <short id>`
       (`short_gate_shell_id`) otherwise.
     - **question_marker:** `_open_question_modal_from_marker(agent)`, else toast
       `No pending question found for this agent`.
     - **workflow_hitl:** `_answer_workflow_hitl(agent)`.
     - **remote_attention:** refactor `action_answer_remote_attention` into
       `_answer_remote_attention_for(agent)` and call it.
   - If the snapshot cache is `None`, perform one off-pump snapshot read and store it
     via `_set_notification_snapshot_cache`. Expose a
     `_ensure_agent_enter_snapshot(then)` helper for phase `wire`.
4. **Leave `,n` untouched in this phase.** `_jump_to_agent_notification`, its leader
   branch, and its tests keep working until phase `wire` deletes them in the same change
   that binds Enter, so no intermediate commit loses a working shortcut. Keep
   `_open_question_modal_from_marker` permanently; the question-marker target reuses it.
5. **Tests** (new `tests/ace/tui/test_agent_enter_targets.py`, plus executor tests;
   reuse the `_ShortcutApp` / `make_agent` / `notification_home` patterns from
   `tests/ace/tui/test_notification_backlog_access.py`). Cover:
   - every scope row kind;
   - a pending sudo gate row;
   - launch, HITL, and custom gate rows;
   - a settled and a `settling` gate being excluded;
   - a container mirroring `gate_state` not producing a phantom gate;
   - creator back-reference matching;
   - dedupe of row plus notification;
   - an old identity-matched PlanApproval behind 120 newer notifications (ported from
     the existing backlog test);
   - the `"~"`/`"unknown"` Patch sentinels;
   - member Patch fallback;
   - ordering and primary;
   - remote attention;
   - WAITING INPUT;
   - the QUESTION marker;
   - the executor's off-thread detail fallback and its revalidation toast.

## 5. Phase `chooser`: `AgentActionChooserModal`

Independent of `resolve`: the modal takes a generic view model and knows nothing about
`Agent` or notifications.

1. Create `src/sase/ace/tui/modals/agent_action_chooser_modal.py`:
   - `@dataclass(frozen=True) class AgentActionChoice` with `result: str` (opaque id
     returned on selection), `section: Literal["gate", "patch"]`, `label`,
     `detail: str | None`, `glyph`, `glyph_style`, `badge: str | None`,
     `badge_style: str | None`, `age: str | None`.
   - `AgentActionChooserModal(ModalScreen[str | None])` built as
     `(choices, *, title: str)`. It assigns keys per §2.2, exposes the assigned keys
     (for tests and the guidance line), and renders the §2.2 layout.
   - Model the structure and keyboard handling on `AgentViewModal`
     (`src/sase/ace/tui/modals/agent_view_modal.py`): `compose` with container, title,
     guidance, `VerticalScroll` list, section headers, and footer; `on_key`; `on_click`;
     `_dismiss_once`.
   - Row ids: `agent-action-row-<n>`.
2. Add CSS in `src/sase/ace/tui/styles.tcss` next to the AgentView rules:
   `AgentActionChooserModal { align: center middle; }`, plus `#agent-action-container`,
   `#agent-action-title`, `#agent-action-guidance`, `.agent-action-section`,
   `.agent-action-row` / `-focused`, and `#agent-action-footer`, per §2.2.
3. Register the lazy export in `src/sase/ace/tui/modals/_export_table.py` and
   `modals/__init__.pyi`.
4. Unit tests in `tests/ace/tui/modals/test_agent_action_chooser_modal.py`, modeled on
   `test_agent_view_modal.py`. Cover:
   - key assignment for (1 gate + Patch), (3 gates + Patch), (2 gates, no Patch), with
     the `g` alias;
   - enter selecting the highlight, starting on the primary row;
   - j/k wrap or clamp (match AgentViewModal);
   - esc/q returning `None`;
   - swallowed printable keys;
   - a single dismiss;
   - the guidance line naming the primary label;
   - long-label and subtitle truncation at 60 and 120 columns.
5. Add one PNG golden, `agent_action_chooser_modal_120x40`: one pending tale-plan gate
   plus a Mailed Patch with a PR, pushed directly as in
   `tests/ace/tui/visual/test_ace_png_snapshots_jump_action.py` but on the Agents tab.
   Put the test in `tests/ace/tui/visual/test_ace_png_snapshots_agents_modals.py`.
   - Generate it with `just fix-tui-screenshots -- <selector>`. Use `/sase_monitor` if
     the run is long, and follow the tui_screenshot golden-inspection rules.
   - Inspect the PNG: badge flush right, focused-row accent, readable subtitle, no
     wrapping.

## 6. Phase `wire`: Enter keymap swap and `,n` retirement

1. **Keymaps.**
   - `src/sase/default_config.yml`: add `act_on_agent: "enter"` with a comment
     describing the Enter contract; set `jump_to_agent_patch: "unbound"`; remove leader
     `jump_to_notification: "n"`.
   - `AppKeymaps` (`keymaps/app_keymaps.py`): new `act_on_agent` field.
   - `_BINDING_META` (`keymaps/metadata.py`): `("act_on_agent", "Act on agent", False)`.
   - Default binding table (`tui/bindings.py:270`): enter → `act_on_agent`, and
     `jump_to_agent_patch` unbound the way other unbound actions are.
   - Palette metadata (`commands/_app_metadata_actions.py`).
   - `LeaderModeKeymaps` defaults (`keymaps/mode_keymaps.py`): remove
     `jump_to_notification`.
   - `_RELOCATED_LEADER_KEYS` entry (§3).
   - Update the `widgets/agent_list.py` comment that names the enter binding.
2. **Action availability** (`_app_action_availability.py`).
   - `act_on_agent` must be unavailable while the prompt input owns keys, like
     `_LOCAL_AGENT_ROW_ACTIONS`.
   - It must stay available on remote rows (for remote attention), so it must _not_ join
     the remote-exclusion set.
   - Palette availability (`commands/_availability_agents.py`): add `app.act_on_agent`
     to `_REQUIRES_AGENT` but not to `_REMOTE_AGENT_LOCAL_COMMANDS`. It is available iff
     a new `CommandContext.agent_enter_available` (computed in `commands/context.py` via
     the resolver) is true. Remove the `leader.jump_to_notification` branch.
3. **`action_act_on_agent`** (in `AgentEnterActionMixin`):
   - Guard on the Agents tab, a selected agent, and no focused group banner.
   - Ensure the snapshot exists, then resolve.
   - Dispatch per §2.1: toast, run, or push `AgentActionChooserModal`. Map targets to
     `AgentActionChoice`, with `result = target.key` and
     `title = f"Act on {scope_title}"`.
   - In the modal callback, re-find the agent by the captured identity and re-resolve.
     Run the target whose key matches, or toast `That action is no longer available` /
     `Agent is no longer visible`.
4. **Retire `,n`.**
   - Delete the `_leader_mode.py` dispatch branch and `_jump_to_agent_notification`
     (with its now-unused imports).
   - Delete the `has_notification` plumbing in `_update_leader_footer`,
     `_display_detail_footer.py`, and `_keybinding_modes.update_leader_bindings`.
   - Delete the `_LEADER_LABELS` / `_LEADER_TABS` entries in
     `commands/_mode_commands.py` and the help row.
5. **Footer.** In `widgets/_keybinding_bindings.py` / `_keybinding_modes.py` /
   `_display_detail_footer.py`, replace `can_jump_to_patch` with an optional
   `enter_action_label` computed from the resolver (§2.3). Show a `jump_to_agent_patch`
   hint only if the user bound that action to a real key and a Patch resolves.
6. **Help, onboarding, docs.**
   - Help and onboarding per §2.3.
   - `docs/ace.md`: rewrite the Agents key-table Enter row; add a short "Enter: act on
     an agent" subsection (scope table summary, chooser keys, Enter-Enter primary); drop
     the `,n` leader row; rewrite the Notifications-modal paragraph and the
     question-shell sentence that cite `,n`.
   - `docs/notifications.md`: rewrite the paragraph that cites the `,n` complete-dataset
     guarantee to describe Enter.
7. **Tests.**
   - Update the assertions that break:
     - `tests/test_command_catalog_guards.py` (the `leader.jump_to_notification`
       presence check);
     - `tests/test_keymaps_defaults.py` (the "Go to Patch" label and yml coverage);
     - `tests/test_command_availability_agents_navigation.py`;
     - `tests/test_command_availability_agents_fleet.py`;
     - `tests/ace/tui/test_jump_to_changespec.py`;
     - `tests/ace/tui/widgets/test_agent_list_bindings.py`;
     - `tests/ace/tui/widgets/test_agent_onboarding.py`;
     - `tests/ace/tui/test_notification_backlog_access.py` (port the real-App `comma`,
       `n` test to an `enter` press test that opens the old PlanApproval).
   - Add retirement tests modeled on `tests/test_keymaps_defaults.py` (~206–231) and
     `tests/test_keymaps_registry_loading_legacy.py`: a stale `jump_to_notification`
     override warns and is dropped.
   - Add Textual pilot tests for Enter:
     - Patch-only row → Patches tab;
     - sudo gate row → SudoRequest handler;
     - gate + Patch → chooser, then `p` → Patches tab and `g` → gate handler;
     - enter, enter → primary;
     - esc → nothing;
     - clan → toast;
     - remote row with attention → remote modal.
   - Footer label tests for 0, 1, and 2+ targets.
8. **Live check.** Capture the real TUI with `sase screenshot` (tui_screenshot workflow)
   on an Agents row that has a pending gate plus a Patch after pressing Enter, and
   inspect the PNG. Optionally run `SASE_TUI_PERF=1` j/k over a large family to confirm
   p95 stays under 16 ms with the footer resolver.

## 7. Verification (every phase)

Follow the `lint_and_test` memory recipe (`just check`; `just check-full` only if
explicitly instructed). Keep `just fix-tui-screenshots` targeted to the new or affected
goldens, and inspect every golden change. Phase workers record `PROPOSED FOLLOW-UP:`
notes on their own bead instead of creating beads. One likely follow-up: drop the unused
pre-dispatch `_read_notification_pending_actions_from_provider()` sync read.

## 8. Non-goals

- Changing how gate handlers or their modals work internally.
- Clan-level "all gates in this clan" actions.
- Acting on Patch banners.
- A new configurable keymap scope for the chooser. It hard-codes keys like every
  existing chooser modal.
- Any change to `A` (`accept_proposal`) on the Agents tab.
