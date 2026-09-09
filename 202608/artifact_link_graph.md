---
tier: epic
status: done
title: First-class artifact link graph
goal:
  "Every SASE artifact has a defined artifact markdown file and a typed, committed link
  graph. Agents write links with `sase artifact link` (required relation and
  description), read artifacts with `sase artifact read`, and see a beautiful
  GitHub-hyperlinked Links table plus Referenced By citations. Prompt refs, audited
  reads, and the RELATED: bead-note convention become first-class edges behind the
  `artifact_links` beta flag."
phases:
  - id: home
    title: Tracked sidecar home for link truth
    depends_on: []
    size: small
    description:
      "home: stop writing the Referenced By index under .sase/, put it in a tracked
      links/ tree, and start committing it."
  - id: core
    title: Link graph types and managed tables in sase-core
    depends_on: []
    size: medium
    description:
      "core: add the link-row wire types, relation registry, ManagedTableBlock
      primitive, Links table, companion paths, and bead link events."
  - id: store
    title: Python store, flag, and aggregate index
    depends_on:
      - home
      - core
    size: medium
    description:
      "store: create the artifact_links beta flag, the per-artifact truth adapter, the
      rebuildable aggregate index, and the v1-to-v2 migration."
  - id: cli
    title: sase artifact link and sase artifact read
    depends_on:
      - store
    size: medium
    description:
      "cli: ship sase artifact link add/list/rm and sase artifact read with doctor
      checks and retention protection."
  - id: render
    title: Rendered link tables, prompt-ref cites, and companions
    depends_on:
      - store
    size: medium
    description:
      "render: project the Links and Referenced By tables, write prompt-ref cites,
      create companion markdown files, and exclude them from inventory."
  - id: beads
    title: "Bead link events, pages, and RELATED: migration"
    depends_on:
      - store
    size: medium
    description:
      "beads: persist bead links in the event stream, render them on bead pages, and
      migrate RELATED: notes."
  - id: ace
    title: ACE relation source for the link graph
    depends_on:
      - store
    size: small
    description:
      "ace: add a links/linked_by relation source on every Artifacts pane without
      blocking the event loop."
  - id: adopt
    title: Glossary, docs, skills, and agent adoption
    depends_on:
      - cli
      - render
      - beads
      - ace
    size: medium
    description:
      "adopt: add the Artifact and Artifact Markdown File glossary terms, rewrite the
      skills and docs, and snapshot the relation registry."
proposed_by: bbugyi200.athena.08f
bead_id: sase-r8
create_time: 2026-09-09 19:49:53
---

- **PROMPT:**
  [prompts/202608/artifact_link_graph.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifact_link_graph.md)
- **BEAD:**
  [sase-r8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-r8/README.md)

# First-class artifact link graph

Inspiration and measurements come from
`research:202608/artifact_link_graph/artifact_link_graph.md` (consolidated 2026-08-18).
This plan is the design of record: where that report left a choice open, the decision is
here. Where this plan and the report disagree, this plan wins.

Do not mention ephemeral workspace directories in commits, comments, or follow-up plans.
Open every non-primary repo with `/sase_repo` before reading or writing it, including
`sase-core` and `sase-research-artifacts`.

## 1. Why

SASE can already _cite_ an artifact (`@plan:…`, `sase bead ref add`, `Referenced By`)
and _resolve_ one (`sase artifact path` / `show` / `open`). It cannot say **why two
artifacts are linked**, persist that as typed, bidirectional, GitHub-navigable state, or
remember that an agent _read_ something. The hole is precise:

| Surface                 | What it answers                         | Why it is not this                             |
| ----------------------- | --------------------------------------- | ---------------------------------------------- |
| Bead `refs`             | this bead cites that artifact           | no relation, no description                    |
| `RELATED:` notes        | this bead relates to that one, because… | free text, no schema, no inverse, no URL       |
| Plan header `ARTIFACTS` | SDD provenance for one plan             | closed section set; `header-invalid` validator |
| `Referenced By`         | who cited me, when, how often           | citation-shaped, bottom-anchored, no why       |
| ACE relation rail       | presentation over declared properties   | derives edges; stores none                     |
| `consumption.jsonl`     | citation counting                       | records the use, not the described edge        |

The product is one **artifact link graph**. Every artifact has a defined **artifact
markdown file**. Deliberate links render as a `## Links` table of GitHub hyperlinks near
the top of that file. Automatic citations stay in `## Referenced By` at the bottom.
Agents write with `sase artifact link` and read with `sase artifact read`.

## 2. Verified current state

Re-check at implementation time; these were true when this plan was authored.

