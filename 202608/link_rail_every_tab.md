---
tier: epic
status: done
title: A link rail on every tab
goal: "Every ACE tab shows the selected entity's typed artifact links in one place, in
  one line, in the same place; `$` plus one key follows any of them across tabs and
  panes; the surface is invisible when the selection has no links; and the read model it
  draws from stops silently losing whole relation classes.

  "
phases:
  - id: converge
    title: One projection for the machine-local read model
    depends_on: []
    size: medium
    description: "converge: make every writer of `artifact-links.json` produce the same
      row set, so the read model stops flapping by 233 rows an hour and `cites`/`read`
      stop vanishing.

      "
  - id: freshness
    title: A stale clone may not prove deletion
    depends_on:
      - converge
    size: medium
    description: "freshness: stop treating a companion file's mere existence as proof
      that a row was deleted, so a behind-HEAD sidecar clone can no longer delete rows
      another workspace added.

      "
  - id: project
    title: Projected edges from facts SASE already owns
    depends_on:
      - converge
    size: large
    description: "project: add a recomputed projection layer that turns commit trailers,
      agent metadata, and chop launch identity into typed edges without growing the
      durable store.

      "
  - id: truthread
    title: A way to read durable truth and see the drift
    depends_on:
      - converge
      - project
    size: medium
    description: "truthread: give the CLI and doctor a durable-truth read path and a
      row-level drift report, and build the harness that tests any index against the
      store rather than against the aggregate.

      "
  - id: subject
    title: One selected-entity ref and one O(1) link index
    depends_on:
      - truthread
    size: medium
    description: "subject: resolve the selected entity to a canonical ref on all three
      tabs and build the app-owned, alias-keyed edge index the render path reads with
      one dict lookup.

      "
  - id: rail
    title: The Link Rail, read-only
    depends_on:
      - subject
    size: medium
    description: "rail: mount the one-line, app-owned rail above the footer on every
      tab, with the chip anatomy, stable ordering, degradation ladder, and the
      invisibility contract.

      "
  - id: follow
    title: The `$` grammar and a jump that always lands
    depends_on:
      - rail
    size: large
    description: "follow: bind `$` as a one-shot prefix for `$1`-`$9`/`$$`, gate it on
      availability, and make a cross-tab jump reveal, expand, and load whatever it takes
      to land on the target.

      "
  - id: trail
    title: Walking back across surfaces
    depends_on:
      - follow
    size: medium
    description: "trail: add the app-level link trail and breadcrumb so
      `Ctrl+O`/`Ctrl+Shift+O` walk link hops across tabs and restore what a forward hop
      changed.

      "
  - id: panel
    title: The `$0` Links panel
    depends_on:
      - follow
    size: medium
    description: "panel: build the overflow inspector that owns the long tail, the full
      `why`, provenance, the index-staleness notice, and the authoring verbs.

      "
  - id: land
    title: Retire the duplicates and land the rail
    depends_on:
      - trail
      - panel
    size: medium
    description: "land: move link relations out of the pane relation panel, delete the
      two bespoke `L` jumps the rail generalizes, remove the flag, and verify the whole
      surface.

      "
proposed_by: bbugyi200.athena.0eh
create_time: 2026-09-09 19:50:45
bead_id: sase-ug
---

- **PROMPT:**
  [prompts/202608/link_rail_every_tab.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/link_rail_every_tab.md)
- **BEAD:**
  [sase-ug](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ug/README.md)

# Plan: A link rail on every tab

## Problem

ACE has three top-level tabs and a typed artifact-link graph that only one of them can
see. `RelationPanel` renders link relations, but `_app_action_availability.py:282`
hard-gates `toggle_relation_panel` to the Artifacts tab, and the panel's host mixin
reads a pane API (`self.contract`, `self.relation_index()`,
`self.selected_entry_target()`) that the Agents tree and the AXE sidebar do not have. So
an agent row cannot show what it cited, a chop cannot show what it launched, and a bead
on the Beads pane shows its links in a rail whose link segment has no key at all —
`_rail_mode_key` returns `""` for `RelationRole.LINK` (`relation_panel.py:~470`), and
`build_relation_view` assigns `key=""` to every link row
(`artifact_relation_layout.py:262,268`).

The codebase has already hand-built two special-cased single-link jumps to work around
this: `beads_open_plan` and `plans_open_bead`, both bound to `L`
(`bindings.py:154,173`), both calling `keymap.first_link_target(...)`. Two bespoke
instances of "jump to the linked thing" is the strongest available evidence that the
general verb is missing.

Underneath the UI gap is a worse one. **The read model every ACE surface consumes is
currently missing 100% of two relation classes, and the loss recurs.**

## Diagnosis

Measured on this machine on 2026-08-26 against `sase` at `0f51e8b77` (master, clean).

### The read model has three writers with three different projections

`~/.sase/projects/<key>/artifact-links.json` is the machine-local aggregate that
`load_artifact_links_snapshot()` (`relations/artifact_links.py:48`) feeds to every
Artifacts pane, that the Agent pane's `linked:`/`relation:`/`artifact:` filters read
through `build_agent_catalog_link_facets`, and that `sase artifact link list` prints
when given no ref (`artifact_cli/link_ops.py:110-112`). Three code paths write it, and
they do not agree:

| Writer                                                         | Projection                                                                |        Rows, this store, this moment |
| -------------------------------------------------------------- | ------------------------------------------------------------------------- | -----------------------------------: |
| `upsert_row` → `rebuild_aggregate()`                           | this workspace's sidecar clones + bead events + carry-forward, unfiltered |                            **1,277** |
| hourly `artifact_link_backfill` chop → `reconcile_aggregate()` | every visible workspace's clones, filtered by `_row_is_publishable`       | **1,303** here, **1,047** as written |
| `_upsert_aggregate_row` / `_remove_aggregate_rows`             | single-row edits                                                          |                                  n/a |

There is no generation, no ordering, and no merge: the last writer replaces the whole
file. The file on disk right now holds 1,047 rows. `store.preview_aggregate()` on the
same store at the same moment returns 1,277, and `store.preview_reconciled_aggregate()`
returns 1,303 and keeps 224 agent-endpoint rows.

The 236 rows the unfiltered rebuild holds and the on-disk file does not are not a random
tail:

| Relation       | Rows in durable sidecars but absent from the aggregate |
| -------------- | -----------------------------------------------------: |
| `cites`        |                                                    142 |
| `read`         |                                                     91 |
| `derives-from` |                                                      2 |
| `related`      |                                                      1 |

**233 of those 236 rows have an `agent:` endpoint, and they are 100% of `cites` and 100%
of `read`.** The observable symptom: `sase agent search 'linked:true'` returns **5**
agents today. The Agents half of the graph is not degraded; it is absent.

The mechanism is `_row_is_publishable` (`_artifact_link_store_reconcile.py`), which
drops any row whose `agent:` endpoint does not resolve through `resolve_cli_reference`
to `exact`/`drifted`/`vcs_backed`. Whether it resolves depends on the
`ArtifactRefContext` the writer happens to have. From a numbered workspace the context
resolves and the rows survive — verified: 37 of the first 40 agent rows are publishable
here, and `_agent_ref_is_published('agent:094')` is `True`. From the housekeeping chop's
primary-checkout cwd, `_artifact_ref_context_for_store()` returns `None` and the code
falls back to `launch_artifact_ref_context(is_home_mode=False)`; under that context the
rows are not publishable and all 233 are dropped. So the row set of a machine-local
cache is a function of **which process last wrote it and from where**, and an unfiltered
`rebuild_aggregate` restores the rows on the next `link add` so the whole cycle repeats
every hour.

That is the real defect, and it is more specific than the ticket that tracks it.
`bead:sase-ua` describes it as "the aggregate is 231 rows stale". It is not staleness.
It is two projections of the same store contending for one file, one of which drops an
entire relation class by design.

**Publishability is a publication concern being applied to a local read model.** The
outbox (`artifact-link-outbox.jsonl`, 18 pending rows) already exists to hold rows back
until their agent endpoint publishes. The local cache should show everything locally
durable; the publication path is where holding back belongs.

### Staleness is indistinguishable from deletion

`preview_aggregate()` carries a prior row forward only when
`_authoritative_source_was_consulted(row)` is false. That predicate
(`_artifact_link_store_core.py:63`) is true when either:

- `_sidecar_truth_was_consulted` — the companion `links/<relpath>.json` **file exists**;
  or
- `_bead_endpoint_is_authoritative` — **either endpoint is a bead** and `beads_dir` is
  not `None`.

