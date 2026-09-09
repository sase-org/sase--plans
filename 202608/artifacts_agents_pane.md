---
tier: epic
title: Artifacts Agent pane — a queryable agent catalog with revival
goal: 'The Artifacts tab gains an "Agent" pane that catalogs every agent SASE has ever
  named, filters it with the shared query dialect, resolves the `agent:` half of the
  artifact-link graph, and revives dismissed agents — with `sase agent search` giving
  the same catalog and dialect headless.

  '
phases:
  - id: grammar
    title: Widen the shared boolean query dialect's value grammar
    depends_on: []
    size: medium
    description: "grammar: teach the boolean profile dialect to spell dotted and
      digit-leading values in Python and Rust, so dates, durations, integers, and agent
      names are expressible at all; fixes an existing Patches canonicalization
      round-trip defect.

      "
  - id: catalog
    title: Textual-free agent catalog row model
    depends_on: []
    size: medium
    description: "catalog: build the registry-spined, index-enriched catalog snapshot as
      a widget-free module shared by the pane and the CLI, with a measured budget.

      "
  - id: dialect
    title: The agents query profile
    depends_on:
      - grammar
    size: medium
    description: "dialect: author agents_query_schema(), register it as a built-in pane
      profile, and add cross-language query goldens and conformance coverage.

      "
  - id: pane
    title: Feature flag, pane contract, and the mounted list
    depends_on:
      - catalog
      - dialect
    size: medium
    description: "pane: create the disabled beta flag, register the built-in
      adapter/descriptor before Files, and mount a snapshot-backed list that renders the
      shared shell and passes contract conformance in both flag states.

      "
  - id: query
    title: Filter bar, Rust evaluation, saved queries, and history
    depends_on:
      - pane
    size: medium
    description: "query: wire the profile filter bar and ArtifactQuerySession over a
      Rust query index, with limit-capped presentation, facet completions, and lazy
      content gating.

      "
  - id: detail
    title: Detail panel, grouping, relations, link targets, and copy
    depends_on:
      - pane
    size: medium
    description: "detail: add lazy agent detail, family/state grouping banners, declared
      relations plus artifact-link edges, the agent branch in link target resolution,
      copy targets, and marks.

      "
  - id: revive
    title: Revival from the pane, with one mutation implementation
    depends_on:
      - pane
    size: medium
    description: "revive: give the existing revive path a completion delta, make the
      main tab a consumer of it, bind revive on the pane, and reroute the R modal's
      custom search into the pane.

      "
  - id: cli
    title: sase agent search
    depends_on:
      - catalog
      - dialect
    size: small
    description: "cli: ship the headless catalog search command over the same row model
      and dialect, with pretty and JSON output.

      "
  - id: land
    title: Remove the flag and land the pane
    depends_on:
      - query
      - detail
      - revive
      - cli
    size: medium
    description:
      "land: delete the flag's Off branch, close the flag bead, refresh visual goldens
      and help text, and gate landing on a monitored full verification run."
proposed_by: bbugyi200.athena.0da
bead_id: sase-tj
create_time: 2026-09-09 19:49:54
status: wip
---

- **PROMPT:**
  [prompts/202608/artifacts_agents_pane.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifacts_agents_pane.md)
- **BEAD:**
  [sase-tj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-tj/README.md)

# Plan: Artifacts Agent pane

## 1. What this epic ships, and what it deliberately does not

The Artifacts tab gets a sixth built-in pane, **Agent**, inserted immediately before
Files. It is a durable, query-first catalog of every agent SASE has ever named — 12,525
names on this machine today — backed by the agent name registry and enriched from the
agent artifact index and the dismissed bundle archive. It answers "find the dismissed
agent I want back" in one line of query, and it is the destination the `agent:` half of
the artifact-link graph has been pointing at since the graph existed.

`sase agent search` ships in the same epic over the same row model and the same dialect,
so the catalog is not TUI-only.

**Out of scope, by the owner's framing:** the artifact-link defect fixes (the lossy
aggregate rebuild, uncommitted read rows, one-sided bead endpoints, rename following)
and the later "every tab is link-aware" project. §9 lists the seams this epic must leave
open for them, and §6.4 states the one thing this epic must _not_ claim about links.

**Also out of scope:** migrating the main Agents tab onto this dialect and deleting
`src/sase/ace/agent_query/` (1,189 lines). That change touches the app's default startup
tab and must be revertible on its own. It is the natural successor epic, and §3.1
explains why the dialect is authored so that it becomes a mechanical change.

### 1.1 The two surfaces are different products

The **Agents tab** is the live control room: attention state, folds, orchestration, live
workflow children, panel layout. The **Artifacts → Agent pane** is the historical
catalog: complete, queryable, revivable, link-addressable. They share the query dialect
and one revival implementation. They do not share a widget, and this epic does not
deprecate either.

Every piece of user-facing prose this epic writes — help modal rows, empty states,
`--help` text — must be unambiguous about which surface it means. Spell them "Agents
tab" and "Artifacts → Agent pane".

## 2. The row model

### 2.1 The spine is the agent name registry

Measured on this machine on 2026-08-25 against the live link aggregate for
`gh_sase-org__sase` (107 rows, 63 `agent:` refs, 47 distinct payloads):

| Candidate spine                                              |   `agent:` refs resolved |
| ------------------------------------------------------------ | -----------------------: |
| **Agent name registry** (`~/.sase/agent_name_registry.json`) |              **47 / 47** |
| Artifact index and dismissed archive together                |                 ~28 / 47 |
| Live Agents tab in-memory list                               | visible working set only |

46 of 47 resolve on the bare local key; one (`agent:bbugyi200.athena.002--1`) resolves
on `canonical_global_name`. The registry is the only complete spine, because naming is
exactly the job an `agent:` ref does.

Do **not** back rows on the artifact index alone, on the published agents sidecar pages
(git-synced, can lag or be absent on a fresh machine), or on the live Agents tab list
(which excludes dismissed agents by construction — the very rows revival needs).

### 2.2 One row set, one `kind`, two display levels

A row is a registry entry. Its display identity is the name; its run identity is carried
alongside, because a name is not unique across time.

