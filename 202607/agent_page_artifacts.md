---
tier: epic
title: Agent sidecar pages link commits, neighbors, and output variables
goal: 'Every published agent and family page in the `--agents` sidecar links its commits,
  its lane neighbors, and the SASE output variables the run set, using one consistent,
  deterministic page anatomy.

  '
phases:
- id: shell
  title: Page shell, breadcrumbs, and golden refresh tooling
  depends_on: []
  size: medium
  description: 'shell: split the browsing renderer into focused modules, add breadcrumb
    navigation to agent and family pages, and add the pytest flag that rewrites the
    agents-sync markdown goldens.

    '
- id: commits
  title: Commit artifacts on agent and family pages
  depends_on:
  - shell
  size: medium
  description: 'commits: resolve the primary repository remote during inventory and
    render per-run and per-family commit tables whose SHAs link to the hosted commit
    page when the remote is GitHub.

    '
- id: vars
  title: Published SASE output variables
  depends_on:
  - shell
  size: medium
  description: 'vars: carry sanitized `output_variables` across the v2 wire under
    strict validation, render them on agent and family pages, and restore them into
    imported local artifacts.

    '
- id: neighbors
  title: Lane neighbors on agent and family pages
  depends_on:
  - shell
  size: medium
  description: 'neighbors: build the name-derived lane kinship projection for a hood
    snapshot and render the Neighbors section with ancestor, descendant, and hood-group
    rows that link to their pages.

    '
- id: polish
  title: Whole-page integration, docs, and consistency pass
  depends_on:
  - commits
  - vars
  - neighbors
  size: small
  description: 'polish: add the all-sections integration golden, reconcile section
    order and anchors across the three feature phases, and document the complete page
    anatomy.

    '
create_time: 2026-07-27 16:35:19
status: wip
bead_id: sase-a9
---

- **PROMPT:** [202607/prompts/agent_page_artifacts.md](prompts/agent_page_artifacts.md)
- **BEAD:** [sase-a9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-a9/README.md)

# Plan: Agent sidecar pages link commits, neighbors, and output variables

## Context

`sase` publishes each project's agent history into that project's hidden `agents` sidecar repository. Alongside the
strict JSON wire (`snapshot.json`, `meta.json`, `state.json`, `commits.json`), it renders a browsable Markdown tree so
the sidecar's GitHub page is readable by a human:

```text
README.md                                                  root index
users/<user>/README.md                                     user index
users/<user>/machines/<machine>/README.md                  machine index
users/<user>/machines/<machine>/hoods/<hood>/README.md     hood roster
agents/<global-name>/README.md                             one agent page
families/<global-family>.md                                one family page
```

All of that Markdown is produced by one pure function, `render_browsing_payload` in `src/sase/agents_sync/rendering.py`,
which receives every validated owner manifest and every hood snapshot in the sidecar and returns `{path: bytes}`.
`src/sase/agents_sync/publication.py::_publish_hoods` calls it on every publication, so the entire Markdown tree is
rebuilt from validated wire data each time and written through `apply_payload_atomic`. Identical inputs must produce
byte-identical output; the renderer contains no clock, no randomness, and no host-local paths.

Today an agent page carries an identity block, a `Summary` list (model, provider, timing, commit **count**), and a
`Files` line linking `prompt.md` / `chat.md`. Three useful artifacts are missing:

1. The commits a run produced are counted but never listed or linked.
2. There is no navigation to related agents; only the hood roster lists siblings, and agent pages do not even link back
   to it.
3. Output variables set with the `/sase_var` skill (`sase var set KEY=VALUE`) never reach the sidecar at all: they live
   in `agent_meta.json` under `output_variables`, and `V2_METADATA_FIELDS` in `src/sase/agents_sync/v2_io.py` does not
   allowlist that key, so publication strips it.

This epic adds all three, under one page anatomy shared by agent and family pages.

## Target page anatomy

Both page kinds use the same ordered skeleton. A section is omitted entirely when it has no rows, so simple runs stay as
short as they are today.

### Agent page — `agents/<global-name>/README.md`