- `sase-core` `0.29.2`, Python pin `sase-core-rs>=0.29.0,<0.30.0`. Additive wire only;
  stay inside the 0.29 window.
- `referenced_by.rs` (659 lines) is already column-generic (`ReferencedByColumnWire` +
  `BTreeMap` rows + `link_targets`). The only hardcoded pieces are three consts and
  bottom-concat in `upsert_referenced_by_block`. `MAX_RENDERED_REFERENCED_BY_ROWS = 50`.
- `plan/artifact_link.rs` owns the top-of-plan header block (schema v3) with a
  `header-invalid` validator. The Links table must sit **after** that header, **before**
  the first ATX heading.
- Structured Referenced By truth is written to
  `.sase/referenced-by/<provider>/<relpath>.json`
  (`src/sase/sdd/referenced_by_index.py`) and appended to `changed_paths`, but `.sase/`
  is in `.git/info/exclude` on every sidecar
  (`src/sase/workspace_provider/git_exclude.py`, patterns `.sase/` and `/sase/repos/`)
  and in the global gitignore. **The index has never been committed.** Four rendered
  `Referenced By` blocks exist in this project; `find` turns up zero committed index
  files. `commit-failed` fires only when _no_ commit is created, so the missing JSON is
  silent. This is a landed defect, in scope for `home`.
- One hash-strip call site: `src/sase/core/prompt_artifact_staging.py` calls
  `referenced_by_block_strip`. Extend it to both marker pairs. Do not put links in
  hashed frontmatter.
- Bead events already have `ReferenceAdded` / `ReferenceRemoved`
  (`BeadEventOperationWire`). Notes are append-only (`NoteAppended`). There is no
  note-delete. `IssueWire.refs` is the projected citation list.
- `HostedLinkResolver` (`src/sase/sdd/hosted_links.py`) already builds GitHub URLs and
  degrades to an unlinked label. Do not build a second resolver.
- ACE relation sources (`src/sase/ace/tui/relations/`) are pure snapshot→edge
  projections with no I/O. `tui_perf.md` rule 1 forbids synchronous disk I/O and JSON
  parsing on the event loop.
- `sase artifact` subcommands today:
  `create doctor list open pane path prune reclaim show stats trash`. Bare
  `sase artifact` defaults to `list`. Alphabetical insertion puts `link` between
  `doctor` and `list`, and `read` between `prune` and `reclaim`.
- `RELATED:` lives in `/sase_new_task` at two call sites
  (`src/sase/xprompts/skills/sase_new_task.md`). Corpus (re-measure at implementation):
  on the order of 260 notes / 150 beads, ~92% mechanical, ~5% manual worklist, ~290
  edges.
- Glossary has `Artifact Reference` only. There is no `Artifact` parent term.
- `links:` frontmatter already means mkdocs-material related-links in
  `docs/blog/posts/*.md`. It is not ours.
- Flag kinds are `beta` | `sunset` (no `wip`). Keys are snake_case. Create only with
  `sase flag new`.
- Bead `sase-ka` (v2 agent `ref-uses.json` publication) was closed canceled. Agent
  outbound links are in scope for this epic's render path, not a queued follow-up.
- Research provider inventory glob is year-prefixed `*.md`. Companion
  `foo_infographic.md` next to `foo_infographic.png` would become a listed research
  report unless excluded **before** the first companion is written.

## 3. Decisions a phase worker must not silently revert

1. **One graph, one row schema, three named layers.** Truth, read model, projection.
   Never parse Markdown back for state. Never keep a second citation ledger beside the
   graph.
2. **Truth is committed in the repo that owns the artifact, never under `.sase/`.** Do
   not narrow `git_exclude`. `.sase/` stays workspace scratch. Per-artifact JSON lives
   at `links/<artifact-relpath>.json` inside the owning sidecar (original extension
   preserved: `links/202608/report.md.json`). Beads are the exception: truth is the bead
   event stream. Agents land outbound rows next to existing publication metadata; their
   page is a projection.
3. **The read model is one rebuildable project-local aggregate**,
   `~/.sase/projects/<key>/artifact-links.json` (or a same-directory sibling if JSON
   needs an atomic replace). ACE and `sase artifact link list` read this, never scan
   sidecar JSON on the TUI thread.
4. **`links:` frontmatter is an authoring inlet, not truth.** The refresh pass ingests a
   list of `{ref, relation, description}` mappings, writes truth, and **deletes the
   key**. A `links:` value that does not match that shape (including mkdocs
   `- Label: path`) is left untouched. No path guard.