```text
AgentCatalogRow
  target      : ArtifactEntryTarget("agents", (name,))
  identity    : name, canonical_global_name, kind, project
  run         : raw_suffix, artifacts_dir, bundle_path          (may be absent)
  attributes  : left-joined from the artifact index and dismissed archive
  provenance  : which sources enriched this row, and which did not
```

`kind` is derived and is what makes the two levels work. Live registry counts:

| `kind`           | Derivation                                                 |          Count |
| ---------------- | ---------------------------------------------------------- | -------------: |
| `family`         | `container_kind == "family"`                               |          1,751 |
| `clan`           | `container_kind == "clan"`                                 |            610 |
| `member`         | `reservation_kind == "claimed"` and the name contains `--` |          3,839 |
| `agent`          | `reservation_kind == "claimed"` and no `--`                |          6,280 |
| `workflow`       | `agent_type == "workflow"` — a workflow parent run         |  2,682 indexed |
| `workflow-child` | `is_workflow_child` from the archive join                  | 20,194 bundles |

**19 of the 47 live `agent:` refs (40%) name a family container**, not a run. A family
is a first-class row, not a synthesized grouping artifact. The family banner over member
rows is a _grouping mode over this one row set_, never a second record type. `agent:0b4`
resolving to a family container is correct behavior, not a bug: `0b4` is a family
reservation created `20260822161630`, and `0b4--0` is its member.

Family/member linkage is derived from the name (`0b4--0` → family `0b4`). `role` is the
alphabetic head of the `--` suffix (`mon-0` → `mon`); a numeric-only suffix (`001--2`)
carries no role. Live role distribution: `code` 1,437, `plan` 1,269, `mon` 361.

**Key rows on the name, never on `artifacts_dir`.** `_target_for_ref` already emits
`ArtifactEntryTarget("agents", (payload,))` for every `agent:` ref, and refs address
names. A `(project, artifacts_dir)` shape would silently never match an incoming ref.

### 2.3 Enrichment: degrade a row, never drop it

Measured over all 12,525 registry names:

| Enrichment                                                                 |       Rows |         % |
| -------------------------------------------------------------------------- | ---------: | --------: |
| Artifact-index row (join on `artifacts_dir`)                               |      9,460 |     75.5% |
| Dismissed top-level bundle (join on `raw_suffix`, `is_workflow_child = 0`) |      8,677 |     69.3% |
| **Either**                                                                 | **11,474** | **91.6%** |
| Neither — name-only row                                                    |      1,051 |      8.4% |

The 8.4% still render, showing name, kind, state, project, and created time with an
explicit "no run data" detail state. A thin row is strictly better than today's status
quo, which is a link target that resolves to nothing.

**The dismissed join must filter `is_workflow_child = 0`.** The archive holds 27,033
bundles across only 7,098 distinct `raw_suffix` values; 20,194 are workflow children.
Filtered to top-level, 6,839 bundles carry 6,839 distinct suffixes and the join is
**verified 0-ambiguous across all 12,525 names**. An unfiltered join is many-to-one and
would attach an arbitrary sibling's attributes.

**Read `bundle_path` from the dismissed summary index, never from the registry.** Only
99 of 12,525 registry entries carry a top-level `bundle_path` (they are the
`source: dismissed_bundle` entries).

**Never `SELECT *` the artifact index.** `record_json` is ~117 MB across 8,022 rows.
Project the columns §3.3 needs and nothing else.

### 2.4 Measured budget

Composite snapshot, warm, single-threaded Python, on this machine:

| Step                                         |       Cost | Result                   |
| -------------------------------------------- | ---------: | ------------------------ |
| Registry load + parse (16 MB JSON)           |     147 ms | 12,525 names             |
| Artifact index, 23-column projection         |      66 ms | 8,022 rows               |
| Dismissed summaries, `is_workflow_child = 0` |      32 ms | 6,839 rows               |
| Join and row derivation                      |      27 ms | 12,525 rows, 0 ambiguous |
| **Total**                                    | **273 ms** | complete, fully enriched |

Budget: **≤ 400 ms** for a complete cold snapshot on a worker thread, asserted by a
bench beside `tests/ace/tui/bench_artifacts_jk.py`.

### 2.5 Where the row model lives, and when it moves to Rust

Build it as a **Textual-free Python module** at `src/sase/agents/catalog/`, imported
identically by the pane worker and by `sase agent search`. One implementation, two
frontends, no duplication.

`rust-core-required`'s litmus test — "if a CLI would need the behavior to match the TUI,
it is core backend logic" — does point at `sase-core`, and this must be recorded
honestly rather than ignored. It is not followed here for one concrete reason: **the
agent name registry is a Python-owned JSON store whose writer is 7,153 lines of Python**
(`src/sase/agent/names/`). A Rust reader of a Python-written registry would create a
_second_ schema owner for the same file — precisely the silent-divergence failure the
decision exists to prevent, relocated rather than removed. Two of the three sources
already have Rust readers (`agent_scan/index.rs`, `agent_archive/mod.rs`); the registry
does not, and porting the reader without the writer is the wrong half.

The part where the TUI and the CLI genuinely must agree — **query parsing,
canonicalization, and evaluation** — already runs through Rust via
`compile_artifact_query_index` / `evaluate_artifact_query_many`, and this epic keeps it
there.

**Promotion trigger, to record in the module docstring:** promote `sase.agents.catalog`
to `sase-core` when the agent-name registry itself moves to a Rust-owned store, or when
a third frontend needs the row model. The 147 ms registry parse is the measurement that
would justify it if the 400 ms budget is ever missed.

## 3. The query dialect

### 3.1 A shared profile, not a third dialect

Author `agents_query_schema()` in `src/sase/ace/query_profile/profiles.py` and register
`"agents"` in `_BUILTIN_SCHEMA_BUILDERS`
(`src/sase/ace/query_profile/pane_registry.py`). Declare **`boolean=True`**.

Why a profile rather than extending `src/sase/ace/agent_query/` (1,189 lines, whose own
tokenizer docstring says it was "Adapted from `sase.ace.query.tokenizer`"):

- Saved queries and query history are pane-namespaced and digest-stamped. A profile
  dialect earns both stores, the slot picker, `^`/`_` history navigation, and
  dialect-change invalidation for free. A forked dialect earns none of them, ever.
