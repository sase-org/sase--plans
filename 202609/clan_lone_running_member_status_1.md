---
tier: tale
title: Clan nodes always show their lone running member's status
goal:
  An agent clan node with exactly one running member node shows that node's exact status
  — including the FINALIZING overlay, the RETRYING countdown, and STARTING — on the clan
  row, the CLAN detail header, and the editor agent catalog, and the glossary records
  the rule for clans and nodes.
size: medium
proposed_by: bbugyi200.athena.0tg
create_time: 2026-09-28 06:24:46
status: wip
---

# Clan nodes always show their lone running member's status

## Decision to confirm at review

The rule below makes the clan node mirror its lone running member **exactly** whenever
that member drives the clan's status. It keeps one existing, previously approved
exception: a member that needs the user (asking a `QUESTION`, awaiting `PLAN`/`TALE`/
`EPIC` review, or `FAILED`) still outranks the running member, so such a clan keeps
reading `QUESTION`/`PLAN`/`FAILED` and stays in the attention groups of `BY_STATUS`
grouping. The existing test
`test_clan_status_preserves_precedence_over_lone_running_member` pins that behavior.

If the reviewer wants the literal "always" instead (the lone running member wins even
over failed/asking members), the change is local to the shared rule in step 1: drop the
"aggregate bucket is Running" condition for in-flight sources. That variant moves such
clans out of the Stopped/Failed groups into Running, and it also requires deciding what
the tribe `ATTENTION` section should show for them. It is not part of this plan.

## Problem

In the reported screenshot, clan `research.2v` has seven members: four `DONE`, two
`WAITING`, and one running member, `research.2v.gem`. That member's Agents-tab row reads
`(FINALIZING)`. The clan row reads `(RUNNING) [R1 W2 D4]`, and the selected clan's
`CLAN` detail header also reads `RUNNING`. Both should read `FINALIZING`.

The invariant to establish is this. An agent clan node with exactly one running member
node shows that member's status exactly as the member's own row shows it: the label, its
styling, and any render-time overlay of the status word.

## Root cause

- `FINALIZING` is not a stored status. It is a render-time overlay in
  `append_agent_row_status`
  (`src/sase/ace/tui/widgets/_agent_list_render_agent_status.py`). In the `RUNNING`
  branch, `_glance_finalizer_state(agent).is_finalizing` checks the row's **own**
  `finalizer_status`. It dispatches to `finalizer_row_state` or, for session container
  rows, `session_finalizer_row_state` (both in
  `src/sase/ace/tui/models/finalizer_row_state.py`).
- `apply_clan_container_status` (`src/sase/ace/tui/models/_agent_clan.py`) mirrors a
  lone active member's `status`, `status_bucket`, and monitor/gate presentation fields
  onto the synthetic clan container. A clan container never has its own
  `finalizer_status`, and it keeps no link to the member it mirrors. The overlay
  therefore never fires for the clan.