5. **Route rendered blocks by curation, not direction.** Top `## Links` shows
   `origin ∈ {manual, migrated}` in **both** directions, with the relation label taken
   from the current document's perspective (the registry inverse when this artifact is
   the target). Bottom `## Referenced By` shows `origin ∈ {prompt_ref, read}`. One
   pointer line between them when the automatic count is > 0:
   `_Plus N automatic references — see [Referenced By](#referenced-by)._` Empty tables
   omit the whole block.
6. **Relations are a closed registry, not an enum in call sites and not free text.** No
   free-slug escape hatch in v1. Snapshot to `sase/artifact_relations.json` (task-type
   snapshot shape), regenerated by `sase memory init`, rendered into `AGENTS.md`.
   `sase artifact link add` **errors** on `blocks` and `depends-on` with a pointer to
   `sase bead dep`. Do not add `duplicates`; that is `sase bead +1`.
7. **`sase artifact link add` always takes an explicit source ref.** Do not default
   source to the current agent. Agent-as-source edges come from `sase artifact read` and
   from prompt-ref expansion.
8. **Reasons are positionals, not required options.**
   `sase artifact read <ref> <reason>` and
   `sase artifact link add <source> <relation> <target> <why>`. Do not add a fifth
   required `-r`. Do not edit `sase/memory/cli_rules.md` in this epic.
9. **Lazy artifact markdown files.** Every kind has a defined path; the file is created
   on first link. Do not materialize 9k automatic PNG companions. `stitch:` is the
   documented exception (SASE does not own a commit; links render on the peer).
   Unpublished `file:` artifacts get a local page under
   `~/.sase/artifacts/pages/<id>.md`, never a sibling inside the content-addressed
   object store, and never a fabricated `main` GitHub URL.
10. **Companion naming uses the stem** (`diagram.png` → `diagram.md`). On collision with
    an existing first-class document, refuse and use `diagram.png.md`. Never overwrite a
    research report. Exclude companions from document-provider inventory **before**
    writing the first one.
11. **Binary companion pages embed the asset** when GitHub can display it
    (`![diagram.png](./diagram.png)`), then the Links table. That is the beautiful page,
    not an empty stub.
12. **Only `read` writes a `read` edge.** `show`, `path`, and `open` stay silent.
    `sase repo open` keeps modification and broad tree exploration.
13. **Ship behind `sase flag new artifact_links -k beta`.** Disabled: no new sidecar
    link rows, no bead link events, no prompt-ref cites, no Links tables, no companions.
    Parsers exist and say so. The `home`-phase v1 Referenced By index in `links/` keeps
    updating regardless of the flag (that is a bugfix, not the new feature). Test both
    flag states from `store` onward.
14. **Do not delete historical `RELATED:` notes.** Append
    `MIGRATED: linked as related/<id>` per converted row. The event store is
    append-only; `sase bead history --lost-notes` depends on the original text.
15. **Treat `agent:<name>` as an ordinary node from day one.** Prompt-ref and read edges
    accumulate so a future Agents sub-tab lights up with real data. Do not build that
    sub-tab in this epic.
16. **Existing `referenced_by_block_*` Python bindings remain working facades** over
    `ManagedTableBlock`. Additive sase-core APIs only.

## 4. The object

### 4.1 Row

```json
{
  "schema_version": 2,
  "source_ref": "research:202608/artifact_link_graph/artifact_link_graph.md",
  "relation": "implements",
  "target_ref": "bead:sase-js",
  "description": "extends the ref contract this epic landed",
  "origin": "manual",
  "created_by": "bbugyi200.athena.y2",
  "created_at": "2026-08-18T23:40:00Z",
  "uses": 1
}
```

- `origin ∈ {manual, migrated, prompt_ref, read, derived}`.
- Directed dedup key: `(source_ref, relation, target_ref)`. A rewrite updates
  `description` and leaves `created_at` / `created_by` stable; `uses` increments for
  `prompt_ref` / `read`.
- Undirected `related`: one stored edge; dedup key is `(relation, frozenset({a, b}))`.
  Adding B→A after A→B is idempotent (description updates). Never store both directions.
- Self-links rejected. The same pair may carry several _different_ relations.
- Canonicalize refs through the existing path, with or without a leading `@`. Historical
  aliases (`commit:` → `stitch:`, `plans:` → `plan:`) canonicalize on write. Bead ids
  store full ids.
- Description is required, non-empty after trim, single line, max 240 characters.
  Machine origins synthesize a bounded excerpt (prompt snippet, xprompt name, or bead
  title) — never the constant "Cited in launch prompt".

Per-artifact JSON holds **every row touching that artifact, both directions**, so one
read answers `sase artifact show` / the Markdown projection. The aggregate index is the
same rows flattened for the project.

