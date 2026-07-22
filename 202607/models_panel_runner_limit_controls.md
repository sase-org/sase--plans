---
tier: tale
title: Models panel running-agent limit controls
goal: 'The Models panel clearly shows the effective maximum-running-agents limit and
  lets users safely edit its persistent configuration or apply and clear a machine-wide
  temporary override with Ctrl+R, while runner admission, capacity displays, and documentation
  all agree on the same live value.

  '
create_time: 2026-07-22 07:12:30
status: done
---

- **PROMPT:** [202607/prompts/models_panel_runner_limit_controls.md](prompts/models_panel_runner_limit_controls.md)

# Plan: Models panel running-agent limit controls

## Background and scope

SASE already has a schema-validated top-level `max_running_agents` setting with a package default of `10`. The runner
slot gate rereads that setting while implicit global-cap waiters are parked, the Agents header renders current occupancy
as `R/L`, and the Statistics runners view uses the current limit as a present-day reference. Persistent edits are
possible through the Config Center, but the Models panel—the existing home for model and launch-default controls— does
not show the setting or offer the focused edit/override experience that default effort now has.

The recently added `Ctrl+E` default-effort workflow establishes the product language to follow: an authoritative status
in the Models header, a compact edit/override/clear action card, truthful configured-versus-temporary state,
source-preserving persistent writes, shared duration and exact-time cards, bounded worker-backed state access, and
strong mounted plus visual coverage. The running-agent limit differs in two important ways: it is an arbitrary positive
integer rather than a fixed vocabulary, and changing it affects live admission decisions and capacity displays even
though it must never terminate an already-admitted agent.

This is a **tale**. The shared state contract, effective-limit accessor, TUI flow, and admission/display integration
form one end-to-end behavior that one follow-up coding agent should implement and verify together. It crosses into the
linked Rust core, but splitting that contract from its first consumer would create an avoidable period where the panel
and runner gate disagree.

The existing default remains `10`; this work does not add a second configuration key or change the schema's minimum of
`1`. It also does not make `max_running_agents` an absolute ceiling for launches carrying an explicit `%wait(runners=N)`
threshold—the current explicit-threshold behavior remains intact and must be explained wherever the temporary control is
presented.

## Product and interaction design

### Authoritative Models-panel status and entry point

- Add a fixed panel-local `Ctrl+R` binding named for managing the running-agent limit. It is intentionally distinct from
  lowercase `r`, which continues to reset the highlighted model alias, and it does not become a new configurable
  leader-mode key. `Ctrl+R` must work from an alias, a collapsed bucket, or a drilled-in bucket because the limit is
  global rather than selection-specific.
- Extend the centered Models title with a separate launch-control line below default effort:
  - configured/default value only: `max running agents: 10`;
  - active temporary value: `max running agents: 4  override · 42m left  configured 10`;
  - until-cleared value: the same layout with `override · until cleared`. Use the established cyan capacity color for
    the number, violet for temporary-override state, dim text for provenance, and explicit words in addition to color.
    Format `1 agent` versus plural `agents` where a sentence uses the unit.
- Rework the footer deliberately instead of allowing the new hint to wrap accidentally. Give the two global controls a
  stable, concise lane such as `ctrl+e=Effort  ctrl+r=Limit`, then retain the existing row-sensitive alias actions and
  navigation lane. Both top-level bucket and alias variants must remain balanced at 120 columns and readable without
  clipping at the supported 80×40 size.
- Initialize the limit line to the known package default of `10`, then replace it from one off-thread snapshot. A small
  snapshot-only timer may update remaining time and locally expire the captured override, but title/footer/highlight
  rendering must never read configuration, acquire a state lock, or touch disk.

### Edit, override, or clear chooser

`Ctrl+R` opens a compact **Max Running Agents** action card parallel to **Default Effort**:

- Its status block is labeled `Current global-cap limit`. Without an override it shows the configured integer. With an
  override it shows the temporary integer plus remaining time and a second `Configured: N agents` line.
- `e  Edit permanently` says that it will preview and write the user `sase.yml`, or the chezmoi source when home config
  is managed there.
