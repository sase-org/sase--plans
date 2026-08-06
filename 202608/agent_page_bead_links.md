---
tier: tale
title: Link published agent pages back to their bead pages
goal:
  Every published agent and family page in the agents sidecar links to the bead page in the beads sidecar for the bead
  that agent worked, closing the one-way bead-to-agent link that exists today.
proposed_by: bbugyi200.athena.u0
create_time: 2026-08-06 09:30:21
status: done
---

- **PROMPT:**
  [prompts/202608/agent_page_bead_links.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/agent_page_bead_links.md)

# Plan: Link published agent pages back to their bead pages

## Problem

Generated bead pages in the `--beads` sidecar already render an `## Agents` section whose rows are absolute GitHub URLs
into the `--agents` sidecar. The reverse link does not exist: an agent page in the `--agents` sidecar never mentions the
bead its run was launched against, so navigation is one-way. Someone reading an agent's page has to guess the bead from
the agent's name and hand-build the beads-sidecar URL.

This plan adds the return link. Scope is limited to agent runs that correspond to a bead — the same population that
writes the `SASE_BEAD=` commit-footer tag through `sase commit`.

## Where things stand today (context for the implementer)

Read these before editing. Everything below was verified against the current tree.

**Agent pages are rendered here.**

- `src/sase/agents_sync/rendering.py` —
  `render_browsing_payload(manifests, snapshots, *, commit_url_base, commit_repo_name)` renders every page in the
  sidecar. Its module docstring is _"Pure deterministic Markdown rendering"_: it performs no I/O and takes
  already-resolved values. Keep it that way.
- `src/sase/agents_sync/rendering_agent_page.py` — `render_agent_page(...)` writes `agents/<global_name>/README.md`.
- `src/sase/agents_sync/rendering_family_page.py` — `render_family_page(...)` writes `families/<global_name>.md`.
- `src/sase/agents_sync/publication_planning.py` — `plan_hoods(...)` is the impure planner that assembles the payload
  and is the _only_ caller of `render_browsing_payload`. It already resolves one hosted-URL value there
  (`_commit_url_base(inventory.primary_remote_url)`), which is the precedent this change follows.

**The bead association is already in the published wire.** No schema change, no bundle change, no migration.

`src/sase/agents_sync/v2_validation.py::V2_METADATA_FIELDS` already allowlists `bead_id`, `epic_bead_id`, and
`phase_bead_id`, and `src/sase/agents_sync/io.py::PORTABLE_META_FIELDS` already transports them. Published snapshots in
the real sidecar carry them today, e.g.

```json
{
  "local_name": "sase-ar.6",
  "metadata": { "bead_id": "sase-ar.6", "epic_bead_id": "sase-ar", "phase_bead_id": "sase-ar.6", "model": "opus" }
}
```

`bead_id` originates from the `%id(..., bead=<id>)` launch directive that `sase bead work` writes for both task workers
(`src/sase/bead/work.py`, the `%id(!{bead_id}, bead={bead_id})` line) and epic phase/land workers, and it is the same
value the commit workflow turns into the `SASE_BEAD=` footer tag. So `metadata["bead_id"]` is authoritative for exactly
the population this plan targets.

**Bead page addresses and URLs already have an authority.** Do not re-derive either.

- `src/sase/bead_pages/paths.py::bead_page_path(bead_id)` → `pages/<lineage-root>/README.md` for a lineage root,
  `pages/<lineage-root>/<bead-id>.md` for a descendant. It is lexical and offline-derivable by design, and raises
  `ValueError` for an id that cannot be a path.
- `src/sase/sdd/hosted_links.py::HostedLinkResolver.bead_url(bead_id)` → the absolute beads-sidecar blob URL, or `None`
  when the beads sidecar is missing, has no hosted remote, or has no resolvable branch. One resolver instance memoizes
  every remote/branch lookup, so build it **once** per publication run.
- `src/sase/bead_pages/links.py::known_bead_ids_for_store(store)` → every bead id in the store, or `None` when the store
  cannot be read.

**The reciprocal direction, for symmetry.** `src/sase/association_agents.py` plus `src/sase/bead_pages/associations/`
derive bead→agent rows, and `src/sase/bead_pages/rendering_tables.py::render_agents` renders them. Read `render_agents`
to match tone and link style.

## Design

### 1. Two sources for "which bead did this agent work", with different trust levels