v1 Referenced By JSON (`schema_version: 1`, `agent` / `canonical_ref` / `uses` /
`published` / …) remains readable. `store` migrates it in place to v2 `cites` /
`origin: prompt_ref` rows.

### 4.2 Relation registry (v1)

| Slug           | Inverse          | Directed | Written by                                  |
| -------------- | ---------------- | -------- | ------------------------------------------- |
| `cites`        | `cited-by`       | yes      | prompt-ref expansion (`origin: prompt_ref`) |
| `read`         | `read-by`        | yes      | `sase artifact read` (`origin: read`)       |
| `related`      | `related`        | no       | CLI; `RELATED:` migration                   |
| `supersedes`   | `superseded-by`  | yes      | CLI                                         |
| `implements`   | `implemented-by` | yes      | CLI                                         |
| `derives-from` | `derived-into`   | yes      | CLI                                         |

Reserved (error + pointer, not stored): `blocks`, `depends-on`.

Builtins live in sase-core. Python assembles the snapshot the way task types do
(builtins + future plugins + config). v1 ships builtins only; the assembly shape must
not paint plugins out.

### 4.3 Artifact markdown file

| Kind                                                           | Artifact md file                                          | Status                                |
| -------------------------------------------------------------- | --------------------------------------------------------- | ------------------------------------- |
| Markdown document (`plan:`, `research:`, other document kinds) | itself                                                    | exists                                |
| `bead:`                                                        | generated page (`pages/<lineage>/README.md` or `<id>.md`) | projection; truth is the event stream |
| `agent:`                                                       | generated agent/family page                               | projection                            |
| `patch:`                                                       | published Patch page                                      | exists                                |
| `file:` markdown                                               | the file itself when it is a real `.md`                   | exists                                |
| `file:` binary, published                                      | sibling `<stem>.md` in the owning repo                    | **new**                               |
| `file:` unpublished                                            | `~/.sase/artifacts/pages/<id>.md`                         | local only                            |
| `stitch:`                                                      | **none**                                                  | links render on the peer              |

`artifact_md_path(ref) → path` is a sase-core function. Create on first link.
Unpublished pages say so in the glossary, not with a fake URL.

### 4.4 Rendered `## Links` block

```markdown
<!-- sase:links:start -->

## Links

| Relation   | Artifact          | Why                                       |
| ---------- | ----------------- | ----------------------------------------- |
| implements | [bead:sase-js][1] | extends the ref contract this epic landed |
| related    | [bead:sase-ct][2] | shares the ACE-TUI flake root cause       |

_Plus 12 automatic references — see [Referenced By](#referenced-by)._

[1]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-js/README.md
[2]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-ct/README.md

<!-- sase:links:end -->
```

Placement: after YAML frontmatter, after the plan header block when one exists, before
the first ATX heading. Numbered link definitions are allocated through existing
`markdown_link_refs` so they never renumber the document's own refs. URLs from
`HostedLinkResolver`. Documents pin a blob SHA when known; bead/agent pages use the
hosted default branch (they are regenerated). Missing remotes degrade to an unlinked
`` `bead:sase-js` `` label, never a guessed `main`.

Sort: relation slug, then artifact ref. Cap at 50 with an `_… and N more_` line, same as
Referenced By. Relation cell is the slug agents type, from this document's perspective,
so the table teaches the vocabulary.

Binary companion skeleton:

```markdown
# diagram.png

![diagram.png](./diagram.png)

_Typed links for this image. This file is generated; do not hand-edit._

<!-- sase:links:start -->

…

<!-- sase:links:end -->
```

PDFs and other non-image binaries get a GitHub blob link instead of an embed. Agents
never hand-edit generated pages (bead, agent, companion).

`Referenced By` (flag on) uses the same row schema, columns
`Relation | Artifact | Why | Uses`, still bottom-anchored. Flag off keeps today's
citation columns on the v1 index.

### 4.5 CLI

```text
sase artifact link add  <source-ref> <relation> <target-ref> <why>
sase artifact link list [<ref>] [-d in|out|both] [-j] [-l N] [-o origin] [-R relation]
sase artifact link rm   <source-ref> <target-ref> [-R relation]
sase artifact link migrate-notes [--apply] [-j]

sase artifact read <ref> <reason> [-f json|markdown|rich] [-n LINES]
```

Bare `sase artifact link` delegates to `list` via the central default-list convention
and prints the usual notice. `list` without a ref shows the current project's recent
links (default limit 50, newest first). `list` with a ref shows that artifact's
neighborhood.

- `add` is idempotent: unchanged edges print `unchanged` and exit 0.
- `rm` without `-R` removes every edge between the pair and says so.
- Refs accept a leading `@`.
- Help is excellent, colored, alphabetically ordered (group
  `add / list / migrate-notes / rm`; parent insertion `link` after `doctor`, `read`
  after `prune`). Every public long option has a short alias. No required options;
  required values are positionals.
