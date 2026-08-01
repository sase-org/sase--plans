---
tier: epic
title: Bead pages associate every repo's commits and always link their agents
goal: 'A published bead page lists every commit and agent actually associated with
  that bead lineage — no matter which of the project''s repositories the commit landed
  in — links each commit to its own owning repository, links every agent to its agents-sidecar
  page, and can never again be emptied by a commit made outside the primary repository.

  '
phases:
- id: anchor
  title: Sidecar-aware primary checkout and owning-project resolver
  depends_on: []
  size: small
  description: 'anchor: add a shared resolver that maps any path inside a managed
    checkout — including a sidecar or linked-repo clone nested under it — to that
    checkout''s primary repository root and canonical project name, with marker-first
    resolution and explicit fallbacks.

    '
- id: publish
  title: Anchor bead-page publication and refresh on the resolved primary repository
  depends_on:
  - anchor
  size: small
  description: 'publish: stop letting the committing repository masquerade as the
    primary repository by routing the post-commit bead-page publication and the pages-refresh
    CLI through the shared resolver, so a commit made in a sidecar can no longer erase
    a lineage''s associations.

    '
- id: agentlinks
  title: Resolve agent links from any repository in the workspace
  depends_on:
  - anchor
  size: medium
  description: 'agentlinks: make the hosted agents-sidecar remote lookup and the SASE_AGENT
    commit-footer tag resolve the owning project through the shared resolver instead
    of naming the current working directory''s repository, so agent labels stop rendering
    and committing unlinked.

    '
- id: multirepo
  title: Associate bead commits across every repository the project owns
  depends_on:
  - anchor
  - publish
  size: medium
  description: 'multirepo: walk the primary repository plus every locally cloned sidecar
    and linked repository when deriving bead associations, carry each commit''s owning
    repository through the association records, and resolve commit URLs against that
    repository''s own remote.

    '
- id: repair
  title: Regenerate degraded pages and verify the sase-b3 lineage end to end
  depends_on:
  - publish
  - agentlinks
  - multirepo
  size: small
  description: 'repair: republish every generated bead page from the corrected projection,
    confirm the sase-b3 lineage reports the associations its commits prove, and add
    a guard that fails when a published page links a commit to a sidecar remote.

    '
create_time: 2026-07-30 07:19:49
status: done
bead_id: sase-b5
---

- **PROMPT:** [prompts/202607/bead_page_association_anchors.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/bead_page_association_anchors.md)
- **BEAD:** [sase-b5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-b5/README.md)

# Plan: Bead pages associate every repo's commits and always link their agents

## Problem

The published page for the `sase-b3` lineage in the `beads` sidecar (`pages/sase-b3/README.md`) is almost entirely empty
of associations:

- Its **Phases** table reports `Agents 0 / Commits 0` for `sase-b3.1` through `sase-b3.8`, and `1 / 1` only for
  `sase-b3.9` — even though all nine phases are closed.
- Its **Commits** table lists exactly one commit, `6c21bbb`, whose link points at
  `https://github.com/sase-org/sase--plans/commit/…` — the **plans sidecar**, not the primary repository.
- Its **Agents** table lists exactly one agent, rendered as bare text (`bbugyi200.athena.sase-b3.9`) with no link to its
  page in the `agents` sidecar, while ~969 of the 994 pages that have an Agents table do render that link.

Three independent defects combine to produce this. All three are reproducible today.

## Root cause 1: a sidecar checkout is allowed to masquerade as the primary repository

`src/sase/workflows/commit/workflow_publication.py` passes `primary_root=cp.cwd` into `publish_committed_bead_pages`.
`cp.cwd` is whatever repository `sase commit` ran in. SASE materializes sidecars and linked repos _inside_ the workspace
checkout (`<checkout>/sase/repos/{plans,beads,research}` and `<checkout>/sase/repos/linked/<name>`), so an agent that
commits a plan file from the plans sidecar hands publication a root that is not the project's primary repository.
Nothing downstream re-derives the real primary checkout, so:

- `read_history_associations` (`src/sase/bead_pages/associations/_history.py`) runs its single `git log` in the
  **sidecar**. Only sidecar commits carrying a `SASE_BEAD` footer are discovered.
- Publication rewrites whole pages (`src/sase/bead_pages/publication.py`, `_write_changed_pages`), so every genuine
  association for the entire lineage is **overwritten with the sidecar's much smaller set**. Nothing restores them
  unless another primary-repo commit later lands for that lineage. `sase-b3` is closed, so its page stays wrong
  permanently.