Neither checks whether this clone is current. A workspace whose plans clone is behind
HEAD still has the file, so it "proves" deletion of rows it has simply never seen. Beads
are 1,231 of the 1,868 endpoints in this graph, so the bead branch alone hands blanket
deletion authority over most of the graph to any clone with a beads directory. This is
the same hazard `sase-tw.1` set out to close ("a rebuild may delete only what it can
prove was deleted"); the guard it added proves _visibility_, not _freshness_.

### The CLI cannot show durable truth

`handle_link_list` reads `store.load_aggregate()` when given no reference. There is
therefore no user-facing way to observe the store, and no way to audit the read model
against it. This is worth stating plainly because the consolidated research report
measured "the store" with `sase artifact link list -j -l 0` and was in fact measuring
the aggregate; its 1,267-row figure was the aggregate at a fresh moment, not the store.
The per-ref path is different and does read truth: `load_artifact_rows(ref)` goes to
sidecar JSON and bead events.

### Some rows have no durable home at all

`NON_SIDECAR_KINDS = {"agent", "bead", "stitch"}`. A row whose endpoints are both in
that set and neither of which is a bead — `stitch:` → `agent:`, `agent:` → `agent:` —
writes no sidecar JSON and no bead event. `_is_aggregate_only` routes it to
`_upsert_aggregate_row`, so it lives only in one machine's cache, is never published,
and cannot be recovered if that file is lost. Any design that wants stitch-to-agent or
agent-to-agent edges must not store them.

### What is _not_ broken

The consolidated report's §4.2 says the store writes `agent:` refs in three incompatible
spellings and that "118 of 194 do not match any name returned by `sase agent search`",
making build-time alias normalization "mandatory". **That is wrong and this plan does
not act on it.** All 196 distinct agent refs in the durable sidecars resolve through the
catalog's existing alias index: `_agent_ref_candidate_index`
(`agents/catalog/_query.py:199`) already inserts bare, owner-qualified, and
machine-qualified spellings for all 12,930 catalog rows, and role suffixes such as
`--plan` are ordinary parts of real agent names (`bbugyi200.athena.000--plan` is a real
agent directory), not a spelling variant to normalize away. Verified: 196 refs in, 196
resolved, 0 missed. The report conflated `sase agent search`'s bounded default
presentation scope with the catalog's ability to resolve, and `linked:true` returning
few rows is the aggregate defect above, not a resolution defect.

The real render-path hazard is different and does exist: `_known_target_for_ref`
(`relations/artifact_links.py:201`) resolves a ref by **iterating every known target**,
an O(n) scan on a path that must be O(1).

### The graph, measured

Measured against `store.preview_aggregate()` — the _whole_ graph, including the rows the
on-disk file is missing: 1,277 rows, 1,868 linked nodes.

| Links on the selected entity | Nodes | Share | Cumulative |
| ---------------------------- | ----: | ----: | ---------: |
| 1                            | 1,522 | 81.5% |      81.5% |
| 2–3                          |   274 | 14.7% |      96.1% |
| 4–9                          |    68 |  3.6% |  **99.8%** |
| 10–26                        |     4 |  0.2% |       100% |

Median degree 1, p90 2, max 26. Cross-kind edges 802/1,277 = **63%**, so following a
link is normally a cross-pane and often a cross-tab move: the jump verb must belong to
the app, not to a pane. Endpoint kinds: bead 1,231, plan 744, research 337, agent 236,
file 6. Relations: `implements` 544, `related` 359, `cites` 142, `derives-from` 138,
`read` 94; there are **no live `supersedes` rows**, and the UI must still treat one as
high-priority when it appears. **Stitches, patches, and chops have zero rows** — they
will never show a rail until something writes or projects those edges, which is what
phase `project` does.

The `why` description is median ~120 characters and truncates at 240, which is why no
one-line surface can carry it verbatim on more than one chip.

Two numbers decide the interface. **81.5% of linked entities have exactly one link and
99.8% have nine or fewer**, so this is a one-chip problem with a tiny tail, not a
graph-browsing problem. And only a small minority of agents and beads have any link at
all, so **invisible-when-empty is the dominant case, not an edge case.**

### Facts SASE already owns but does not link

| Source                                                                               | Already durable |                   Candidate edges |
| ------------------------------------------------------------------------------------ | --------------- | --------------------------------: |
| `SASE_BEAD=` commit trailer                                                          | git history     |                 1,341 stitch→bead |
| `SASE_AGENT=` commit trailer                                                         | git history     |                4,112 stitch→agent |
| `metadata.bead_id` / `epic_bead_id` / `phase_bead_id` in published agent `meta.json` | agents sidecar  | 7,091 agent→bead over 2,212 beads |
| `.chop.<lumberjack>.` in the agent's own name                                        | agents sidecar  |                     98 chop→agent |

The whole graph is 1,277 rows today. These four sources are worth roughly 12,600 more
edges, and every one of them is machine-owned, immutable, and already published.

## Design

### The shape

One new app-owned widget, **`LinkRail`**: full-width, single-line, yielded by
`AppLayoutMixin.compose` between `Horizontal(id="main-container")` and
`yield KeybindingFooter(...)` (`_app_layout.py:102`). That seam is the only object that
is literally the same on all three tabs. `display: none` when the selection has no
links.

`$` is a **one-shot prefix, never a toggle**:

| Gesture                   | Effect                                            | Keys     |
| ------------------------- | ------------------------------------------------- | -------- |
| `$1` … `$9`               | Follow link _N_, switching tab and pane as needed | 2        |
| `$$`                      | Follow the lead chip                              | 2        |
| `$0`                      | Open the Links panel                              | 2        |
| `Ctrl+O` / `Ctrl+Shift+O` | Walk back / forward along the link trail          | existing |

Two keystrokes reach 99.8% of linked entities directly and 100% by way of `$0`. There is
no armed mode to remember because the chips are painted _before_ the prefix is pressed:
the rail is simultaneously the display, the legend, and the keymap.

`$` is free — a programmatic walk of every value under `ace.keymaps` in
`default_config.yml` finds `dollar_sign` used 0 times across 109 distinct key strings —
and it completes a coherent family: `<` ancestors, `>` children, `~` family, `$` links.

**Why not let a bare `$` jump when there is exactly one link**, which would save a
keystroke 81.5% of the time: `bindings.py:127-129` binds bare digits to Artifacts
sub-panes, so a `$` that sometimes fires immediately would make `$2` at a one-link bead
follow link 1 _and then_ switch to the Stitches pane — two surfaces from where the user
aimed, with no error and no undo hint. A uniform two-key grammar makes that unreachable.
`$$` recovers the ergonomics without the ambiguity, and prefix-doubling is already the
house idiom (`%%`, `!!`, `,,`, `zz`).

Because 81.5% of entities have exactly one link, **the rail renders that one chip's key
as `$$` rather than `$1`**, so the dominant case is taught by the surface rather than by
documentation. `$1` stays live; only the label differs.

### Anatomy

```
 LINKS 3 · $1 ← implemented-by ✎ artifact_link_durability — "lands the design" · $2 ↔ rel ◈ sase-u3 · $0 all
 └──┬──┘   └┬┘ └────┬───────┘ └┬┘ └──────┬─────────┘   └───────┬────────┘   └── sigil chips ──┘  └─┬─┘
 header    key  perspective   kind    short target label     elided why (lead only)            overflow
           chip     label    icon+
                             accent
```

- **Header** — `LINKS` plus count, in the _selected entity's_ accent. The count is
  omitted at n=1; the single chip already says it.
- **Key chip** — `$N`, or `$$` at n=1, in the `[key]` style the relation panel uses.
- **Direction glyph** — `→` this entity is the source, `←` this entity is the target,
  `↔` undirected. Always relative to the selected entity, using the
  perspective-corrected label the Rust binding
  `artifact_relation_label(relation, this_is_source)` already returns for the
  audited-read footer. Never display the bare stored slug.
- **Relation label** — full word on the lead chip, four-letter sigil on the rest
  (`impl`, `cite`, `read`, `rel`, `sup`, `deriv`).
- **Kind icon and accent** — from the existing `ARTIFACTS_ICONS` / `ARTIFACTS_ACCENTS`
  (`_artifact_tab_model.py:52,64`), painted in the **destination** pane's accent. The
  hue tells you where the key will take you, at zero cell cost.
- **Short target label** — kind prefix removed and shortened: `bead:sase-u3` →
  `sase-u3`; `stitch:sase-org/sase@f4b827af6` → `sase@f4b827a`.
- **Lead-chip `why`** — elided, dim, in typographic quotes. **On by default** (owner
  decision), and the first thing dropped under width pressure.
- **Overflow** — `$0 all` always terminates the rail; when chips do not fit it becomes
  `$0 +7 more`.

Rendered per tab at 120 cells:

```
Artifacts ▸ Beads, sase-tw selected (3 links)
 LINKS 3 · $1 ← implemented-by ✎ 202608/artifact_link_durability — "lands the approved design" · $2 ↔ rel ◈ sase…

Agents, live agent sase-tj.land.w3 selected (1 link)
 LINKS · $$ → cites ✎ 202608/artifact_link_derivation.md — "expanded into the launch prompt at one hop"

AXE ▸ hooks / epic_launch_flush selected (2 launched agents)
 LINKS 2 · $1 → launched ⬡ sase-u6.1.code — "chop launch 2026-08-26T14:02Z" · $2 → launched ⬡ sase-u6.2.code

Anything with no links
 (no row: the rail is display:none and $ is not bound)
```

### Ordering and truncation must be provably stable

A rail becomes untrustworthy the moment a chip's key moves when the terminal is resized.

- Chips lay out in a fixed order: semantic relations first (`supersedes`, `implements`,
  `derives-from`, `related`), then observational (`cites`, `read`), then projected;
  within each group by perspective label, then neighbor ref. That is the sort
  `neighborhood_footer` already uses (`sdd/artifact_link_neighborhood.py`), so the rail,
  the audited-read footer, and the `$0` panel agree.
- A superseding replacement pins first and colours amber; it changes how the current
  artifact should be read. There are no live `supersedes` rows today, and the ordering
  must still handle one.
- **`$N` is assigned from that order and never from what fits.** A chip that does not
  fit is absorbed into `$0 +k more` and is _not_ renumbered. `$4` works when chip 4 is
  off-rail, because the key map is complete even when the render is not.
- Degradation ladder under width pressure: drop the lead chip's `why` → abbreviate the
  lead label to a sigil → drop trailing chips into `+k more` → collapse the breadcrumb →
  drop the header count.
- Duplicate observational endpoints render once with `N uses`.
- **Same-rule, same-kind projected edges collapse into one counted chip.** A bead with
  14 stitches gets `$3 → 14 stitches`, not fourteen chips. Pressing a counted chip's key
  opens the `$0` panel scoped to that group instead of jumping, which is the only place
  fourteen destinations can be chosen between anyway. This is what keeps the rail at
  nine or fewer chips regardless of how many edges the projection adds, and it is better
  UI independently of that: nobody wants nine near-identical stitch chips competing for
  the same glance.

### The invisibility contract

Three mechanisms, one predicate — `link_edges_for_selection() != ()`:

1. **Rail** — `display = False`, zero height, out of the layout. Not an empty bordered
   box, not a `0 links` badge, no placeholder, no toast.
2. **Keymap** — `check_app_action(app, "follow_artifact_link", …)` returns `False`, the
   same mechanism that already hides `toggle_relation_panel` off Artifacts. An
   unavailable action produces no binding, so `$` is inert and passes through.
3. **Footer, help, and command palette** — all driven by action availability, so no `$`
   chip, no help row, no palette entry, and `$0` is unreachable.

An entity ACE cannot resolve to a ref — a synthetic clan banner, a workflow grouping
row, an AXE lumberjack, a background command — is presented **identically to a linkable
entity with zero edges: nothing.** Never fabricate a ref from a row label to make the
rail appear.

**Dangling links still count as links.** The rail shows, the chip renders dim with `⊘`
and `(missing)` — the vocabulary `relation_panel.py:236` already uses — and `$N` on it
emits a toast instead of navigating. Hiding a dangling link would under-report the
graph, which is worse than an honest dead end.

### Traversal

Following a link makes the destination the selected entity, whose own rail paints
immediately, so `$1 $1 $1` walks three hops in six keys with no mode.

Going back needs new plumbing, because every existing back-stack is per-surface while
link hops cross surfaces: `_artifacts_jump_history` is
`dict[ArtifactsPaneKey, ArtifactEntryTarget]` (`actions/artifacts_navigation.py:36`) —
one slot per pane — and `_entry_jump_agents_anchor_stack` is a separate per-tab list. A
cross-tab link walk has nowhere to record itself.

- An app-level `_link_trail`, bounded at 32, mirroring `memory_panel_travel.py`'s
  `_trail` and `_MAX_TRAIL_LENGTH`. A hop records
  `(tab, ArtifactEntryTarget, pane query digest, fold state)` — enough to restore what a
  forward hop widened or expanded.
- `Ctrl+O` pops the link trail when the most recent navigation was a link hop and
  otherwise falls through to the existing per-surface anchor stacks unchanged.
  `Ctrl+Shift+O` walks forward. **No new back key.**
- Failed or dangling jumps do not mutate history.
- With a non-empty trail the rail grows a leading breadcrumb chip, which is both the
  affordance and the legend; entries beyond the last two collapse to `⟨ …3 › ✎plan ⟩`.
  The trail clears when the user navigates by any other means, so it never lies about
  how you arrived.

### A `$N` jump always lands

The highest-value reliability requirement, because 63% of jumps cross panes and every
pane has a filter query. `bead:sase-tw` is closed; the Beads pane default query is
`-status:closed`; today `_navigate_to_relation_target` handles a cross-pane target by
calling `_request_artifacts_entry(target)` and **returning immediately** — the reveal
path below it is never reached — so the jump silently lands nowhere. Order of attempts:

1. Switch tab if needed, then `_request_artifacts_entry(target)`, which already handles
   sub-pane switching and deferred selection.
2. If the target is not in the pane's results, widen the query minimally through the
   existing `reveal_entry_target` / `_change_query_for_navigation` path, which must be
   reached on the cross-pane path too.
3. If the target is folded, expand only the ancestors needed. If it is outside a bounded
   head slice, request it by stable identity from a worker. If it is in another project
   scope, change scope under the same reversible treatment. Never report "missing"
   before resolving the target's owning project.
4. Toast exactly what changed — `query widened: -status:closed removed` — and record the
   pre-jump digest so `Ctrl+O` restores it.
5. Only then report a dead end, naming the ref it could not resolve.

**Destination policy** is resolved in the snapshot so it cannot change between paint and
jump: `bead:`/`patch:`/`stitch:`/`file:` and provider documents route to their Artifacts
pane; `chop:` routes to AXE. `agent:` is the one intentional dynamic route — a loaded
live agent goes to the Agents tab, anything else to Artifacts ▸ Agent. If the target is
already represented in the current pane, prefer same-pane selection over a gratuitous
tab switch.

### The `$0` Links panel

A `ModalScreen` shaped like `AgentNeighborModal`, with row keys `a`–`z` so `$0` then `k`
reaches the 26th link in three keystrokes. It owns the four jobs the rail cannot do:

1. **The tail** — the entities with 10+ links.
2. **The `why`** — median 120 characters, each on its own indented line rather than
   wrapped into a 12-cell column the way `sase artifact link list` does today.
3. **Provenance** — `origin` (`derived`, `migrated`, `manual`, `read`, `prompt_ref`,
   `projected`), `uses`, `created_at`, `created_by`, and the projection rule for a
   projected row. Being able to see that a link was inferred rather than asserted is
   what makes the graph trustworthy. This is also where the index-staleness notice
   belongs.
4. **Authoring** — `add`/`rm`, folding in the existing `ArtifactLinkModal` and
   `artifacts_link_marked`, offered only for store-backed rows.

No query language in v1: degree is tiny and direct keys beat a filter input. No
transitive expansion: this is a one-hop neighbourhood inspector, and every jump exposes
the next hop.

### Projected edges, and why not a backfill

The owner asked that stitches link to their beads, that old stitches be backfilled, and
that "in general, we should try to backfill links for old artifacts". The four sources
in the diagnosis table are worth ~12,600 edges. Writing them durably would grow the
store twelve-fold, would need a `why` per row, would take a resumable multi-hour job,
and would put ~4,200 of them (`stitch:`→`agent:`) into the aggregate-only class that has
no durable home and cannot publish.

So: **project them instead of storing them.** A projection rule reads a fact SASE
already owns and emits typed edges at index-build time, marked `origin: projected`:

| Rule           | Source fact                                                                                      | Edge                                               |
| -------------- | ------------------------------------------------------------------------------------------------ | -------------------------------------------------- |
| `stitch-bead`  | `SASE_BEAD=` commit trailer                                                                      | `stitch:<repo>@<sha>` `implements` `bead:<id>`     |
| `stitch-agent` | `SASE_AGENT=` commit trailer                                                                     | `stitch:<repo>@<sha>` `produced-by` `agent:<name>` |
| `agent-bead`   | published `meta.json` `metadata.bead_id` / `epic_bead_id` / `phase_bead_id`                      | `agent:<name>` `implements` `bead:<id>`            |
| `chop-agent`   | `metadata.chop_name` / `chop_lumberjack`, falling back to the `.chop.<lumberjack>.` name segment | `chop:<lj>/<chop>` `launched` `agent:<name>`       |

This is a _better_ backfill than a backfill: a projection is recomputed from immutable
facts on every rebuild, so it is complete the moment it ships, can never drift, can
never be lost, and needs no migration. It also makes the aggregate-only hazard moot,
because projected rows are never stored. Projected rows are materialized into the
machine-local read model, so `sase artifact link list` and the Agent pane's
`linked:`/`relation:`/ `artifact:` filters see them exactly as the rail does.

The honest cost, which the `$0` panel states rather than hides: a projected row is not
published to other machines — each machine recomputes it — and it cannot be removed with
`link rm`, only invalidated by its source fact changing. Every chip therefore carries a
writability flag, and only store-backed chips are offered `add`/`rm`.

`chop:` stays a **virtual** subject kind: the AXE rail resolves it, the index keys it in
both directions, and no new entry joins the closed ref-kind catalog. Promote it to a
real kind only when chops must be link _targets of user-authored edges_. The rail does
not change when that happens, which is the point of doing it in this order.

### Performance

The rail paints on every highlight move, undebounced, per `tui_perf` rule 7 ("debounce
detail panels, never the highlight"). Budget: p95 < 16 ms key-to-paint.

- **Index.** One app-level `dict[str, tuple[LinkChip, ...]]` built off-thread from the
  already-cached `ArtifactLinksSnapshot` plus the projection rules, keyed by every alias
  spelling using the same build-time pattern `_agent_ref_candidate_index` already proves
  out. Rebuild is gated by the existing mtime+size signature (`_aggregate_signature`),
  satisfying `tui_perf` rule 8 by construction.
- **Render path.** One `dict.get` plus at most nine chip renders into a `rich.Text`,
  `no_wrap`, `overflow="ellipsis"`. Never call `_known_target_for_ref`'s O(n) scan from
  a render path.
- **Generation guards.** `tui_perf` rule 4: re-capture tab, entity, and generation after
  every await before applying. The rail clears _immediately_ on selection change; a
  stale summary from the prior row is worse than a brief absence.
- **Startup.** Rule 9: the rail must not block first paint. Before the index exists it
  renders nothing; the first successful build triggers one coalesced refresh.
- **Instrumentation.** `tui_trace("widget.link_rail.update")` as `relation_panel.py:99`
  does, plus the rail in `bench_tui_jk.py`.

The genuine risk is not the lookup — it is the temptation to resolve a ref to a _label_
lazily on the render path by reading a bead title or statting a plan file. **Do not.**
Labels derive purely from the ref string; enrichment comes from the pane's
already-loaded `relation_entry_facts()` when available and is simply absent otherwise.

### Rejected alternatives

- **Extend `RelationPanel` to every tab.** It cannot be app-level:
  `RelationPanelHostMixin` reads a pane API the Agents tree and AXE sidebar do not have,
  so making it app-level means doing the rail's work anyway while keeping the pane
  widget. It also fuses all four relation axes, so `$` could never be gated
  independently of `<`/`>`/`~`.
- **Modal only.** Cannot satisfy the display half of the brief; a modal shows nothing
  until opened, and adding a "you have links" indicator to make it discoverable means
  building the rail anyway.
- **Detail-panel chips.** In a different place on every tab, which the brief forbids.
  Worse, detail panels scroll (chips can scroll off while their keys stay live — exactly
  the "armed mode you cannot see" failure) and are debounced at 150 ms by
  `DetailPanelDebouncer`, so chips would lag the cursor and show the previous entity's
  links. That is a correctness bug, not a performance one.
- **Per-row badges.** Deferred at the owner's direction. They answer "which rows are
  worth pressing `$` on", which nothing else does, and they compose on top of the rail
  rather than competing with it.

## Implementation

### Phase `converge` — One projection for the machine-local read model

1. Extract a single `project_aggregate_rows(...)` used by `preview_aggregate`,
   `preview_reconciled_aggregate`, and every incremental writer, so the three paths
   differ only in _which stores they scan_, never in _which rows they keep_.
2. Remove `_row_is_publishable` from the local read-model path. Publication holdback
   stays the outbox's job; `durable_sidecar_rows()` keeps the filter because it feeds
   publication, and its docstring must say so.
3. Make writes to `artifact-links.json` merge rather than replace: carry a monotonic
   generation and reject a write derived from an older snapshot, so a chop and a
   workspace cannot clobber each other.
4. Add a convergence test that seeds a store with a `cites` row and a `read` row, runs
   every writer in every order, and asserts an identical row set each time. This is the
   test that would have caught the current defect.
5. Re-measure and record the before/after: `sase agent search 'linked:true'` must go
   from 5 to the full agent-endpoint population.

Record the corrected diagnosis as a note on `bead:sase-ua` rather than rewriting it: it
is not "231 rows stale", it is competing projections, and rescoping or closing it is the
owner's call. `bead:sase-t0` (uncommitted `read` rows) and `bead:sase-u9` (rename-repair
budget) stay independent and are not gated by this phase.

### Phase `freshness` — A stale clone may not prove deletion

1. Give `_authoritative_source_was_consulted` freshness evidence: a clone proves
   deletion for a row only when the clone's own commit time for that companion file (or
   the beads store's) is at least as new as the row's `created_at`. Otherwise the row is
   carried forward.
2. Apply it to both branches. `_bead_endpoint_is_authoritative` is the more dangerous
   one: it currently grants blanket authority over 1,231 of 1,868 endpoints on the
   strength of `beads_dir is not None`.
3. Regression test: two workspace stores over one project, one deliberately behind,
   prove that the behind clone's rebuild no longer deletes the ahead clone's rows. This
   reproduces the live failure and must fail before the fix.
4. Verify no user-visible slowdown: freshness evidence is per-companion-file, cached per
   pass, and must not add a git subprocess per row (`bead:sase-u9`'s hazard).

### Phase `project` — Projected edges from facts SASE already owns

1. Add a projection module with one rule per row of the design table, each pure, each
   taking already-loaded inputs, each emitting rows tagged `origin: projected` plus the
   rule id.
2. Register `produced-by` and `launched` in the relation registry in `../sase-core` with
   their inverses, since the registry is closed and lives in the Rust core; add the
   binding and Python-side declarations. Follow the Rust-core boundary rule: wire and
   tests in `sase-core`, callers here.
3. Materialize projected rows into the machine-local read model through the single
   projection point built in `converge`, never into sidecar JSON or bead events.
4. Cache trailer extraction by commit sha and agent metadata by file mtime; the
   projection must not walk 13,412 commits or 10,011 agent directories on any
   interactive path.
5. Coverage tests per rule with a fixture repo and fixture agents sidecar, plus an
   idempotence test (two projections of the same facts produce byte-identical rows) and
   a volume smoke test asserting the projection completes inside the chop's budget.

### Phase `truthread` — A way to read durable truth and see the drift

1. `sase artifact link list --source store|index` (default `index`, preserving today's
   behaviour). `store` assembles from sidecar JSON and bead events, so a user can
   finally see truth.
2. Make `inspect_artifact_link_health` (`artifact_cli/link_health.py:99`, reached
   through `sase artifact doctor`) report the row-level diff — counts by relation, by
   origin, by endpoint kind, and the first N differing rows — instead of a boolean, and
   surface it in the command's output. Upgrade the `project.artifact_links_aggregate`
   doctor check to summarize the same diff in its `next_steps`, so "artifact-links
   aggregate is missing or stale versus sidecar links/" becomes "missing 142 cites, 91
   read".
3. Build the shared test harness that loads durable rows and asserts a given index
   resolves every one of them. **Tests assert against the store, never against the
   aggregate**; a repeat of the current defect must fail the suite.
4. Extend `_check_artifact_links_aggregate` coverage to the projected layer, so a broken
   projection rule is reported rather than silently emitting nothing.

### Phase `subject` — One selected-entity ref and one O(1) link index

1. Add `LinkSubject` (canonical ref, optional `ArtifactEntryTarget`, accent, icon) and
   an app-level `selected_link_subject()` resolved by three thin adapters:
   - **Artifacts** — from `pane.selected_entry_target()`, inverting `_target_for_ref`.
   - **Agents** — from `_get_selected_agent()` via `reference_for_agent_name`.
   - **AXE** — from `selected_axe_item_key()`; `("chop", lj, name)` becomes the virtual
     `chop:<lj>/<name>`; lumberjack and bgcmd rows return `None`. Resolution is
     synchronous, allocation-light, and does no I/O. Anything unresolvable returns
     `None`.
2. Build `LinkIndex` off-thread from the snapshot plus the projection layer: a
   `dict[str, tuple[LinkChip, ...]]` keyed by every alias spelling, both directions,
   including the virtual `chop:` keys. Reuse `_agent_ref_candidate_index`'s build-time
   pattern; add stitch short/full sha and `plan:`/`ref:plan` variants.
3. Add `link_edges_for_selection()` on the app, returning the ordered chip tuple for the
   current subject, and the `follow_artifact_link` availability predicate.
4. Delete no UI yet. Test headlessly on all three tabs against the store harness from
   `truthread`, including every ref kind in **both** endpoint positions, directed
   primary and inverse labels, symmetric `related`, and duplicate observational rows
   converging to `uses`.

### Phase `rail` — The Link Rail, read-only

1. Create the `beta` feature flag with `sase flag new` covering the rail, the keys, and
   the panel, so intermediate phases can land without exposing an unfinished surface.
   `land` removes it.
2. Implement `LinkRail` and yield it at the `_app_layout.py:102` seam. Rendering per the
   anatomy above, ordering and the degradation ladder per the design, `$$` for the n=1
   lead chip.
3. Implement the invisibility contract's rail mechanism and wire the selection watchers
   on all three tabs, honouring `tui_perf` rules 4, 7, 8, and 9. Implement counted-chip
   collapsing for same-rule, same-kind projected edges, and assert in a test that no
   subject in the live graph produces more than nine chips.
4. Add `tui_trace("widget.link_rail.update")` and a `bench_tui_jk.py` case.
5. PNG goldens: the same rail location on all three tabs; 1-link, 3-link, and 12-link
   renders; an inverse-direction label; a dangling row; 120×40 and 60×30. The strongest
   no-links assertion is **exact pixel equality with a view that has no rail mounted at
   all**, not "the panel says empty".

### Phase `follow` — The `$` grammar and a jump that always lands

1. Add `"dollar_sign": "$"` to `_KEY_DISPLAY` (`keymaps/key_validation.py:8`) and a
   `"$"` friendly alias beside the existing `"+"`/`"-"` spellings.
2. Generalize `modals/numbered_link_keys.py` from a hard-coded `.` to a parameterized
   prefix and reuse it app-level: arm on prefix, resolve on the next decimal digit,
   cancel on anything else, never fire while an `Input` has focus. Add `$$` and `$0`.
3. Implement the app-level jump verb and the availability gating for
   `follow_artifact_link`, including the footer, help, and command-palette suppression.
4. Make a jump always land: extend `_navigate_to_relation_target` so the cross-pane
   branch reaches `reveal_entry_target`, then fold expansion, head-slice request by
   stable identity, and project-scope change — each reversible, each toasting what it
   changed.
5. Tests per pane for filtered, folded, outside-head-slice, and cross-project targets;
   dangling target visible but disabled; a failed jump leaves history untouched;
   selection changing mid-async-refresh never paints a stale rail.

### Phase `trail` — Walking back across surfaces

1. Add the app-level `_link_trail` bounded at 32, recording tab, target, query digest,
   and fold state per hop.
2. Route `Ctrl+O` / `Ctrl+Shift+O` through it when the last navigation was a link hop,
   and fall through to the existing per-surface anchor stacks otherwise, unchanged.
3. Render the breadcrumb chip and its `⟨ …3 › ✎plan ⟩` collapse; clear the trail on any
   other navigation.
4. Test the round trip across **all pairs** of top-level tabs, and that a restore undoes
   a query widening and a fold expansion.

### Phase `panel` — The `$0` Links panel

1. Build the modal on the `AgentNeighborModal` shape with `a`–`z` row keys, sharing the
   rail's index and ordering so the two can never disagree. Support opening it scoped to
   one counted chip's group, which is where `$N` on a collapsed chip lands.
2. Render the full `why` per row on its own indented line, plus origin, uses,
   `created_at`, `created_by`, and the projection rule id for projected rows.
3. Surface index staleness here — never as a validation on the render path — using the
   drift summary from `truthread`.
4. Offer `add`/`rm` through the existing `ArtifactLinkModal` and
   `artifacts_link_marked`, gated by the writability flag; label non-store rows by
   source.
5. Goldens at 120×40 and 60×30 for a 3-link and a 26-link neighbourhood, a dangling row,
   and the staleness notice.

### Phase `land` — Retire the duplicates and land the rail

1. Filter `RelationKind.LINK` sections out of `build_relation_view`'s output. Structure
   stays in the pane; links become app-level. This shrinks the collapsed relations rail
   to `▸ . expand · > 2 children` and removes the unkeyed `1 plans` segment visible in
   `artifacts_beads_collapsed_relations_120x40.png`.
2. Retire `beads_open_plan` and `plans_open_bead` (`L`), which `$1` subsumes, and delete
   the `first_link_target` plumbing they need. This frees `L` on two panes and is a net
   deletion — the strongest sign the boundary is right.
3. Remove the feature flag: delete the Off branch, make the On branch unconditional,
   drop the registry entry, and close the flag bead in the same change.
4. Rebaseline the affected goldens and run the full verification below.

## Verification

Beyond each phase's own tests:

- Every ref kind in **both endpoint positions**; directed primary and inverse labels;
  symmetric `related`; neighbourhoods of 1, 2, 9, 10, and 26.
- **The index matches the store, not the aggregate** — the assertion that would have
  caught the live defect.
- Every writer of `artifact-links.json`, run in every order, converges on one row set.
- A behind-HEAD clone cannot delete an ahead clone's rows.
- Each projection rule is idempotent and complete against a fixture repo and fixture
  agents sidecar; a chop-launched agent's edge survives run-history pruning and chop
  disable.
- Zero-link entity: no widget, no footer action, no help row, no palette entry, and
  pixel equality with an unmounted rail. Synthetic Agents banner, AXE lumberjack, and
  background command likewise.
- Filtered, folded, outside-head-slice, and cross-project targets each reveal and
  restore; `Ctrl+O` / `Ctrl+Shift+O` round-trip across all tab pairs.
- `p95 < 16 ms` key-to-paint on every tab under `SASE_TUI_PERF=1`, with the rail present
  and absent, plus the `bench_tui_jk.py` case.
- `just check-full` through `/sase_monitor` before landing.

## Risks

| Risk                                                                                              | Severity | Mitigation                                                                                                                                                                                                                                   |
| ------------------------------------------------------------------------------------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The rail under-reports because its read model is wrong                                            | **High** | It is wrong today and this epic fixes it first: `converge`, `freshness`, and `truthread` all land before any UI. Tests assert against the store.                                                                                             |
| Removing `_row_is_publishable` from the local path leaks unpublished agent refs to other machines | Medium   | It is removed only from the _local read model_. `durable_sidecar_rows()` keeps it, and publication continues to go through the outbox. Covered by a test that a locally visible row is not published.                                        |
| Projected rows confuse users who cannot delete them                                               | Medium   | Writability flag on every chip; `$0` labels the source and the rule; `link rm` refuses with a pointer to the source fact.                                                                                                                    |
| The projection is slow enough to be felt                                                          | Medium   | Cached by commit sha and file mtime, built off-thread, gated by the existing signature, and smoke-tested inside the chop budget. If it still costs, narrow `stitch-agent` first — it is the largest and least valuable rule.                 |
| The graph grows a long tail and two keystrokes stop reaching everything                           | Medium   | Counted-chip collapsing bounds the rail at nine chips by construction, so degree growth costs a group open rather than a key. Still **re-measure the degree distribution at the end of `project`** to confirm the grouping keys stay stable. |
| Cross-pane query widening surprises the user                                                      | Medium   | Always toast the change, always restore on `Ctrl+O`, one test per pane.                                                                                                                                                                      |
| Rail appearing and disappearing causes layout jitter while scrolling a mixed list                 | Medium   | It sits above a fixed footer, so only its own row moves. If jitter is felt, reserve the row permanently on tabs where more than 30% of rows are linked.                                                                                      |
| Two rails on Artifacts confuse rather than clarify                                                | Medium   | `land`'s boundary: structure stays in the pane, links go app-level. If it still confuses, merge by moving hierarchy into the rail and deleting the pane panel — deliberately deferred.                                                       |
| `$` collides with a future Textual or plugin binding                                              | Low      | Unclaimed today across all 109 configured key strings, validated centrally, user-overridable like every other action.                                                                                                                        |

## Sequencing notes

- The reliability phases are not gated by any in-flight epic. `bead:sase-ua` should be
  rescoped by `converge` rather than waited on; `bead:sase-t0` and `bead:sase-u9` remain
  independent.
- `project` needs a relation-registry addition in `../sase-core`, so it carries a Rust
  release plus binding update. Sequence that work first inside the phase.
- `rail`, `follow`, `trail`, and `panel` all move goldens. Rebaseline once per phase and
  keep `land`'s rebaseline separate so the boundary change is reviewable on its own.