- The `CLAN` header prints `agent.display_status`. Both the full `Status:` line in
  `append_clan_identity_fields` and the compact first row in `build_clan_compact_lines`
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_identity.py`) do this, so
  that surface has no overlay either.

## Other violations found (fixed by this plan)

1. **Lone `STARTING` member.** `apply_clan_container_status` deliberately treats a lone
   `STARTING` member as a non-inheritable source, so the clan reads `RUNNING` while its
   only running member reads `STARTING`. The test
   `test_clan_stays_running_for_lone_starting_member` pins this behavior; the plan
   reverses it.
2. **`RETRYING` countdown.** The member row renders `RETRYING (12s)` from its own
   `retry_next_at_epoch`. The clan inherits the `RETRYING` label but has no
   `retry_next_at_epoch`, so it renders a bare `RETRYING`.
3. **Editor helper agent catalog.** `_clan_entries` in
   `src/sase/integrations/_editor_helper_agents.py` computes a clan's `status` with the
   bare precedence ladder (`aggregate_agent_group_status`). A clan whose only in-flight
   member reads `STARTING`, `WORKING PLAN`, `RETRYING`, `ANSWERED`, etc. is therefore
   reported as `clan · N members · RUNNING`.

## Surfaces checked and intentionally left unchanged

- The **CLAN NEIGHBORS roster**, **node finder rows**, **tribe roster labels**,
  **BY_STATUS grouping**, **count chips**, **summary counts**, and **rail glyphs** all
  read the clan container's stored `status`/bucket. They become correct automatically
  once the projection is right. The tribe roster still colors a clan unit from an
  attention-oriented aggregate over flattened rows, so a transient mirrored `STARTING`
  label is drawn in the Running color. The label itself matches, and changing that
  bucket would alter the tribe `ATTENTION` section, so it stays.
- **Prompt-bar completion candidates.** `_build_clan_completion_candidates` in
  `src/sase/ace/tui/_agent_completion_candidates.py` shows only a status-colored `●`
  dot. Its member candidates are also raw-status aggregates, and `status_style` colors
  `RUNNING` and `STARTING` identically. Switching the clan to refined node labels would
  make the clan dot `dim` while its session member's dot stays teal, which is worse.
- **Wait-dependency clan buckets.** `collect_agent_wait_status_maps` in
  `src/sase/ace/tui/_agent_completion_wait.py` holds wait semantics, not the node's
  status label.
- **`aggregate_clan_status` itself.** It is also the tribe, hood, tab, session, and
  imported-session aggregator; only clan-node call sites change.
- **Chips outside the status parentheses.** The `⊛` finalizer chip, the `⚒` tool-run
  chip, and the `↻N▸model` retry annotation are not the status word. Clan rows keep none
  of them.
- **Rust boundary.** sase-core has no clan or group status aggregation; it has only
  fleet wire buckets. The bucket mapping and the precedence ladder live in
  `src/sase/agent/status_buckets.py`, so the shared rule goes there, next to them.
  Moving agent status-bucket semantics into sase-core is a separate migration and out of
  scope.

## The rule

A clan's **running member** is a deduplicated direct member agent node, as returned by
`clan_members`, whose effective bucket (`agent_status_bucket`) is `Running` or
`Starting`. This is exactly what the clan count chip counts as `R`.

When a clan has exactly one running member and the precedence-ladder aggregate bucket is
`Running`, the clan node shows that member's status exactly. The aggregate bucket is
`Running` precisely when no member is asking, awaiting plan review, or failed. Mirroring
covers:

- the stored label (`status`), the effective bucket (`Running` **or `Starting`**), and
  the monitor/gate presentation fields, which are copied today already;
- the render-time overlays of the status word, `FINALIZING` and `RETRYING (Ns)`. These
  are resolved through a new `status_display_source` pointer to the mirrored member.

All other behavior stays as it is today:

- the ladder aggregate;
- inheriting the label of a lone `Failed`/`Stopped` member when the aggregate bucket
  matches;
- the lone-queued-member admission rank through `wait_display_source`;
- the empty-member fallback.

## Implementation

### 1. Shared pure rule: `src/sase/agent/status_buckets.py`

Add these next to `aggregate_agent_group_effective_status`, and export them the way this
module exposes its other public helpers:

```python
IN_FLIGHT_STATUS_BUCKETS: frozenset[str] = frozenset({"Starting", "Running"})


def clan_status_source_index(entries: Sequence[tuple[str, str]]) -> int | None:
    """Return the index of the member whose own status a clan shows, else None."""


def aggregate_clan_member_status(entries: Sequence[tuple[str, str]]) -> str | None:
    """Return a clan's display status from per-member (status, bucket) entries."""