- Flag off: `link add` / `link rm` / `migrate-notes --apply` error with the flag name
  and how to enable it. `link list` still reads whatever exists. `read` still prints;
  stderr notes that the read was not recorded as a link.

`sase artifact read` order:

1. Require a non-empty reason (positional).
2. Resolve through the same path as `show` / `path` (every kind, every `#L` / `#page=` /
   `#t=` fragment, VCS-backed materialization).
3. Strip leading frontmatter **and both managed blocks** before printing.
4. Print. Markdown to stdout; page when a TTY; raw when piped. Images / PDFs / video
   print a metadata card plus a pointer to `sase artifact open` and still record the
   read. `stitch:` prints the `show` payload plus the commit subject.
5. Append an audited JSONL row (`~/.sase/projects/<key>/artifact_reads.jsonl`, same
   house pattern as `src/sase/repo_open_log.py`: frozen dataclass,
   `discover_agent_identity`, `locked_file(LOCK_EX)`, `schema_version`) **and** a
   consumption event so `show` and `list --unused` stay honest.
6. When the flag is on, write `from = current agent`, `to = <ref>`, `relation = read`,
   `description = <reason>`, `origin = read`. Outside an agent run, recording the link
   is skipped (the JSONL row still stores the human/user identity) unless `SASE_AGENT`
   is set.
7. Refuse to print if the audit row could not be recorded, matching
   `sase glossary read`.

`sase artifact show` (flag on) gains a Links section. `sase bead show` gains one once
`beads` lands. JSON envelopes include a stable `links` array.

`sase artifact doctor` (from `cli`) reports dangling rows, stale tables (rendered block
disagrees with truth), missing companions, and a rendered block whose `links/` JSON is
not in `HEAD`. `--fix` rebuilds projections from truth; it never parses Markdown for
state.

Retention: add the link store to `collect_protected_artifact_ids`
(`src/sase/core/artifact_file_protection.py`) the way consumed refs already are.
`reclaim` skips linked `file:` rows so IDs do not churn out from under the graph.

### 4.6 `RELATED:` migration

`sase artifact link migrate-notes` is dry-run by default.

1. Scan bead notes for `^RELATED:\s*<bead-id>(\s*,\s*<bead-id>)*\s*[—–]\s*<why>$`.
2. One `related` edge per named target, `origin: migrated`, shared rationale on fan-out.
3. Notes that name a commit become `stitch:<repo>@<sha>` when the sha is unambiguous;
   otherwise they go to the manual worklist.
4. Leave the original note. Append `MIGRATED: linked as related/<id>` per converted row.
5. Print a worklist for the ~5% that do not parse. Do not guess.

Then `/sase_new_task` emits `sase artifact link add` instead of `RELATED:` (see
`adopt`).

## 5. Rust / Python split

Per `rust_core_backend_boundary`. If a web app, CLI, or editor would need the behavior
to match the TUI, it is core.

**sase-core (`/sase_repo` open `sase-core`):**

- Link row + per-artifact index + aggregate-index wire types (`schema_version: 2`).
- Relation registry: slug, inverse, directed, reserved-slug errors, v1 builtins.
- `ManagedTableBlock { start_marker, end_marker, heading, anchor }` extracted from
  `referenced_by.rs`. `Referenced By` (bottom) and `Links` (top) are two instances.
  Existing `referenced_by_block_{parse,render,upsert,remove,strip}` remain byte-stable
  facades.
- Top-anchored placement function: after frontmatter, after plan header if present,
  before first ATX heading.
- `artifact_md_path`, companion naming, collision refusal.
- Tolerant frontmatter-`links:` reader (inlet schema only).
- `BeadEventOperationWire::{LinkAdded, LinkRemoved}` plus payloads
  `{ target_ref, relation, description, origin }` / `{ target_ref, relation }`, and a
  projected `links` field on `IssueWire` (`#[serde(default)]`).
- PyO3 exports for every new public function. Binding names stay snake_case and stable.

**sase (this repo):**

- CLI parsers/handlers (`parser_artifact.py`, new handler module).
- Per-artifact truth I/O, aggregate rebuild, flag, audited log.
- Outbox drain, sidecar commit, `artifact_links` file-hook cause (excluded by default,
  mirroring `referenced_by`).
- Bead mutation path, page renderer, `migrate-notes`.
- Prompt-publication `cites` writer.
- ACE relation source and pane-contract declarations.
- Skill templates, docs, glossary, research-plugin inventory exclusion.
- Retention protection.

