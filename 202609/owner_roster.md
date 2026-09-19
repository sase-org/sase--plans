---
tier: tale
title: Share the real owner roster and retain visible family shells
goal:
  The real owner Agents loader and the real gateway presentation catalog produce the
  same normalized visible-node and nested-shell sets for one production-shaped fixture.
size: medium
proposed_by: bbugyi200.athena.sase-133.5.1
bead: sase-133.5.1
create_time: 2026-09-19 08:57:11
status: wip
---

- **PARENT:**
  [202609/remote_parity_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202609/remote_parity_landing_repairs.md)
- **BEAD:**
  [sase-133.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-133/sase-133.5.1.md)

# Plan: Owner roster parity from a production fixture oracle

Complete phase bead `sase-133.5.1` so gateway presentation selection matches the owner's
actual local Agents loader and modern family-shell lifecycle. This phase owns **who is
in the roster** (visible nodes, nested shells, grouping membership). It does not own
rich status labels, tribe inheritance for rendering, version-skew diagnostics, live
cross-machine captures, parent-epic close, or ancestor plan-status updates.

## Established baseline

Confirmed production gaps from the landing audit, live catalog
`file:explicit:d1d475c9a14db30b47241095`, and current sources. Open `sase-core` with
`sase repo open sase-core` before reading or editing it.

- `decide_fleet_presentation` in `sase-core/crates/sase_core/src/fleet_presentation.rs`
  excludes every `family_member && Dead/NotProcess` candidate
  (`terminal_family_member_is_not_a_standalone_recent_row`,
  `dead_protected_family_member_is_not_a_standalone_row`). The owner TUI still shows
  those shells nested under the family (`×N`, plan/code/monitor/gate/proc chips).
- Gateway candidate construction in `sase-core/crates/sase_gateway/src/fleet_reads.rs`
  sets `family_member: tracked_parent_timestamp(record).is_some()`. Modern owner records
  often carry `family_id` / `family_shell` / `--plan|--code|--gate|--mon|--proc` name
  suffix **without** `parent_timestamp`, so the same object is misclassified as a
  standalone row instead of a member.
- `family_role_for_projection` in `sase-core/crates/sase_core/src/fleet_contract.rs`
  classifies an `AgentShell` with no tracked parent as `Root`. `row_kind_for_record`
  maps `family_shell.kind` `gate` / `monitor` to those row kinds (so `--plan` often
  becomes an unparented `DONE` gate). Live apollo rows such as `0n--plan` / `0k--plan`
  arrive in that shape.
- Gateway `PresentationContext::from_records` indexes roots as served records with
  `family_id` and **no** `parent_timestamp`. A concrete `--plan` shell is not a family
  root. Context is built only from the served set, so omitted members cannot supply
  linkage even when a later viewer pass would nest them.
- Viewer `src/sase/ace/tui/actions/agents/_fleet_refresh.py` requests the default
  catalog (`include_terminal: True`) and paginates that same presentation cursor. It
  never switches to history scope. Missing presentation members cannot contribute
  nesting, counts, chips, or family status.
- Viewer `src/sase/ace/tui/models/_fleet_agents_nodes.py` already treats
  `gate`/`monitor`/`proc`/`member`/`historical_shell` as nested
  (`_is_history_or_nested`) and will attach them to a real served root
  (`_attach_unresolved_members_to_real_roots`). That cannot recover shells the gateway
  never served, and it synthesizes duplicate containers when no real root is present.
- Phase-1 `owner_served_set_matches_visible_identity_set` in `fleet_reads.rs` asserts a
  hand-written gateway label set and never calls `load_tiered_agents`. Python
  `tests/ace/tui/test_fleet_agents_display_parity.py` supplies statuses and the full
  tree by hand. Keep those as unit tests; they are **not** the production oracle.
- Owner listing is not the index query alone.
  `src/sase/ace/tui/models/_agent_loader_artifacts.py` hydrates the compact Agents-list
  projection (`agents_list_projection=True`, `record_shape="list"`), which skips
  marker-only waiting/question rows. `load_tiered_agents` then merges RUNNING-field
  claims, home `running.json`, done/workflow snapshot loaders, and `_filter_dead_pids`.
  Family shells survive a dead PID; a pending gate claim with a killed creator PID is
  held until the shell settles (`_stale_claim_is_releasable` in `_running_loaders.py`).
  Gateway presentation queries `agents_list_projection=false` / `record_shape=Full` and
  does not ingest those claim sources.
- `decide_fleet_presentation` is not currently exported from `sase_core_py`. The Python
  oracle must call the **same** catalog assembly the gateway uses, without standing up
  HTTP.
- Do not equate raw catalog rows, visible nodes, and shell counts. Apollo's settled
  owner capture reports 20 agents (`@default=8`, `@epic=6`, `@research=6`); the
  contemporaneous `machine:apollo` viewer capture reports 30. The oracle compares
  **normalized identity sets**, not those live counts.