- The profile-driven `FilterBar` supplies completion, static value lists, per-key hints,
  and highlighting from the schema. `agent_query` hand-writes `highlighting.py` and
  drives a modal query editor with a one-line hint.
- Rust corpus routing and the cross-language conformance golden
  (`tests/ace/tui/artifacts_contract/test_query_conformance.py`) come with the profile.
- `procs_query_schema()` is the load-bearing precedent that a profile dialect needs no
  pane: it is registered and consumed by `ProcQueryFilter` with no Artifacts pane behind
  it. That is why `dialect` and `cli` can land and be tested headless.

`boolean=True` because the main Agents tab already has AND/OR/NOT with parentheses, and
flat mode would make the eventual tab migration a downgrade. The cost is that boolean
panes have no `key:a,b` OR-lists, no `-key:x` negation, and no bare-bool shorthand;
users write `key:a OR key:b`, `NOT key:x`, and `revivable:true`.

### 3.2 The blocking defect: the boolean dialect cannot spell values

**This is the single most important correction in this plan, and the `grammar` phase
exists for it.** The shared boolean profile parser's unquoted property-value grammar is
`[A-Za-z_][A-Za-z0-9_-]*` (`_PROPERTY_VALUE_RE` in
`src/sase/ace/query/profile_reference_boolean.py:28`). Verified against a compiled
boolean profile on this tree:

```
since:2h           ERROR  Expected property value (at position 6)
until:7d           ERROR  Expected property value (at position 6)
min:5m             ERROR  Expected property value (at position 4)
attempt:2          ERROR  Expected property value (at position 8)
since:2026-08-01   ERROR  Expected property value (at position 6)
name:0b4           ERROR  Expected property value (at position 5)
family:research.12 ERROR  Unexpected character: . (at position 15)
name:sase-r8.9.land ERROR Unexpected character: . (at position 12)
9lives             ERROR  Unexpected character: 9 (at position 0)
```

In boolean mode today **no date, duration, or integer value can be spelled at all**, and
no dotted or digit-leading value can be spelled unquoted. The prior research concluded
that `until:2h` already works and needs no host change; that is true in _flat_ mode
(`min:5m -> min:300` verified) and false in boolean mode. Agent names are dotted,
digit-leading, and `--`-containing by construction (`0b4`, `research.11.cdx`,
`sase-r8.9.land`, `athena.sase-8t`, `001--2`), and time is this pane's
second-most-important axis after `revivable`. Without the fix, `boolean=True` is not
viable and the dialect would have to be flat.

It is also an existing latent defect on Patches, the one boolean pane today:

```
name:"sase-r8.9"  ->  canonical  name:sase-r8.9  ->  re-parse ERROR
```

Canonicalization is not round-trippable for such values, and saved queries and query
history store canonical text — so a saved Patches query on any dotted name is already
unloadable.

There is also a live cross-language parity divergence: Rust's `parse_property_value`
starts scanning at `is_bare_word_byte` (`crates/sase_core/src/query/tokenizer.rs:19`,
alphanumeric + `_` + `-`), so Rust accepts `name:0b4` while Python rejects it.

**The fix, and its exact charter.** Widen the boolean dialect's bare-token character
classes, in Python and Rust in lockstep:

1. An unquoted property value may start with a digit and may contain `.`.
2. A bare free-text word may start with a digit and may contain `.`.

Nothing else. `.` is not currently meaningful anywhere in the boolean grammar (it is an
"Unexpected character" in every position), so this is a **strict widening**: no query
that parses today changes meaning, and previously-invalid queries become valid. No
profile digest changes, because the digest is computed over the schema payload, not the
grammar. Every saved query that works today keeps working, and the Patches round-trip
defect above is repaired as a side effect.

This phase spans two repos. Follow `rust-core-required`'s stated cost: one commit in
`../sase-core` (opened with `/sase_repo`), one in the Python adapter, ratcheted and
released in order.

### 3.3 Field set

Grouped by the question a user is actually asking. **Every field below is backed by a
column the row model already projects** — see §3.4 for what was cut and why.

**Identity and lineage**

| Key        | Kind               | Source                                                            |
| ---------- | ------------------ | ----------------------------------------------------------------- |
| `name`     | string, exact      | registry key; also matches `canonical_global_name`                |
| `kind`     | enum, multi-valued | `agent`, `member`, `family`, `clan`, `workflow`, `workflow-child` |
| `family`   | string, exact      | derived from the name; bare-value match on any member             |
| `clan`     | string, exact      | registry `container_kind`/name, `agent_clan`                      |
| `tribe`    | enum               | `clan_tribe`: `epic`, `chop`, `research`                          |
| `role`     | string             | alphabetic head of the `--` suffix (`code`, `plan`, `mon`)        |
| `workflow` | string             | `workflow_name` / archive `workflow`                              |
| `parent`   | string             | `parent_timestamp`                                                |
| `project`  | string, exact      | **display name and key both match; render the name**              |

**Lifecycle**

| Key         | Kind | Source                                                                                                                                                                                                    |
| ----------- | ---- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `state`     | enum | registry `state`: `active`, `done`, `dismissed`                                                                                                                                                           |
| `status`    | enum | `STARTING`, `RUNNING`, `WAITING`, `DONE`, `FAILED`, `COMPLETED` — declare the uppercase display spellings and normalize the stores' lowercase values into them; enum matching is already case-insensitive |
| `hidden`    | bool | artifact index `hidden`                                                                                                                                                                                   |
| `dismissed` | bool | derived from `state`                                                                                                                                                                                      |
| `revivable` | bool | **derived**: dismissed **and** a readable top-level bundle exists                                                                                                                                         |
| `attention` | bool | derived: failed, or waiting on input                                                                                                                                                                      |
| `retry`     | bool | derived: participates in a retry chain                                                                                                                                                                    |
| `attempt`   | int  | `retry_attempt`, equality-only (§3.5)                                                                                                                                                                     |

**Execution**

| Key        | Kind   | Source                                                            |
| ---------- | ------ | ----------------------------------------------------------------- |
| `model`    | string | `model`                                                           |
| `provider` | enum   | `llm_provider`: `claude`, `codex`, `grok`, … plus observed facets |
| `patch`    | string | see §3.4 — derive, measure, and drop if degenerate                |

