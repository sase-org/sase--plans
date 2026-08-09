---
tier: tale
title: Show the hosted agents-sidecar page URL in the Agents metadata panel
goal:
  Selecting a Done-bucket agent that made commits on the Agents tab shows its hosted `agents` sidecar page URL near the
  top of the metadata panel, on exactly one never-wrapping line.
size: medium
proposed_by: bbugyi200.athena.ro
create_time: 2026-08-02 06:58:55
status: done
---

- **PROMPT:**
  [prompts/202608/agent_page_url_metadata_panel.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/agent_page_url_metadata_panel.md)

# Plan: Show the hosted agents-sidecar page URL in the Agents metadata panel

## Goal

When a terminal (`Done`-bucket) agent that produced commits is selected on the Agents tab, render its hosted `agents`
sidecar page URL near the top of the agent metadata panel, on exactly one line that never wraps.

## Background

### The URL already has an owner

`HostedLinkResolver.agent_url()` in `src/sase/sdd/hosted_links.py` already derives exactly the URL this plan needs:

- it resolves the project's `agents` sidecar target via `resolve_sync_targets()`,
- it resolves the sidecar's hosted branch via `resolve_hosted_branch()`,
- it maps the agent name to its sidecar page path via `agent_link_target()` / `lane_page_path()`
  (`agents/<owner>.<host>.<lane>/README.md` for a solo lane, `families/<global>.md` for a family lane, plus a `#anchor`
  for a concrete family member),
- it composes the final blob URL via `github_blob_url()`, returning `None` instead of guessing whenever the sidecar is
  missing, detached, non-hosted, or non-GitHub.

It is already used by `sase bead show`, bead pages, and plan headers. **Do not reimplement any of this derivation.**
This plan only adds a TUI-side adapter that calls it with the right project/primary-root anchor, plus a rendering lane.

### Why the resolution cannot happen on the render path

