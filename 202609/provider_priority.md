---
tier: epic
title: Temporary provider priority in Provider Routing
goal:
  Let users temporarily favor one provider in model alias pools through Provider
  Routing, with reversible state, reliable fallback behavior, and clear visual feedback
  throughout ACE.
phases:
  - id: priority-core
    title: Add provider priority state and policy to Rust core
    depends_on: []
    size: medium
    description:
      "priority-core: Add independent priority persistence, coherent routing snapshots,
      pure availability policy, Python bindings, and core/binding tests."
  - id: priority-routing
    title: Integrate provider priority into routing and cached previews
    depends_on:
      - priority-core
    size: medium
    description:
      "priority-routing: Add typed Python adapters, one captured routing context per
      resolution, provenance, cache invalidation, and routing parity tests."
  - id: priority-tui
    title: Ship the prioritize interaction and visual treatment
    depends_on:
      - priority-routing
    size: medium
    description:
      "priority-tui: Add duration and clear controls, priority and backup presentation,
      live refresh, documentation, interaction tests, and PNG verification."
proposed_by: bbugyi200.athena.0gs
bead_id: sase-xf
create_time: 2026-09-09 19:52:39
status: wip
---

- **PROMPT:**
  [prompts/202609/provider_priority.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/provider_priority.md)
- **BEAD:**
  [sase-xf](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xf/README.md)

# Temporary provider priority

Users should be able to spend remaining capacity on one provider before its usage window
resets. In SASE Admin Center → Config → Launch → `p` Provider Routing, selecting CODEX
and pressing `p`, then choosing `2h`, will favor CODEX in model alias pools for two
hours. Other providers remain usable backups. A blue `★ priority` badge, an explicit
time window, and a clear action make the preference visible and easy to undo.

This is an epic because shared state and policy cross the Rust/Python boundary, multiple
launch and preview paths must agree, and the interface needs its own interaction and
visual verification. Each phase is bounded direct implementation work. The dependency
chain is intentional: core contract → routing → interface. Plan authoring is xlarge work
under the canonical size guidance.

## Existing implementation and boundaries

Provider Routing currently binds `d`/Enter to hard disable, `s` to soft disable, and `x`
to enable in `src/sase/ace/tui/modals/models_panel_provider_modal.py`. Duration
selection, write workers, rendering, and snapshots are split into the adjacent
`models_panel_provider_*` modules. The pane reuses model alias duration choices and
refreshes its parent through a captured snapshot and `provider_routing_changed`.

`src/sase/llm_provider/provider_disable.py` is a Rust binding facade. The Rust core's
`crates/sase_core/src/provider_disable.rs` persists actual disables in
`llm_provider_disables.json`, with schema-1 migration, schema-2 records, atomic writes,
and a 250 ms bounded state lock. Availability is currently layered in `registry.py` and
`model_alias_resolution_types.py`; the `PREFERRED`, `SPARING`, and `UNAVAILABLE` values
feed the established selection masks in `load_balancing.py`.

All new persistence, validation, eligibility precedence, and priority-derived
availability policy belong in Rust. Python owns provider/plugin facts and thin adapters;
Textual owns interaction and rendering. Open `sase-core` with `/sase_repo` and use the
returned checkout path. Do not assume a sibling or numbered workspace path. Existing
selector grammar and cursor accounting remain the integration seam.

## Product contract

### Setting, changing, and clearing priority

- There is one machine-wide temporary priority record, initially absent. It applies
  wherever the existing provider-disable policy is consumed for a new routing decision.
  It survives ACE restarts and is independent of project selection.
- `p` on an eligible provider opens **Prioritize CODEX**. Reuse `15m`, `30m`, `1h`,
  `2h`, `4h`, Until cleared, custom duration, and exact local date/time. The copy is
  “Prefer CODEX in pools; other providers remain backups.” Exact-time submission says
  **Set priority**. Escape and Back never write state.
- `p` on the current priority opens **Change CODEX priority** to replace its window. `p`
  on another eligible provider opens **Prioritize CLAUDE**, with a subtitle explaining
  that it replaces the current CODEX priority after a duration is chosen. Cancelling
  preserves the current priority. Do not silently carry over an old duration or guess a
  provider's reset time.
- A conditional `c` **Clear priority** action is available from every row while any
  priority exists. It clears just this one record, immediately and idempotently. It
  remains reachable when the preferred provider has been uninstalled or hidden, or the
  list has no selectable rows. This also makes recovery independent of row selection.
  The footer names the provider being cleared in the description strip.
