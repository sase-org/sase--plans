---
tier: epic
title: Validate and display %wait agent-tribe references correctly
goal: 'A `%wait(@<tribe>)` target is understood end to end: reserved pseudo-tribe
  references such as `@default` are rejected at launch instead of parking an agent
  forever, and the ACE Agents tab renders a real tribe wait as a pending-or-bound
  tribe target instead of a missing agent name.

  '
phases:
- id: reserved-tribe-guard
  title: Reject reserved tribe references in wait and fork targets
  depends_on: []
  size: small
  description: 'reserved-tribe-guard: add a canonical reserved-tribe concept to core
    and reject `@default` (and any other reserved pseudo-tribe) wherever a tribe reference
    is used as a `%wait` or `#fork` target, so the launch fails with a clear message
    instead of parking forever.

    '
- id: tribe-wait-binding
  title: Shared tribe wait binding resolver
  depends_on: []
  size: medium
  description: 'tribe-wait-binding: extract the `tribe_candidate` ordering and aggregation
    rules into a pure, snapshot-driven resolver in Python core that both the wait
    index and the TUI can call, and add a pending/bound/reserved classification the
    display layer can consume without touching disk.

    '
- id: ace-tribe-wait-display
  title: Tribe-aware wait rendering in the Agents tab
  depends_on:
  - tribe-wait-binding
  size: medium
  description: 'ace-tribe-wait-display: stop classifying tribe wait targets as missing
    agent names, give them their own wait lane tag and tribe identity styling, show
    the bound entity and its status once one exists, and refresh the affected render-cache
    keys, help text, and PNG snapshots.

    '
- id: unresolvable-wait-surface
  title: Surface waits that can never resolve
  depends_on:
  - reserved-tribe-guard
  - ace-tribe-wait-display
  size: small
  description: 'unresolvable-wait-surface: give already-parked reserved-tribe waits
    a distinct "this wait can never resolve" presentation in ACE so agents launched
    before the guard landed are diagnosable rather than silently pending.

    '
create_time: 2026-07-28 17:04:51
status: wip
bead_id: sase-ak
---

- **BEAD:** [sase-ak](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ak/README.md)

# Plan: Validate and display `%wait` agent-tribe references correctly

## Background

`%w(@<tribe>)` is documented (`src/sase/xprompts/skills/sase_run.md:60-70`) as binding to the _next_ agent or clan
launched into that tribe after the waiting launch, resolving to the earliest qualifying successful entity.

The **runtime already implements this correctly**:

- `src/sase/axe/run_agent_directives.py:216-228` skips `normalize_owned_agent_name` for names that parse as tribe
  references, so `@epic` survives into the waiting marker verbatim.
- `src/sase/axe/run_agent_directive_identity.py:63-72` keeps a lone tribe wait from seeding the launched agent's name.
- `src/sase/core/wait_dependency_resolution/_resolution.py:43-56` passes `newer_than=waiter_launch_cutoff` only when
  `name.startswith("@")`.
- `src/sase/core/wait_dependency_resolution/_index_queries.py:334-355` (`is_resolved`) dispatches `@` names to
  `tribe_candidate` (`_index_queries.py:34-130`), which returns a `TribeCandidate` (`_types.py:48-57`) for the earliest
  _complete_ agent or clan launched strictly after the waiter.

Two defects sit on either side of that working core.

### Defect A — `@default` is a reserved pseudo-tribe that parks an agent forever

`DEFAULT_AGENT_TRIBE = "default"` (`src/sase/ace/tui/models/agent_panels.py:36`) is the reserved _display_ identity of
the untagged panel. `normalize_panel_key` maps `"default"` to `None`, and `is_reserved_default_panel`
(`agent_panels.py:60`) keys off that.

`WaitDependencyIndex.tribes` is populated only from explicit assignments — artifact `meta["tribe"]` or
`~/.sase/agent_tribes.json` (`src/sase/core/wait_dependency_resolution/_index.py:281-309`). Untagged agents are never
indexed under `"default"`. Inspection of the live store confirms no agent carries the tribe `default`, so `%w(@default)`
can never resolve.

ACE's interactive paths already know this:

- `src/sase/ace/tui/actions/agents/_wait_helpers.py:186-193` refuses with _"The reserved @default panel cannot be used
  as a wait target"_ / _"The reserved @default panel cannot be forked"_.