`kind` is the single identity axis and is deliberately **multi-valued**: a workflow
parent that is also a family member carries `("member", "workflow")`, and any declared
value matching wins. This is the idiom `_bead_query_entry` already uses for `type` /
`status` / `tier`. Do **not** add a second `type:` key for `agent_type` — its `agent`
value would collide with `kind:agent` and mean something different.

**Time.** `since` / `until` over started-at; `after` / `before` over finished-at; `min`
/ `max` over runtime seconds. See §3.5.

**Search-only** (`filterable=False, searchable=True`): `label`, `text`. Free text
searches indexed metadata only — see §3.6.

Two fields carry the pane:

- **`revivable` is the highest-value key in the schema.** The flagship gesture is "find
  the dismissed agent I want back".
  `revivable:true AND project:sase AND role:code AND since:7d` answers in one line a
  question that today means paging an archive modal over 27,033 JSON bundles.
- **`kind` must distinguish containers from runs.** 2,361 of 12,525 names are family or
  clan reservations. Without `kind`, `name:0b4` returns a container and its member with
  equal weight.

Completions for `status`, `role`, `model`, `provider`, `project`, `tribe`, and
`workflow` merge static canonical values with **observed snapshot facets**, so an
unreleased model or a new provider stays queryable.

Worked examples for the help text and the `--help` epilog:

```text
revivable:true AND project:sase AND role:code
provider:codex AND status:FAILED AND since:7d
family:"research.12" AND NOT kind:workflow-child
state:active AND (attention:true OR status:WAITING)
retry:true AND model:"gpt-5.6-sol" AND min:5m
```

### 3.4 Fields deliberately not in v1, and why

`effort`, `bead`, `epic`, `workspace`, and `xprompts` are **not** v1 fields. None of
them is a column in either index; they live only inside `record_json`, which is the 117
MB blob §2.3 forbids reading. Adding them means an artifact-index schema migration plus
a reindex, and the dismissed archive has no equivalent at all — so half the corpus could
never answer them. Shipping a filter that silently matches nothing on 8,677 dismissed
rows would be worse than not shipping it.

Record them as a `PROPOSED FOLLOW-UP:` note on the `dialect` phase bead, naming the
artifact-index columns each would need.

`patch` is a judgment call the `catalog` phase must resolve with a measurement, not an
assumption. `cl_name` in the artifact index is dominated by project keys
(`gh_sase-org__sase` on 7,211 of 8,022 rows), not patch names. Derive `patch` from
`cl_name`/`meta_patch` **excluding values that equal a known project key**, then report
the resulting distinct-value cardinality on the phase bead. If it is degenerate, **drop
the field** rather than ship a filter that lies.

### 3.5 Time, and the `Nm` collision

`HOST_DATE_BOUND_KEYS = {since: >=, after: >=, until: <=, before: <=}` and
`HOST_DURATION_BOUND_KEYS = {min: >=, max: <=}`, mirrored byte-for-byte in
`crates/sase_core/src/query/profile.rs`. Comparison direction is keyed off the field
name, not a separate operator, so a dialect gets **two date ranges and one numeric
range**, and every other `int` field is equality-only.

| Intent                                | Spelling             |
| ------------------------------------- | -------------------- |
| Started at least 2h ago (`age>2h`)    | `until:2h`           |
| Started within the last 2h (`age<2h`) | `since:2h`           |
| Finished inside a window              | `after:` / `before:` |
| Ran at least 5 minutes                | `min:5m`             |
| Ran at most 2 hours                   | `max:2h`             |

Do **not** add a `duration` value kind or general `<`/`>` operators. That would touch
the Rust profile, parser, tokenizer, and evaluator plus the Python registry, compiler,
both reference parsers, evaluator, highlighting, and completion — for expressiveness the
host already has once §3.2 lands.

**One trap that must be documented in both hints and tested.** In a **date** bound,
`_RELATIVE_MONTH_RE = ^(\d+)m$` makes `5m` five _months_; in a **duration** bound, `5m`
is five _minutes_. `since:5m` and `min:5m` differ by a factor of ~43,000. This is also
the reason not to add `age>5m` input sugar: the sugar would have to pick a target
silently. Put the collision in the `since`/`until` hints and in the `min`/`max` hints,
and cover it with a golden case.

`attempt` is equality-only for the same structural reason (`attempt:2` means exactly two
attempts). If a third expressiveness gap appears, reopen a general comparison operator
as its own shared-infrastructure change with its own justification — never as a rider on
this pane.

### 3.6 Content search stays lazy, and `content:` is not offered

`AgentContentSearchCache` (`src/sase/ace/tui/models/agent_content_search.py`) reads
prompt, reply, and attempt replies, capped at 512 KiB per file and keyed on
`(path, mtime_ns)`. That is right for a ~200-row live inbox. Applied to 12,525 rows it
turns a metadata keystroke into tens of thousands of file stats.

Port the gating shape from `_query_needs_output` in `src/sase/ace/tui/_proc_query.py`:
parse first, walk the AST for content-needing keys, and only build the expensive
haystack when the query actually asks. Default free text searches indexed metadata only.

Do **not** offer a `content:` key in v1. A persistent FTS index over transcripts is the
correct answer, and until it exists a `content:` key would silently walk the archive.
`chats_catalog.sqlite`'s generated `agent-links` index is the right lookup for a row's
transcript path and a cheap first-pass snippet haystack when the detail panel needs one.

## 4. Pane contract