- Existing `d`, Enter, `s`, and `x` retain their disable/enable meanings. In particular,
  `x` clears a real disable; it does not clear global priority. On a priority-derived
  backup row with no actual disable, explain “Enabled; CODEX has priority. Press c to
  clear priority.” Do not report it as manually soft-disabled.
- Only a registered, user-facing provider with an available CLI and no active real hard
  or soft disable can become the priority. A disabled target explains “Enable CODEX with
  x before prioritizing it”; a missing CLI explains its unavailability. Recheck
  eligibility when saving, since the duration picker may have stayed open. A rejected
  write preserves any previous priority.
- A real disable or missing CLI appearing after priority is set retains the priority
  intent and its original expiry. The display explains **priority unavailable** or
  **priority soft-disabled**, with the real reason. Routing uses available backups; the
  preference becomes effective again if the provider recovers before expiry. Disabling,
  enabling, or automatically disabling any provider never rewrites the priority record.
  Clearing priority never restores stale copies of disable state.

### Exact routing semantics

At one captured clock, priority P has the same selection effect as applying a virtual
soft disable to every other provider. Existing hard disables always win. Actual disable
records and their source/expiry continue to exist independently. The priority overlay is
derived at resolution time and never materializes synthetic disable records into the
store.

| Provider condition                                   | Effective state with priority P | Explanation                                    |
| ---------------------------------------------------- | ------------------------------- | ---------------------------------------------- |
| Unregistered, CLI missing, or actually hard-disabled | UNAVAILABLE                     | Ordinary unavailability wins, including for P. |
| Actually soft-disabled, including P                  | SPARING                         | An existing soft disable retains its meaning.  |
| Usable P with no real disable                        | PREFERRED                       | Pool members on P receive the preference.      |
| Any other usable provider                            | SPARING                         | It is a backup because P has priority.         |
| Priority absent or expired                           | Existing disable-derived state  | Current real disables determine routing.       |

This applies the overlay even when P is unavailable or absent from a particular pool:
the other usable members are all sparing and the existing all-sparing rule keeps the
pool rotating. Preserve the exact soft-disable behavior in that case, including pools
that already contain manually soft-disabled members. Do not replace it with a separate
“suspend overlay and use baseline preferences” policy.

- A `|` pool selects among usable P members when at least one is preferred. Multiple P
  models keep their existing weights, efforts, and cursor behavior.
- If a pool has no preferred member, it rotates among its usable sparing members.
  Priority never injects a provider/model that is not in the pool.
- `||` retains its order: both preferred and sparing candidates are usable. In
  `(A | B) || C`, priority on C cannot pull it into the primary pool; the last-resort
  tail is consulted only when the primary pool is unavailable.
- Concrete `%model`, explicit provider choices, and concrete temporary alias/launch
  overrides keep working on backups. Existing hard-disable guards still apply. Priority
  never opens the disabled-provider launch dialog or provider-drain prompt.
- Existing bootstrap reservations retain the same semantics as soft disabling: an
  already reserved target remains redeemable while routable. Each subsequent unresolved
  launch, follow-up, or fallback decision sees current routing state. Running processes
  are not restarted and priority does not consume pool cursors.
- Expiry is `now >= expires_at`. Until cleared uses null expiry. Each resolution
  captures state once so expiry or a concurrent edit cannot split one decision.

## Visual design

Keep stable provider ordering, recognizable provider colors, keyboard highlight, and the
current compact card. Priority gets the existing blue/cyan accent family and a star plus
text; it must be understandable without color. Hard-disable amber and soft-disable
yellow remain distinct. Backup rows use a quiet neutral/cool tone and the word
**backup**, preserving their provider label color.

Illustrative content at a normal terminal width:

```text
                         Provider Routing
          ★ CODEX priority · 2h left
          Pools prefer CODEX; other providers remain backups.

  CLAUDE        11 models   backup · CODEX priority
› CODEX          7 models   ★ priority · 2h left
  GROK           3 models   soft · manual · 30m left
  GEMINI         2 models   CLI missing

  CODEX is preferred in pools that contain a usable CODEX model.
  Explicit choices and || order still apply. c clears CODEX priority.

  p Prioritize/change   c Clear priority   d/enter Disable
  s Soft disable       x Enable          j/k Navigate   esc Back
```

