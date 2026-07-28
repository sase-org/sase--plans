---
tier: epic
title: Published bead pages and the SASE_BEAD commit tag
goal: 'Every bead that a SASE agent commits against has a beautiful, self-healing
  GitHub page in the project''s `--beads` sidecar that links out to its plan, its
  agents, its commits, and its related beads, and every commit reaches it through
  a `SASE_BEAD` footer tag instead of a parenthetical bolted onto the headline.

  '
phases:
- id: pathing
  title: Bead page address contract and hosted bead URLs
  depends_on: []
  size: small
  description: 'pathing: define the deterministic bead page path, carry the beads
    sidecar remote on the resolved store, and teach the shared hosted-link resolver
    to turn a bead id into an absolute GitHub URL.

    '
- id: tag
  title: SASE_BEAD commit tag replaces the headline parenthetical
  depends_on:
  - pathing
  size: medium
  description: 'tag: stop appending ` (<bead_id>)` to commit and PR headlines and
    write a linked `SASE_BEAD=` footer tag instead, resolved best-effort and local-only.

    '
- id: associations
  title: Derived bead association index
  depends_on:
  - pathing
  size: medium
  description: 'associations: derive each bead''s commits and agents in one repository
    walk from `SASE_BEAD` tags, the legacy headline parenthetical, `SASE_AGENT` tags,
    and bead-derived agent names, then roll descendants up into their root bead.

    '
- id: rendering
  title: Bead page rendering
  depends_on:
  - pathing
  - associations
  size: medium
  description: 'rendering: render one bead page per bead — identity, plan, description,
    lineage, phases, dependencies, agents, and commits — as deterministic, timestamp-free,
    bounded Markdown.

    '
- id: publish
  title: Post-commit lineage publication
  depends_on:
  - rendering
  size: medium
  description: 'publish: after a primary commit carrying a `SASE_BEAD` tag, re-render
    that bead''s whole lineage into the beads sidecar, write only what changed, commit,
    and push asynchronously, strictly best-effort.

    '
- id: conflicts
  title: Regenerable-page conflict class
  depends_on:
  - pathing
  size: small
  description: 'conflicts: teach the bead conflict resolver that generated pages are
    regenerable projections so a page conflict resolves automatically instead of wedging
    every bead rebase in the sidecar.

    '
- id: reconcile
  title: Bulk refresh command and lineage roster
  depends_on:
  - publish
  size: medium
  description: 'reconcile: add `sase bead pages refresh` and `sase bead pages url`,
    and let the refresh path own the `pages/README.md` roster that per-commit publication
    deliberately never touches.

    '
- id: planlink
  title: Reciprocal BEAD bullet in the plan header block
  depends_on:
  - pathing
  size: medium
  description: 'planlink: add a `BEAD` section to the sase-core plan-header block
    grammar and render it from each plan''s existing bead frontmatter so plans link
    back to bead pages.

    '
- id: docs
  title: Documentation and discoverability surfaces
  depends_on:
  - reconcile
  - tag
  size: small
  description: 'docs: update the generated beads sidecar README, the commit-workflow
    and bead documentation, and make `sase bead show` print the bead''s page URL.

    '
- id: rollout
  title: Publish every project's bead pages
  depends_on:
  - reconcile
  - conflicts
  - docs
  size: small
  description: 'rollout: run the bulk refresh against every enabled project, verify
    the published links resolve, and confirm a real `sase commit` produces a working
    `SASE_BEAD` link.

    '
create_time: 2026-07-28 14:20:20
status: done
bead_id: sase-ai
---

- **PROMPT:** [202607/prompts/bead_pages.md](prompts/bead_pages.md)
- **BEAD:** [sase-ai](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ai/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ai.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ai.1/README.md)
  - [bbugyi200.athena.sase-ai.10](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ai.10/README.md)
  - [bbugyi200.athena.sase-ai.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ai.2/README.md)
  - [bbugyi200.athena.sase-ai.3](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ai.3/README.md)
  - [bbugyi200.athena.sase-ai.4](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ai.4/README.md)
  - [bbugyi200.athena.sase-ai.5](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ai.5/README.md)
  - [bbugyi200.athena.sase-ai.6](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ai.6/README.md)
  - [bbugyi200.athena.sase-ai.7](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ai.7/README.md)
  - [bbugyi200.athena.sase-ai.8](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ai.8/README.md)
  - [bbugyi200.athena.sase-ai.9](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ai.9/README.md)
  - [bbugyi200.athena.sase-ai.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ai.land/README.md)