- `o  Override temporarily` says that it leaves configuration unchanged and will ask for a duration.
- `x  Clear temporary override` appears and is actionable only when a temporary runner-limit override is active.
- A concise, always-visible note states both safety rules: already-running agents continue even if the limit is lowered,
  while launches with an explicit `%wait(runners=N)` keep their own initial-admission threshold. `Esc`/`q` cancels.

Use the same aligned one-key rows, calm surface, strong status inset, double-border visual language, and keyboard-only
speed as the effort card, but give capacity values a cyan identity so the two controls remain easy to distinguish. Share
card layout/style primitives where doing so prevents drift; keep the domain-specific status copy and color rather than
forcing both controls through opaque generic strings.

### Positive-integer value card

Both Edit and Override continue to a focused **Running Agent Limit** value card rather than reusing the effort ladder.
The mode-specific subtitle says whether the value will be persistent or temporary. The card should:

- prefill and select the configured value in Edit mode, and the current effective value in Override mode;
- accept an ordinary base-10 whole number with no whitespace surprises, reject empty, signed, fractional, boolean-like,
  or nonnumeric input, and enforce the existing schema minimum of `1` without inventing a new product maximum;
- show live inline validation, `minimum 1 · package default 10`, and an Enter-to-continue / Esc-to-cancel footer;
- retain focus and the entered text after an invalid submission, and emit no write or duration modal until the value is
  valid.

Use the repository's single-line input conventions, including select-all on mount and familiar cursor editing. The
number should read as the visual focal point, not as a generic form field. Edit proceeds to a persistent preview;
Override proceeds to the existing duration picker. There is no pseudo-value for a temporary “default”: cancelling or
clearing honestly resumes configured behavior. The full Config Center remains the place to remove a user-layer key or
target arbitrary overlay scopes.

### Persistent edit flow

Plan a source-preserving `set` of the top-level `max_running_agents` field against the writable **user base** config
layer, matching the scope policy of the Models-panel default-effort editor and never writing a project-local config.
Reuse/generalize the existing scalar preview scaffold so the confirmation screen shows:

- the exact `max_running_agents` path and requested positive integer;
- the actual target path, including an explicit chezmoi-source annotation when remapped;
- configured before/after values from the Rust-backed config edit plan;
- schema diagnostics, no-op handling, and the exact YAML diff with comments/order preserved;
- when a temporary runner override is active, a prominent note that it remains admission-effective until expiry or clear
  even though the configured value is changing.

Only `y`, Enter, or `Ctrl+S` writes. Planning, file I/O, chezmoi application, Git discovery, and any follow-up commit
work stay off the Textual event loop. Do not report success if planning, validation, writing, or chezmoi application
fails. After success, reload the configured value from the effective config snapshot rather than assuming the requested
integer won over every higher-precedence overlay; preserve any active temporary override, show a precise notification,
and offer the established tracked commit/pull/push flow with a focused subject such as
`chore: update max running agents`.

Immediately request an Agents refresh through the existing coalesced refresh entry point so the `R/L` header and queued
context see the new effective value without closing the Models panel or rebuilding rows directly.

### Temporary override flow

After a valid Override value, push the exact shared `DurationPickerModal`. Preserve every existing choice and back path:
`15m`, `30m`, `1h`, `2h`, `4h`, until cleared, custom combined durations, and `t` for the shared configured-timezone
absolute-time panel. Setting a new value replaces the prior runner-limit override.

Set, replace, and clear override state in worker threads. Suppress duplicate writes while one is active, keep the parent
panel from closing as though a write had completed, cancel owned workers on unmount, and re-check mount state before
applying results. On success, update the captured snapshot without moving the selected alias, notify with the chosen
limit and compact duration, and request the standard Agents refresh. On error, retain the prior snapshot and display an
actionable error. Expiry needs no destructive action: parked implicit-cap agents and normal Agents refreshes will
re-evaluate the effective accessor, while the open Models panel locally falls back to its configured snapshot as soon as
the captured deadline passes.

## Shared backend state and effective-limit semantics