Raise the `sase-core-rs` floor in the first sase phase that calls a new binding
(`store`), once the core release is in the 0.29 window. Do not open 0.30.

## 6. Feature flag

Created at the start of `store` with `sase flag new` (the one bead this epic is allowed
to create; do not use `sase bead create` or `/sase_new_task`):

```bash
sase flag new artifact_links -k beta \
  --when-enabled "Agents add typed artifact links, sase artifact read records a read edge, prompt refs write cites, and Markdown artifacts render Links and Referenced By tables from the unified graph." \
  --when-disabled "Link-writing commands report that artifact_links is disabled; no new sidecar link rows, bead link events, prompt-ref cites, Links tables, or companion files. Existing v1 Referenced By projections in links/ keep updating." \
  --remove-when "The link graph has been the default for a full minor release, doctor is clean on the sase sidecars, and no caller depends on the v1-only Referenced By schema."
```

Paste the printed registry entry; do not hand-add a `FeatureFlag` member. Tests cover
both states.

## 7. Phases

`home` and `core` are independent. `store` is the join. `cli`, `render`, `beads`, and
`ace` are independent once `store` lands. `adopt` is last.

### home

Move Referenced By structured truth off `.sase/`.

- Change `referenced_by_index_path` to `<sidecar>/links/<artifact-relpath>.json` with
  the original extension preserved. Drop the extra `referenced-by/<provider>/` prefix;
  the sidecar _is_ the provider.
- Keep writing **v1** JSON. Do not wait for the unified schema.
- `commit_sdd_store_files` already commits `changed_paths`; once the path is tracked,
  the JSON lands. Confirm with a test that the path is not matched by
  `SASE_GIT_INFO_EXCLUDE_PATTERNS`.
- Do **not** change `git_exclude.py`.
- Backfill the four existing rendered blocks by running the refresh (or a one-shot
  equivalent) so each cited document has a committed v1 JSON file.
- Doctor: a committed Markdown document with a `Referenced By` block and no `links/`
  JSON in `HEAD` is an error.
- Update `docs/artifact_references.md` Publication so it no longer claims
  `.sase/referenced-by/`.
- Tests: path helper, commit includes JSON, exclude still ignores `.sase/`, existing
  upsert goldens unchanged, doctor fires on a missing index.

This phase is a bugfix of landed code. No feature flag.

### core

Implement in the linked `sase-core` repo (open with `/sase_repo`).

- New module `artifact_link` (name bikeshed: do **not** steal `plan/artifact_link.rs`).
  Wire types from §4.1–4.3.
- Extract `ManagedTableBlock`. Re-express `Referenced By` through it. Golden tests:
  current `referenced_by` fixtures stay byte-identical. New fixtures for the
  top-anchored Links instance, including a plan document that already has a v3 header
  block (Links must not trip `header-invalid`).
- Relation registry with inverses, reserved slugs, undirected `related`.
- `artifact_md_path` + companion naming + collision refusal.
- Frontmatter inlet reader.
- Bead `LinkAdded` / `LinkRemoved` + projected `links`. Reducers ignore unknown
  historical streams; new ops round-trip. `#[serde(default)]` on `IssueWire.links`.
- Export through `sase_core_py`. Keep existing `referenced_by_block_*` names.
- `cargo test` in sase-core. No sase user-reaching behavior yet. A thin sase commit is
  allowed only to bump the binding floor / satisfy `tools/validate_sase_core_rs` if the
  release lands in-phase.

### store

- `sase flag new artifact_links` as specified in §6.
- Python adapter over the new core types. Read/write per-artifact JSON under `links/`.
  Atomic replace. Kind-native: beads are **not** written here (the adapter asks the bead
  store once `beads` lands; until then, skip `bead:` rows or keep them only in the
  aggregate as a cache of what the bead phase will own — do not create a second bead
  truth).
- Rebuildable aggregate at `~/.sase/projects/<key>/artifact-links.json`. Rebuild is a
  command the later `doctor --fix` calls, and a post-write hook from the adapter.
- Migrate v1 `links/*.json` to v2 `cites` / `origin: prompt_ref` rows. Keep a v1 reader
  until the migration has a test that round-trips the four live blocks.
- Audited-read JSONL helper (write path used by `cli`).
- Flag off: adapter refuses new v2 writes and reports disabled. v1 Referenced By refresh
  from `home` still runs.
- Both-states tests. No CLI yet beyond whatever doctor hook is needed to prove the
  adapter.

### cli

- Parsers in `src/sase/main/parser_artifact.py` (alphabetized, excellent `-h`, short
  aliases, RawDescriptionHelpFormatter examples). Nested `link` group with `list` so
  bare `sase artifact link` delegates.