# Plan: Published bead pages and the `SASE_BEAD` commit tag

## Goal

Two recently landed epics set this up. `sase-ag` gave plan files a self-healing header block whose bullets link out to
prompts, parent plans, agents, and commits as real github.com hyperlinks. `sase-a8` moved bead state out of the plans
sidecar into a dedicated `<project>--beads` repository whose root already holds a generated README and an infographic.

What is missing is the bead half of the graph. A bead is the unit of work that agents are actually launched against, it
is what commits are attributed to, and it is what plans are approved into — and yet it has no page. A commit that
belongs to bead `sase-ag.4` says so only by carrying `(sase-ag.4)` at the end of its subject line, which is inert text
that leads nowhere.

This epic gives every bead a page in its project's beads sidecar and rewires commit attribution to link to it:

- `pages/<root>/README.md` and `pages/<root>/<id>.md` in the `--beads` repository, rendered from durable bead state and
  from associations derived out of the primary repository's history.
- `SASE_BEAD=[<bead-id>][n]` in the commit footer, resolving to that page, replacing the headline parenthetical.
- Outbound links from each page to the plans sidecar, the agents sidecar, the primary repository's commits, and the
  pages of parent, child, and dependency beads.

Like the plan header block, **a bead page is a projection of durable state, never an accumulator.** Nothing new is
persisted. Every page is fully re-derivable from the bead event store plus one `git log` walk, so a stale, duplicated,
or wrong entry disappears the moment the underlying source is corrected, every write point can be best-effort, and a
later reconcile repairs anything a write point dropped.

## Where the pages live

```text
<owner>/<repo>--beads/
├── README.md                    # generated sidecar guide (exists)
├── assets/…                     # generated infographic (exists)
├── config.json, metadata.json   # bead store (exists)
├── issues.jsonl, events/**      # bead store (exists)
└── pages/                       # NEW — generated bead pages
    ├── README.md                # lineage roster, owned by `sase bead pages refresh`
    ├── sase-ag/
    │   ├── README.md            # the root bead's page
    │   ├── sase-ag.1.md
    │   └── sase-ag.land.md
    └── sase-a8/…
```

The directory name is **`pages`, and it must not be `beads`.** `BEADS_DIRNAME_NON_VC` is the literal string `"beads"`,
and `resolve_beads_dir` in `src/sase/bead/conflict_resolver.py` locates the bead directory by collecting candidates from
`(root / BEADS_DIRNAME, root / BEADS_DIRNAME_NON_VC)` plus `root` itself when `root/config.json` exists, then bailing
out when the candidates are not unique. Creating `<beads-repo-root>/beads/` would make that set ambiguous and
`resolve_beads_dir` would return `None`, which turns every conflicted bead rebase in the sidecar into
`non-bead conflicts remain`. Phase `pathing` pins this with a test rather than a comment.

**Addressing is lexical and derivable offline.** A bead id's root is the segment before its first `.`, so `sase-ag.1`,
`sase-ag.land`, and `sase-26.1.1` live under `pages/sase-26/` and `pages/sase-ag/`. The root bead takes `README.md` so
GitHub renders it when browsing the directory; every descendant takes `<full-id>.md`. Nothing consults the bead store to
compute a page path, which is what lets the `SASE_BEAD` tag be written _before_ the page exists — exactly how
`agent_link_target` lets `SASE_AGENT` link to an agents-sidecar page that the same commit's post-commit step publishes.

Sharding by lexical root also keeps the tree browsable: the `sase` project has 2,215 beads under 331 roots, so `pages/`
lists 331 directories of roughly seven files each instead of one flat directory of 2,215.

## Rendered format

A root bead page, `pages/sase-ag/README.md`:

```markdown
# Bead: sase-ag — Plan-file provenance header block

[Bead Pages](../README.md) / sase-ag

**Status:** ✅ closed · **Resolution:** done · **Type:** plan · **Tier:** epic **Owner:** `bryanbugyi34@gmail.com` ·
**Assignee:** `sase-ag.land` **Created:** 2026-07-28 09:48:54 UTC · **Closed:** 2026-07-28 18:02:11 UTC **Plan:**
[202607/plan_header_provenance.md](https://github.com/sase-org/sase--plans/blob/main/202607/plan_header_provenance.md)

## Description

Every committed plan file opens with one beautiful, self-healing header block whose bullets link the plan to its prompt,
its parent plan, every agent that worked it, and every commit it produced.

## Phases

| Bead                      | Title                                                | Status    | Size   | Agents | Commits |
| ------------------------- | ---------------------------------------------------- | --------- | ------ | -----: | ------: |
| [sase-ag.1](sase-ag.1.md) | Rust-owned plan header block grammar                 | ✅ closed | medium |      2 |       4 |
| [sase-ag.2](sase-ag.2.md) | Hosted URL resolution for plans, agents, and commits | ✅ closed | small  |      1 |       2 |

## Dependencies

- **Blocks:** [sase-a8.6](../sase-a8/sase-a8.6.md) ✅

## Agents

| Agent                                                                                                                        | Bead                      | Commits |
| ---------------------------------------------------------------------------------------------------------------------------- | ------------------------- | ------: |
| [bbugyi200.athena.sase-ag.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ag.1/README.md) | [sase-ag.1](sase-ag.1.md) |       4 |

## Commits

| Commit                                                          | Subject                                   | Bead                      | Committed (UTC)     |
| --------------------------------------------------------------- | ----------------------------------------- | ------------------------- | ------------------- |
| [`9701511`](https://github.com/sase-org/sase/commit/97015111f…) | feat(sdd)!: write plan provenance headers | [sase-ag.4](sase-ag.4.md) | 2026-07-28 14:22:07 |
```

A descendant page is the same document without `Phases`, with a `Parent` breadcrumb linking `README.md` in its own
directory, and with `Agents`/`Commits` scoped to itself.

Rules that make this beautiful and stable, mirroring the conventions already used by the agents sidecar's pages:

- **No generation timestamps anywhere.** Every rendered instant comes from bead state or commit metadata. A page that
  embedded "generated at" could never be idempotent, and idempotence is what stops this feature from producing an
  endless stream of empty commits to the beads sidecar.
- Sections with nothing to show are omitted entirely; an empty `## Commits` heading is never rendered.
- Titles, subjects, agent names, and assignees go through the existing `md_cell` / `md_escape` / `md_code` helpers.
- Free-form bead prose (`description`, `notes`) is rendered as body Markdown, bounded in length, with any line that
  would open a fence or an ATX heading neutralized so authored text can never break the page's own structure.
- List sections are capped at the shared `MAX_RENDERED_COMMITS = 50` with a visible `… and N more` line. The cap is
  always visible in the document, never a silent truncation.
- Commit entries show the seven-character short SHA as link text and link to the full SHA.
- A root bead page rolls its descendants' agents and commits up into its own tables, attributing each row to the bead it
  actually belongs to; a descendant page lists only its own. This matches the epic roll-up the plan header block already
  does.
- Ordering is deterministic: phases and dependencies by bead id, agents by global name, commits by committed timestamp
  then SHA.

**Every link degrades rather than guessing.** Link generation is local-only and best-effort, exactly like
`format_sase_plan_link` and `resolve_agent_commit_tag`. A store with no hosted remote, no resolvable branch, or a
non-GitHub host without authoritative provider metadata renders an unlinked label. Sibling bead pages inside the same
repository always use relative hrefs, so they work even with no remote at all. Visible labels stay identical across
those fallbacks, so a project that gains a remote later produces a link-only diff.

## The `SASE_BEAD` commit tag

`enforce_bead_id_in_message` currently rewrites the first line of `payload["message"]` to `<subject> (<bead_id>)`, and
`CommitWorkflow.run` calls it once for every method plus a second idempotent time inside `handle_beads`. That is
replaced by one linked footer tag written through the existing tag machinery:

```text
feat(bead): publish bead pages from sase commit

SASE_BEAD=[sase-b3.5][1]
SASE_PLAN=[202607/bead_pages.md][2]
SASE_AGENT=[bbugyi200.athena.sase-b3.5][3]

[1]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-b3/sase-b3.5.md
[2]: https://github.com/sase-org/sase--plans/blob/main/202607/bead_pages.md
[3]: https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-b3.5/README.md
```

The tag value is the bare bead id, matching the way `SASE_AGENT` carries the globally unique agent name, so the same
string appears in the commit footer, in the page title, and in every table row that references the bead. Writing it
where `enforce_bead_id_in_message` runs today puts `BEAD` ahead of `PLAN` and `AGENT` in the footer, which reads as work
→ design → author.

Two accepted consequences of dropping the parenthetical, both direct results of the request:

- `git log --oneline` and release-please changelog entries no longer show the bead id in the subject. The footer tag
  carries it, and `sase bead pages` and the pages themselves carry the association.
- Historical commits keep their parentheticals, because commit messages are immutable. Phase `associations` reads them
  as a first-class legacy source, which is what backfills 2,035 existing `sase` commits onto bead pages on day one.

## Sources of truth

Nothing here introduces a new store.

- **Bead facts** — the bead store in the beads sidecar, read through `BeadProject` and the existing
  `resolve_issue_detail` relationship resolution in `src/sase/bead/cli_detail.py`, which already assembles ancestors,
  phases, child epics, `depends_on`, `blocks`, and the plan link.
- **Commits** — one `git log --format=%H%x00%ct%x00%s%x00%B%x00` walk of the primary repository, parsed with
  `sase.core.commit_footer_facade.parse_commit_footer`, indexed by the `SASE_BEAD` tag value when present. A commit with
  no `SASE_BEAD` tag falls back to a trailing `(<bead-id>)` in its subject, accepted **only** when the captured id is a
  bead that actually exists in the store, so an ordinary parenthetical is never mistaken for an association.
  `src/sase/sdd/associations/_history.py` is the existing precedent for this exact walk.
- **Agents** — the union of `SASE_AGENT` tag values on commits already indexed under a bead, and agents whose name
  derives to that bead through `sase.agent.bead_display.derive_agent_bead_id_from_name`. The second source is what makes
  phase agents and `.land` agents appear on a page even when they produced no commit of their own, and it needs no new
  metadata: `sase bead work` already names those agents after their bead.

Agent artifact metadata carries `phase_bead_id` and `epic_bead_id`, but the agent artifact index wire does not expose
them, so reaching them would mean a `sase-core` wire change for associations that the two sources above already cover.
That is explicitly out of scope.

## Reliability

**Publication never affects the primary commit.** It runs post-commit, in the same seam as
`refresh_committed_plan_header`, and every failure is logged and returned as an outcome, never raised across the commit
boundary. A missing beads sidecar, an unhosted remote, a busy write lock, or a wedged beads repository all degrade to
"nothing published"; `sase bead pages refresh` is the repair path.

**Idempotence is a hard requirement.** Rendering produces final bytes with no prettier pass — the agents sidecar
precedent, and the thing that keeps output byte-stable. Publication compares rendered bytes against what is on disk and
writes only files that actually changed, so a second run of the same publication commits nothing.

**Page conflicts must not wedge the bead store.** Today `_is_bead_path` classifies anything at the beads-repo root
outside `{events, issues.jsonl, metadata.json, config.json}` as a non-bead conflict, and any non-bead conflict makes
`resolve_bead_conflicts` refuse the whole resolution. Because phase agents in one clan work the same lineage in parallel
from different workspaces, page conflicts are not hypothetical, and under today's classifier the first one would block
every subsequent bead rebase in that sidecar. Phase `conflicts` fixes this before phase `rollout` puts real traffic on
it.

**Contention is bounded by design.** Per-commit publication touches exactly one lineage directory, so two agents working
different epics never touch the same file. The one genuinely shared file, the `pages/README.md` roster, is owned solely
by `sase bead pages refresh` and is deliberately never written by per-commit publication. The roster therefore lags
behind newly created root beads until the next refresh; the per-lineage pages, which are what every link actually points
at, are always current.