Temporary global runner capacity is cross-process domain state, so implement it in the linked `sase-core` repository and
expose a stable PyO3 wire API. Python should provide strict typed rehydration, configuration composition, and TUI
orchestration—not a second file-locking implementation.

Store the override separately at `~/.sase/max_running_agents_override.json` with its own bounded lock. A versioned wire
record contains the positive integer limit, `created_at`, optional `expires_at`, and non-empty `source`. The Rust domain
must provide get, set-relative, set-until, and idempotent clear operations and must:

- reject zero/negative or non-integer values and invalid, non-finite, non-positive, or non-future time inputs;
- accept the SASE home and clock at the domain seam so `$SASE_HOME` and deterministic tests continue to work;
- serialize writers with a bounded advisory lock and atomically replace a same-directory temporary file so readers never
  observe partial JSON;
- treat `now >= expires_at` as expired and self-clean expired, malformed, wrong-version, missing-field, or invalid-value
  state;
- preserve the already-published effort-override wire shape and behavior if low-level bounded-lock, atomic-write, and
  timed-expiry helpers are extracted for reuse.

Give configuration two explicit meanings in Python:

- a configured accessor that validates the merged `max_running_agents` value and falls back to package default `10`;
- an effective snapshot/accessor that chooses an active temporary override first and the configured value second.

Keep the public `get_max_running_agents()` name as the effective admission limit so its established consumers acquire
the new behavior without parallel resolution logic. Add a clearly named configured-only accessor for the Models snapshot
and persistent preview. Strictly validate the Rust record in the Python facade (exact fields/version, positive integer
excluding `bool`, finite timestamps, non-empty source) before it can influence admission or rendering.

The effective limit applies to every existing global-cap decision:

1. launches without an explicit `%wait(runners=N)` use `effective_limit - 1` as their existing-runner threshold;
2. already-parked implicit waiters recompute it on every poll, so raising or lowering the override takes effect within
   the normal polling interval;
3. a question continuation reacquires against the current effective global limit, as it already does for live config
   changes;
4. an explicit `%wait(runners=N)` initial admission retains its stored explicit threshold and does not get rewritten by
   the global override.

Lowering the effective limit below current occupancy is non-preemptive: no process is killed or forced to yield, and no
new implicit-cap participant is admitted until occupancy falls enough for the new threshold. Raising it allows eligible
parked agents to advance through the existing priority/FIFO gate; it does not bypass queue ordering. If temporary-state
access fails because of bounded lock or I/O contention, admission must fail closed for that poll—keep/publish the agent
as waiting, release locks, and retry—instead of crashing the agent or silently admitting against a possibly higher
configured value. TUI reads retain their last good snapshot and surface a warning.

## TUI integration, naming, and performance

- Add a focused runner-limit Models-panel mixin plus small render/action/value/edit modules, keeping `models_panel.py`
  as the stable facade and monkeypatch seam. Reuse duration/exact-time components directly and share the scalar preview
  and action-card mechanics with effort where the abstraction stays readable.
- Capture configured limit, active override, chezmoi mode, and one clock value off-thread. Coordinate this with the
  effort snapshot load when practical so opening the panel does not launch redundant config reads, but keep typed effort
  and runner snapshots rather than an unstructured dictionary.
- Treat the Agents header and Statistics runners view as consumers of the **effective** current limit. Rename internal
  fields such as `configured_runner_limit`/`configured_limit` where they would otherwise become misleading, while
  preserving the atomic in-memory `RunnerCapacitySnapshot` boundary and local-refilter behavior.
- Route successful edits/sets/clears through `request_agents_refresh(...)` and the current cached-then-background reload
  path. Do not add a second agent scan or full-list rebuild path, and do not do state reads from render, key, timer, or
  highlight handlers.
- Keep programmatic OptionList highlight changes behind the existing synchronous guard and preserve the exact alias or
  bucket cursor across every limit action. The new input modal owns focus while typing; global panel bindings must not
  intercept its digits or editing keys.
- Do not add a new top-bar pill. The Models title/action card are the configuration authority, while the existing Agents
  `R/L` capacity chip is the operational view. This avoids another always-visible status element and keeps the two
  surfaces complementary.