```markdown
# Agent: foo.bar.baz--code

[Agent Hoods](../../README.md) / [alice](../../users/alice/README.md) /
[athena](../../users/alice/machines/athena/README.md) / [foo](../../users/alice/machines/athena/hoods/foo/README.md) /
[foo.bar.baz](../../families/alice.athena.foo.bar.baz.md) / foo.bar.baz--code

**Global name:** `alice.athena.foo.bar.baz--code` · **State:** completed · **Source run:** `run-01`

**Owner:** `alice.athena` · **Project:** Project · **Hood:** foo

## Summary

- Model: gpt
- Provider: codex
- Timing: 2026-07-23T12:00:00+00:00 → 2026-07-23T12:01:00+00:00
- Commits: [2](#commits)
- Variables: [3](#variables)

## Files

[Chat](chat.md) · [Prompt](prompt.md)

## Commits

| Commit                                                          | Subject             | Committed (UTC)     |
| --------------------------------------------------------------- | ------------------- | ------------------- |
| [`0a1b2c3`](https://github.com/alice/project/commit/0a1b2c3...) | feat: add the thing | 2026-07-23 12:00:31 |

## Variables

| Variable    | Value            |
| ----------- | ---------------- |
| `plan_file` | `sdd/plans/x.md` |
| `status`    | ok               |

## Neighbors

| Agent                                                                    | Relation     | State       |
| ------------------------------------------------------------------------ | ------------ | ----------- |
| [foo](../alice.athena.foo/README.md)                                     | ancestor     | completed   |
| [foo.bar](../alice.athena.foo.bar/README.md)                             | ancestor     | completed   |
| [foo.bar.kazam](../alice.athena.foo.bar.kazam/README.md)                 | foo.bar hood | failed      |
| [foo.rootless](../../families/alice.athena.foo.rootless.md) (family · 2) | foo hood     | completed 2 |
```

The breadcrumb's family segment appears only for a run that belongs to a family container; it replaces today's free-text
sentence "This run is represented in its family lineage."

### Family page — `families/<global-family>.md`

Keeps its `# Family:` heading, identity line, `## Lineage` mermaid diagram, and member table, and gains: the same
breadcrumb (ending at the family), then `## Commits`, `## Variables`, and `## Neighbors`. The family-level `Commits` and
`Variables` tables gain a leading `Role` column so a reader can attribute each row to a member, mirroring how ACE
attributes output variables across family members. The family `Neighbors` table is the projection for the family lane,
so it never lists the family's own members — they are already in the member table directly above.

Hood, machine, user, and root pages are unchanged by this epic.

## Design decisions

**One lane, one roster.** The neighbor projection operates on _lanes_, not raw runs, exactly like the Agents tab: a
family is one lane that owns its members, and every other run is its own lane. A family member's agent page therefore
shows its family's neighbors, not a sibling-versus-sibling list, and its siblings are reached through the breadcrumb's
family link. This is what makes the page reliable to read: each relationship appears exactly once, in exactly one place.

**Neighbor semantics mirror the TUI's NEIGHBORS roster, by specification not by code sharing.**
`tests/agents_sync/test_import_boundaries.py` forbids `sase.agents_sync` from importing `sase.ace`, so the ACE
implementation (`src/sase/ace/tui/models/agent_hoods.py`, `agent_lane_neighbors.py`) cannot be reused; it also operates
on live rendered rows, folds, and clan-reveal state that a static page does not have. The sidecar projection is instead
derived purely from names, on top of the Rust identity primitives already exposed through
`sase.core.agent_identity_facade` (`parse_agent_family_name`, `agent_name_ancestors`, `agent_link_target`), which is
where the actual relationship semantics live. The remaining logic — grouping order, relation labels, row caps — is page
layout policy. If a third consumer ever needs this projection, promote it into `sase_core` then; doing it now would
force a cross-repo `sase-core` release cycle for a Markdown rendering change, so it is deliberately deferred.