`agent_url()` shells out to git (`git symbolic-ref`, `git remote get-url`) and reads the project registry / repo
inventory on its first call for a project. Per `sase/memory/tui_perf.md` rule 1 and rule 8, render paths must never do
subprocess or disk work. The Agents detail panel already has the correct home for this: the threaded detail-header
enrichment worker (`start_agent_detail_header_enrichment` in
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_async_agent.py`), whose result is the cached `DetailHeaderSummary`
that `build_header_text()` reads. Everything expensive in this plan goes there.

`hosted_link_resolver()` memoizes resolver instances process-wide, and each resolver memoizes its remote/provider/
branch lookups forever, so only the first resolution per project pays the git cost.

### Why wrapping is the interesting rendering problem

The metadata header is one large Rich `Text` rendered inside a `VerticalScroll`, so a long URL would wrap across several
lines by default. The header already has an established mechanism for width-aware sub-blocks: a _responsive section_.
`build_header_text()` collects `(start, end, section)` triples into `AgentHeaderRenderable`
(`_agent_display_header_renderable.py`), which yields each section as its own Rich renderable at render time while the
section's `logical_text` stays in the logical header for search/inspection. `ResponsiveWaitSection`,
`ResponsiveBeadSection`, `ResponsivePlanSection`, and `ResponsiveSlowToolCallsSection` are the existing examples. Use
that mechanism — it is the only place in this header where a block can set `no_wrap`.

## Behavior specification

### Visibility predicate

Render the `Page:` lane only when **all** of these hold:

1. The agent's status bucket is `Done` — use `status_bucket_for_values(agent.status)` from `sase.agent.status_buckets`,
   not a literal `== "DONE"` comparison. _Rationale / assumption to flag at review:_ the request said "DONE agent". The
   `Done` bucket additionally covers `PLAN DONE`, `TALE DONE`, `PLAN REJECTED`, `EPIC CREATED`, `STOPPED`, and the
   feedback status. An agent in any of those states that made commits is published to the sidecar just as much as a
   plain `DONE` one, so gating on the bucket avoids an arbitrary hole. If review prefers the literal status, narrowing
   this is a one-line change.
2. The agent made at least one commit — `agent_commit_groups(agent)` from
   `src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py` is non-empty. That accessor parses in-memory
   `step_output["meta_commits"]` only and performs no disk I/O, which is what makes it safe to call from the predicate.
   Commits are the trigger for `publish_committed_agent_hood()`, so "made commits" is the correct proxy for "published
   to the `agents` repo".
3. `agent.agent_name` is set and the row is not a synthetic container — skip when `agent.is_clan_container` is true (a
   clan is a hood, not a lane, and has no lane page).
4. The resolved URL is not `None`.

When any condition fails, render nothing at all. Do **not** render a placeholder, a dimmed "unavailable" string, or a
bare label.

### Placement

Render the lane inside the identity block of the metadata header, immediately after the `Name:` line and its optional
`Bead:` line, and before the retry-chain line — i.e. it is the second field group the user sees, above `ChangeSpec:` /
`Project:` / `Workspace:`.

_Assumption to flag at review:_ "somewhere at the top" is read here as "in the top identity block". Leading with `Name:`
is kept because it is the panel's established first line and the primary identifier. Moving the lane above `Name:` would
be a one-line reordering if review prefers the literal top.

### Format

```
Page: https://github.com/<owner>/<agents-repo>/blob/<branch>/agents/<owner>.<host>.<lane>/README.md
```

- Label `Page: ` styled `bold #87D7FF`, matching every other field label in the header.
- URL styled `bold underline #569CD6`, matching the existing hosted-link styling used for `BUG:` and `cl_num`.
- Exactly one line; the lane ends with a single newline.

### Non-wrapping

The rendered lane must occupy exactly one terminal line at every panel width. Render the lane's `Text` with
`no_wrap=True` and `overflow="ellipsis"` so a URL wider than the pane is truncated with an ellipsis instead of wrapping.
The complete, untruncated URL must still be present in the logical header text (`header.plain`) so content search and
copy-of-visible paths keep seeing it.

## Implementation

### 1. TUI-side URL adapter

Add `src/sase/ace/tui/models/agent_page_url.py` with:

```python
def agent_publishes_page(agent: Agent) -> bool: ...
def resolve_agent_page_url(agent: Agent) -> str | None: ...
```

`agent_publishes_page()` implements conditions 1–3 of the visibility predicate above. It must stay pure and I/O-free so
callers can use it as a cheap gate.

`resolve_agent_page_url()`:

1. Return `None` immediately unless `agent_publishes_page(agent)`.
2. Derive the project key from `Path(agent.project_file).parent.name`; return `None` when `agent.project_file` is empty.
3. Derive the project's stable primary root with `parse_workspace_dir(agent.project_file)` from
   `sase.workspace_provider.utils`, falling back to `agent.workspace_dir`. **Do not anchor on the agent's own ephemeral
   `sase_<N>` workspace as the first choice** — a terminal agent's numbered workspace may have been recycled and
   reassigned, which would make `resolve_checkout_anchor()` inside `_resolve_agents_remote()` fail or resolve against
   the wrong checkout. Return `None` when neither is available.
4. Build the store with `resolve_sdd_store(primary_root, 1)` and the resolver with
   `hosted_link_resolver(store, project=project_key, primary_root=primary_root)`. The store is only needed because
   `HostedLinkResolver` requires one for its plans/beads accessors; the `agents` accessor never touches it.
5. Call `resolver.snapshot_agent_name_registry()` before `resolver.agent_url(agent.agent_name)`, so a solo-vs-family
   lane is classified from the reservation registry rather than degrading to a solo lane. This is safe on the worker
   thread: `load_name_registry()` is mtime-cached.
6. Return `resolver.agent_url(agent.agent_name)`.
7. Wrap steps 2–6 in a broad `except Exception: return None`. This is a best-effort presentation affordance and must
   never break the detail panel; that matches how `hosted_links.py` and `agents_sync/links.py` already treat
   unresolvable link targets.

Keep a module-level memo of `project_key -> primary_root` (or of the resolved store) if profiling shows
`resolve_sdd_store` is hot; `hosted_link_resolver` already memoizes the resolver itself.

Note on the Rust core boundary (`sase/memory/` → `rust_core_backend_boundary`): no new domain logic is introduced here.
The URL derivation stays owned by `sase.sdd.hosted_links` / `sase.agent_lanes`; this module is a thin presentation-side
adapter that supplies the project anchor a TUI row knows and the CLI callers get from `cwd`.

### 2. Carry the URL on `DetailHeaderSummary`

In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_state.py`, add to `DetailHeaderSummary`:

```python
agent_page_url: str | None = None
```

Keep it a trailing field with a default so every existing construction site and test keeps working.

### 3. Populate it in the enrichment worker

In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_summary.py`:

- Add an `include_agent_page_url: bool = True` keyword to `build_detail_header_summary()`, mirroring the existing
  `include_slow_tools` flag.
- When enabled, populate the new field with `resolve_agent_page_url(agent)`.
- Pass `include_agent_page_url=False` from `src/sase/ace/tui/widgets/prompt_panel/_agent_clan_member_content.py`, which
  builds a summary per clan member for aggregation and has no use for a per-member page lane.

This function already runs off the event loop in the threaded worker started by `start_agent_detail_header_enrichment`,
and its result is TTL-cached per agent identity, so no new refresh path, timer, or worker is needed. **Do not add one.**

### 4. Responsive rendering lane

Add `src/sase/ace/tui/widgets/prompt_panel/_agent_page_section.py` exporting:

- `AGENT_PAGE_SECTION_ID = "agent-page"`
- `AGENT_PAGE_FIELD_LABEL = "Page: "` and its label style constant
- `ResponsiveAgentPageSection`, a slotted dataclass holding the URL, with:
  - a `logical_text` property returning the full `Page: <url>\n` styled `Text` (used for search/inspection), and
  - `__rich_console__` yielding a single `Text(no_wrap=True, overflow="ellipsis")` carrying label + URL.

Follow the shape of `ResponsiveWaitSection` in `_agent_wait_section.py` exactly (same `logical_text` /
`__rich_console__` contract, same `__all__` discipline).

Register the section type in the `ResponsiveHeaderSection` union in
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_renderable.py`.

### 5. Wire the lane into the header

In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py`:

- Give `_append_identity_fields()` access to `responsive_ranges` and have it, after the `Name:`/`Bead:` lines and before
  the retry-chain line, append `section.logical_text` when `summary is not None and summary.agent_page_url` is set,
  recording `responsive_ranges[AGENT_PAGE_SECTION_ID] = (start, len(text))` and returning the section.
- Change `append_agent_metadata_fields()` to return the page section alongside its existing results. Prefer a small
  frozen dataclass return over growing the tuple, and update both call sites: `build_header_text()` in
  `_agent_display_header.py` and `tests/ace/tui/widgets/test_agent_wait_section.py`.

In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py`, append the page section to `responsive_sections`
exactly like the wait/bead/plan/slow-tool sections are appended (guarded on both a non-`None` section and the section id
being present in `responsive_ranges`). The existing `responsive_sections.sort(key=...)` already puts it in document
order, so no ordering special case is needed.

## Testing

Add unit tests under `tests/ace/tui/widgets/` (follow the file-per-behavior split the repo has been converging on — see
recent `test: split ...` commits — rather than appending to one large module):

1. **Predicate — shown.** A `Done`-bucket agent with `step_output["meta_commits"]` populated and a stubbed resolver
   produces a `Page:` line whose value is the resolved URL.
2. **Predicate — hidden.** No `Page:` line for: a `RUNNING` agent with commits; a `DONE` agent with no commits; a `DONE`
   agent with commits whose resolution returns `None`; a clan container row.
3. **Bucket coverage.** A `PLAN DONE` agent with commits renders the lane (locks in the bucket-not-literal decision so
   it is a deliberate, reviewable choice).
4. **Placement.** In the rendered header `plain`, the `Page:` line appears after `Name:` and before `ChangeSpec:` /
   `Project:`.
5. **No wrap.** Render the header through `rich.console.Console(width=40)` (a width far narrower than a realistic URL)
   and assert the `Page:` lane occupies exactly one output line and ends in an ellipsis, while the full untruncated URL
   is still present in `header.plain`.
6. **Adapter.** Direct tests for `resolve_agent_page_url()` covering: missing `project_file`; missing primary root; a
   raising `resolve_sdd_store` / `agent_url` (must return `None`, not propagate); and the happy path asserting
   `hosted_link_resolver` is called with the project key and the project's primary root — _not_ the agent's ephemeral
   workspace dir.
7. **Summary flag.** `build_detail_header_summary(agent, include_agent_page_url=False)` leaves `agent_page_url` as
   `None`.

Stub the resolver boundary in these tests (patch `hosted_link_resolver` / `resolve_sdd_store` in the new adapter
module). Do **not** let any test shell out to git or touch a real sidecar clone.

### Visual snapshots

Adding a header line changes the ACE Agents PNG goldens for any fixture whose selected agent is a `Done`-bucket row with
commits. Run `just test-visual`, inspect the diffs under `.pytest_cache/sase-visual/`, confirm the only change is the
new `Page:` line, and only then re-run with `--sase-update-visual-snapshots` to accept them. If a fixture's resolution
reaches real git, pin it in the fixture instead of accepting a machine-dependent golden.

## Validation

Run `just install` first (workspace virtualenvs go stale), then `just check`.

## Out of scope

State these explicitly rather than silently implementing them:

- Making the URL clickable/openable. `append_text_with_file_hints()` deliberately excludes HTTP(S) URLs from the `[N]`
  hint numbering, and the lane is rendered by its own section so it is not hint-scanned at all. No hint, OSC-8
  hyperlink, or "open in browser" action is added.
- A new `agents` copy target for the page URL (`COPY_TARGETS` in `src/sase/ace/tui/copy_targets.py`). Worth a follow-up
  task bead; not part of this change.
- Showing the URL for non-terminal agents, for agents published only because a hood neighbor committed, or on the
  clan/tribe/family aggregate documents.
- Any change to publication itself (`sase/agents_sync/`), to commit footers, or to `HostedLinkResolver`.