## Testing and visual acceptance

Add focused coverage at every seam:

1. **Rust core and binding** — positive limits including the boundary `1`; invalid values and times; relative, exact,
   no-expiry, replacement, boundary-expiry, malformed-state cleanup, idempotent clear, bounded lock behavior, atomic
   persistence, deterministic SASE-home paths, registered PyO3 functions, and stable round-trip records. If shared timed
   state helpers are extracted, retain all effort-override regressions.
2. **Python facade and admission** — strict wire rejection (including `bool`), configured fallback `10`, temporary-over-
   configured resolution, local expiry fallback, binding errors, and effective `get_max_running_agents()` behavior.
   Exercise immediate and parked implicit launches, live override replacement/expiry, explicit `%wait(runners=N)`
   isolation, question-slot reacquisition, fail-closed read errors, priority/FIFO preservation, and a lowered limit
   below current occupancy without preemption.
3. **Capacity consumers** — Agents worker/apply snapshots and header colors/counts use the effective limit and survive
   local refilters; Config Center and Models writes schedule the existing refresh; the Statistics runners reference
   reads the same current effective value without pretending it is historical configuration.
4. **Persistent edit** — user-base targeting, values `1`, `10`, and a larger integer, schema validation, effective
   before/after preview, no-op and higher-precedence-overlay truthfulness, YAML comment/order preservation, cache clear,
   chezmoi remap/apply success and failure, active-override warning, snapshot reload, non-Git skip, and the focused
   commit offer.
5. **Mounted TUI flows** — `Ctrl+R` and the redesigned footer in alias/collapsed/open-bucket states; configured and
   active-override title/status forms; conditional Clear; Edit/Override prefill differences; valid and invalid numeric
   input; persistent preview; shared relative/custom/exact duration paths; replace/clear/expiry success; worker errors,
   duplicate/busy guards, Agents-refresh request, cursor preservation, cancel/back behavior, and clean teardown.
6. **PNG and narrow layouts** — update all Models-panel goldens intentionally for the new title/footer, and add 120×40
   goldens for an active runner-limit override, the action chooser, both numeric-editor modes, and the persistent
   preview (including chezmoi or active-override provenance). Inspect every changed PNG and add 80×40 region assertions
   so the three-line title, two-lane footer, and new cards stay within the viewport without incidental wrapping or
   clipping.

Run focused Rust and Python tests while iterating. Before handoff, run `cargo fmt --all -- --check`, strict workspace
clippy, and `cargo test --workspace` in the linked core; rebuild/install the local binding consumed by the Python
checkout; run the focused Models/admission/capacity suites and dedicated visual snapshots; inspect generated PNG
actual/expected/diff artifacts; then run the repository-required `just install` followed by `just check` in SASE.

## Documentation and acceptance criteria

Update the Models-panel section of `docs/ace.md`, the `max_running_agents` configuration reference, runner-slot
troubleshooting, `%wait(runners=N)` documentation, and Statistics wording so they consistently distinguish configured
and effective limits, describe the fixed `Ctrl+R` flow, identify the state file, and explain expiry, replacement,
question reacquisition, explicit-threshold precedence, queue timing, and non-preemptive lowering. Correct stale “root
agents only” descriptions in docs/schema/default-config comments to the current slot-participant definition (top-level
user agents and eligible parallel family members; serial follow-ups, workflow Python/bash steps, and axe runners remain
excluded). Document `Ctrl+R` as a fixed Models-modal binding, not a leader-keymap setting.

The feature is complete when a user can open the Models panel and immediately understand both the configured and
currently effective running-agent limit; press `Ctrl+R` from any row; persist a validated positive integer through a
truthful user/chezmoi preview or apply it temporarily through the familiar duration workflow; clear the temporary value
from the same card; and see implicit runner admission, question reacquisition, Agents capacity, and current Statistics
reference converge on that value without killing active agents, bypassing explicit wait semantics, moving the Models
cursor, blocking the event loop, or introducing visual regressions.