The header summary names the current priority and time independently of selection. With
no priority, use the existing explanatory summary plus a short prioritization hint. A
real disable or missing CLI is always visible on the affected row; add the priority
designation as secondary information when it is the selected priority. Do not hide
manual/automatic provenance behind a generic backup label. Details explain both causes
when a real soft disable overlaps the priority overlay.

Update the Config → Launch title/context, alias descriptions, model picker and
completion provenance, and top-bar indicator consistently. Priority-derived members must
not be labelled “soft-disabled.” Use a compact priority explanation in alias
descriptions, retaining existing selected/skipped markers and available pool counts. An
all-backup pool may select a backup normally. A provider in a `||` chain is still
selected by order; avoid copy implying that priority reorders that chain.

Extend the existing top-bar indicator composition with a small **CODEX ★ 2h** priority
pill and tooltip. Preserve the existing real-disable pill/count when both are present;
never count virtual backups as disables. Make unavailable priority visible in
text/tooltip. Keep the existing click route to Config → Launch.

At 70×32, let summary and footer wrap deliberately, shorten row metadata before clipping
provider/state, and allow the list to scroll. The priority provider, its window, clear
action, and real hard-disable status must remain discoverable. Verify 120×40 and 70×32
with the pinned PNG renderer; inspect the images, not only assertions.

## Phase 1: Rust state and policy

Implement in the opened `sase-core` repository. Add a focused `provider_priority` module
and a routing-context/policy module as needed, register them in
`crates/sase_core/src/lib.rs`, and expose additive bindings in
`crates/sase_core_py/src/lib.rs`.

1. Store one strict version-1 priority record in `llm_provider_priority.json` under the
   supplied SASE home: `version`, `provider`, `created_at`, `expires_at`, and `source`
   (ACE writes `ace`). Validate provider syntax, finite timestamps, future expiry,
   overflow, and source. Missing/expired priority means no preference. Keep the existing
   disable file/schema and legacy migration behavior intact.
2. Expose get, set-relative, set-until, and clear operations, with typed results for
   changed/unchanged, ineligible target, and I/O/lock errors. Clear accepts a valid
   expected current record/revision rather than requiring provider registration. Use
   record/revision comparison for edits and clears to reject stale dialogs instead of
   clearing or replacing a newer priority from another ACE session. A mismatch refreshes
   the UI and asks the user to retry; no automatic retry write. Clearing an already
   absent/expired record is an unchanged success, while a different live record is a
   conflict. Compare the complete stored record or a true revision; provider name alone
   cannot distinguish changed windows.
3. Reuse bounded locking and atomic replace. Coordinate priority operations and
   authoritative combined snapshots with the existing provider-disable lock via narrow
   internal helpers, so eligibility checks see current actual disables and one snapshot
   observes both stores coherently. Registration/CLI facts are supplied by the frontend
   and refreshed immediately before the operation. Avoid nested acquisition. Priority
   writes change only the priority file; automatic disable first-writer behavior
   continues to inspect only actual disables.
4. Expose an immutable routing context containing actual disables, optional priority,
   and captured clock. Add a pure Rust classifier for supplied provider/CLI facts that
   returns the existing tri-state availability plus explicit provenance
   (`actual_hard_disable`, `actual_soft_disable`, `priority_backup`, `priority`, or
   ordinary availability/unavailability). Retain both relevant causes where needed. It
   performs no filesystem access or cursor operations and supports providers added after
   the priority was set.
5. Provide a read-only decode/derive seam for cached display data that never takes the
   shared lock or prunes files. Malformed priority data degrades to no priority with a
   diagnostic, retaining independently valid disable state. Authoritative write failures
   are surfaced; they must not be disguised as a successful set. Keep
   malformed/unsupported priority recovery consistent and covered by tests.
6. Test relative/exact/indefinite windows, exact expiry, restart reads, replacement,
   idempotent clear, rejected/stale writes, invalid values, lock timeout, and atomic
   failure. Exercise concurrent priority changes and automatic disable writes. Verify
   that unrelated disable records retain timestamps, mode, source, and expiry. Test the
   complete precedence table and PyO3 wire round trips/error translation.

Run the core repository's `just check` (or its documented script), including the PyO3
tests with Python >=3.12. Core-only cargo tests are insufficient. Hand the binding
contract and verified build to phase 2. release-plz owns crate versions; do not manually
edit release versions. Coordinate release availability before a published Python
consumer requires the new binding.