## Phases

### Bead page address contract and hosted bead URLs

Add `src/sase/bead_pages/paths.py` as the single authority for page addressing. `bead_pages` is a new top-level package
next to `agents_sync` rather than a subpackage of `src/sase/bead/`, which is already 12k lines; per the Rust core
boundary, page rendering and association derivation are the same class of local projection that `agents_sync/rendering*`
and `sase.sdd.associations` are, and stay in Python.

- `BEAD_PAGES_DIRNAME = "pages"`, `bead_page_root(bead_id)`, `bead_page_path(bead_id)` returning the repo-relative posix
  path, and a lexical `bead_lineage_root(bead_id)` helper. All are pure functions of the id.
- A test asserting `BEAD_PAGES_DIRNAME` is not `BEADS_DIRNAME`, not `BEADS_DIRNAME_NON_VC`, and that `resolve_beads_dir`
  still resolves a beads-sidecar root that contains a `pages/` directory. This is the guard for the ambiguity described
  above.
- A test asserting the lexical root of every id in a representative fixture matches the root reached by following
  `parent_id` in the store, so the addressing scheme and the data agree.

Carry the beads remote on the resolved store, mirroring `research_remote_url` exactly:

- `src/sase/sdd/_store_types.py`: add `beads_remote_url: str | None = None` to `SddStore`.
- `src/sase/sdd/_store_resolution.py`: populate it from `record.beads.remote_url` when the record has a beads sidecar.
- `src/sase/sdd/_commit_store.py`: the beads target built by `sdd_commit_targets` currently sets `remote_url=None`; give
  it `remote_url=store.beads_remote_url` so the beads-rooted target is as honest as the research one.

Extend the existing resolver rather than adding a second one. In `src/sase/sdd/hosted_links.py`, add a `_beads_remote`
coordinate resolved from `store.beads_remote_url` plus `resolve_hosted_branch(store.repo_root_for_kind("beads"))`, and a
`bead_url(bead_id)` method that composes it with `bead_page_path`. It memoizes like its siblings, so a tree-wide refresh
resolves the beads remote once. `HostedLinkResolver` then serves all four link kinds — plans, agents, commits, and beads
— from one cache.

Tests: a hosted store yields the expected absolute URL; a store with no beads sidecar, no remote, or no resolvable
branch yields `None` and never raises; the resolver shells out to git once across many `bead_url` calls.

### `SASE_BEAD` commit tag replaces the headline parenthetical

Add `resolve_bead_commit_tag(bead_id, *, store=None, cwd=None)` to `src/sase/bead_pages/links.py`, modeled directly on
`sase.agents_sync.links.resolve_agent_commit_tag`: it returns the bare id when anything is unresolvable and a
`LinkedCommitTagValue(bead_id, url)` when the beads sidecar is hosted. It must never raise and never touch the network.

In `src/sase/workflows/commit/commit_hooks.py`, replace `enforce_bead_id_in_message` with `apply_bead_commit_tag`, which
writes `{"BEAD": <value>}` through `update_trailing_commit_tags` with `remove_keys={"BEAD"}` so re-application is
idempotent and a resumed commit cannot double-tag. Delete `_message_line_has_bead_id` with it. In
`src/sase/workflows/commit/workflow.py`, call the new helper where `enforce_bead_id_in_message` is called today, and
drop the now-redundant second call inside `handle_beads`; `handle_beads` keeps closing and syncing the bead.

Keep the current method coverage exactly: the tag is applied for `create_commit`, `create_proposal`, and
`create_pull_request`, because the parenthetical is applied for all three today. Confirm the value survives into PR
bodies through `build_pr_body`, and that `filter_runtime_owned_tags` and the inherited-PR-tag paths do not strip it.

Tests: a linked tag when the sidecar is hosted and a plain tag when it is not; the subject line is left byte-identical;
re-applying the tag to an already-tagged message is a no-op; a message that already carries `PLAN`/`AGENT` tags keeps
them and gains `BEAD` first; a commit with no `bead_id` in the payload gains no tag; the checkpoint/resume path does not
re-tag.