- `HostedLinkResolver._resolve_primary_remote` (`src/sase/sdd/hosted_links.py`) reads `origin` from the sidecar, so the
  one surviving commit is linked to the sidecar's remote. **That wrong remote in the published link is the visible
  fingerprint of this bug.**

Evidence:

- `6c21bbb6 docs: add the missing prompt link to the fuzzy completion plan` exists only in the plans sidecar's history;
  it is absent from the primary repository entirely.
- Rebuilding the resolver with `primary_root` set to the plans sidecar reproduces
  `commit_url(...) -> https://github.com/sase-org/sase--plans/commit/…`, while the same resolver anchored on the
  workspace checkout returns `https://github.com/sase-org/sase/commit/…`.
- **21 of the 2360 published pages** currently contain a `sase--plans/commit/` link — every recent lineage from
  `sase-aj` through `sase-b3`. This is ongoing corruption, not a one-off.

## Root cause 2: the owning project is inferred from the current directory's repository

`HostedLinkResolver._resolve_agents_remote` falls back to `_current_project()` → `get_project_from_workspace()`
(`src/sase/workflows/utils.py`), which asks the VCS provider to name `os.getcwd()`. From the plans sidecar that yields
`sase--plans`; `resolve_sync_targets(("sase--plans",))` returns **0** targets, the `len(...) != 1` guard trips,
`agent_url()` returns `None`, and `render_agents` emits a bare label. Confirmed directly: with the cwd-derived name the
agent URL is `None`; passing `project="sase"` returns
`https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-b3.9/README.md`.

The same fallback in `resolve_agent_commit_tag` (`src/sase/agents_sync/links.py`) is why commits made outside the
primary repository also _write_ unlinked footers: the plans commit carries `SASE_AGENT=bbugyi200.athena.sase-b3.9` with
no link target, and every `sase-b3` commit in the `sase-core` linked repo likewise carries a bare `SASE_AGENT=`.
`SASE_BEAD=` stays correctly linked in those same commits because bead URLs do not depend on the primary root — which
isolates the fault to the project/primary-root inputs rather than to link rendering.

A resolver that already gets this right exists: `infer_project_name_from_cwd` (`src/sase/bead/project_name.py`) resolves
the nearest `.sase/checkout.json` marker first and returns the project key from inside any of those sidecar paths.
`resolve_sync_targets` accepts both the project key and the display name, so either spelling works.

## Root cause 3: associations are derived from one repository only

`read_history_associations` walks exactly one repository, so bead work that lands anywhere else is never associated.
This is why phases `sase-b3.1`–`sase-b3.5` would **still** report `0 / 0` even after root causes 1 and 2 are fixed: each
of those phases has exactly one commit, and all five are in the `sase-core` **linked** repository
(`git@github.com:sase-org/sase-core.git`), which matches their titles — "Canonical fuzzy matcher in sase-core",
"Server-side fuzzy completion for editors", and so on. The primary repository only carries `sase-b3.6`, `sase-b3.7`,
`sase-b3.8`, and two `sase-b3.9` commits.

`collect_repo_inventory` (`src/sase/repo_inventory.py`) already enumerates a project's primary, sidecar, linked, and
external repositories with per-workspace clone paths and remote URLs, and that module documents itself as the intended
adapter seam for exactly this kind of consumer.

## Verified target state for `sase-b3`

Once all three defects are fixed, the lineage's own commit footers prove the page must report:

| Bead                                  |                     Commits | Owning repository     |
| ------------------------------------- | --------------------------: | --------------------- |
| `sase-b3.1` … `sase-b3.5`             |                      1 each | `sase-core` (linked)  |
| `sase-b3.6`, `sase-b3.7`, `sase-b3.8` |                      1 each | `sase` (primary)      |
| `sase-b3.9`                           | 2 primary + 1 plans-sidecar | `sase`, `sase--plans` |

Every agent row must render as a link into the `agents` sidecar. Treat these counts as the acceptance target, but
re-derive them from the store and histories at implementation time rather than hard-coding them, since new commits may
land for this lineage in the meantime.

## Rust core boundary