A survey of the real published corpus (5,747 runs) is what drives this design:

| Source                              |  Runs | Bead exists in the store       |
| ----------------------------------- | ----: | ------------------------------ |
| `metadata["bead_id"]` present       |   826 | — (trusted, not checked)       |
| bead-shaped agent name, no metadata | 1,034 | 1,008 real, 26 false positives |
| neither                             | 3,887 | n/a                            |

Metadata alone would miss **56% of the bead-worked corpus**, because `bead_id` was added to the portable metadata
allowlist after most of that history was published. The name fallback is therefore required, not a nicety. But it is a
guess: the 26 false positives are agents like `sase_fix_just-g` and `gha-fix-sase-org-sase-28299141485-a1`, whose names
happen to satisfy the bead-name shape. Confirming a candidate against the bead store removes all 26 exactly.

So the two sources get different rules:

- **`metadata["bead_id"]` — authoritative, trusted lexically.** Always render the row. Link it whenever the beads
  sidecar resolves to a hosted URL. Do **not** gate it on the bead store. Rationale: it came from the launch directive,
  it is the same value that goes into the `SASE_BEAD=` footer, and a lexical link stays correct across machines and
  _self-heals_ — a bead page published later makes an already-written URL valid with no re-render, which is the exact
  property `bead_pages/paths.py` documents for the `SASE_BEAD` commit tag.
- **Name-derived fallback — a guess, must be confirmed.** Used only when `metadata["bead_id"]` is absent. Derive with
  `src/sase/agent/bead_display.py::derive_agent_bead_id_from_name(run.local_name)` — the same helper
  `bead_pages/associations/_artifacts.py` already uses for the bead→agent direction, so both directions agree by
  construction (it handles the `--<role>` family suffix, the `NNNNNN.` dismissed prefix, and `.land`). Render the row
  **only** when the derived id is present in `known_bead_ids_for_store(store)`. If the store cannot be read, that set is
  `None` and every name-derived row is dropped — metadata rows are unaffected.

`phase_bead_id` is deliberately unused: it equals `bead_id` for every phase worker, so rendering it would only repeat a
row.

### 2. Epic context

When `metadata["epic_bead_id"]` is present **and differs from** the resolved bead id, render a second `Epic:` row linked
the same lexical way. For a `.land` agent the two are equal, so only one row renders. Name-derived rows never carry an
epic row (there is no metadata to read).

### 3. Keep rendering pure; resolve in the planner

All policy — source selection, store confirmation, URL resolution — happens in the impure planner. The renderers receive
one already-decided mapping and stay pure and byte-deterministic for a given input.

New module `src/sase/agents_sync/bead_links.py`:

```python
@dataclass(frozen=True, slots=True)
class BeadPageLink:
    """One run's resolved bead association, ready for rendering."""

    bead_id: str
    url: str | None = None
    epic_bead_id: str | None = None
    epic_url: str | None = None


def build_bead_page_links(
    snapshots: Iterable[V2HoodSnapshot],
    *,
    primary_root: Path,
    project: str | None,
) -> tuple[Mapping[str, BeadPageLink], tuple[str, ...]]:
    """Resolve every published run's bead page link, keyed by global name."""
```

- Key the mapping by `run.global_name`; it is globally unique, while `source_run_id` is only hood-unique.
- Return `(links, diagnostics)`. Fold the diagnostics into `V2PublicationCounts.diagnostics` exactly the way
  `plan_hoods` already folds `inventory.diagnostics` and `load_validated_publication` diagnostics.
- Resolve the store the way `src/sase/bead_pages/links.py::_resolve_store` does
  (`workspace_context_for_plan_resolution(primary_root)` → `resolve_sdd_store(...)`), build **one**
  `hosted_link_resolver(store, project=project, primary_root=primary_root)`, and call `known_bead_ids_for_store(store)`
  **once**.
- Never raise. A missing SDD store, missing beads sidecar, unhosted remote, or unreadable bead store must degrade to an
  empty mapping (or unlinked rows) plus a diagnostic — publication must not fail because a bead link could not be
  resolved. Every existing helper on this path already follows that contract.