Update `docs/commit_workflows.md` (the pipeline diagram, the "Bead association" description, and the payload table) and
the `SASE_BEAD_ID` row in `docs/configuration.md` in this phase, since they document the behavior being replaced.

### Derived bead association index

Add `src/sase/bead_pages/associations/` building, in one pass, a map from bead id to its commits and agents.

- Walk the primary repository once and index each commit under its `SASE_BEAD` tag value. Take the tag's label, not its
  rendered Markdown, because the value is a `LinkedCommitTagValue`.
- Legacy fallback: a commit with no `SASE_BEAD` tag whose subject ends in `(<id>)` is indexed under `<id>` **only** if
  `<id>` is a known bead. The known-bead set comes from the store, which the index already needs.
- Index the commit's `SASE_AGENT` tag value as an agent association for the same bead, and add agents whose name derives
  to a bead through `derive_agent_bead_id_from_name`. Skip agents whose artifact metadata marks them hidden, matching
  `sase.sdd.associations`.
- Roll every descendant's associations up into its root bead by walking `parent_id`, guarding against cycles rather than
  following them. Roll-up rows retain the bead they came from so the rendered table can attribute each one.
- Return frozen records carrying the display label, the resolved URL from `pathing`, and the sort key, so the renderer
  makes no further formatting decisions.

One `git log` walk and one bead-store read per invocation, with results reusable across every bead in the tree. Follow
`sase/memory/tui_perf.md` if any of this becomes reachable from ACE.

Tests: a `SASE_BEAD`-tagged commit; a legacy-parenthetical commit for a real bead; a parenthetical that is not a bead id
and must be ignored; an agent associated by name with no commits; epic roll-up; a parent cycle that terminates.

### Bead page rendering

Add `src/sase/bead_pages/rendering_*.py`, split by page element to stay under `_lint-toobig`, reusing
`sase.agents_sync.rendering_markdown` for escaping and URL encoding rather than adding a second set of helpers.

Render the sections described in **Rendered format**: header identity block, plan link, description and notes, parent
breadcrumb, phases table, dependencies, agents table, commits table, and — on root pages only — a bounded Mermaid graph
of the lineage with dependency edges, skipped entirely above a node cap. `md_mermaid` already exists for label escaping,
and GitHub renders Mermaid natively, which is what makes a root page read as a picture of the epic rather than a list.

Every bead-to-bead href is relative within the beads repository. Plans, agents, and commits use the resolver from
`pathing` and degrade to unlinked labels.

Tests: golden fixtures for a root page and a descendant page; re-rendering identical inputs produces byte-identical
output; a bead with a `|`, a backtick, and a `#` heading in its title and description renders a page whose structure is
unchanged; the `… and N more` cap; an unhosted project renders labels with no URLs; a bead with no agents, commits, or
dependencies renders none of those headings.

### Post-commit lineage publication

Add `src/sase/bead_pages/publication.py` with `publish_committed_bead_pages(commit_message, *, primary_root, project)`,
modeled on `sase.sdd.plan_header_refresh.refresh_committed_plan_header`:

1. Read the `SASE_BEAD` tag from the committed message; skip when absent.
2. Resolve the store; skip when the project has no recorded beads sidecar.
3. Take `store_git_write_lock(store.repo_root_for_kind("beads"), mutates_worktree=True)`; a busy lock is an outcome, not
   an exception.
4. Build the association index once, render every page in the bead's lexical lineage — the root and all descendants,
   because the root's roll-up changed too — and write only files whose bytes changed.
5. Commit through
   `commit_sdd_store_files(store, f"Publish bead pages for {root_id}", paths=…, auto_commit_type="beads", push_after_commit="async", already_locked=True)`.
   Page paths sit under the beads clone, so `sdd_commit_targets` routes them to the beads repository with no new
   plumbing.
6. Return a frozen outcome carrying `changed`, `committed`, `skip_reason`, and `error`. Wrap the whole body so nothing
   escapes.

Wire it into `CommitWorkflow._run_agent_publication_step` beside the existing `refresh_committed_plan_header` call, and
record it in `completed_steps` so a `sase commit --resume` does not republish needlessly.