| Fact                                                   | Value                                                                      | Why                                                                                                                     |
| ------------------------------------------------------ | -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `pane_id`                                              | `agents`                                                                   | `_target_for_ref` already emits it                                                                                      |
| Label                                                  | `Agent`                                                                    | fixed panes use singular labels: Patch, Stitch, Bead, File                                                              |
| Position                                               | **immediately before Files**                                               | `assign_artifacts_digit_shortcuts` pins Files to the highest digit by rule; this makes `agents`=6 and moves `files` 6→7 |
| Icon                                                   | one cell, distinct from `◉ ⎇ ◈ ▤ ◆`                                        | `⬡` is the suggested value                                                                                              |
| Accent                                                 | new `ARTIFACTS_ACCENTS["agents"]`                                          | see the trap below                                                                                                      |
| `has_inventory` / `has_fields` / `has_stable_identity` | true                                                                       | earns FILTER_SESSION, QUERY_HISTORY, SAVED_QUERIES, STABLE_MARKS, STABLE_REFERENCE_COPY                                 |
| `has_revisions`                                        | false                                                                      | shell lineage is a relation, not document versions                                                                      |
| `can_mutate`                                           | **true**                                                                   | revive is a `PaneCapability.MUTATION` verb                                                                              |
| `project_scoped`                                       | true                                                                       | required for conformance                                                                                                |
| `has_detail`                                           | true                                                                       |                                                                                                                         |
| Relations                                              | `family`, `clan`, `retry_chain`, `parent`, plus auto `links` / `linked_by` | `with_artifact_link_relations()` appends the link pair for every built-in adapter                                       |
| Grouping                                               | `by_family` (default), `by_state`, `by_project`                            | grouping presents the query result; it is never a second filter                                                         |
| Status counters                                        | one counter on `status`                                                    |                                                                                                                         |
| Copy group                                             | `artifacts_agents` (new)                                                   | do **not** reuse the main tab's `agents` copy group; adding targets there would change the Agents tab's copy footer     |
| Revive key                                             | `agents_revive: "w"`                                                       | see §7.2                                                                                                                |

**The accent trap.** `_provider_accent_for_kind` hashes provider kinds onto
`_PROVIDER_ACCENTS` _after removing every reserved `ARTIFACTS_ACCENTS` value_, and
`hash_palette_index(kind, len(palette))` depends on the palette length. Choosing an
accent that equals an existing provider accent would shrink the palette from 9 to 8 and
**repaint the existing `ref:research` tab**. Pick a colour that satisfies
`tests/ace/tui/test_artifacts_provider_palette.py` — ≥3.3 contrast against both dark and
light shell surfaces and against the `#1A1A1A` chip text, and ≥0.09 OKLab distance from
every other `ARTIFACTS_ACCENTS` value, from `EXTERNAL_ACCENT`, and from every
`_PROVIDER_ACCENTS` entry. That last constraint makes equality impossible, so the
palette length is preserved automatically; the existing test enforces it.

**Add every registration together** — pane id, icon, accent, `FIXED_ARTIFACTS_*`
entries, `BUILTIN_ADAPTERS` row, `fixed_descriptor` label, profile registry entry, and
the `ArtifactsView._compose_pane()` branch — so no half-configured target ever exists.

### 4.1 Conformance is the quality floor, and it is free

`tests/ace/tui/artifacts_contract/harness.py` auto-parametrizes **12 checks** over every
descriptor `resolve_artifacts_subtabs()` returns, the moment `agents` appears there:
declared actions registered, declared copy targets registered, declared relation edges
resolve, declared grouping banners navigable and foldable, declared keys resolve to the
action the contract names with no available conflict, unavailable actions carry OFF
verdicts, and the pane renders the shared shell.

Because the flag defaults **off**, the pane is absent from `resolve_artifacts_subtabs()`
in a default test run and would get **zero** conformance coverage. The `pane` phase must
therefore add an explicit flag-on fixture that sets the flag, calls
`reset_artifacts_subtabs_cache()`, and runs the full conformance sweep — and a flag-off
test asserting today's inventory
(`stitches, patches, beads, ref:plan, ref:research, files`) is reproduced byte-for-byte,
Files still on digit 6.

`test_query_conformance._REQUIRED_PROFILE_PANES` is a hard-coded set; the `dialect`
phase adds `agents` to it and adds the matching corpus to
`tests/ace/tui/artifacts_contract/goldens/query/profile_cases.json`.

## 5. Performance contract

Non-negotiable, and each one is a review checklist item:

- **Index-only.** Build from the registry, the artifact index (23-column projection),
  and the dismissed summary index (`top_level_only=True`). Never `scan_artifacts()`,
  never `SELECT *`, never page bundle JSON files.
- **Cached first, revalidate explicitly.** Activation consumes the last snapshot
  immediately and schedules a refresh off-thread through `ArtifactsSnapshotPane` /
  `SnapshotRequest`, the same lifecycle every other pane uses. Snapshot generation,
  index signatures, and project scope belong to the cache key; the profile digest and
  canonical query are already inside `ArtifactQueryCacheKey`.
- **Full corpus, bounded presentation.** Queries evaluate across all 12,525 rows in the
  Rust index; the option list renders a `limit:`-capped window with an explicit
  `showing N of M`. **Never create 12,000 Textual options.** Reuse the host `limit:`
  token (`src/sase/ace/query/limit_token.py`) and the existing `artifacts_load_more` /
  `artifacts_unload` keys rather than inventing paging.
- **Exact navigation bypasses the cap and the query.** A link jump to a row the current
  query excludes clears the query rather than failing — the pattern other panes already
  use for out-of-scope targets.
- **Lazy detail.** Prompt, chat, reply, commits, output variables, and bundle JSON load
  for the selected stable id after the normal `DetailPanelDebouncer` window, with a
  selection generation that prevents a late worker from overwriting a newer selection.
- **Thin pump callbacks.** Worker completion swaps immutable state and refreshes
  affected rows only. No filesystem reads, subprocesses, or index rebuilds in render,
  watch, or callback paths. Long bodies go through `spawn_pump_free_task()` and are
  cancelled in `on_unmount` with `cancel_pump_free_tasks()`.
- **Guard programmatic highlights.** `OptionList` emits `OptionHighlighted` echoes on
  programmatic assignment; set the guard flag and clear it synchronously in `finally:`.

Target: j/k p95 < 16 ms on the pane at the full 12k corpus, captured with
`SASE_TUI_PERF=1` and reported on the `land` phase bead.

## 6. Link destination work

This epic owes the artifact-link graph exactly four things, and nothing more.

1. **Add an `agent` branch to `_known_target_for_ref`**
   (`src/sase/ace/tui/relations/artifact_links.py:189`). It has no such branch today, so
   the `_target_for_ref` fallback only matches by accident. The branch must match (a)
   the bare local name and (b) a row whose `canonical_global_name` equals the payload.
   That one branch makes all 47 live refs land, containers included.