```

`entries` holds one `(status, effective_bucket)` pair per deduplicated direct member.
`clan_status_source_index` works as follows:

1. `aggregate = aggregate_agent_group_effective_status(entries)`. Return `None` when the
   result is `None`.
2. `aggregate_bucket = status_bucket_for_values(aggregate)`.
3. `relevant` holds the indices whose bucket is not `Queued`, `Waiting`, or `Done`.
4. If exactly one relevant index `i` exists, return it when either:
   - its bucket is in `{"Failed", "Stopped", "Running"}` and equals `aggregate_bucket`
     (today's rule); or
   - its bucket is `"Starting"` and `aggregate_bucket == "Running"` (new).
5. Otherwise return `None`.

A lone in-flight member alongside only queued/waiting/done members always takes branch 4
when no attention member exists. A second in-flight member, or any failed/asking member,
makes `relevant` longer than one.

`aggregate_clan_member_status` returns `entries[i][0]` when an index exists; otherwise
it returns `aggregate`.

Do not change `aggregate_agent_group_status`, `aggregate_agent_group_effective_status`,
or `aggregate_agent_group_bucket`.

### 2. Clan projection: `src/sase/ace/tui/models/_agent_clan.py`

- In `apply_clan_container_status`, keep the identity deduplication. Build `entries`
  from `(member.status, agent_status_bucket(member))` and choose the source with
  `clan_status_source_index`. Delete the now-dead local selection logic
  (`_CLAN_INHERITABLE_STATUS_BUCKETS`, `_CLAN_IGNORED_MEMBER_STATUS_BUCKETS`, and the
  `relevant_members`/`source_bucket` block).
- When a source exists:
  - copy its `status`;
  - set `status_bucket = agent_status_bucket(source)`;
  - copy the presentation fields with the existing `_copy_turn_status_presentation`;
  - set `container.status_display_source = source`.
- Otherwise:
  - set `status` to the aggregate, or to `fallback` when the aggregate is `None`;
  - set `status_bucket = None`;
  - clear the presentation fields;
  - set `status_display_source = None`.
- Leave `_set_clan_queued_wait_display_source` unchanged.
- Rewrite the docstring to state the rule above. Remove the sentence saying a lone
  `STARTING` member leaves the clan at `RUNNING`. Say that clearing on every call keeps
  repeated projections (tree projection and both runner-slot refresh paths in
  `agent_runner_slots.py`) idempotent.
- Add and export `status_display_agent(agent: Agent) -> Agent`, which returns
  `agent.status_display_source or agent`.
- `_remote_agent_session_container` in `_fleet_agents_nodes.py` also calls this
  function, so remote session containers pick up the same `STARTING` mirror and pointer.
  Keep their existing tests passing.

### 3. New runtime-only field on `Agent`

In `src/sase/ace/tui/models/_agent_state_session.py`, next to `wait_display_source`,
add:

```python
    # Member row whose status this synthetic container mirrors (a clan's lone
    # running member). Runtime-only presentation plumbing; not serialized.
    status_display_source: Agent | None = field(
        default=None,
        compare=False,
        repr=False,
    )
```

`compare=False`/`repr=False` are required. The pointer targets one of the container's
own `runtime_children`, so dataclass eq/repr and the repr-based hint digest would
otherwise recurse. The clan row still invalidates correctly for these reasons:

- `runtime_children` stays a compared field, and it carries the member's `status`,
  `finalizer_status`, and `retry_next_at_epoch`, so the incremental display diff
  (`build_agent_display_diff`, which uses dataclass `!=`) still flags the container as
  changed.
- `_runtime_signature` in `src/sase/ace/tui/widgets/_agent_list_render_cache.py` already
  recurses into runtime children.

Register the field everywhere `wait_display_source` is registered as runtime-only:

- `_RUNTIME_ONLY_BUNDLE_FIELDS` and `_AGENT_SIGNATURE_SKIP_FIELDS` in
  `src/sase/ace/tui/models/agent_bundle.py`;
- the skip set in `src/sase/ace/tui/actions/agents/_fleet_projection.py`;
- the pointer list in the `src/sase/ace/tui/models/_agent_graph.py` module docstring.

Run `grep -rn wait_display_source src tests` to catch any other inventory, such as a
field-coverage test.

### 4. One presented-status helper: `src/sase/ace/tui/models/finalizer_row_state.py`

Add and export three helpers:

- `glance_finalizer_state(agent)`. Move the dispatch here from the widget-private
  `_glance_finalizer_state`: session container rows use `session_finalizer_row_state`,
  and all other rows use `finalizer_row_state`.
- `row_status_is_finalizing(agent)`. Returns
  `agent.status == "RUNNING" and glance_finalizer_state(status_display_agent(agent)).is_finalizing`.
- `presented_status_label(agent)`. Returns `"FINALIZING"` when
  `row_status_is_finalizing(agent)` is true, else `agent.display_status`.

Import `status_display_agent` inside the function if a module-level import would create
a cycle (this module already defers its `agent_session_members` import).

### 5. Agents-tab row: `src/sase/ace/tui/widgets/_agent_list_render_agent_status.py`

- `RUNNING` branch: use `row_status_is_finalizing(agent)`.
- `RETRYING` branch: read `retry_next_at_epoch` from `status_display_agent(agent)`.
- `_append_finalizer_chip` keeps using the row's own glance state through
  `glance_finalizer_state(agent)`. Drop `_glance_finalizer_state` unless tests import
  it; keep no dead alias. `_append_tool_run_chip` is unchanged.

### 6. CLAN detail header: `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_identity.py`

Both `append_clan_identity_fields` (the `Status:` line) and `build_clan_compact_lines`
(the row-1 status; the node finder preview reuses it) print
`presented_status_label(agent)` instead of `agent.display_status`. Keep the
`CLAN_MEMBER_STATUS_STYLES[agent_status_bucket(agent)]` styling. `FINALIZING` stays in
the Running bucket, and a mirrored `STARTING` gets the Starting style.

### 7. Editor helper catalog: `src/sase/integrations/_editor_helper_agents.py`

In `_clan_entries`, compute the clan `status` as
`aggregate_clan_member_status([(m.status, status_bucket_for_values(m.status)) for m in generation_members]) or "RUNNING"`.
Keep `_aggregate_status` for sessions, hoods, tribes, and tabs.

Catalog members are flat turn records. A sequential session contributes at most one
in-flight turn, which is the turn the TUI session row mirrors, so the rule carries over.

### 8. Docs

- `docs/ace.md`, the clan-row status paragraph beginning "Clan rows aggregate member
  status using the same operational precedence":
  - state the rule: exactly one running member, `STARTING` included, and no asking,
    reviewing, or failed member, means the clan shows that member's exact status;
  - add that the rule includes the `FINALIZING` word and the `RETRYING (Ns)` countdown,
    on both the clan row and the CLAN `Status:` line;
  - remove any implication that a lone `STARTING` member reads `RUNNING`;
  - keep the existing `TESTING`/`TESTED`/`QUEUED #3/4` examples.