**Import-cycle landmine — read this before writing the module.** `sase/bead_pages/__init__.py` imports
`sase.bead_pages.rendering`, which imports `sase.agents_sync.rendering_markdown`, which initializes the whole
`sase.agents_sync` package. A module-level `from sase.bead_pages... import ...` inside `agents_sync` would therefore
create a partially-initialized-module cycle through `agents_sync/__init__.py` → `publication` → `publication_planning` →
the new module. Keep every `sase.bead_pages.*`, `sase.sdd.*`, and `sase.agent.bead_display` import **function-local** in
`bead_links.py`, matching the style `sase/bead_pages/links.py` already uses.
`tests/agents_sync/test_import_boundaries.py` is the guard; extend it (see Testing).

### 4. Threading

`plan_hoods` already has everything needed: `target.primary_checkout` for `primary_root` and `target.project` for
`project`, and it holds the validated `manifests, snapshots` before it calls the renderer.

```python
bead_links, bead_diagnostics = build_bead_page_links(
    snapshots.values(),
    primary_root=target.primary_checkout,
    project=target.project,
)
payload.update(
    render_browsing_payload(
        manifests,
        snapshots,
        commit_url_base=_commit_url_base(inventory.primary_remote_url),
        commit_repo_name=inventory.primary_repo_name,
        bead_links=bead_links,
    )
)
```

Add `bead_links: Mapping[str, BeadPageLink] | None = None` as a keyword-only parameter with a `None` default to
`render_browsing_payload`, `_render_run_pages`, `render_agent_page`, and `render_family_page`, so every existing call
site and test keeps working unchanged.

Build the links from the **validated** `snapshots` (which include imported foreign-owner hoods), not just this run's
`current_snapshots`. Foreign hoods belong to the same project and therefore the same beads sidecar, and their pages are
re-rendered on every publication run, so they get links too.

### 5. Rendered output

**Agent page** — two new bullets at the **top** of `## Summary`, above `Model`. The bead is the run's purpose; model and
provider are implementation detail, and putting the cross-repo link first makes it discoverable without scrolling.

```markdown
## Summary

- Bead: [sase-ar.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ar/sase-ar.6.md)
- Epic: [sase-ar](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ar/README.md)
- Model: opus
- Provider: claude
- Timing: 2026-07-29T15:00:38+00:00 → 2026-07-29T15:31:02+00:00
- Commits: [1](#commits)
```

Degrade rules, matching every other optional fact on these pages:

- No bead association → neither bullet renders, and the page is byte-identical to today.
- Bead known but no hosted beads sidecar → `- Bead: sase-ar.6` as plain escaped text, no link. This mirrors
  `hosted_links.py`'s stated contract of degrading "to an unlinked label instead of a broken URL", and the existing
  behavior where commit cells fall back to plain short SHAs.
- `epic_bead_id` absent or equal to the bead id → no `Epic:` bullet.

Escape link text with `md_escape` from `src/sase/agents_sync/rendering_markdown.py`. Emit the resolved URL as-is;
`github_blob_url` already encodes it.

**Family page** — append one fact to the existing header line, after `Members:`:

```markdown
Owner: `bbugyi200.athena` · Hood: `sase-al` · Members: 2 · Bead: [sase-al](…/pages/sase-al/README.md)
```

Take the distinct bead ids across the family's members, sorted. One distinct bead (the normal case, since a family is
one lane working one bead) renders `Bead: <link>`; several render `Beads: <link>, <link>` capped at 5 with a trailing
`… +N more`; none renders nothing and leaves the line exactly as today. No `Epic:` on the family page — the member pages
carry it and the header line should stay scannable.

Family pages matter here specifically because `HostedLinkResolver.agent_url` prefers `families/<name>.md` over a member
page, so bead→agent links frequently _land_ on the family page. Without this, half the inbound links would arrive at a
page with no way back.

### 6. Explicitly out of scope

State these in the change description so a reviewer knows they were considered:

- **Bead titles on agent pages.** Tempting and pretty, but bead titles are mutable: retitling one bead would rewrite
  every agent page that references it. The agents sidecar stays a deterministic projection of run metadata, so the link
  carries identity only.
- **Root/user/machine/hood index pages.** They are navigational rosters; every row already links to a page that carries
  the bead link one click away. Keep the tables narrow.
- **Prompt-archive documents** (`src/sase/agents_sync/prompt_archive/`), which already link a plan and an agent page and
  could later link a bead the same way. Follow-up, not this change.
- **`sase-core` / Rust.** This is sidecar page _rendering_, and the whole bead-pages and agents-sidecar rendering stack
  is Python today. Nothing here crosses the Rust core backend boundary.