Tests: a commit with a `SASE_BEAD` tag publishes the lineage; a second identical run commits nothing; a project with no
beads sidecar is skipped without error; a beads repository that fails its health check does not fail the primary commit;
a raised exception inside rendering is captured in the outcome; the workflow's return value is unchanged in every
failure mode.

### Regenerable-page conflict class

In `src/sase/bead/conflict_resolver.py`, add a third path class alongside the store entries and the mergeable store
files: a **regenerable** path, meaning a path whose first component is `BEAD_PAGES_DIRNAME` in the root-layout beads
sidecar (empty `bead_prefix`). Regenerable paths are bead paths, so they no longer trip the `non-bead conflicts remain`
bail-out, and they resolve by taking the upstream side — using the existing `_upstream_and_local_stages` so a rebase
does not silently invert "ours" and "theirs" — and staging it. Accept an upstream deletion as a deletion. The next
publication re-renders the correct content, which is the whole point of a projection.

Add a fast path: when every conflicted file is regenerable, resolve them and return without running the semantic event
merge at all, so a pages-only conflict never rewrites `issues.jsonl` or `events/manifest.json`.

The classifier must stay tight. A conflicted `README.md`, `assets/…`, or `.gitignore` at the beads root is still a
non-bead conflict that refuses auto-resolution, and the non-empty-prefix (plans-embedded) layout gains no regenerable
paths at all, because it has no pages.

Tests: a pages-only conflict resolves and stages the upstream content without touching the store; a mixed
store-plus-pages conflict resolves both; a conflicted root `README.md` still refuses; an upstream page deletion resolves
as a deletion; the legacy prefixed layout is unaffected.

### Bulk refresh command and lineage roster

Add a `sase bead pages` group beside the existing `sase bead` subcommands, following `sase/memory/cli_rules.md` —
alphabetically sorted subcommands and options, a short alias for every public long option, excellent `-h` output, and
colored output where it aids scanning. There is no `list` child, so a bare `sase bead pages` prints help.

- `sase bead pages refresh` — dry run by default with a summary of what would change, `-w/--write` to apply,
  `-b/--bead <id>` to scope to one lineage, `-j/--json` for machine-readable output. It rebuilds every lineage from the
  association index, rewrites only files whose bytes changed, and commits the batch through `commit_sdd_store_files` in
  one commit. Report removed pages for beads that no longer exist rather than leaving orphans.
- `sase bead pages url <bead-id>` — print the bead's page URL, or a clear message when the project has no hosted beads
  sidecar. Local-only.

`refresh` also owns `pages/README.md`: a roster of root beads with title, type, tier, status, phase count, agent count,
and commit count, each linking to its lineage. Per-commit publication must not write this file; add a test that pins
that, because it is what keeps parallel agents off a shared file.

Tests: dry run writes nothing and reports the same set the write run changes; a second write run is a no-op; `--bead`
scopes to one lineage; `--json` shape; an orphaned page is reported and removed; the roster is regenerated
deterministically.

### Reciprocal `BEAD` bullet in the plan header block

A plan file records its bead as `bead:` (tales, stamped at propose) or `bead_id:` (epics, stamped when the epic bead is
created from the approved plan). Both render on github.com as inert frontmatter text — the same problem the `parent:`
property had before `sase-ag` replaced it with a `PARENT` bullet. Now that bead ids address real pages, give them a
bullet too.

Work in `../sase-core` (open it with `/sase_repo`), in the plan header block grammar and its PyO3 surface:

- Add a `BEAD` variant to the header section-kind enum with no legacy YAML-property mapping, positioned in the fixed
  render order as `PROMPT`, `PARENT`, `BEAD`, `AGENTS`, `COMMITS`.
- Bump the block wire schema version and update `PLAN_HEADER_BLOCK_WIRE_SCHEMA_VERSION` in
  `src/sase/sdd/plan_header_block.py` to match. Follow the `sase-ag` precedent for landing this: the core change ships
  first, then this repository raises its published `sase-core-rs` floor in the same phase, exactly as
  `build(deps): require sase-core-rs 0.12.4 (sase-ag)` did.