**Commit links come from the publisher's primary remote, not from the wire.** A sidecar serves exactly one project, and
that project's primary repository is the same repository for every owner in it, so the commit URL base does not need to
be published per owner. Publication resolves `remote.origin.url` from the primary checkout it already runs `git log`
against, and the renderer builds `https://<host>/<owner>/<repo>/commit/<sha>` only when that remote is recognizably
GitHub. This keeps the strict wire schema untouched. When no hosted remote is recognized, commits still render as plain
short SHAs with subjects, so a bare-git project loses the hyperlink and nothing else.

**Output variables are the one wire change, and it is a strict one.** They are real published data, so they go into the
run's portable metadata with explicit shape validation rather than being smuggled into free-form text. The v2 decoders
reject unknown metadata keys by design, which means a machine running an older `sase` will raise `AgentsSyncFormatError`
when it reads a snapshot published by this version. That failure is loud, specific, and fixed by upgrading; see "Risks"
below.

**Everything stays deterministic and bounded.** Every new section sorts by a total order, renders timestamps as UTC from
integer epochs, escapes all interpolated text through the existing Markdown escapers, and caps its row count with an
explicit "and N more" tail so no page can grow without bound.

## Page shell, breadcrumbs, and golden refresh tooling

Foundation phase. Ships one visible change (breadcrumbs) and prepares the module layout and test tooling the other
phases rely on.

**Split the renderer.** `src/sase/agents_sync/rendering.py` is 417 lines and would pass the `toobig` warning threshold
(`just _lint-toobig` uses `1000 850 700`) once three sections are added. Split it into flat sibling modules, matching
the existing `agents_sync` naming style (`v2_import_*.py`, `incoming_cache_*.py`):