- `_build_tribe_completion_candidates` (`src/sase/ace/tui/_agent_completion_candidates.py:278+`) builds candidates from
  `agent.tribe` / `clan_tribe`, both `None` for untagged rows, so `@default` is never offered as a completion.

But the directive parse path has no reserved-name check: `_parse_wait_tribe_reference`
(`src/sase/xprompt/_directive_values.py:45-58`, also used at `:311`) only calls
`sase.core.agent_tribe.parse_tribe_reference` → `validate_tribe_name` (`src/sase/core/agent_tribe.py:27-48`), which
accepts `default`. A hand-typed `%w:@default` is therefore launched and parked indefinitely. The same gap applies to
`#fork:@default`, which becomes an implied wait name through `fork_agent_names`
(`src/sase/agent/names/_resume.py:94-96`) appended to `wait_names` in `run_agent_directives.py:201-203`, and is
separately resolved by `_resolve_tribe_fork_source` (`src/sase/scripts/agent_chat_from_name.py:363-390`).

### Defect B — ACE renders every tribe wait target as a missing agent name

`missing_wait_dependency_names` (`src/sase/ace/tui/_agent_completion_wait.py:149-164`) returns every `waiting_for` entry
absent from the agent-name → status-bucket map. Tribe references are never in that map, so **every** tribe wait —
including a perfectly valid `%w(@epic)` — is reported as missing:

- `src/sase/ace/tui/widgets/_agent_list_build.py:295-298` sets `has_missing_wait_target`, rendering the `?` glyph next
  to `WAITING` on the row (`src/sase/ace/tui/widgets/_agent_list_render_agent.py:306-315`, `_MISSING_WAIT_TARGET_GLYPH`
  at `src/sase/ace/tui/widgets/_agent_list_styling.py:108-109`).
- `src/sase/ace/tui/widgets/prompt_panel/_agent_wait_section.py:106-139` renders `Wait: [agents] @default ?`.

Additionally `wait_dependencies_satisfied` (`_agent_completion_wait.py:133-146`) can never be true for a tribe wait, and
the wait lane tags tribe targets `[agents]` (`_WAIT_TAG_STYLES`, `_agent_wait_section.py:29-34`) with no tribe identity
styling, even though `compose_tribe_identity_style` / `named_tribe_identity_colors`
(`src/sase/ace/tui/models/tribe_display.py`) already exist and are used by the wait modal
(`src/sase/ace/tui/modals/wait_modal.py:276-330`).

### Boundary and constraint notes for every phase

- **Rust core boundary.** Wait-dependency resolution has no Rust counterpart: `crates/sase_core` carries
  `agent_clan_tribe.rs` but no `wait_dependency` module. The whole wait-resolution domain already lives in Python at
  `src/sase/core/wait_dependency_resolution/`. New shared logic in this epic therefore belongs beside it in Python core,
  **not** in `../sase-core`. Porting wait resolution to Rust is a separate, much larger effort and is explicitly out of
  scope here.
- **TUI performance** (`sase/memory/tui_perf.md`, rules 6 and 8). Render paths must never stat, glob, or read disk.
  Tribe binding for display must be computed from already-loaded rows, once per agent snapshot, alongside the existing
  `collect_agent_wait_status_maps` work — never per row and never per keypress. Do not build a `WaitDependencyIndex` in
  the TUI.
- **Memory files.** No phase may edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims (`CLAUDE.md`,
  `GEMINI.md`, `OPENCODE.md`, `QWEN.md`). Permission for those has not been granted.
- **`just check`.** Every phase that changes files under `src/` or `tests/` must run `just install` then `just check`
  before finishing.

---

## Phase `reserved-tribe-guard`: Reject reserved tribe references in wait and fork targets

### Goal

`%w(@default)` and `#fork:@default` fail fast with an actionable message instead of launching an agent that waits
forever.

### Design

Add the reserved-tribe concept to **core**, not the TUI:

1. In `src/sase/core/agent_tribe.py`, add a canonical reserved name and predicate, e.g.

   ```python
   #: Reserved display identity for the untagged agent bucket. It is a panel
   #: label, never a real tribe assignment, so it can never be a wait target.
   RESERVED_DEFAULT_TRIBE = "default"
   RESERVED_TRIBE_NAMES: frozenset[str] = frozenset({RESERVED_DEFAULT_TRIBE})

   def is_reserved_tribe_name(tribe: str) -> bool: ...
   ```

   Export both from `__all__`.

