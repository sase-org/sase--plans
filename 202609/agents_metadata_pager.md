---
tier: tale
title: Open the Agents-tab metadata panel in the pager with V
goal:
  Pressing V on any local Agents-tab row (running or done) opens that agent's metadata
  as a beautiful sectioned pager document with agent-aware links, agent chrome, and a
  live r-refresh, while V on other tabs keeps opening the Agent Run Log modal.
size: medium
proposed_by: bbugyi200.athena.0kf
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0kf](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0kf.md)
  - [bbugyi200.athena.chop.refresh_docs.sase.0_289632.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.chop.refresh_docs.sase.0_289632.1/README.md)
  - [bbugyi200.athena.chop.refresh_docs.sase.0_289632.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.chop.refresh_docs.sase.0_289632.2/README.md)
  - [bbugyi200.athena.toobig-5d.test_agent_wait_resume.0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.toobig-5d.test_agent_wait_resume.0.md)
  - [bbugyi200.athena.toobig-5d.test_artifact_cli_link_health.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5d.test_artifact_cli_link_health.0/README.md)
  - [bbugyi200.athena.toobig-5d.test_continuation_facade.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5d.test_continuation_facade.0/README.md)
  - [bbugyi200.athena.toobig-5d.test_gate_capacity_e2e.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5d.test_gate_capacity_e2e.0/README.md)
  - [bbugyi200.athena.toobig-5d.test_monitor_followup.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5d.test_monitor_followup.0/README.md)
  - [bbugyi200.athena.toobig-5d.test_monitor_followup_prompt.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5d.test_monitor_followup_prompt.0/README.md)
  - [bbugyi200.athena.toobig-5d.test_prompt_panel_section_navigation_targets.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5d.test_prompt_panel_section_navigation_targets.0/README.md)