Two things worth a sentence each in the change description because a reviewer will ask:

- **Hidden runs.** A run with `metadata["hidden"]` is excluded from the bead page's `## Agents` table but still gets an
  agent page. Its bead link renders. That is intended: the page already exists and the link exposes nothing the page
  does not.
- **Privacy.** Bead ids already appear in agent names, hood names, and commit subjects throughout the agents sidecar, so
  linking them adds no new disclosure. Both sidecars are private to the same project.

## Implementation steps

1. **`src/sase/agents_sync/bead_links.py` (new).** `BeadPageLink` and `build_bead_page_links` per §3, with the
   source-selection rules from §1–§2 factored into a small pure helper that takes
   `(local_name, metadata, known_bead_ids)` and returns a candidate id plus whether it came from metadata — that helper
   is what the unit tests drive. Keep the heavy imports function-local. Sort `__all__` (keep-sorted runs in
   `just check`).
2. **`src/sase/agents_sync/rendering.py`.** Add the `bead_links` keyword through `render_browsing_payload` and
   `_render_run_pages`; look each run up by `run.global_name`.
3. **`src/sase/agents_sync/rendering_agent_page.py`.** Prepend the `Bead:`/`Epic:` bullets to `summary` per §5.
4. **`src/sase/agents_sync/rendering_family_page.py`.** Extend the header line per §5, from the members' links.
5. **`src/sase/agents_sync/publication_planning.py`.** Build the mapping, pass it, and merge the diagnostics into
   `V2PublicationCounts.diagnostics` alongside the existing ones.
6. **Docs.**
   - `docs/agents_sidecar.md`, `## Browsing page anatomy`: extend the Summary bullet to name the bead and epic rows and
     their position; add a short paragraph next to the existing commit-URL paragraph stating when a bead link resolves,
     that it degrades to a plain bead id otherwise, that metadata ids are trusted lexically while name-derived ids are
     confirmed against the bead store, and — mirroring "The sidecar wire does not store commit URL bases" — that the
     wire stores bead ids but not bead page URLs; add the family-page header sentence.
   - `docs/beads.md`, the bead-pages section: one sentence noting that published agent pages link back, so the
     bead↔agent relationship is navigable in both directions.

## Testing

- **`tests/agents_sync/test_bead_links.py` (new).** Unit-test the builder against an injected fake resolver and an
  explicit `known_bead_ids` set: metadata source links; metadata source with an unresolvable URL yields an unlinked row;
  name-derived candidate confirmed in the store links; name-derived candidate _not_ in the store yields no row (cover
  `sase_fix_just-g` and `gha-fix-sase-org-sase-28299141485-a1` by name — they are the real false positives); an
  unreadable store (`known_bead_ids is None`) drops name-derived rows but keeps metadata rows; `epic_bead_id` equal to
  `bead_id` renders no epic; a bead id that `bead_page_path` rejects yields an unlinked row rather than raising; the
  `--<role>` family suffix, the `NNNNNN.` dismissed prefix, and `.land` all derive the expected id.
- **`tests/agents_sync/test_rendering.py`.** Drive `render_browsing_payload` directly with an explicit `bead_links`
  mapping: agent page shows `Bead:` then `Epic:` above `Model:`; the unlinked degrade renders plain escaped text; a run
  absent from the mapping renders exactly as before; family page header renders one bead, several beads, and the
  `… +N more` cap. Add one new golden under `tests/agents_sync/goldens/` (e.g. `bead-linked-agent.md`) rendered through
  this path so the finished shape is reviewable as a file; wire it to the existing `--sase-update-agents-goldens` flag
  the same way `tests/agents_sync/test_publication.py` does.
- **`tests/agents_sync/test_publication.py`.** The existing fixture publishes against a synthetic `ProjectTarget` with
  no SDD store, so it must keep producing byte-identical goldens — that is the regression guard that publication
  degrades cleanly with no beads sidecar. Do not modify the existing goldens.
- **`tests/agents_sync/test_import_boundaries.py`.** Add `sase.agents_sync.bead_links` to the fresh-interpreter import
  parametrization so a future module-level `sase.bead_pages` import fails loudly instead of producing a partially
  initialized module.

## Verification

Run `just install` first (workspaces are ephemeral and dependencies may have moved), then `just check`. This change
touches rendering, publication planning, and package import structure, so also run `just check-full` before landing.