2. **Do not** change `validate_tribe_name` or `parse_tribe_reference`. This is critical:
   - `parse_tribe_reference` is used as a _classifier_ in many call sites that only ask "is this a tribe reference?"
     (`src/sase/axe/run_agent_directives.py:223`, `src/sase/agent/repeat_launcher.py:172`,
     `src/sase/agent/multi_prompt_reference_allocator.py:99`, `src/sase/agent/multi_prompt_reference_rewriting.py:196`,
     `src/sase/xprompt/_directive_alt_naming.py:291`, `src/sase/agent/names/_resume.py:330`,
     `src/sase/history/chat_resume.py:106`). Raising there would either escape uncaught (several call sites have no
     `try`) or silently reclassify `@default` as an _agent name_, which is worse than today.
   - `validate_tribe_name` is also applied to _stored_ and _configured_ tribe values.
     `src/sase/default_config.yml:80-83` legitimately configures `tribes.default` to style the reserved panel, and
     `WaitDependencyIndex._valid_tribe` (`_index.py:292-299`) validates values read back from artifacts. A blanket
     rejection would break both.

   The guard must therefore be scoped to _references used as targets_.

3. Reject reserved references at the two target entry points:
   - **`%wait`**: in `_parse_wait_tribe_reference` (`src/sase/xprompt/_directive_values.py:50-58`), after a successful
     parse, raise `DirectiveError` when `is_reserved_tribe_name(tribe)`. Wording should match the existing TUI message
     so users see one vocabulary, e.g.
     `"Invalid '%wait' tribe reference '@default': the reserved @default panel is the untagged bucket, not a real tribe, so it can never resolve — wait on a named tribe, an agent, a family, or a clan instead."`
     This covers both the direct parse at `:45` and the expansion path at `:311`.

   - **`#fork:@<tribe>`**: a fork reference becomes an _implied_ wait after directive parsing, so the `%wait` guard does
     not cover it. Reject it where the fork-implied wait names are finalized in
     `src/sase/axe/run_agent_directives.py:200-228` (the loop over `fork_agent_names(fork_reference_prompt)`), raising
     the launch-time error used by neighbouring failures in that module, with a `#fork`-flavoured message.

   - **Defence in depth**: in `src/sase/scripts/agent_chat_from_name.py`, have `_resolve_tribe_fork_source` (`:363-390`)
     raise the same explanatory `RuntimeError` for a reserved tribe _before_ it reports the generic
     `"No completed @default entity launched after …"`, so a fork workflow that somehow reaches that code still explains
     itself.

4. **`sase agent tribe set <agent> default`** (`src/sase/agents/cli_tribe.py:92` `_validate_or_exit`): reject it too.
   Assigning the reserved name produces an agent that ACE folds into the untagged panel while the wait index treats it
   as a real tribe — an inconsistency that would let `@default` waits resolve non-deterministically once phase
   `reserved-tribe-guard` claims they cannot. Reject on _assignment_ only; keep reads of pre-existing `default`
   assignments working (there are none today, and silently dropping stored values is worse than tolerating them).

### Tests

- New xprompt directive tests: `%wait(@default)`, `%w:@default`, and the `%wait` expansion path each raise
  `DirectiveError` with the reserved-tribe message; `%wait(@epic)` still parses unchanged.
- New axe test: a prompt containing `#fork:@default` fails the launch with the reserved-tribe error.
- New core test for `is_reserved_tribe_name` and for the unchanged behaviour of `validate_tribe_name("default")` and
  `parse_tribe_reference("@default")` (both must still succeed — this is the regression guard for item 2 above).
- New `sase agent tribe set` CLI test for the rejected assignment.
- Existing suites to keep green: `tests/xprompt/`, `tests/axe/`, `tests/core/` wait-resolution tests.

---

## Phase `tribe-wait-binding`: Shared tribe wait binding resolver

### Goal

One implementation of "which entity does this tribe wait bind to?" that both `WaitDependencyIndex` and the ACE display
layer use, so the two can never drift.

### Design