2. **Load an `ArtifactLinksSnapshot` on the pane worker** and pass `known_targets` to
   `artifact_link_edges`, the way `files_pane` does. `with_artifact_link_relations`
   appends `links` / `linked_by` automatically, so the declaration itself is free.
3. **Normalize the ref form on read, not on write.** 46 of 47 live refs are bare local
   names, 1 is owner-qualified; match either. Do not rewrite stored rows — that is
   link-graph write-path scope with open work already on it.
4. **Emit a canonical `agent:<canonical_global_name>` reference** from the pane's
   `reference` copy target, so newly written links are owner-qualified even while reads
   tolerate both forms.

### 6.1 Ship link _navigation_, not link _filtering_

The machine-global link aggregate is rebuilt destructively from whichever workspace
clone triggers it, and it does not preserve rows it cannot see. It has been measured at
137, 84, 95, and 107 rows on successive reads. A `linked:false` or `-has:links` filter
over an index that silently drops rows returns confidently wrong answers.

**Do not ship `relation:`, `artifact:`, or `linked:` filter keys in v1.** Reserve the
field seam on the row and enable them only after the derivation work's Phase 0 lands.
Showing edges is safe; filtering on their absence is not.

## 7. Revival

### 7.1 One mutation implementation, plus a completion delta

`_do_revive_agent(agent)` and `_do_revive_agents(agents)` live on
`AgentReviveExecutionMixin` (`src/sase/ace/tui/actions/agents/_revive_execution.py`, 394
lines) mixed into `AceApp`. They restore on-disk markers, clear the dismissed set, purge
bundle summaries in a specific order relative to the index sync, upsert the artifact
index, and schedule a refresh. Do **not** write a second implementation, and do not
attempt a full widget-free extraction in this epic — it is a delicate ordering contract
and it is not what makes the pane valuable.

The coupling is real and specific, in both methods:

```python
self._refilter_agents()                    # main-tab work, run unconditionally

def _on_revive_loaded() -> None:
    if self.current_tab != "agents":       # pane callers get no refresh at all
        return
```

The fix is contained:

1. Keep `_do_revive_agent(s)` as the single mutation implementation.
2. Give it an explicit **completion delta**: revived identities and artifact dirs,
   skipped/failed identities with their stage, and the dismissed/index generation
   change.
3. Make the main tab's refilter and reselect a _consumer_ of that delta rather than an
   unconditional side effect, so the Artifacts pane can consume the same delta to
   invalidate its snapshot generation, re-run the current query, and select the revived
   row.

### 7.2 Keys and selection

Bind `agents_revive: "w"` in the `app` keymap and register it under
`CAPABILITY_HOST_ACTIONS[PaneCapability.MUTATION]` — conformance fails otherwise. Add it
to `NON_PRS_ARTIFACT_ACTIONS` and gate it to `current_artifacts_pane_key == "agents"` in
`_app_action_availability.py`, mirroring `FILES_ARTIFACT_ACTIONS`.

`R` stays `refresh` in the `app` keymap; do not shadow it on one pane. `w` is safe on
this pane: `reword` is filtered out by the `NON_PRS_ARTIFACT_ACTIONS` gate on every
non-Patches Artifacts pane, and `beads_launch_work` is gated to the Beads pane — but the
conformance key check will verify this rather than trusting it.

Selection semantics:

- A dismissed run row revives that row and its workflow children by the existing parent
  rule.
- Marked rows revive as one batch. `STABLE_MARKS` is derived from `has_inventory`, so
  `m` / `u` come free and `_do_revive_agents` already takes a list. Do not hand-roll
  multi-select.
- A family or clan banner with exactly one revivable member revives it; with more than
  one, it **seeds the pane query** to that family's revivable members rather than
  opening a modal.
- A non-dismissed row reports "not revivable" without mutation.
- A row whose registry entry exists but whose bundle is missing stays visible with a
  diagnostic and is not `revivable:true`.

Single-row revive proceeds on the key. Batch revive names the concrete count and the
first few names first, because a collapsed group can hold more rows than fit on screen.

### 7.3 Keep the `R` panel; retire the weakest surface

`R` on the Agents tab opens `SavedAgentGroupRevivalModal` — saved dismissal _groups_
plus recent groups, with a `custom_search` escape hatch into
`DismissedAgentSelectModal`. Reviving a previously saved group is a genuinely different
and faster gesture with its own durable store (`~/.sase/dismissed_agent_groups`,
`times_revived`, `revived_at`).

**Keep `R` exactly as is. Change one thing:** point `_open_custom_revival_search`
(`src/sase/ace/tui/actions/agents/_revive_flow.py:147`) at the Artifacts → Agent pane
with a seeded `state:dismissed AND revivable:true` query, instead of pushing
`DismissedAgentSelectModal`. That retires the offset-paging archive modal — the surface
that can only filter what it has already loaded, which is the defect actually being felt
— without touching the gesture that works.

Do **not** delete `DismissedAgentSelectModal` in this epic. That is a later, separate
call after a release of real use; the retired Chats pane (`ad11756e6`) is the precedent
for doing it properly.

## 8. `sase agent search`

A thin wrapper over `sase.agents.catalog` and the compiled `agents` profile, in the
`ProcQueryFilter` shape: parse with `parse_query_for_profile`, evaluate through the Rust
index, render.

```
sase agent search [-h] [-j] [-l LIMIT] [-p PROJECT] [QUERY]
```

- Positional `QUERY`; a bare invocation lists the catalog under the default presentation
  scope. Options are never required (`cli_rules.md`).
- `-j/--json` emits a stable machine-readable array; `-l/--limit` caps rows;
  `-p/--project` scopes.
- Every public long option gets a short alias. Subcommands and options stay
  alphabetically sorted in `sase agent -h`; `search` slots between `revert` and `show`.
- Colored output using `sase.agents.status_style.agent_status_text`, so the CLI, the
  Agents tab, and the pane share one status vocabulary and one colour scheme.
- The `--help` epilog carries the §3.3 worked examples and the §3.5 `Nm` warning.
- Wire it in `src/sase/main/parser_agent.py` and dispatch in
  `src/sase/main/agent_handler.py`.

This command is the reason §2.5's row model is Textual-free from day one.

## 9. Seams to leave open for the follow-on work