## Phase 2: Routing integration

Implement the thin Python adapter, propagation, and cached display seam in `sase`. Use
the actual installed binding contract from phase 1, with strict typed hydration and the
existing required-Rust loader. Do not implement the precedence table again in Python or
disguise priority as `TemporaryProviderDisable` records.

1. Introduce an immutable `ProviderRoutingContext` with real disables, priority,
   captured time, and Rust-derived provenance. Add explicit `routing_context` plumbing
   at routing roots and through nested resolvers. Keep legacy explicit
   `provider_disables` inputs deterministic as disable-only contexts; callers must not
   silently recapture ambient priority mid-resolution. Reject conflicting explicit
   inputs. Update production roots to capture the complete context once. Legacy
   compatibility applies to supplied data, not a backend fallback; the derived states
   still come from the same Rust policy. Avoid losing priority through `dict(disables)`
   or `disables or None` shortcuts, especially when priority is the only active state.
2. Integrate `registry.py`, `model_alias_resolution_types.py`,
   `model_alias_resolution_resolve.py`, `model_alias_resolution_selector.py`,
   `alias_view.py`, `launch_selection.py`, `model_launch_settings.py`, and
   `temporary_override_defaults.py`. Feed Rust-derived states to the existing selector
   masks and weighted cursor implementation. Preserve public availability booleans for
   sparing members and expose provenance on selector/view metadata.
3. Audit all capture and forwarding sites, including `_invoke.py`, workflow prompt
   launch, `agent/launch_guard.py`, TUI launch guards, `tmux_agent/catalog.py`, model
   picker rows/modal, model/directive completion, and
   `doctor/checks_config_model_aliases.py` diagnostics (which currently assume every
   sparing member is soft-disabled). Actual-disable getters and try-set functions retain
   their actual-state meaning for usage limits and drains. Cover reservation/redeem
   behavior explicitly without inventing a new reservation invalidation rule for
   priority.
4. Extend the existing lock-free, time-gated peek and launch-default token
   infrastructure to include priority and its expiry transition. Store both real
   disables and priority in cached snapshots. New disk reads/parsing and authoritative
   refreshes run in existing workers/pump-free refresh tasks; keystroke/render paths use
   cached data and pure derivation, never state locks or pruning writes. Expiry must
   invalidate effective routing even when file mtime has not changed.
5. Extend the TUI's `ProviderRoutingSnapshot` and route-change comparison to carry
   priority and provenance, ready for phase 3. Include priority set, replacement, clear,
   and expiry in change detection even if actual disables are unchanged. Explicit local
   writes invalidate display caches immediately; external edits flow through bounded
   existing refresh cadence and coalescing, without idle reload storms.
6. Add parity tests: for a fixed pool/cursor/context, prioritizing P selects exactly as
   virtual soft-disabling every other provider while retaining hard disables. Cover
   weighted/multiple-P pools, no-P/all-backup pools, real soft overlaps,
   hard/CLI-unavailable P, all unavailable, alias indirection, efforts, `||`, and
   `(pool) || tail`. Test concrete and temporary overrides, provider autodetection, new
   launch/follow-up resolution, and existing reservation redemption. Include a
   real-binding integration test rather than relying entirely on mocked snapshots.
7. Test priority-only contexts through every relevant boundary, combined-snapshot
   consistency, clear/expiry restoration using the latest real disables, externally
   changed state, plugin addition/removal, and lock-free cached read behavior.

Reference tests include `tests/llm_provider/test_provider_disable_soft_routing.py`,
`test_provider_disable_routing.py`, `test_provider_disable_peek.py`,
`test_launch_default_peek.py`, pool last-resort tests, launch selection/reservation
tests, and `tests/test_models_panel_provider_routing.py`.

Build/install the opened core through the existing `just install`/Rust development flow
when required. The current consumer floor is `sase-core-rs>=0.32.25,<0.33.0`; use the
repository's existing core-floor/release reconciliation tooling to enforce availability
of new symbols. Do not guess the next release number or add a fallback.

## Phase 3: Provider Routing experience

1. Add fixed modal-local `p` and conditional `c` bindings. Reuse duration and exact time
   cards with priority-specific titles, subtitles, submit labels, and cancel/Back flow.
   Maintain an explicit pending action so callbacks cannot confuse priority,
   hard-disable, or soft-disable writes. Refresh eligibility and expected revision
   before accepting a result; saving the duration is the only priority set mutation.