The rules encoded in `tribe_candidate` (`src/sase/core/wait_dependency_resolution/_index_queries.py:34-130`) are:

1. Only entities launched **strictly after** the waiter's launch timestamp qualify (`timestamp <= newer_than` is
   skipped); the waiter's own artifact dir is excluded.
2. A tribe-assigned clan member enrolls its **whole generation**; clan entities are keyed by `(clan_name, generation)`
   and additionally pulled in through `effective_clan_tribes`.
3. A clan's ordering timestamp is its generation's **earliest member launch**.
4. An agent qualifies as complete when `is_resolved and is_done`; a clan qualifies only when **every** aggregated member
   is `is_resolved and is_done`.
5. The winner is `min` by `(timestamp, kind, name)`.

Add a pure module — suggested `src/sase/core/wait_dependency_resolution/_tribe_binding.py`, re-exported from the package
`__init__` — that expresses those rules over a **minimal, backend-agnostic member record** rather than over
`ArtifactCandidate`. Something like:

```python
@dataclass(frozen=True, slots=True)
class TribeMemberRow:
    """One tribe-enrolled entity row, from artifacts or from a TUI snapshot."""

    tribe: str
    launch_timestamp: str      # artifact-dir basename ordering key
    identity: str              # artifact dir (agents) or clan key
    name: str
    clan_name: str | None = None
    clan_generation: str | None = None
    is_complete: bool = False  # caller maps its own resolved/done notion
    is_terminal: bool = False  # entity reached a terminal state (for "pending" vs "failed")


@dataclass(frozen=True, slots=True)
class TribeWaitBinding:
    """How one `%wait(@tribe)` target currently stands."""

    tribe: str
    state: Literal["reserved", "pending", "bound"]
    kind: Literal["agent", "clan"] | None = None
    name: str | None = None
    generation: str | None = None
    timestamp: str | None = None


def resolve_tribe_wait_binding(
    tribe: str,
    rows: Iterable[TribeMemberRow],
    *,
    newer_than: str | None,
    exclude_identity: str | None = None,
) -> TribeWaitBinding: ...
```

Then:

- Rewrite `WaitDependencyIndex.tribe_candidate` to project its `self.tribes` / `self.effective_clan_tribes` /
  `self.clans` / `self.artifacts_by_dir` state into `TribeMemberRow`s and delegate the selection to
  `resolve_tribe_wait_binding`, mapping the result back to `TribeCandidate` (which must keep returning its `members`
  tuple for `#fork:@tribe`, so keep the projection keyed so members can be recovered). Its observable behaviour must not
  change — this is a refactor, verified by the existing wait-resolution tests.
- Have `resolve_tribe_wait_binding` return `state="reserved"` for names in `RESERVED_TRIBE_NAMES` (from phase
  `reserved-tribe-guard`, if landed; if that phase has not landed yet, gate the branch on
  `sase.core.agent_tribe.is_reserved_tribe_name` and let the import be the coupling point). `WaitDependencyIndex` treats
  `reserved` exactly like "no candidate", preserving today's runtime semantics.
- `state="pending"` means the tribe is real but no qualifying complete entity exists yet — the honest state for a
  freshly launched tribe wait.

Consider also exposing the **pending in-flight** entity (the earliest qualifying entity launched after the waiter that
has not completed yet) as an optional field on `TribeWaitBinding`, because the display phase wants to say "waiting on
`sase-aj`, still running" rather than only "nothing yet". Keep it optional so the wait index ignores it.

### Boundary

Per the note above, this stays in Python `src/sase/core/`. Do not add a `sase-core` Rust module for it; the resolver it
factors out is Python-only today, and splitting the domain across two languages mid-refactor would be worse than the
status quo.

### Tests

- New unit tests for `resolve_tribe_wait_binding` covering: strictly-after ordering, self-exclusion, clan generation
  enrollment through both `clan_tribe` and `effective_clan_tribes`, clan earliest-member timestamp, all-members-complete
  requirement, the `(timestamp, kind, name)` tie-break, `pending` when nothing qualifies, and `reserved`.
- Existing `WaitDependencyIndex` / `dependency_resolution_status` tests must pass **unchanged**; if any needs editing,
  the refactor changed behaviour and is wrong.

---

## Phase `ace-tribe-wait-display`: Tribe-aware wait rendering in the Agents tab