- **COMMITS:**
  - [a59ded7](https://github.com/sase-org/sase/commit/a59ded7669c59b7e80fc5d1794ff6e15f7e26c93)
    — feat(ace,pager): open agent metadata in the pager with V
  - [aecb02d](https://github.com/sase-org/sase/commit/aecb02db26d979e6d64bed8acfa552fc2f84900c)
    — docs(agents): clarify fleet capacity and recovered waits
  - [fd30496](https://github.com/sase-org/sase/commit/fd30496f76baacdf73776879c4c1d9017869c31c)
    — docs(agents): tighten fleet capacity and shell-retry wait wording
  - [e87a5b4](https://github.com/sase-org/sase/commit/e87a5b459528ecc3a3dfc039e5c117b48ee1beb1)
    — test(ace): split test_agent_wait_resume.py into focused files
  - [5d26cff](https://github.com/sase-org/sase/commit/5d26cff74615ca25be6025973bb856fa798187e8)
    — test(ace): split prompt panel navigation target tests
  - [20a7d39](https://github.com/sase-org/sase/commit/20a7d397e66b8ffba7bcc43f260c5b2053ff6f4f)
    — test(core): split test_continuation_facade.py into focused files
  - [fc9a831](https://github.com/sase-org/sase/commit/fc9a8317c5cf7bf7d79863c1a82e952d7486d175)
    — test(fakey): split gate capacity e2e tests
  - [70712f0](https://github.com/sase-org/sase/commit/70712f0c806f5b3bb1f7984e18b460b8cd958b57)
    — test(main): split test_artifact_cli_link_health.py into focused files
  - [33c3c4c](https://github.com/sase-org/sase/commit/33c3c4caa6183fadcbad754844cb503e2e858855)
    — test(monitor): split monitor follow-up tests
  - [d6f35ed](https://github.com/sase-org/sase/commit/d6f35ed6d8b97fadcbe267c3d6bab72953949823)
    — test(monitor): split monitor follow-up prompt tests

# `V` on the Agents tab: open the metadata panel in the pager

## Motivation

The Agents-tab metadata panel (`AgentDetail` — "metadata panel" in `docs/ace.md`) is the
richest per-agent surface in ACE, but it lives in a cramped side panel: no full-screen
reading, no vim `/` search (only the `,/` leader-mode panel search), no goto-line, no
section jumps, no labeled link targets. The sase pager already provides all of that.
This tale adds a `V` keymap on the Agents tab that renders the selected agent's metadata
as a first-class sectioned pager document — for running and done agents alike — and adds
the pager support needed to host it excellently: an `agent` origin with its own
glyph/accent chrome, and a live document-refresh hook so `r` re-snapshots a running
agent instead of merely recomposing a frozen document.

## Current state (verified against the tree)

- `V` is bound app-wide to `show_agent_run_log` (`src/sase/default_config.yml:628`,
  `src/sase/ace/tui/bindings.py:273`, `src/sase/ace/tui/keymaps/metadata.py:168`,
  `src/sase/ace/tui/keymaps/app_keymaps.py:151`) but is **inert on the Agents tab**: the
  action body early-returns unless
  `current_tab in {"artifacts", "patches", "changespecs"}`
  (`src/sase/ace/tui/actions/patch/_core.py:432-448`). It is, inconsistently, listed in
  `_LOCAL_AGENT_ROW_ACTIONS` (`src/sase/ace/tui/_app_action_availability.py:68`).
- The config already documents a tab-disjoint key precedent: `r` is `agents_refresh` on
  Agents while other tabs keep `refresh`/`run_workflow`; the disambiguation lives in
  `check_app_action` (`src/sase/ace/tui/_app_action_availability.py:118-123`). Textual
  dispatches a shared key to the binding whose `check_action` allows it.
- Lowercase `v` (`view_files`) is the end-to-end precedent for "Agents tab → off-thread
  document build → `PagerScreen`": `src/sase/ace/tui/actions/hints/_files.py`
  (`_view_agent_files_impl` at :270, `build_pager_document` at :57,
  `_view_files_with_pager_screen` at :435-458) and
  `src/sase/ace/tui/actions/hints/_view_processing.py` (off-thread materialization at
  :259-331, agent link-context capture at :333-346, stale-render session/identity
  guards).
- The pager (`src/sase/pager/`) has no metadata-panel concept and no agent origin:
  `PagerOrigin` is `bead|file|diff|research` (`src/sase/pager/link_scan.py:39-50`). Its
  `r` refresh only recomposes the already-frozen document (`action_refresh`,
  `src/sase/pager/_screen_body.py:68-74`); sections are parsed once at document
  construction. `PagerSection.body` accepts any Rich renderable
  (`src/sase/pager/document.py:67-118`), and `agent_link_context()` already exists
  (`src/sase/pager/link_context.py:60-74`).
- Running and done agents share one dataclass (`Agent`,
  `src/sase/ace/tui/models/agent.py:60`, over `AgentState`,
  `src/sase/ace/tui/models/_agent_state.py`). Liveness is `status` (buckets in
  `src/sase/agent/status_buckets.py`) plus `runner_is_live`; done rows add
  `response_path`, `stop_time`, archive fields. Remote fleet rows (detected via
  `fleet_origin_alias`) carry far less metadata and are excluded from every local row
  action (`src/sase/ace/tui/_app_action_availability.py:139-140`).
- Metadata renderers to reuse, so the panel and the pager never tell different stories:
  `build_header_text()`
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py:79-120`),
  `append_agent_metadata_fields()`
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py:505-560`),
  heading/key-value helpers (`src/sase/ace/tui/widgets/prompt_panel/_helpers.py`), the
  aligned bead table
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_bead_section.py:46-115`), and the SASE
  CONTEXT lanes (`src/sase/ace/tui/widgets/prompt_panel/_agent_context.py`).
- No Rust core change is needed: pager section bodies are supplied entirely by Python
  callers, and this feature is presentation-only, so it stays on the Python side of the
  core boundary.

## Design

### UX

- On the **Agents tab**, with a **local** agent row selected (starting, running,
  waiting, or done — provenance does not matter), `V` opens the pager full-screen with
  that agent's metadata as a multi-section document.
- The subject line reads like every other pager document: agent glyph + accent, the
  presented agent name as the document title, current section title, `i/N`, scroll
  percentage.
- Sections mirror the metadata panel's existing grouping and order (skip any that would
  be empty):
  1. **IDENTITY** — name, family/clan/tribe, status (+ bucket), retry chain.
  2. **MODEL** — model, alias, provider, reasoning effort, fallback model.
  3. **WORKSPACE** — workspace number and directory, project, VCS provider, linked
     repos.
  4. **TIMELINE** — start/run/wait/stop timestamps and derived durations; a running
     agent shows elapsed-so-far instead of a total.
  5. **CONTENT** — response/diff/output/plan paths, artifacts dir, extra files.
  6. **SASE CONTEXT** — the context lanes.
  7. **BEAD** — the aligned bead field table, when the agent has one.
- Every pager affordance works: `/` search, `;` goto, `ctrl+n`/`ctrl+p` section jumps,
  `y` copy labels, the trail. Artifact refs inside the metadata resolve through the
  agent's link context exactly as in the `v` (view files) flow.
- `r` inside the pager re-snapshots the agent — fresh row state, rebuilt document —
  which is the payoff for running agents whose status, timestamps, and paths evolve.
- Remote fleet rows: `V` is unavailable (footer hides it), consistent with every other
  local row action. Other tabs keep their current `V` (Agent Run Log modal).
- No selected agent (whole panel focused, empty list): the action is unavailable.

### Key-conflict resolution

Introduce a **new action id `view_agent_metadata`** bound to `V`, tab-disjoint with
`show_agent_run_log` exactly like the `r` precedent. Do not repurpose or rename
`show_agent_run_log`.

## Implementation steps

### 1. Keymap plumbing for `view_agent_metadata`

- `src/sase/ace/tui/keymaps/app_keymaps.py`: add field `view_agent_metadata: str`.
- `src/sase/ace/tui/keymaps/metadata.py`: add
  `("view_agent_metadata", "Agent Metadata", False)`. (An import-time check in
  `src/sase/ace/tui/keymaps/types.py:84-96` enforces the 1:1 match with `AppKeymaps`;
  both edits must land together.)
- `src/sase/default_config.yml`: in the `ace.keymaps.app` Agents sub-tab comment group
  (around lines 545-562), add `view_agent_metadata: "V"` with a comment mirroring the
  existing `r` note, e.g. "Tab-disjoint with show_agent_run_log: Agents uses V to open
  the metadata panel in the pager; other tabs keep V for the Agent Run Log modal."
- `src/sase/ace/tui/bindings.py`: add the fallback
  `Binding("V", "view_agent_metadata", "Agent Metadata", show=False)`.

### 2. Availability gating

In `src/sase/ace/tui/_app_action_availability.py`:

- Add `"view_agent_metadata"` to `_LOCAL_AGENT_ROW_ACTIONS` so the existing prompt-input
  and remote-fleet-row suppressions apply.
- Remove `"show_agent_run_log"` from `_LOCAL_AGENT_ROW_ACTIONS` (it is not an Agents row
  action; this is the latent inconsistency) and add an explicit gate
  `if action == "show_agent_run_log" and app.current_tab == "agents": return False`
  alongside the `refresh`/`run_workflow` gates, so `V` cleanly falls through to the new
  action on Agents and behaves exactly as before everywhere else.
- Gate `view_agent_metadata` itself: available only when `current_tab == "agents"` and a
  local (non-fleet) agent row is selected (`_selected_agent` is not `None`). This also
  drives correct conditional footer display.

### 3. Metadata pager document builder

New module `src/sase/ace/tui/actions/agents/_metadata_pager_document.py` (pure build
code, no widget access) exposing roughly
`build_agent_metadata_document(agent: Agent) -> PagerDocument`:

- Compose one `PagerSection` per UX section above, reusing the prompt_panel helpers
  (headings, `format_meta_key` label styling, the bead table, context lanes) so styling
  stays identical to the panel. Bodies are Rich `Text`/renderables; skip empty sections;
  never assume done-only fields (`stop_time`, `response_path`) are present, and handle
  family/clan container rows without crashing (render the same aggregate content the
  panel renders for them, or a minimal IDENTITY section).
- Document title: the presented agent name; `origin`: the new agent origin (step 4);
  give sections a stable `identity` so pager refresh can keep the reader's place.
- Set `link_anchors`/plain text such that artifact refs in the metadata are scannable,
  mirroring `build_pager_document` in `src/sase/ace/tui/actions/hints/_files.py:57-85`.
  Making raw file paths followable targets is explicitly out of scope (candidate
  follow-up bead).
- This function must be safe to call off the event loop (it may stat paths, so it will
  only ever be called from a worker thread).

### 4. Pager support: `agent` origin + chrome

- Add an `agent` member to `PagerOrigin` (`src/sase/pager/link_scan.py:39-50`) and sweep
  every `PagerOrigin` match/dispatch site for exhaustiveness (link scanning should treat
  it like `bead` document scanning unless a site clearly needs otherwise).
- Add glyph + accent entries for the agent document/section kind in the tables used by
  `subject_line()`/`section_rule()` (`src/sase/pager/_chrome.py:15-46`). Reuse an
  existing agent-ish icon from `ARTIFACTS_ICONS`/`ARTIFACTS_ACCENTS` if one exists;
  otherwise add one consistent with the set. The result must look deliberate in the
  subject line, section rules, and footer — this is the "beautiful" bar.

### 5. Pager support: live document refresh

- `PagerScreen.__init__` (`src/sase/pager/screen.py:78-110`) gains keyword-only
  `refresh_document_fn: Callable[[], PagerDocument | None] | None = None` (thread the
  same parameter through `SasePager` in `src/sase/pager/app.py` for parity, defaulting
  to `None`).
- `action_refresh` (`src/sase/pager/_screen_body.py:68-74`): when the provider is
  `None`, behavior is unchanged. When present, run it via a thread worker
  (`run_worker(..., thread=True)` — never on the pump, per the TUI perf rules), with a
  coalescing "refresh in flight" guard released in `finally`. On the UI thread, swap in
  the returned document: reset `_body`/`_body_width`/label layer/dangling refs/goto and
  search transients, re-run subject/trail/footer updates, and restore the scroll
  position clamped (prefer keeping the current section via section `identity`). A `None`
  return or an exception keeps the current document and shows a brief footer status
  (`_set_footer_status`, `src/sase/pager/_screen_chrome.py:17-19`).
- Contract note on the parameter docstring: the provider is invoked from a worker thread
  and must be thread-safe.

### 6. The `V` action on the Agents tab

New small mixin module `src/sase/ace/tui/actions/agents/_metadata_pager.py`, mixed into
the agents action family the same way existing row actions are:

- `action_view_agent_metadata()`: thin, `tui_trace("agents.view_agent_metadata")`,
  self-gate on `current_tab == "agents"`, resolve the row via `_get_selected_agent()`
  (`src/sase/ace/tui/actions/agents/_selection.py:236-245`), bail quietly on `None` or a
  fleet row.
- Build off the pump: snapshot the agent row and link context on the UI thread (reuse
  the capture pattern of `_capture_view_link_context`,
  `src/sase/ace/tui/actions/hints/_view_processing.py:333-346`), build the document via
  `asyncio.to_thread(build_agent_metadata_document, agent)`, and re-check the selected
  identity/session after the await before pushing (stale-render guard, as in
  `_render_agent_hint_document`).
- Push
  `PagerScreen(document, links_enabled=True, resolve_ref_fn=..., refresh_document_fn=...)`
  mirroring `_view_files_with_pager_screen`
  (`src/sase/ace/tui/actions/hints/_files.py:435-458`), with the document's
  `link_context` built by `agent_link_context(...)`.
- The refresh provider closure: look up the agent's current row by stable identity from
  app state via `call_from_thread` (it may have progressed, finished, or been
  dismissed), then build the fresh document in the calling worker thread; return `None`
  when the row no longer exists.

### 7. Discoverability surfaces

- Agents help modal: add the `V` row in
  `src/sase/ace/tui/modals/help_modal/agents_bindings.py` (near `view_files` at :124);
  keep the patches/axe help rows for `show_agent_run_log` untouched. Respect the
  help-modal width conventions in `src/sase/ace/CLAUDE.md` (description ≤ 32 chars).
- Conditional footer: add to `_compute_agent_bindings()`
  (`src/sase/ace/tui/widgets/_keybinding_bindings.py:105`) — it qualifies because
  availability depends on the selected row (local vs remote vs none); alphabetical
  ordering is handled by the footer convention.
- Command palette: `src/sase/ace/tui/commands/_availability_agents.py` and
  `src/sase/ace/tui/commands/_app_metadata_actions.py` gain `app.view_agent_metadata`
  entries.
- Docs: `docs/ace.md` — the metadata panel section (~:4678) and the Agents key tables
  mention `V`; `docs/pager.md` — document the agent origin and the refresh-provider
  behavior of `r`.

### 8. Tests

- `tests/ace/tui/`:
  - `V` on a **running** local row and on a **done** local row builds a document with
    the expected sections and pushes `PagerScreen` (assert section titles and that
    done-only fields are absent for the running row).
  - `V` is suppressed on remote fleet rows, with no selection, and while a prompt input
    owns keys (availability tests for the `_LOCAL_AGENT_ROW_ACTIONS` membership changes,
    including `show_agent_run_log` now disabled on Agents).
  - Patches-tab `V` still opens `AgentRunLogModal` (regression for the tab-disjoint
    split).
  - Family/clan container row: `V` does not crash and renders the container document.
- `tests/pager/`:
  - Agent origin: subject-line glyph/accent and section rules render (extend
    `test_chrome.py` patterns).
  - Refresh provider: `r` swaps the document and preserves/clamps scroll; provider
    returning `None` keeps the document and sets a footer status; provider raising keeps
    the document; no provider → legacy recompose behavior unchanged.
- Keymap defaults: existing import-time parity check plus any defaults-parse test pick
  up the new id; update whatever availability/footer snapshot tests enumerate agent
  bindings.

## Verification

- `just check` must pass (includes the `sase validate` freshness gate).
- Targeted suites while iterating: `pytest tests/pager`,
  `pytest tests/ace/tui -k "metadata or availability or keymap or view"`, and
  `pytest tests/main/test_pager_command.py tests/test_cli_pager.py` (to prove the
  standalone pager is unregressed by the constructor change).
- Manual smoke: `sase ace` → Agents tab → `V` on a running agent, `r` inside the pager
  while it progresses, `V` on a done agent, `q` back; confirm `V` on the Patches tab
  still opens the Agent Run Log modal.

## Risks and notes

- Two `V` bindings coexist in one `BindingsMap`; correctness depends on the
  `check_app_action` gates being disjoint (agents vs non-agents). The `r` key is the
  proven precedent; the regression test in step 8 pins it.
- `keymaps/types.py` enforces `_BINDING_META` ↔ `AppKeymaps` parity at import time:
  partial keymap edits fail fast rather than subtly.
- Pager perf: the document is built off-thread and sections are frozen once, per the
  existing pager contract (`_ensure_body` recomputes only width-dependent layout); the
  refresh provider must never run on the message pump.
- Do not add a `pager` keymap-config scope in this tale; pager keys stay Textual
  `BINDINGS` (no new pager keys are introduced).