No change to `sase-core` is required. The defect is in Python-owned orchestration: which repository path gets walked and
which project name gets resolved. Commit-footer parsing already goes through the Rust facade and is correct — it parsed
these footers fine. Linked-repository configuration and SDD store records are Python-owned by design, and
`src/sase/repo_inventory.py` is the declared seam for consumers of that inventory, so the multi-repo walk belongs here.
If the association projection is ever moved into `sase-core`, this plan's resolver and inventory calls are the migration
points.

## Phase: Sidecar-aware primary checkout and owning-project resolver

Add one shared, best-effort resolver that answers both questions the rest of this plan needs: given an arbitrary path,
what is the primary repository root of the checkout that owns it, and what is that checkout's canonical project name?

Place it where both `sase.bead_pages` and `sase.agents_sync` can import it without a cycle — a small module such as
`src/sase/sdd/checkout_anchor.py` is a reasonable home; use judgment if a better existing module presents itself.
Resolution order:

1. `find_marker_from_cwd` (`src/sase/workspace_provider/marker.py`) — the nearest ancestor `.sase/checkout.json`. This
   is authoritative and already handles every nested sidecar and linked-repo clone, because those clones live under the
   checkout and carry no marker of their own. Return the marker's checkout directory as the primary root and its
   `project_name` as the project.
2. Otherwise, if the path lies under a `sase/repos/...` sidecar or linked-repo directory, strip that suffix to recover
   the enclosing checkout root. This covers a marker-less primary checkout, where `find_marker_from_cwd` deliberately
   returns nothing.
3. Otherwise, return the path unchanged and no project name, matching today's behavior.

Canonicalize the project name the way existing callers do (see `_canonicalize_project_ref` and
`infer_project_name_from_cwd` in `src/sase/bead/project_name.py`) and only return a name that resolves to a real project
record. Prefer reusing `infer_project_name_from_cwd` for the project half rather than duplicating its
marker/provider/scan ladder; this phase's new work is the _primary root_ half plus a single call site that returns both
together.

The resolver must never raise — every consumer is on a best-effort, post-commit path — and must be purely local (no
network, no git subprocess beyond what the marker walk already does).

Cover with unit tests: a path that is a managed checkout root; a path inside each of that checkout's sidecar clones; a
path inside a nested linked-repo clone; a marker-less checkout root; a marker-less checkout's sidecar path; and an
unrelated directory. Assert both returned values.

## Phase: Anchor bead-page publication and refresh on the resolved primary repository

Route every bead-page entry point through the new resolver so the committing repository can no longer stand in for the
primary one.

- In `src/sase/workflows/commit/workflow_publication.py`, resolve `cp.cwd` to its primary root and project before
  calling `publish_committed_bead_pages`, and pass the resolved project explicitly instead of leaving it `None`.
- In `src/sase/bead_pages/publication.py`, resolve defensively as well rather than trusting the caller, since this
  function is a public best-effort boundary. Keep the store resolution working as it does today — it already lands on
  the right beads sidecar because it walks up to the workspace.
- In `src/sase/bead/cli_pages.py`, `_page_context` derives `primary_root` from
  `workspace_context_for_plan_resolution(Path.cwd())`, which returns the invocation path unchanged for a marker-less
  checkout. Use the shared resolver so `sase bead pages refresh` and `sase bead pages url` behave identically from a
  sidecar and from the primary checkout.

Also apply the resolver to `resolve_bead_commit_tag`'s `primary_root` in `src/sase/bead_pages/links.py`. Bead URLs
happen not to depend on it today, but leaving a known-wrong value threaded through a link resolver invites the next
regression.

Add regression tests that a bead-page publication driven from a sidecar path produces the same associations and the same
commit-URL remote as one driven from the primary checkout. The sharpest assertion is the one that would have caught
this: a published commit link must never point at a sidecar remote.

## Phase: Resolve agent links from any repository in the workspace

Make both agent-link resolvers identify the owning project from the checkout rather than from the current directory's
repository name.

- `HostedLinkResolver._resolve_agents_remote` (`src/sase/sdd/hosted_links.py`): replace the `_current_project()`
  fallback with the shared resolver applied to `self._primary_root`.
- `resolve_agent_commit_tag` (`src/sase/agents_sync/links.py`): replace its `get_project_from_workspace()` call the same
  way, so commits authored in sidecar and linked-repo clones write linked `SASE_AGENT=` footers.

While in `_resolve_agents_remote` and `resolve_agent_commit_tag`, re-examine the `target.primary_roots` containment
guard. Nested clones satisfy it today only incidentally, because `<checkout>/sase/repos/...` is a descendant of the
registered checkout root. Once the resolver hands these functions a true primary root, compare against that root rather
than against the raw cwd so the guard tests what it means to test.