### Goal

`Wait: [agents] @default ?` becomes an honest tribe rendering, and a real tribe wait such as `%w(@epic)` shows what it
is bound to.

### Design

1. **Snapshot-level binding, computed once.** Extend `collect_agent_wait_status_maps`
   (`src/sase/ace/tui/_agent_completion_wait.py:53-98`) to also produce, per tribe reference appearing in any loaded
   row's `waiting_for`, the `TribeWaitBinding` for that waiter. Build `TribeMemberRow`s from already-loaded `Agent` rows
   — every field needed is in memory:
   - `agent.raw_suffix` is the launch timestamp and the artifact-dir basename (it is the third element of the identity
     tuple that `WaitDependencyIndex._posthoc_tribe` looks up at `_index.py:300-308`), so it is the ordering key.
   - `agent.tribe`, `agent.clan_tribe`, `agent.clan_tribes`, `agent.agent_clan`, `agent.agent_clan_generation` give
     tribe enrollment.
   - `status_bucket_for_values(agent.status)` gives completion (`"Done"` → complete; the existing
     `aggregate_clan_status` path in this module already handles clan aggregation and should be reused).

   The binding is per _(waiter, tribe)_ because `newer_than` differs per waiter. Key the map accordingly, e.g.
   `dict[tuple[AgentIdentity, str], TribeWaitBinding]`.

   Rule 8 of `sase/memory/tui_perf.md` applies: this is O(rows) work inside the existing snapshot pass. Do not stat,
   glob, or read `~/.sase` from here, and do not recompute per row.

2. **Return-shape change.** `collect_agent_wait_status_maps` currently returns a 2-tuple unpacked with `or (None, None)`
   at three call sites (`_agent_display_hints.py:426-430`, `_agent_display_render.py:216-220`,
   `_agent_display.py:207-211`) and destructured at `_agent_completion_wait.py:49`. Rather than growing to a 3-tuple,
   introduce a frozen dataclass (e.g. `AgentWaitStatusMaps` with `buckets`, `clan_member_statuses`, `tribe_bindings`)
   and update `agent_wait_status_maps_for_app`, the re-exports in `src/sase/ace/tui/agent_completion.py:25-51`, and all
   three call sites. Three unpack sites with `or (None, None)` fallbacks are exactly where a silent index shift would
   hide.

3. **Stop calling tribe targets missing.** In `missing_wait_dependency_names` (`_agent_completion_wait.py:149-164`),
   skip names that parse as tribe references. A tribe wait is pending, not missing. Consequently
   `has_missing_wait_target` (`_agent_list_build.py:295-298`) stops firing, and the `WAITING ?` row badge disappears for
   tribe waits.

4. **Satisfaction.** In `wait_dependencies_satisfied` (`_agent_completion_wait.py:133-146`), treat a tribe wait as
   satisfied only when its binding state is `bound`. Untangle it from the plain `status_buckets.get(name) == "Done"`
   test.

5. **Wait lane rendering** (`src/sase/ace/tui/widgets/prompt_panel/_agent_wait_section.py:106-139`):
   - Split tribe targets into their own lane with a new `[tribes]` tag in `_WAIT_TAG_STYLES` (`:29-34`), so the
     `[agents]` tag stops lying. Keep the existing lane order sensible (agents, tribes, beads, time, runners) and
     confirm `_wait_gutter_width` (`:220-224`) still aligns with the wider tag.
   - Style the `@<tribe>` token with the tribe's identity color via `compose_tribe_identity_style` /
     `named_tribe_identity_colors` (`src/sase/ace/tui/models/tribe_display.py`), matching how the wait modal already
     renders tribe identities (`src/sase/ace/tui/modals/wait_modal.py:276-330`).
   - Render the binding state after the token:
     - `bound` → `→ <entity name>` plus that entity's status badge through the existing `_append_wait_status_badge`
       helper; for a clan binding, reuse the clan-member detail shape already built at `:114-132`.
     - `pending` with a known in-flight entity → name it with its live status badge.
     - `pending` with nothing in flight → a dim explanatory suffix such as `(next launch)`, and **no** `?` glyph.
   - The `?` missing-target glyph must never appear for a tribe target.