2. Extend the existing provider write worker/outcome path for set/change/clear.
   Serialize writes, suppress repeated submissions, keep the modal open, preserve the
   current selected provider when completion arrives, and ignore obsolete snapshot
   completions. Keep blocking I/O off both the loop and serial pump. Preserve existing
   disable drain behavior; priority outcomes cannot request drains.
3. Distinguish write failure from a committed write followed by refresh failure. On a
   confirmed commit, mark parent routing changed and invalidate caches even if reload
   fails, then show an honest refresh warning and schedule a retry. Surface
   stale-revision conflicts with the refreshed state. Do not optimistically report
   success or overwrite a concurrent update. Closing/unmounting must preserve the
   existing worker lifecycle protections and never mutate an unmounted widget.
4. Implement the visual contract in `models_panel_provider_rendering.py`, modal
   options/state/workers, `models_panel_providers.py`, and `styles.tcss`. Add cached
   countdown refresh to the routing modal and parent: reaching expiry triggers one
   coalesced off-thread reload and parent routing signal. Refresh the header, row,
   description, and footer together. Preserve highlight guards and stable ordering.
5. Add truthful priority/backup provenance to alias descriptions and picker/completion
   displays, with full real-disable provenance where applicable. Extend
   `widgets/provider_disables_indicator.py` and `_override_pill.py` for priority plus
   actual-disable composition, then update the existing top-bar refresh route in
   `actions/agent_workflow/_leader_mode.py`. Keep narrow-screen content usable.
6. Update `docs/ace.md` Provider routing controls and `docs/llms.md` routing guidance:
   exact navigation, `p` and `c`, windows, precedence, explicit choices, fallback
   semantics, reservations, expiry, and the distinction between backups and real
   disables. Update the relevant `?` help/guide content and local footer. These are
   fixed inner-modal bindings like `d/s/x`, so no new leader configuration is needed;
   verify `src/sase/default_config.yml` remains consistent with that choice.
7. Add interaction tests for the real Admin Center → Config → Launch → Provider Routing
   path, set/change/switch/clear, all duration forms, cancel/Back, ineligible target,
   stale duration window, external conflicting write, orphan priority, write failure,
   post-commit refresh failure, repeated keys, expiry while open, preserved selection,
   parent refresh, and coexistence with `d/s/x`. Preserve existing hard-disable/drain
   and soft-disable tests.
8. Extend the established PNG suite under `tests/ace/tui/visual/`: priority active,
   until cleared, priority plus manual/automatic disables, unavailable priority,
   duration card, Launch alias description, and combined top-bar indicators. Cover
   120×40 and 70×32; use representative combined states rather than a Cartesian snapshot
   explosion. Inspect actual/expected/diff outputs and intentionally update goldens only
   for this feature's visual changes.

## Verification and completion

Every implementation worker reads the applicable repository instructions and
`lint_and_test.md`; read `tui_perf.md` before changing refresh/render behavior. Run
focused meaningful tests during implementation, then `just check` for each changed SASE
tree. Use `just install` first when the ephemeral environment requires it. Use
`/sase_monitor` for long commands; `just check-full` is exclusively monitor-run.

The final combined epic tree must pass Rust `just check` including binding tests, SASE
`just check-full` through a TESTING/TESTED monitor, and `just test-visual` with reviewed
PNGs. Do not treat the scoped lane as the combined epic landing gate. No performance
regression is acceptable from state reads on keystroke/render paths; verify the existing
idle-refresh and responsiveness checks for touched seams.

Acceptance: setting CODEX priority for two hours is visible immediately throughout ACE
and affects the next eligible pool decision; another provider becomes a usable,
truthfully labelled backup; explicit selections and ordered fallbacks retain their
rules; hard disables win; clearing/expiry restores current real-disable behavior;
concurrent updates and errors cannot clobber state or leave success-looking stale views;
and the experience remains readable and keyboard-complete at both tested terminal sizes.
The initial no-priority state retains existing routing behavior.

This feature does not add quota prediction, automatic reset-time selection, per-alias
priority rules, multiple ranked providers, new CLI commands, config rewrites, or a
migration of the existing selector engine. It is an additive temporary routing
preference implemented after this plan is approved.