- `rendering.py` — keeps `render_browsing_payload` and the payload assembly loop.
- `rendering_markdown.py` — the shared text helpers, made public because Symvision forbids importing `_name` symbols
  across files: `md_escape` (today's `_md`), `md_cell` (`_cell`), `md_code` (`_code`), `md_mermaid` (`_mermaid`),
  `md_html_id` (`_html_id`), `page_url` (`_url`), `relative_page_url` (`_relative_url`), `page_bytes` (`_bytes`), plus
  `state_counts` (`_state_counts`) and `run_timing` (`_timing`).
- `rendering_index_pages.py` — root, user, machine, and hood renderers.
- `rendering_agent_page.py` — the agent page.
- `rendering_family_page.py` — the family page.

Keep every rendered byte identical through the split itself, then apply the breadcrumb change as a separate, reviewable
step so the golden diff shows only breadcrumbs.

**Add breadcrumbs.** Agent and family pages gain a breadcrumb line directly under the `#` heading, in the same style the
hood page already uses. Build every hop with `relative_page_url` from the page's own path so the links stay correct
regardless of nesting:

- agent page: root / user / machine / hood / _family, when the run is in a family_ / current agent name (unlinked).
- family page: root / user / machine / hood / current family name (unlinked).

The hood hop targets `users/<user>/machines/<machine>/hoods/<hood>/README.md`; derive `<hood>` from the snapshot being
rendered (`snapshot.local_hood`), never from the agent name. The family hop targets `families/<global-family>.md` and
uses the localized family name as its text. Remove the now-redundant "This run is represented in its family lineage"
sentence and its `in_family` plumbing only if the breadcrumb covers it — it does, because `_render_run_pages` already
knows which runs belong to a family container.

**Add golden refresh tooling.** `tests/agents_sync/goldens/*.md` are byte-compared in
`tests/agents_sync/test_publication.py`. Three later phases rewrite those files, so add a pytest option mirroring the
existing visual-snapshot precedent in `tests/conftest.py` (`--sase-update-visual-snapshots`):

- register `--sase-update-agents-goldens` in `pytest_addoption`;
- in the golden assertion in `test_publication.py`, write the rendered text to the golden path instead of asserting when
  the flag is set (still failing the run afterward if any golden changed, so a refresh run is never silently green);
- document the flag in a comment at the assertion site.

**Verify.** `just install && just check`, plus `just test tests/agents_sync`. Goldens change by exactly one added
breadcrumb line per agent and family golden.

## Commit artifacts on agent and family pages

**Resolve the primary remote.** `ProjectHoodInventory` (`src/sase/agents_sync/inventory_models.py`) gains a trailing
defaulted field `primary_remote_url: str | None = None` — trailing and defaulted so existing positional constructions in
tests keep working. `src/sase/agents_sync/inventory.py` populates it next to `_historical_associations`, which already
runs git against `target.primary_checkout` through the injected `git_runner`: run `git config --get remote.origin.url`,
strip it, and return `None` on any non-zero exit or empty output. Never raise from this path; a missing remote must
degrade to unlinked commits, not fail a publication.

`_publish_hoods` in `publication.py` resolves the commit URL base once and passes it into `render_browsing_payload` as a
new keyword-only argument (`commit_url_base: str | None = None`), so the renderer stays a pure function of its inputs.

**Build the URL.** Add `github_commit_url(remote_url, *, provider, sha)` to `src/sase/_git_remote.py` next to the
existing `github_blob_url`, reusing `parse_hosted_git_remote` and the same host rule: `github.com` is accepted directly,
any other host only when authoritative provider metadata says GitHub. Resolve that provider argument with the same
configured-GitHub-hosts logic `src/sase/agents_sync/links.py::_hosted_provider` uses today; promote that helper to a
public function in a module both callers can import (`links.py` and the publication path) rather than duplicating it.
Validate `sha` as 7–64 lowercase hex before interpolating.

**Render `## Commits` on agent pages.** One table, `| Commit | Subject | Committed (UTC) |`:

- rows sorted by `(committed_at, sha)`, the same order `CommitRecord` tuples already arrive in;
- commit cell is `` `<sha[:7]>` `` wrapped in a link to the commit URL when one is available, and a bare code span
  otherwise;
- subject escaped with `md_cell`;
- timestamp formatted from the integer epoch as `datetime.fromtimestamp(committed_at, tz=UTC)` rendered
  `%Y-%m-%d %H:%M:%S` — no local timezone, no `datetime.now()`;
- cap at 50 rows and append a final row or trailing line reading `… and N more commits` when truncated.

The `Summary` bullet becomes `- Commits: [<n>](#commits)` when `n > 0` and stays `- Commits: 0` otherwise.

**Render `## Commits` on family pages.** Same table with a leading `| Role |` column holding the member role (the `root`
fallback already used by the member table), rows sorted by `(committed_at, sha, role)`, capped identically. The member
table's existing `Commits` count column becomes a link to that member's `agents/<global>/README.md#commits` when the
count is non-zero.

**Tests.** Extend `tests/agents_sync/test_rendering.py` with: a GitHub remote producing linked SHAs; an unrecognized
remote (e.g. `ssh://git@example.invalid/x/y.git`) producing unlinked SHAs and no crash; a `None` remote; a subject
containing `|`, backticks, and angle brackets staying escaped; truncation past the cap; and epoch-to-UTC formatting
independent of `TZ` (set a non-UTC `TZ` in the test). Extend `tests/agents_sync/test_inventory.py` for remote resolution
including the failure path. Refresh goldens.

## Published SASE output variables

**Publish them.** Add `"output_variables"` to `V2_METADATA_FIELDS` in `src/sase/agents_sync/v2_io.py`. Sanitize on the
way out in `src/sase/agents_sync/inventory_io.py::portable_metadata`: keep only `str -> str` entries whose key matches
`[A-Za-z_][A-Za-z0-9_]*` (the key rule `src/sase/core/agent_output_variables.py` already enforces at write time), drop
the key entirely when nothing survives, and emit the entries sorted so the canonical JSON is stable. A malformed local
`agent_meta.json` must never fail a hood publication.

**Validate them strictly on the way in.** In the metadata validation path used by `_run` in `v2_io.py` (and the parallel
decode in `v2_run_io.py`), add explicit shape validation for the `output_variables` key: a JSON object, at most 256
entries, every key matching the identifier rule, every value a string of at most 8192 UTF-8 bytes. Anything else raises
`AgentsSyncFormatError` with a message naming the offending key, consistent with the surrounding validators.

**Render `## Variables`.** Agent page: `| Variable | Value |`, rows sorted by key, key as a code span, value escaped
with `md_cell` (which already turns newlines into `<br>`). Truncate a rendered value past 200 characters with a trailing
`…`, and when any value was truncated add one line under the table:
`Values are truncated for display; see [meta.json](meta.json) for the full values.` Family page: the same table with a
leading `| Role |` column, rows sorted by `(role, key)`, listing every member that set variables. Add
`- Variables: [<n>](#variables)` to the agent `Summary` list only when `n > 0`; omit the bullet entirely otherwise,
since most runs set none.

**Round-trip on import.** `src/sase/agents_sync/v2_import_rendering.py::artifact_payload` rebuilds a local
`agent_meta.json` from an explicit key list; add `output_variables` to it so an imported foreign run shows its variables
in ACE's `OUTPUT VARIABLES` panel exactly as a local run does.

**Tests.** `tests/agents_sync/test_v2_io.py`: accept a valid map; reject a non-object, a bad key, a non-string value, an
over-long value, and an over-large map. `tests/agents_sync/test_inventory.py`: sanitization drops junk and keeps valid
pairs. `tests/agents_sync/test_rendering.py`: rendering, sorting, escaping of a value containing `|` and a newline, and
truncation with the note line. `tests/agents_sync/test_v2_importer.py` (or the integration variant): the imported
`agent_meta.json` carries the variables. Refresh goldens with a fixture run that sets variables.

## Lane neighbors on agent and family pages

**Build the projection.** New module `src/sase/agents_sync/rendering_kinship.py`, pure and snapshot-scoped, with no I/O:

1. _Lanes._ For one `V2HoodSnapshot`, a run belongs to a family lane when its `source_run_id` appears in a
   `V2ContainerRecord` with `kind == "family"` — the same rule `_render_run_pages` already uses to decide which runs get
   a family lineage link. That lane's name is the container's global name with the `<username>.<machine>.` prefix
   stripped; its page is `families/<global-family>.md`. Every other run is its own lane, named by its `local_name`, with
   page `agents/<global-name>/README.md`. Clan containers are not lanes; clan members are `<clan>.<suffix>` names and so
   surface naturally as `<clan> hood` neighbors, which is exactly how the Agents tab groups them.
2. _Chains._ For each lane, `agent_name_ancestors(lane_name)` returns its hood chain, top hood first, with the lane
   itself as the final element. Use the facade function rather than splitting on `.` locally: it normalizes historical
   names such as `fi--code.f0` that predate current naming rules.
3. _Relations for lane `L`._ Ancestors are the lanes named by `chain(L)[:-1]`, nearest first. Descendants are the lanes
   whose chain contains `L` before their own final element, sorted by case-folded name. Hood groups walk `chain(L)` from
   most specific to least specific; each hood's group is the lanes whose chain contains that hood and that have not
   already been assigned to `L` itself, an ancestor, a descendant, or an earlier group; groups are labelled
   `<hood> hood` and their rows are sorted by case-folded name. This is the Agents-tab NEIGHBORS ordering.

Return an ordered, immutable projection of rows carrying: lane name, relation label, page path, whether the lane is a
family, and the member count for a family lane.

**Render `## Neighbors`.** Same table on both page kinds, `| Agent | Relation | State |`:

- agent cell links the lane name to its page, relative to the page being rendered; a family lane appends
  ` (family · <n>)`;
- relation cell is `ancestor`, `descendant`, or `<hood> hood`;
- state cell is the run's state for a solo lane, and the member state summary (`state_counts`, e.g. `completed 2`) for a
  family lane;
- rows keep projection order (ancestors, descendants, then hood groups);
- cap each group at 50 rows, appending `… and N more in the [hood roster](<hood README>)` so nothing is silently
  dropped;
- omit the whole section when a lane has no neighbors, which is the common case for a lone top-level agent.

An agent page renders the projection for _its lane_, so every member of a family renders the family's neighbor list and
the family page renders the same list. Cross-owner and cross-machine relationships are out of scope: a different owner's
names live in a different global namespace and a different snapshot.

**Tests.** New `tests/agents_sync/test_rendering_kinship.py` covering: ancestors nearest-first; descendants sorted and
excluded from hood groups; hood groups nearest-hood-first with no duplicate lane across groups; a family lane never
listing its own members; a family member page rendering the family lane's neighbors; a lone lane rendering no section;
historical `--` names classified through the facade; and cap-with-tail behavior. Extend `test_rendering.py` for the
rendered table and link correctness (relative paths resolve to files that exist in the same payload). Refresh goldens —
the existing multi-agent hood fixture in `test_publication.py` already produces rich ancestor/descendant/hood relations.

## Whole-page integration, docs, and consistency pass

Runs after the three feature phases and owns the end state.

- Add one integration golden that exercises every section at once: a family whose members carry commits and output
  variables and that has ancestors, descendants, and hood neighbors. Assert section order is exactly Summary, Files,
  Commits, Variables, Neighbors on the agent page, and Lineage, member table, Commits, Variables, Neighbors on the
  family page, and that every `#`-anchor referenced from a Summary bullet exists as a heading on that page.
- Assert idempotence explicitly: publishing twice with unchanged inputs leaves every byte identical (the existing
  `before == after` assertion covers the mechanism; extend the fixture so it covers the new sections too).
- Reconcile whatever drifted between the parallel phases: duplicated helpers, inconsistent cap wording, table header
  casing, and golden churn. Re-run `just fmt` so the goldens and any touched Markdown match prettier's
  `--prose-wrap=always --print-width=120`.
- Update `docs/agents_sidecar.md`: describe the page anatomy (breadcrumb, Summary, Files, Commits, Variables,
  Neighbors), state that commit links require a recognized GitHub remote for the project's primary repository and
  degrade to plain SHAs otherwise, state that neighbor rosters are lane-scoped and owner-scoped and mirror the Agents
  tab's NEIGHBORS grouping, and state that published output variables are the sanitized `sase var set` values and must
  not contain secrets. Add an explicit compatibility note that snapshots published by this version require every machine
  reading the sidecar to run at least this version.

## Risks and constraints

- **Cross-machine version skew (accepted, documented).** The v2 decoders reject unknown metadata keys. Once any machine
  publishes `output_variables`, a machine on an older `sase` fails its next sidecar publication or import with
  `AgentsSyncFormatError: ... metadata has unsupported fields: output_variables`. The failure is loud and is fixed by
  upgrading that machine. This is the deliberate cost of a strict allowlist; do not weaken the allowlist to work around
  it inside this epic.
- **Secrets in variables.** The `/sase_var` skill already warns against storing secrets because variables are shown in
  ACE and Telegram. Publishing them widens that exposure to anyone who can read the sidecar remote. The docs phase must
  say so plainly; no redaction heuristics — they would be unreliable and would silently corrupt legitimate values.
- **Determinism.** Every publication rebuilds the whole tree, and a competing owner's publication rebuilds it too. Any
  non-determinism (local time, dict ordering, config-dependent link rendering) shows up as sidecar churn and
  non-fast-forward retries. Sort every collection, format every timestamp from the integer epoch in UTC, and derive the
  commit URL from the project's own remote — which is identical for every machine cloning that project.
- **Golden conflicts across parallel phases.** `commits`, `vars`, and `neighbors` all rewrite
  `tests/agents_sync/goldens/*.md`. If two run concurrently, the second resolves the conflict by re-running the suite
  with `--sase-update-agents-goldens` from the `shell` phase and reviewing the regenerated diff — never by hand-merging
  golden text.
- **Boundaries to respect.** `sase.agents_sync` must not import `sase.ace`
  (`tests/agents_sync/test_import_boundaries.py`). All agent-name classification goes through
  `sase.core.agent_identity_facade`, never through local string splitting. `just check` is mandatory after every phase,
  and `just install` first because these workspaces are ephemeral.

## Out of scope

- Clan pages, and any clan-specific section: clan-mates already appear as `<clan> hood` neighbors.
- Cross-owner or cross-machine neighbor links.
- Changes to the hood, machine, user, or root index pages beyond what the breadcrumbs link to.
- Migrating ACE's `agent_lane_neighbors` projection, or moving either projection into `sase_core`.
- Publishing any new artifact kind (plans, beads, diffs) or any new file kind in the strict `_FILE_KINDS` allowlist.