6. **Render cache.** `agent_render_key` (`src/sase/ace/tui/widgets/_agent_list_render_cache.py:136-235`) documents that
   every input affecting `format_agent_option` must be an explicit key element. `has_missing_wait_target` and
   `wait_deps_satisfied` are already keys, so if the row's visible output changes only through those two, no key edit is
   needed — verify that. If the row gains any new tribe-derived visual, add the corresponding key element in the same
   change.

7. **Help text.** Per `src/sase/ace/CLAUDE.md` ("Help Popup Maintenance"), update the wait-related help entries:
   `src/sase/ace/tui/modals/help_modal/binding_common.py:30`, `src/sase/ace/tui/modals/help_modal/agents_bindings.py:88`
   and the wait-badge legend at `:387`.

8. **Skill docs.** `src/sase/xprompts/skills/sase_run.md:60-70` describes the semantics; extend it only if the phase
   changes user-visible vocabulary (e.g. the `[tribes]` lane). This file is a generated-skill _source_, not a memory
   file, so editing it is in scope — but regenerate per `sase/memory/generated_skills.md` if the phase touches it.

### Tests

- Extend `tests/ace/tui/widgets/test_agent_display_waiting_warning.py`: a tribe wait must not appear in
  `missing_wait_dependency_names` and must not set `has_missing_wait_target`; a plain missing agent name still must.
- New tests for `wait_dependencies_satisfied` under pending vs bound tribe bindings.
- New wait-lane tests asserting the `[tribes]` tag, the absence of `?`, and the bound `→ <name>` rendering, following
  the existing `ResponsiveWaitSection.logical_text` inspection seam (`_agent_wait_section.py:233-248`).
- PNG visual snapshots: the wait lane and the agent row both change. Run `just test-visual`, inspect
  `.pytest_cache/sase-visual/` artifacts, and only then accept intentional changes with
  `--sase-update-visual-snapshots`. `tests/ace/tui/visual/test_ace_png_snapshots_agents_panels.py` and
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py` are the likely movers; add a fixture covering a
  waiting agent with a tribe target if none exists.

---

## Phase `unresolvable-wait-surface`: Surface waits that can never resolve

### Goal

Agents already parked on `%w:@default` before the guard landed are diagnosable in ACE rather than indistinguishable from
a legitimately pending tribe wait.

### Design

Phase `reserved-tribe-guard` blocks _new_ reserved-tribe launches, but agents launched earlier stay in `WAITING` forever
— including the `ne` agent that prompted this epic. Phase `ace-tribe-wait-display` would render them as an ordinary
pending tribe wait, which is honest about the mechanism but hides that they will never proceed.

- Use the `state="reserved"` branch that `resolve_tribe_wait_binding` already returns (phase `tribe-wait-binding`) to
  drive a distinct presentation: keep the tribe identity styling, but mark the target as unresolvable — for example a
  dedicated glyph and style alongside a dim `(reserved — never resolves)` suffix in the wait lane, plus a row-level
  badge distinguishable from the generic missing-target `?`.
- If that row badge is new visible output, add the corresponding element to `agent_render_key`
  (`src/sase/ace/tui/widgets/_agent_list_render_cache.py:136-235`) in the same change, per that function's docstring.
- Update the wait-badge legend in `src/sase/ace/tui/modals/help_modal/agents_bindings.py:387` and any wait entries in
  `binding_common.py:30` to document the new marker, per `src/sase/ace/CLAUDE.md`.

Deliberately out of scope: auto-killing or auto-rewriting already-parked agents. Killing them is a one-keystroke user
action (`x`) once the TUI makes the situation legible, and silently mutating a user's parked launches is a bigger
decision than this epic should make.

### Tests

- Unit test that a reserved-tribe wait renders the unresolvable marker and not the pending-tribe rendering.
- Extend the PNG snapshot fixture set with a reserved-tribe waiter if the row badge is visible there; same
  `just test-visual` accept-only-after-inspection workflow as the previous phase.

---

## Verification for the epic

- `just install && just check` in every phase that touches `src/` or `tests/`.
- `just test-visual` in the two display phases, with snapshot updates accepted only after inspecting the
  `.pytest_cache/sase-visual/` diff artifacts.
- Manual confirmation in `sase ace`: a live `%w(@epic)`-style waiter shows a tribe lane with no `?`; a `%w(@default)`
  launch is now rejected at submit time with the reserved-tribe message.