- Handlers for `add` / `list` / `rm` / `read` against the store adapter. Colored tables.
  JSON `-j`. Idempotent `add`. Completions for relation slugs and artifact refs.
- `sase artifact show` Links section (flag on).
- Doctor checks from §4.5. Retention protection + reclaim skip.
- Flag-off behavior from §4.5, tested.
- `read` strips frontmatter and both managed blocks using the core strip helpers. TTY
  paging vs pipe. Non-text kinds. `stitch:` subject. Refuse to print when the audit log
  cannot be written.

### render

- Generalized refresh + outbox: `enqueue_referenced_by_request` becomes (or grows into)
  `enqueue_artifact_link_request` carrying `relation`, `origin`, and a synthesized
  description. Drain writes v2 truth (flag on) and upserts **both** blocks. Lock order
  remains **artifact repos first, `agents` last**.
- File-hook cause `artifact_links`, excluded by default, same as `referenced_by`.
- Extend `prompt_artifact_staging.py` strip to both marker pairs. A link must not change
  a document digest.
- Prompt publication (`agents_sync/prompt_archive/publish.py`) emits `cites` /
  `origin: prompt_ref` rows. Pointer kinds still create the edge. Idempotent across
  retries and `%repeat`.
- Companion creation on first link to a published binary, **after** inventory exclusion.
  Open `sase-research-artifacts` with `/sase_repo` and exclude companion files from the
  research inventory glob (stem siblings of a non-md asset, plus `*.png.md`
  disambiguation names). Builtin `plan` provider glob stays `**/*.md` but must not list
  companion pages if any plan-sidecar binaries appear.
- Agent pages render outbound links from publication metadata. Carry this plumbing; do
  not wait on canceled `sase-ka`.
- Flag off: no Links block, no companions, no `cites` rows. v1 Referenced By path from
  `home` continues.
- Tests: placement relative to plan headers, strip invariance, outbox idempotency,
  companion collision refusal, inventory exclusion, both flag states.

### beads

- Mutation path (lock → event → commit → page refresh) for `link add` / `rm` when either
  endpoint is a `bead:`. `sase bead show` displays the projected `links` field. Do not
  write link rows into generated pages as truth.
- Page renderer is the **only** writer of the bead-page Links / Referenced By tables.
- `sase artifact link migrate-notes` as specified in §4.6. Dry-run default; `--apply`
  writes events + `MIGRATED:` notes. Re-measure the corpus; do not hard-code the
  2026-08-18 counts into assertions.
- Flag off: mutation errors with the flag name; migrate-notes dry-run may still print
  what it _would_ do.
- Tests: event round-trip, page clobber (a second refresh does not drop links),
  undirected `related` idempotency, reserved-slug errors pointing at `sase bead dep`,
  migration worklist for unparseable notes.

### ace

- One new relation source in `src/sase/ace/tui/relations/` that reads the
  **already-loaded** aggregate snapshot (loaded off-thread, cached, mtime-keyed; never
  parse JSON in an action/render handler — `tui_perf.md` rules 1, 8, 9).
- Declare `links` / `linked_by` (`RelationKind.LINK`, inverse of each other,
  `directed=True` for the pair, `transitive=False`) on **every** pane contract in
  `src/sase/ace/tui/_artifact_tab_contract_adapters.py` and the provider compiler,
  including files / stitches / patches / beads / `ref:plan` / other document panes. This
  earns the relation rail, `<` / `>` / `~` navigation, and cross-pane reveal-lens jumps
  without building an Agents sub-tab.
- `source` for the declaration is a new snapshot property (`artifact_links`), not a disk
  path.
- Flag off: no edges from this source (decls may exist; the source returns empty).
- If the relation rail visual snapshot changes, update PNG goldens via
  `just test-visual` / `--sase-update-visual-snapshots` only for intentional diffs.
- Tests: pure snapshot→edge (no I/O in the source), inverse labels, undirected `related`
  shown once, both flag states.

### adopt

This phase is the user-requested terminology, skill, and documentation update. Use
`sase glossary add` (not a hand-edit of `sase/sase.yml`), then `sase memory init` so
`AGENTS.md` and provider shims regenerate. Skill **source templates** live in
`src/sase/xprompts/skills/` and are not memory files; edit those, commit, and do not
`sase skill init --force` from a dirty tree.