On the Python side, render the section from the plan's own frontmatter, resolving `bead_id` first and falling back to
`bead`, with the URL from `pathing` and an unlinked label otherwise. Write it wherever `PROMPT` and `PARENT` are written
today in `src/sase/sdd/plan_header_writes.py`, and include it in `sase plan links refresh` so the existing bulk path
backfills committed plans.

**Keep the frontmatter properties.** Unlike `parent:`, `bead:` and `bead_id:` are read by real code —
`plan_propose_handler` stamps them and `epic_from_plan` enforces `bead_id` uniqueness — so this section is purely
additive and needs no migration.

Tests: a plan with `bead_id` renders a linked `BEAD` bullet; a plan with only `bead` renders the same bullet; a plan
with neither omits the section; an unhosted project renders an unlinked label; a second write is a no-op; a
prettier-wrapped `BEAD` bullet re-parses to identical logical content.

### Documentation and discoverability surfaces

- `src/sase/sdd/templates/sidecar-beads-README.md`: add `pages/` to the Directory Layout table, describe the page
  addressing rule and what a page links to, and add `sase bead pages refresh` and `sase bead pages url` to the Commands
  list. `sase repo init` regenerates the sidecar README from this template, so this is how the published guide gets the
  new section.
- `docs/beads.md`: document bead pages, the `SASE_BEAD` commit tag, and the refresh command.
- `docs/sdd_storage.md`: note that the beads sidecar now holds generated pages in addition to bead state.
- `sase bead show`: print the bead's page URL when one resolves, and include it in the `--json` output. This is the
  cheapest possible discovery path from a terminal to the page.
- Do not touch `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims. `sase/memory/glossary.md` describes the
  sidecar repositories and will want a line about bead pages, but memory edits require explicit user permission in the
  conversation that makes them. Report it as a follow-up instead.

### Publish every project's bead pages

Re-derive the project list with `sase project list -j` rather than trusting a planning-time snapshot; as of planning
there are three enabled projects, and only projects with a recorded beads sidecar can publish.

Per project, from its primary workspace:

1. `sase bead pages refresh` (dry run) and read the summary.
2. `sase bead pages refresh --write`, confirming one batch commit lands in the `--beads` sidecar and pushes.
3. Open the published `pages/README.md` and two lineages on github.com and confirm the plan, agent, commit, and
   sibling-bead links all resolve. A 404 here means a link policy bug, not a data bug.
4. Make one real `sase commit` against a bead and confirm the commit footer carries a `SASE_BEAD` tag whose URL
   resolves, that the subject carries no parenthetical, and that the lineage pages were republished.
5. Delete the beads clone, run `sase bead list` to confirm lazy materialization still works, then re-run
   `sase bead pages refresh` to confirm it is a no-op.

Order matters: migrate the low-traffic projects first as a rehearsal and the `sase` project last, since its agents run
bead commands continuously from ephemeral workspaces. Before the `sase` run, confirm the new build is installed
everywhere that touches the project.

## Constraints and verification

- Run `just install` before `just check` in this workspace; ephemeral workspace clones may have stale dependencies.
  `just check` is mandatory for every phase that touches files in this repository.
- Changes under `../sase-core` must be opened through `/sase_repo` and verified with that repository's own test command.
  Because `just install` builds `sase_core_rs` from the local checkout, the `planlink` phase can validate the full stack
  in one workspace before the published floor is raised.
- Respect `_lint-toobig` and `_lint-symvision`; the new package is large enough to warrant focused modules from the
  start. Read `sase/memory/symvision.md` before suppressing anything, and prefer `--epic-symbol` entries for symbols a
  later phase of this epic will consume.
- Read `sase/memory/cli_rules.md` before adding the `sase bead pages` group.
- The highest-value regression tests, which every reviewer should look for: an idempotence test proving a second
  publication writes nothing; a fallback test proving an unhosted project renders labels rather than broken URLs; a
  best-effort test proving a beads-sidecar failure leaves the primary commit's exit status untouched; a conflict test
  proving a pages-only conflict resolves; and the `resolve_beads_dir` guard proving the pages directory name can never
  shadow a bead store.
- Do not publish anything to a real `--beads` sidecar while implementing `pathing` through `planlink`. The first
  tree-wide publication belongs to `rollout`, where it is a single reviewable batch per project.