Shared inclusion, retirement, family-root-versus-shell classification, and bounded
family context belong in `crates/sase_core`. Gateway supplies injectable host
observations and cache/HTTP. Python keeps loader/adapters/bindings. Do not put selection
policy in the TUI event loop.

## Design decisions

1. **One production fixture, two real producers.** Seed a hermetic on-disk lifecycle
   that matches current persisted shape, then run it through `load_tiered_agents` and
   the real presentation catalog builder. Do not replace either side with a
   hand-authored expected roster.
2. **Three independent signatures**, each a sorted identity tuple, not a count:
   - **visible nodes**: family/agent containers the owner counts as Agents-tab nodes
     (roots and standalone agents; not nested shells). Use the same default-fold
     visibility the owner Agents tab uses (`filter_agents_by_fold_state` with a default
     `FoldStateManager`), applied to both producer outputs after identity normalization.
   - **nested shells**: concrete plan/code/monitor/gate/proc/historical members under
     those nodes, including pending shells.
   - **grouping**: tribe/clan panel membership of visible nodes only (`panel_keys_for` /
     `agents_for_panel` keys, not rendered titles).
3. **A real root and a concrete shell are different objects.** Classify membership from
   modern facts, in this order: `family_shell.kind`, `agent_family_role` /
   `role_suffix`, plan-chain name suffix (`--plan`, `--code`, `--gate`, `--mon`,
   `--proc`), `family_id`, then `parent_timestamp` when present. A `--plan` gate with
   `family_id` and null `parent_timestamp` is a nested shell, never a root.
4. **Visible historical shells of a currently presented family stay in the presentation
   served set** so the viewer can nest them without switching to history. They are
   catalog rows with nested `family_role`, not extra visible nodes. Dead members of a
   missing, dismissed, or out-of-window family stay excluded (keep
   `orphan_terminal_family_members_are_hidden_from_presentation`). Do not fetch the full
   archive on refresh and do not publish hidden/dismissed history merely to obtain `×N`.
5. **Pending shells remain visible.** A pending gate whose creator PID is dead is not
   obsolete. Preserve unknown-liveness conservatism, dismissal-first exclusion, and
   recycled-PID / wrong-command identity mismatch exclusion from both presentation and
   history.
6. **Current and compact-index paths both have to pass.** Compact-index is the owner's
   `agents_list_projection=True` hydration. Current is the gateway's existing
   presentation index query (`agents_list_projection=false`, Full records, active +
   bounded recent-completed). If reconciling them requires the gateway to use the
   compact projection, it must still ingest the claim/marker sources the owner loader
   uses for rows that projection omits (fresh launches, waiting-only). Do not blindly
   switch the gateway query and drop those identities.
7. **Fresh launches and cache invalidation are in scope.** The fixture includes a newly
   launched shell whose index row can lag the RUNNING claim. After the same
   revalidate/invalidation the owner uses, both producers must include it. Stale
   recycled-PID and dismissed identities must stay absent.
8. **No duplicate family containers.** Deduplicate real roots versus synthetic
   containers; timestamps and project identities must be collision-safe. Counts must not
   inflate because a root and its `--plan` shell both counted as nodes.
9. **Out of scope for this phase:** owner `display_status` pipeline (`TESTING`,
   `TALE DONE`, `EPIC CREATED √`), chip/runtime rendering, capability-versus-fleet
   version comparison, live athena/apollo screenshot acceptance, parent `sase-133` /
   `sase-133.5` close. Coarse wire status may remain; `family_role` / linkage / served
   membership must be correct enough that later owner-facts can attach facts to the
   right objects.
10. **Schema.** Prefer serving nested shells as existing catalog rows with corrected
    role and linkage. Bump the fleet contract only if a new bounded family-context field
    is strictly required; keep the change additive and older-host tolerant. Explicit
    history pagination stays a separate API.

## Implementation

1. **Production fixture.** Add one shared on-disk fixture (Python test helper is the
   natural writer; Rust tests may consume the same directory layout) that includes:
   - project RUNNING claims, including a newly launched shell
   - running / waiting / pending-question markers
   - done records and a completed family with several historical shells
   - family root **and** concrete plan, code, monitor, gate, and proc shells
   - a pending gate whose creator PID is dead and whose own gate markers are still
     pending
   - a settled monitor
   - an active coder
   - at least one modern `--plan` / `--code` shell with `family_id` (or name suffix) and
     **no** `parent_timestamp` (the live `0n--plan` / `0k--plan` shape)
   - an old dismissed identity
   - a recycled-PID / identity-mismatch running record Rebuild the artifact index the
     same way production does. Inject host liveness so tests do not depend on PIDs that
     happen to exist on the runner.

2. **Oracle.** From that fixture, call the real owner loader (`load_tiered_agents` / the
   production snapshot+claims path, not a hand-built `Agent` list) and the real
   presentation catalog builder (`build_snapshot_blocking` / `FleetReadService.catalog`,
   not a reconstructed candidate list). Lift catalog-from-index assembly (selection,
   root-versus-shell classification, bounded family context, row projection inputs) into
   `sase_core` as needed so:
   - gateway `build_snapshot_blocking` stays a thin observer/cache wrapper
   - Python can invoke the **same** catalog builder through `sase_core_py` without
     standing up HTTP Normalize both outputs to the three signatures above. Fail with
     the extra/missing identities **and** the admitting/removing rule (dismissal,
     identity mismatch, dead member of a visible family, pending-gate hold,
     compact-projection skip, cache generation, duplicate container). Run the oracle
     against both the compact-index owner hydration and the current gateway presentation
     query.