- Glossary terms (verbatim, then `sase memory init`):

  **Artifact**

  > A SASE artifact is any durable record addressable by an artifact reference: a
  > sidecar document such as a plan or a research report, a bead, a completed agent, a
  > Patch, a stitch, or an indexed file. Every artifact has a canonical
  > `<kind>:<argument>` identity and an artifact markdown file that carries its typed
  > links.

  **Artifact Markdown File** (aliases: `artifact md file`, `artifact md`)

  > An artifact markdown file is the Markdown document that carries one artifact's typed
  > links. A Markdown artifact is its own artifact md file. A non-Markdown file uses a
  > sibling `<stem>.md`; beads, agents, and Patches use their generated page, which is a
  > projection of the artifact's own store. A commit has none — links to a stitch render
  > on the other artifact. The file is created the first time the artifact acquires a
  > link. SASE renders those links as a table of hyperlinks near the top of the file.
  > Agents write links with `sase artifact link` and read artifacts with
  > `sase artifact read`; they never hand-edit a generated page.

  Update the `Artifact Reference` definition only if a single sentence of
  cross-reference is needed; do not rewrite it.

- Snapshot `sase/artifact_relations.json` via `sase memory init`. Render a short
  always-loaded `AGENTS.md` section listing the six slugs, their inverses, and the
  reserved `blocks` / `depends-on` pointer.
- Docs: new `docs/artifact_links.md`; update `docs/artifact_references.md`,
  `docs/cli.md`, `docs/beads.md` (retire the `RELATED:` recipe).
- Skills:
  - `/sase_artifact_file`: document `link` and `read`; prefer `read` over `path` + a
    silent file open when the agent is consuming an artifact as context.
  - `/sase_new_task`: replace both `RELATED:` recipes with
    `sase artifact link add <new> related <other> "<why>"`.
  - `/sase_repo`: one paragraph distinguishing `sase repo open` (modify or explore a
    repo tree) from `sase artifact read` (one artifact, recorded).
- Tests for skill-source examples (`test_init_skills_source_content.py` currently
  asserts the `RELATED:` string).

## 8. Hazards

| #   | Hazard                                                | Mitigation                                                                         |
| --- | ----------------------------------------------------- | ---------------------------------------------------------------------------------- |
| 1   | Index silently uncommitted                            | `home` moves off `.sase/`; doctor fails when a block exists without JSON in `HEAD` |
| 2   | Hash feedback loop                                    | Strip both marker pairs; never store links in hashed frontmatter                   |
| 3   | Dangling links after prune / reclaim ID change        | Protect linked ids; reclaim skips linked rows                                      |
| 4   | File explosion from automatic PNGs                    | Lazy companions only                                                               |
| 5   | Two scheduling truths                                 | Reserved slugs error to `sase bead dep`                                            |
| 6   | Generated-page clobber                                | Page renderer is the only table writer; bead truth is events                       |
| 7   | Inventory pollution by companions                     | Exclude before first write                                                         |
| 8   | File hooks on projection commits                      | Cause `artifact_links`, excluded by default                                        |
| 9   | TUI stall                                             | Aggregate index, off-thread load                                                   |
| 10  | Outbox delay vs GitHub                                | Already true of Referenced By; document it                                         |
| 11  | Unflagged user-reaching behavior                      | `artifact_links` beta from `store`                                                 |
| 12  | Link spam once `read` is adopted                      | Curation routing, 50-row cap, dedup                                                |
| 13  | mkdocs `links:` collision                             | Inlet accepted only on `{ref, relation, description}` shape                        |
| 14  | 3-arg `link add` footgun (agent as accidental source) | Source ref is always required                                                      |
| 15  | Companion overwrites a research report                | Stem collision refuses and uses `stem.png.md`                                      |
| 16  | sase-core 0.30 break                                  | Additive APIs, keep `referenced_by_block_*` facades                                |

## 9. Non-goals

- The Agents sub-tab (do not preclude it).
- Generated `stitch:` pages.
- Replacing bead dependencies, plan-header `ARTIFACTS`, or citation counting.
- Deleting historical `RELATED:` notes.
- Stored inverse edges (inverse is computed from the registry).
- Narrowing `git_exclude` or committing `.sase/`.
- A free-slug relation escape hatch.
- Editing `sase/memory/cli_rules.md` or converting `memory read` / `glossary read` to
  positionals (propose as land-agent follow-up if desired).
- Flag removal (owned by the flag bead `sase flag new` creates).

## 10. Verification

Each phase runs `just install` then `just check` in this repo after its sase edits.
`core` runs `cargo test` in sase-core. `ace` additionally runs visual snapshots if rail
rendering changes. `just check-full` is the land agent's job via `/sase_monitor`, not an
inline phase command.

`tools/select_tests --explain` if a scoped run looks wrong. Touching the broadening set
(artifact CLI, bead events, ACE contracts, sase-core bindings) is a reason for the land
agent to prefer `check-full`.