The owner's steps 2 (artifact-link fixes) and 3 (link-aware tabs) are out of scope, but
this pane will carry the graph's largest edge population, so it must not foreclose them:

- Do not assume `agent:<name>` is unique across time. 6,732 of 12,525 registry entries
  carry `collision_owners`. Row identity, cache keys, and copy targets carry the run.
- Do not resolve `agent:` refs through the published sidecar alone.
- Do not flatten relation metadata further. `_emit_link_edge` already discards
  `relation`, `description`, `origin`, and `uses`, keeping the slug only inside
  `_edge_key`. Leave room for a typed relation rail; do not add new flattening.
- Every row exposes, separately: a stable `ArtifactEntryTarget`, a canonical artifact
  reference, its source project, its available relations, open/copy locators, and its
  mutation capability facts. That is enough for a later generic "link this artifact to
  …" host action — and for chops to point at canonical agent targets — without importing
  an Agents widget.

## 10. Phases

### grammar — Widen the shared boolean dialect's value grammar

Cross-repo. Open `sase-core` with `/sase_repo`.

- Rust (`crates/sase_core/src/query/tokenizer.rs`): allow `.` in the bare-token
  character class and allow a bare free-text word to begin with a digit.
- Python (`src/sase/ace/query/profile_reference_boolean.py`): widen `_PROPERTY_VALUE_RE`
  to permit a leading digit and interior `.`, and widen bare-word scanning to match, so
  Python and Rust agree byte-for-byte.
- Verify the canonicalization round trip: for every value shape,
  `parse → canonical → parse` is stable and Python and Rust produce identical canonical
  text.
- Add parity cases to the shared query goldens covering: dotted values
  (`sase-r8.9.land`), digit-leading values (`0b4`), `--` values (`001--2`), dates
  (`since:2h`, `since:2026-08-01`), durations (`min:5m`), integers (`attempt:2`), and
  bare digit-leading free text.
- Add a regression test proving the existing Patches saved-query round trip on a dotted
  `name:` is repaired, and that every currently-valid boolean query is unchanged.
- Ratchet the `sase-core-rs` dependency window and release the two commits in order.

**Do not** add `,` to the class, add comparison operators, or touch the flat parser.

### catalog — Textual-free agent catalog row model

New package `src/sase/agents/catalog/`, no Textual imports.

- Load the registry, project the artifact index by `artifacts_dir` (23 columns, never
  `record_json`), and load dismissed summaries with `top_level_only=True` through the
  existing `load_dismissed_bundle_summaries` facade.
- Derive `kind`, `family`, `role`, `revivable`, `dismissed`, `attention`, `retry`, and
  per-row provenance. Emit an immutable snapshot of typed rows plus observed facets.
- Resolve §3.4's `patch` question by measurement and report the cardinality on the phase
  bead.
- Assert the join is 0-ambiguous over the whole registry.
- Bench beside `tests/ace/tui/bench_artifacts_jk.py` with a **≤400 ms** budget against
  the measured 273 ms.
- Fixtures covering: solo, family, clan, member, owner-qualified name, collision-history
  name, workflow child, and a name-only row with no enrichment.
- Record §2.5's promotion trigger in the package docstring.

### dialect — The agents query profile

- Author `agents_query_schema()` with the §3.3 field set and `boolean=True`; register
  `"agents"` in `_BUILTIN_SCHEMA_BUILDERS`.
- Add `agents` to `_REQUIRED_PROFILE_PANES` and a corpus to
  `goldens/query/profile_cases.json`, including both date-bound pairs, the duration
  pair, boolean precedence with parentheses, and the `Nm` months-versus-minutes
  collision.
- Document the `Nm` collision in the `since`/`until` and `min`/`max` hints.
- File the §3.4 deferred-field note on this phase's bead.
- No UI.

### pane — Feature flag, pane contract, and the mounted list

- `sase flag new artifacts_agents_pane -k beta` with authored `--when-enabled`,
  `--when-disabled`, and `--remove-when` sentences. Do not hand-add the registry entry;
  do not reuse this epic as the removal bead.