3. **Shared inclusion/retirement in `sase_core`.**
   - Stop using `tracked_parent_timestamp(record).is_some()` as the only `family_member`
     bit. Classify root versus concrete shell from the modern facts in decision 3. Put
     that classifier next to `decide_fleet_presentation` (or in `fleet_contract` if
     projection also needs it) and call it from gateway candidate construction **and**
     `family_role_for_projection`.
   - Keep live/unknown undismissed identities current, including pending shells.
   - Admit dead/not-process **visible historical shells of a currently presented
     family** into the presentation served set (bounded by the existing seven-day /
     200-row terminal window unless the owner loader proves a tighter visible subset). A
     family is "currently presented" when its root or a live/unknown member is already
     current/recent-terminal after the existing dismissal and identity-mismatch filters.
   - Continue excluding dismissed identities, process-identity mismatches, and orphan
     terminal members that would not form an owner-visible family.
   - Preserve unknown-liveness safety. Do not infer that every dead-PID shell is
     obsolete.
   - Invoke this one policy from gateway presentation **and** from any Python listing
     path that currently duplicates incompatible rules. Prefer calling the core decision
     through the binding over persisting a second copy of the policy. Persist a
     lifecycle fact into the index only when the original approved plan's
     one-source-of-truth rule requires it and porting the rule would be
     disproportionate; record that choice in the phase note.
   - Keep host observations injectable.

4. **Bounded family context.** Rebuild `PresentationContext` (moved into core with the
   catalog assembly) so it can resolve a served shell's family without requiring the
   root to look like a parentless served record. Use bounded sibling/root facts from the
   presentation candidate set; do not scan the full archive. Deduplicate real versus
   synthetic roots. Keep timestamps/project identities collision-safe. Do not expose
   hidden/dismissed history. Leave explicit history scope on its existing separate API
   and safety filters (dismissal + identity mismatch, no presentation age/count window).

5. **Gateway query and cache.** Keep presentation bounded (active + recent-completed; no
   `include_full_history` on the refresh path). Cover cache invalidation so a newly
   launched claim appears after revalidate. Preserve resource capabilities, content
   identity, and older-host tolerant readers. Pending gate shells may outlive their
   creator PID.

6. **Tests to add or rewrite.**
   - Production oracle (required acceptance): equal visible-node and nested-shell
     signatures; grouping membership equal; stale dismissed/recycled rows absent;
     pending gate visible; fresh launch present; no duplicate family containers. Place
     the Python oracle under `tests/ace/tui/` (or a sibling helper module imported by
     that test), not inside the display-parity fixture.
   - Compact-index and current presentation query paths both pass that oracle.
   - Core unit tests: `--plan` without `parent_timestamp` is a shell not a root; dead
     member of a visible family is served for nesting; orphan dead member stays
     excluded; pending dead-creator gate stays current; dismissal and identity mismatch
     still exclude.
   - Replace or extend `owner_served_set_matches_visible_identity_set` so it is no
     longer a hand-written expected set. Update
     `terminal_family_member_is_not_a_standalone_recent_row` /
     `terminal_family_root_represents_completed_members` to the
     nested-shell-in-presentation rule (members are served, not standalone visible
     nodes). Keep `orphan_terminal_family_members_are_hidden_from_presentation`.
   - Leave the Python display-parity fixture as a unit test of the renderer; do not
     treat it as this phase's oracle.

## Verification

- Iterate with focused tests: core `fleet_presentation` / catalog assembly, gateway
  `fleet_reads` fixtures, and the Python production-oracle module.
- In the linked `sase-core` repo run `just check` (fmt, clippy, workspace tests).
- In the primary repo run `just fix` then `just check` per `lint_and_test.md`. Use a
  monitor for a long `just check`. Do not run `just check-full` here; that belongs to
  the later parity-acceptance phase.
- If TUI goldens change, inspect the visual report; this phase should not need a second
  synthetic visual oracle. Include generated goldens through the host finalizer if they
  do change.
- Run `sase bead epic-symbols sase-133.5.1`. Resolve every remaining `--epic-symbol` or
  re-key its Justfile line to a still-open bead (parent epic `sase-133.5` or a later
  phase). `sase bead close` refuses while leftovers remain.
- Close **only** `sase-133.5.1` with
  `sase bead close sase-133.5.1 --note "<oracle, compact-index, current path, and canonical checks verified>"`.
  Do not close `sase-133.5`, `sase-133`, or any other ancestor. Record discovered
  follow-up as `sase bead note sase-133.5.1 'PROPOSED FOLLOW-UP: …'` — do not create
  beads.