Deliberately **out of scope**: changing `get_project_from_workspace` itself. It has ~38 call sites across commit, PR,
and accept workflows, and broadening it here would put unrelated flows at risk for no gain. Fix the two link resolvers
that demonstrably misbehave, and note the wider cleanup as possible follow-up work.

Tests: assert that `agent_url` resolves from a sidecar-anchored resolver; that `resolve_agent_commit_tag` returns a
linked value when invoked with a cwd inside a sidecar or linked-repo clone; and that a genuinely unresolvable project
still degrades to a bare label rather than an invented URL.

## Phase: Associate bead commits across every repository the project owns

Extend the association projection from one repository to every repository the project owns and has cloned locally.

Enumerate candidates with `collect_repo_inventory(project=...)`. For each record, choose the clone path via
`RepoRecord.clone_for_workspace(workspace_num)` and fall back to `RepoRecord.path`, and skip any that does not exist on
disk — a project can register far more repos than a given workspace has materialized. Include `primary`, `sidecar`, and
`linked` kinds; leave `external` out, since those are read-only research checkouts rather than places this project's
bead work lands.

Then:

- Give `read_history_associations` a repository identity per walk, and have the caller in
  `src/sase/bead_pages/associations/_build.py` fold the per-repository results into one index. `HistoricalBeadCommit`
  needs to carry its owning repository so the SHA is never ambiguous.
- Carry that identity through `BeadCommitAssociation` (`src/sase/bead_pages/associations/models.py`) and resolve each
  commit's URL against **its own** repository's remote. `HostedLinkResolver` currently exposes only `commit_url(sha)`
  against the primary remote; add a per-repository variant and keep its remote/provider lookups memoized the way the
  existing ones are.
- Keep the rendered label churn small. `render_commits` (`src/sase/bead_pages/rendering_tables.py`) should keep emitting
  a bare short SHA for primary-repo commits and qualify only non-primary ones (for example `sase-core@1290667`), so the
  ~970 pages whose commits all come from the primary repository stay byte-identical. Do not add an unconditional new
  table column; that would rewrite every page for no information gain on most of them.
- Keep sort keys total and deterministic. `(committed_at, sha)` is no longer unique across repositories in principle;
  include the repository name so page bytes stay stable.

Watch the cost. This turns one `git log` into one per existing clone on a post-commit path, and
`sase/memory/tui_perf.md` is required reading before changing anything that affects responsiveness. Walk only existing
clones, keep the inventory and remote lookups memoized per index build, and record a measured before/after for a full
`sase bead pages refresh` dry run. If the cost is material, surface it as a diagnostic rather than silently dropping
repositories — `BeadAssociationIndex` already carries `diagnostics` for exactly this.

Tests: multi-repository fixtures where the same bead is touched from a primary repo, a sidecar, and a linked repo;
assert per-repository commit URLs, deterministic ordering, that an unreadable or missing clone degrades to a diagnostic
instead of an exception, and that a single-repository project's rendered bytes are unchanged.

## Phase: Regenerate degraded pages and verify the sase-b3 lineage end to end

With the projection corrected, republish and verify.

1. Run `sase bead pages refresh` as a dry run first and review the action list — it should show the 21 pages that
   currently carry sidecar commit links, plus any page gaining newly discovered linked-repo associations. Confirm pages
   unaffected by all three defects are reported unchanged; that is the check that the label-churn constraint above
   actually held.
2. Run `sase bead pages refresh --write` to publish the repaired pages in one batched beads-store commit.
3. Verify `pages/sase-b3/README.md` against the "Verified target state" table above: every phase reports the commits its
   footers prove, each commit links to its own owning repository, and every agent row links into the `agents` sidecar.
4. Add a durable guard so this class of corruption fails loudly instead of silently. A test asserting that rendered
   commit links never resolve against an SDD sidecar remote is the minimum; consider a `sase doctor` check that scans
   published pages for sidecar commit links, since the symptom is cheap to detect and was invisible for 21 lineages.

Note in passing: `sase-core` also carries commits tagged `sase-b3.10` and `sase-b3.10.1`, bead IDs beyond the nine
phases visible on the current page. The `known_bead_ids` filter already drops tags for beads the store does not hold, so
this needs no special handling — but if those beads do exist, expect them on the regenerated page and confirm that is
correct rather than surprising.