- Register the pane id, icon, accent (§4's trap), `FIXED_ARTIFACTS_*` entries,
  `BUILTIN_ADAPTERS` row, `fixed_descriptor` label, and the `_compose_pane` branch — all
  together. Gate insertion in `resolve_artifacts_subtabs()` on the flag, and **include
  the flag value in `_ARTIFACTS_TAB_CACHE`'s token** so a runtime flag change through
  the Config → Flags pane is not served a stale inventory.
- Mount `ArtifactsAgentsPane` on the `ArtifactsSnapshotPane` lifecycle, modelled on
  `files_pane.py` (the closest structural analogue: index-backed, snapshot-loaded).
  Render the shared shell states — loading, stale, empty, degraded, results — and a
  bounded option list.
- Flag-on conformance fixture (§4.1) and a flag-off inventory test.

**Build the pane as a thin class over sibling mixin modules**, the way `files_pane`
does. The `query`, `detail`, and `revive` phases run in parallel against this file; each
must add a new sibling module and one base-class entry, not grow the class body.

### query — Filter bar, Rust evaluation, saved queries, and history

- `AgentFilterBar` from the compiled profile; `ArtifactQuerySession` over an
  `ArtifactQueryIndex` built with `compile_artifact_query_index` on the worker.
- Host `limit:` token handling with an explicit `showing N of M`; wire
  `artifacts_load_more` / `artifacts_unload`.
- Facet completions merged with static values (§3.3).
- AST content gating ported from `_query_needs_output` (§3.6).
- Saved queries and query history come from the contract; verify the slot picker,
  `^`/`_` navigation, and dialect-change invalidation actually work on this pane.
- Stale-worker rejection: a result whose generation or profile digest no longer matches
  is dropped, never rendered.

### detail — Detail panel, grouping, relations, link targets, and copy

- Lazy detail panel behind `DetailPanelDebouncer` with a selection generation guard.
  Render local truth first — identity, kind, state, timing, model, provider,
  family/clan/tribe, retry lineage, provenance, links — then a "Published page" section
  for the agents sidecar page, which is enrichment and never identity.
- Grouping modes `by_family` (default), `by_state`, `by_project` via
  `ArtifactGroupFoldMixin` and `build_grouped_rows`; family banners render members
  beneath their family row.
- Declared relations `family`, `clan`, `retry_chain`, `parent`, plus the automatic
  `links` / `linked_by` pair; load `ArtifactLinksSnapshot` on the worker and add the
  `agent` branch to `_known_target_for_ref` (§6).
- New `artifacts_agents` copy group: `reference` (canonical
  `agent:<canonical_global_name>`), `name`, `link`, `path`, `chat`, `prompt`, `json`,
  `handoff`, `snapshot`. Every declared target needs a registered implementation or
  conformance fails.
- Entry navigation, `jump_to_entry`, and marks. A link jump to a row the current query
  excludes clears the query rather than failing.

### revive — Revival from the pane, with one mutation implementation

- Add the completion delta to `_do_revive_agent(s)` and make the main tab's refilter and
  reselect a consumer of it (§7.1).
- Bind `agents_revive: "w"`, register it under `MUTATION`, add it to
  `NON_PRS_ARTIFACT_ACTIONS`, and gate it to the `agents` pane (§7.2).
- Wire single, marked-batch, and family/clan-banner selection (§7.2).
- Reroute `_open_custom_revival_search` to the pane with a seeded query (§7.3). Keep `R`
  and the saved-group flow untouched.
- Test partial batch failure, workflow-child cascade, the bundle-purge-before-index-sync
  ordering, and cross-surface cache invalidation in both directions.

### cli — `sase agent search`

Per §8. Include a `--help` snapshot test and JSON schema stability test.

### land — Remove the flag and land the pane

- Delete the flag's Off branch, make the pane unconditional, remove the registry entry,
  and close the flag bead in the same change (`sase_flags.md`'s removal rule).
- Refresh the PNG visual goldens: the sub-tab strip gains a pane and Files moves from
  digit 6 to 7, so every `artifacts_*` snapshot changes. Run `just test-visual` and
  accept with `--sase-update-visual-snapshots`. This is why the flag was worth having:
  goldens stay green through phases 1–8 and change exactly once.
- Add new pane snapshots: populated, empty, family-grouped, filter bar with completion,
  filter parse error, and narrow layout.
- Help-modal rows and pane description distinguishing the Agents tab from the Artifacts
  → Agent pane (§1.1).
- Capture j/k p95 with `SASE_TUI_PERF=1` at the full corpus and report it on the bead.
- Gate landing on monitored `just check-full`:

  ```bash
  sase monitor start --command 'just check-full' \
    --start-status TESTING --stop-status TESTED --next '...'
  ```

## 11. Verification

| Area        | Required evidence                                                                                                                                                                                                 |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Grammar     | Python/Rust parity on dotted, digit-leading, `--`, date, duration, and integer values; canonical round-trip stable; every currently-valid boolean query unchanged; the Patches dotted-`name:` round trip repaired |
| Contract    | `sase artifact pane show agents` reports the expected capabilities, relations, grouping, and profile; the Off state reproduces `stitches, patches, beads, ref:plan, ref:research, files` with Files on digit 6    |
| Conformance | all 12 harness checks pass for `agents` under an explicit flag-on fixture                                                                                                                                         |
| Dialect     | Rust/Python golden parity; boolean precedence with parens; both date-bound pairs; the duration pair; the `Nm` collision tested and documented                                                                     |
| Identity    | bare local, canonical global, owner-qualified, member, family, clan, collision-history, and name-only rows all resolve and copy durable references                                                                |
| Row model   | 0 ambiguous dismissed joins over all 12,525 names; ≥91% enrichment; thin rows render a "no run data" state; no `record_json` read                                                                                 |
| Scale       | 12k-row corpus; bounded option count; `showing N of M`; stale-worker rejection; j/k p95 < 16 ms                                                                                                                   |
| Detail      | lazy hydration and cancellation; selection generation prevents a late overwrite                                                                                                                                   |
| Revival     | single, marked batch, family banner, workflow children, partial failure, correct bundle/index ordering; both surfaces refresh; the `R` modal, saved groups, and main-tab selection are unchanged                  |
| Relations   | family/clan/retry/parent edges; inbound and outbound `agent:` navigation for all four ref spellings; a jump to an excluded row clears the query                                                                   |
| CLI         | `sase agent search` matches the pane's results for the same query and scope; `-h` is complete and alphabetically sorted; JSON schema is stable                                                                    |
| Visual      | populated, empty, family-grouped, filter-bar, parse-error, and narrow layouts; the sub-tab strip change is intentional and accepted once                                                                          |

Every phase runs `just install` first (ephemeral workspaces), then `just check`. The
`land` phase runs monitored `just check-full`.

## 12. What not to do

- **Do not build a third agent query dialect.** The reason `src/sase/ace/agent_query/`
  is worth deleting later is that it was built this way once already.
- **Do not ship `boolean=True` without the `grammar` phase.** Dates, durations,
  integers, and dotted agent names are unspellable until it lands (§3.2).
- **Do not add a `duration` value kind or general `<`/`>` operators** (§3.5).
- **Do not back the pane with the artifact index alone, the sidecar pages, or the live
  Agents tab list** (§2.1).
- **Do not join the dismissed archive without `is_workflow_child = 0`.** 27,033 rows,
  7,098 distinct suffixes.
- **Do not read `bundle_path` from the registry.** 99 of 12,525 entries have one.
- **Do not `SELECT *` the artifact index.** `record_json` is ~117 MB.
- **Do not offer `effort:`, `bead:`, `workspace:`, or `content:` in v1** (§3.4, §3.6).
- **Do not ship completeness-sensitive link filters** (§6.1).
- **Do not rewrite stored `agent:` link rows to canonical form here.** Match both forms
  on read.
- **Do not show hidden rows or workflow children by default.** 4,583 of 8,022 indexed
  rows are hidden and ~75% of archived bundles are workflow children. The query corpus
  includes everything; the default presentation scope excludes them, and navigating to
  an excluded target clears the query rather than failing.
- **Do not use the pure-Python profile evaluator as the production matcher.** It is
  parity-test-only by design.
- **Do not delete `DismissedAgentSelectModal` or the `R` revival panel in this epic**
  (§7.3).
- **Do not migrate the main Agents tab in this epic.**
- **Do not pick an accent that collides with the provider palette** (§4).