- `docs/ace.md`, the "At a glance: FINALIZING rows and ⊛ chips" paragraph: add one
  sentence saying a clan whose lone running member is finalizing also reads
  `FINALIZING`, while the `⊛` chip stays on the member row.
- `docs/editor.md` ("clan rows also have aggregate `status`") and `docs/integrations.md`
  ("clan rows also include aggregate status"): say that the clan `status` follows the
  Agents-tab clan rule (a lone running member's status, otherwise the aggregate).

### 9. Glossary (the user asked for this update)

Edit the canonical strands, then run `sase memory init` to regenerate the generated
instruction files. Never hand-edit `AGENTS.md`/`CLAUDE.md`. The glossary web uses
implicit links, so plain term mentions link themselves.

- `sase/memory/glossary/agent-clan.md`: append to the paragraph:

  > An agent clan node's status aggregates its member agent nodes, except that a clan
  > with exactly one running member (`STARTING` included) shows exactly that node's
  > status — its label, styling, and row overlays such as `FINALIZING` — rather than a
  > generic `RUNNING`. Only a member that needs the user (asking, awaiting plan review,
  > or failed) outranks it.

- `sase/memory/glossary/sase-node.md`: append:

  > A node's status is the word its row shows, including render-time overlays such as
  > `FINALIZING`; a container node derives its status from its members, and an agent
  > clan node with exactly one running member node shows that node's status.

## Tests

- **`tests/test_agent_status_buckets.py`** — table tests for `clan_status_source_index`
  and `aggregate_clan_member_status`:
  - lone `RUNNING`, `TESTING`, `WORKING PLAN`, or `RETRYING` among `DONE`/`WAITING`
    returns the index or label;
  - lone `STARTING` returns its index or label (new);
  - `STARTING` plus `RUNNING` returns `None`, giving `RUNNING`;
  - a lone in-flight member plus `FAILED`, `QUESTION`, or `PLAN` returns `None`, giving
    that status;
  - a lone `TESTED` entry with bucket `Failed` among waiting/done members returns its
    index;
  - only queued, waiting, or done members return `None`;
  - an empty list returns `None`.
- **`tests/ace/tui/models/test_agent_tree_clan_status.py`**:
  - Replace `test_clan_stays_running_for_lone_starting_member` with
    `test_clan_mirrors_lone_starting_member`. Expect:
    - `status == "STARTING"`;
    - `agent_status_bucket(container) == "Starting"`;
    - `status_display_source` is the member;
    - `format_agent_option` plain text starts with `(STARTING`, with the label style
      equal to the member row's (reuse `_style_at`).
  - Screenshot case: members `DONE`×4, `WAITING`×2, and one `RUNNING` member whose
    `finalizer_status` comes from `finalizer_status_from_mapping(...)`
    (`sase.core.agent_scan_wire_markers`, as in
    `tests/ace/tui/widgets/test_finalizer_glance_surfaces.py`) with phase `declaring`.
    Expect:
    - the container status stays `RUNNING`;
    - the pointer is the member;
    - the clan row text starts with `(FINALIZING)` and includes `[R1 W2 D4]`;
    - the `FINALIZING` style equals the member row's.
  - Repeat with phase `executing` and a running instance.
  - A lone running **session** member whose current turn is finalizing makes the clan
    row read `(FINALIZING`. Reuse the session fixtures from
    `test_clan_mirrors_lone_queued_agent_session_turn_rank`.
  - Two running members, one finalizing, give `(RUNNING` and no pointer.
  - A finalizing member plus a `FAILED` member gives `(FAILED` and no pointer.
    Parametrize or extend
    `test_clan_status_preserves_precedence_over_lone_running_member` to also assert
    `status_display_source is None`.
  - Settling: after the member's summary becomes phase `settled`, re-rendering the same
    container reads `(RUNNING`. After the member turns `DONE` and `project_clan_tree`
    reprojects, the pointer is `None`.
  - `RETRYING` countdown: a lone member with `status="RETRYING"` and
    `retry_next_at_epoch = time.time() + 30` makes both the member and clan rows match
    `RETRYING \(\d+s\)`.
  - Cache invalidation: `agent_render_key(container, 0, ...)` from
    `_agent_list_render_cache` differs before and after changing only the member's
    `finalizer_status`.
- **`tests/test_agent_clan.py`** — extend
  `test_apply_clan_container_status_empty_input_uses_fallback_and_clears_source` to
  assert that `status_display_source` is cleared.
- **`tests/ace/tui/test_agent_runner_slots_agent_sessions.py`** — a clan with a lone
  `STARTING` member plus a `WAITING` member keeps `STARTING` and its pointer after both
  `refresh_runner_slot_context(projected)` and
  `refresh_runner_slot_context(projected, effective_limit=10)`. Keep
  `test_first_refresh_promotes_all_slot_waiters_and_clan_aggregate` unchanged.
- **`tests/ace/tui/widgets/test_identity_header_compact.py`** (or the existing
  CLAN-header test module) — a clan whose lone running member is finalizing shows
  `FINALIZING` on both the expanded `Status:` line and the compact first row. A lone
  `STARTING` member shows `STARTING`.
- **`tests/ace/tui/widgets/test_finalizer_glance_surfaces.py`** —
  `presented_status_label` and `row_status_is_finalizing` cover:
  - a plain agent;
  - a session container;
  - a clan container with and without a pointer;
  - a non-`RUNNING` clan whose pointer member has an active summary, which stays
    un-overlaid.
- **`tests/test_editor_helper_agent_catalog.py`** — a clan catalog entry with one
  `STARTING` record plus `DONE` records reports `status == "STARTING"` and a detail
  ending `· STARTING`. One `RUNNING` plus one `FAILED` reports `FAILED`. Two `RUNNING`
  records report `RUNNING`.

Update any other existing expectation that encoded the old lone-`STARTING` collapse, and
name it in the final summary. Do not weaken precedence, bucket, or count assertions.

## Verification

- Run `just install` if the workspace virtualenv is stale, then `just fmt`.
- Run `sase tool run check`, which covers every lint gate (ruff, mypy, symvision) plus
  the diff-scoped test lane.
- Visual goldens: no visual fixture currently holds a clan whose lone running member is
  `STARTING`, `RETRYING`, or finalizing, so no PNG golden should change. Confirm with a
  targeted check-only run over the clan, tribe, node-finder, and fleet visual modules
  (`just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py ...`).
  Use `/sase_monitor` if it will not fit the synchronous limit. If a golden legitimately
  changes, inspect it and update it with `just fix-tui-screenshots` for those selectors.
- Manual smoke check (optional): in `sase ace`, a clan whose only running member is
  finalizing shows `(FINALIZING)` on the clan row and `FINALIZING` in the CLAN header.
