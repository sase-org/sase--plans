---
tier: epic
title: Artifact reference contract
goal: 'Artifact references stop being xprompts and become a first-class, versioned
  ref contract: five builtin kinds (`@stitch`, `@patch`, `@bead`, `@agent`, `@file`)
  plus artifact-repo document kinds (`@plan`, `@research`, ...) configured inline
  or with `use: <provider>` from an installed plugin. Every ref expands deterministically,
  is recorded per occurrence against the agent that used it, publishes as a numbered
  Markdown reference link, writes a `Referenced By` table back into the cited artifact,
  and gets a generated "Artifacts" sub-tab in ACE.

  '
phases:
- id: core
  title: Ref contract wire types in sase-core
  depends_on: []
  size: large
  description: 'core: define the versioned provider/entry/resolution/use wire types,
    the closed expansion formatter, quoted-argument grammar, the `@stitch`/`@patch`/path-`@file`
    kinds with permanent legacy aliases, the shared numeric Markdown-link allocator,
    and the `Referenced By` footer block; release the binding.'
- id: retire
  title: Retire the ref xprompt surface
  depends_on: []
  size: medium
  description: 'retire: delete the `#ref/<kind>` xprompt adapter, its packaged renderer
    bodies, the synthetic source schemes, the precedence table, and the catalog, completion,
    LSP, and docs surfaces built on it, falling builtins back to hardcoded rendering.'
- id: registry
  title: Provider registry, plugin hooks, and config
  depends_on:
  - core
  - retire
  size: large
  description: 'registry: add the `sase_artifact` pluggy project with ref-provider
    and file-hook provider hookspecs, the spec registry with `use:`/inline merge and
    validation, the config schema deltas, the builtin `plan` provider, the `sase init`
    writer, and fail-soft diagnostics.'
- id: builtins
  title: Builtin refs and prompt ref context
  depends_on:
  - registry
  size: large
  description: 'builtins: thread an explicit per-segment `PromptRefContext` through
    late prompt processing, implement `@stitch`, `@patch`, `@bead`, and `@agent` resolution
    on it, record one immutable use row per ref occurrence, and land the legacy parse
    aliases.'
- id: files
  title: The @file ref and the content-addressed store
  depends_on:
  - registry
  size: large
  description: 'files: add the `artifact_refs.file.roots` allow-list, launch-time
    capture with a single byte read and full SHA-256, the one-object-per-digest store
    in the agents sidecar, and durable logical-path/version indexing at publication.'
- id: linking
  title: Reference links and Referenced By write-back
  depends_on:
  - builtins
  - files
  size: large
  description: 'linking: rewrite published prompts to numbered `[@kind:arg][N]` reference
    links with revision-pinned destinations, and reconcile a managed `Referenced By`
    table plus structured index into each cited artifact repo through the publication
    outbox.'
- id: ace
  title: Generated Artifacts sub-tabs and the new Files pane
  depends_on:
  - builtins
  - files
  size: large
  description: 'ace: make the Artifacts sub-tab set dynamic, collapse the plans/chats/files
    panes into one provider-driven documents pane, and rebuild Files as one row per
    logical file with version toggling and a visible origin badge.'
- id: research
  title: The sase-research plugin repository
  depends_on:
  - registry
  size: large
  description: 'research: build `sase-org/sase-research` as an installable plugin
    owning the `research` ref provider, the `research-highlights` file-hook provider,
    and the `#research*` xprompts, with wheel-level CI, tests, and documentation.'
- id: adopt
  title: Adoption, glossary, and documentation
  depends_on:
  - linking
  - ace
  - research
  size: medium
  description: 'adopt: link and install the research plugin, move Bryan''s config
    to `use:` plus overrides, add the `@file` roots for `~/bob`, add the Artifact
    Reference glossary term, and rewrite the affected documentation end to end.'
proposed_by: bbugyi200.athena.y2
create_time: 2026-08-11 13:20:32
status: wip
bead_id: sase-js
---

- **PROMPT:** [prompts/202608/artifact_ref_contract.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifact_ref_contract.md)
- **BEAD:** [sase-js](https://github.com/sase-org/sase--beads/blob/main/pages/sase-js/README.md)

# Plan: Artifact reference contract

## 1. Why

Epic `sase-ho` modeled artifact references as contextual `#ref/<kind>` xprompts. That
was a category mistake. An xprompt is a prompt fragment; a ref is a typed resolver with
an inventory, a property schema, a prompt projection, a publication target, and a usage
event. Rendering — the only part the xprompt framing could express — is the smallest
part of the contract. The framing forced synthetic source schemes
(`sidecar_ref_config:`, `generated_sidecar_ref:`), a seven-level renderer precedence
table, read-only special cases in the xprompt browser, and a `#ref/<kind>` invocation
path that bypasses usage tracking entirely.

This epic replaces that adapter with an explicit **ref contract**, fulfilled by builtin
refs, the special `@file` ref, and artifact-repo configurations that can be authored
inline or pulled from an installed provider plugin with one `use:` line.

**Do not `git revert` `sase-ho`.** It shipped three separable layers and only the
xprompt adapter is wrong. The path-filter contract (`artifact_ref/filter.rs`,
`filter_artifact_ref_path_payloads`, the `filtered` resolution status) and the document-
root plumbing (`ArtifactRefDocumentRootWire`, `ArtifactRefContextWire`,
`effective_sidecar_ref_policies`) are precisely the primitives the new contract needs.
Reverting the Rust would also be a wire-schema downgrade against an already-released,
already-installed binding that `tools/validate_sase_core_rs` gates. The `retire` phase
does a surgical deletion instead.

## 2. Verified current state

Read at the revisions in this workspace. Where the
`202608/ref_provider_contract/ref_provider_contract.md` research report is stale, this
section wins.

**The eight refs today.** `sase xprompt list` reports exactly eight ref-kind entries:
`#ref/agent`, `#ref/bead`, `#ref/bug`, `#ref/chat`, `#ref/commit`, `#ref/file` from
`src/sase/xprompts/refs/*.md`, plus `#ref/plans` and `#ref/research` from synthetic
`generated_sidecar_ref:<role>` sources, both carrying the default `**/*.md` globs. The
authored plans ref is `@plans`, plural, because
`sidecar_ref_config._sidecar_role_ref_kind` defaults the ref kind to the sidecar _role_.

**ACE has already moved on, and the research report describes an older tree.** Today
`src/sase/ace/tui/artifact_tabs.py` declares
`ArtifactsSubTab = Literal["patches", "stitches", "beads", "files"]` with
`ARTIFACTS_SUBTAB_ORDER = ("stitches", "patches", "beads", "files")`,
`FilesSubTab = Literal["plans", "chats", "other"]`, and
`LEGACY_ARTIFACTS_SUBTABS = {"prs": "patches", "bugs": "beads"}`. There is no Bugs tab
and no PRs tab to preserve — `sase-jd` already merged them. Plan against this tree, not
the report's `["prs","stitches","bugs","beads","files"]`.

**Numbering primitives exist but are incomplete.**
`commit_footer.rs::allocate_numeric_id` really does take the first available positive
integer, and `assign_reference_id` reuses one label per distinct destination. But
`reference_ids_in_body` scans reference _definitions_ (`[n]: dest`) only, never numeric
_uses_ (`[text][n]`), and has no fenced- code awareness. Both are fine for commit
messages and unsafe for prompt documents.

**The scanner has no quoting.** `artifact_ref/scanner.rs` runs a candidate from `@` to
whitespace with a trailing-punctuation trim, and `trim_candidate_end` already special-
cases the two-colon `file:` payload. Any argument containing a space — a Patch name, a
path — is currently unrepresentable. Quoting is new grammar work and a prerequisite for
`@patch` and real-world `@file` paths.

**The artifact pool duplicates content two ways.** `artifact_pool_filename` is
`{12-hex-prefix}-{basename}`, so identical bytes under different basenames get different
filenames; and `prompt_archive/paths.py::relative_artifact_link` publishes to
`artifacts/<yyyymm>/<name>`, so identical bytes cited in different months land in
different directories. "Exactly one location per unique contents" needs a real full-
SHA-256 store, not a reshuffle.

**Reusable plumbing.** `sdd/plan_links_refresh.py` is already a locked, batched, cross-
repo section reconciler. `sdd/plan_header_block.py` + `sections.rs` are already a
Rust-owned, idempotent, capped projected section. `agents_sync/publication_outbox*.py`
already defers and quarantines publication work. `core/prompt_artifact_staging.py`
already stages with sha256, MIME, a size cap, and `fcntl` JSONL.
`prompt_artifact.rs::rewrite_prompt_artifact_links` already rewrites idempotently while
protecting literal zones. `main/_repo_init_config.py` already makes surgical `ruamel`
writes to a project's `sase/sase.yml`. `config/file_hooks.py` is the fail-soft-per-entry
parsing model to copy.

**The research plugin repo is greenfield.** `sase-org/sase-research` has one commit, a
0-byte README, and the GitHub description `sase--research Artifact Repo Plugin` — which
is itself the ambiguity this epic has to fix.

**The research xprompts are mostly not files.** `#research`, `#research/image`,
`#research/more`, and `#research/prompt` live in the `xprompts:` YAML block of chezmoi's
`home/dot_config/sase/sase.yml`. Only `#research_swarm` is a file
(`home/sase/xprompts/research_swarm.md`), and it depends on the `research_a` /
`research_b` / `research_lead` model aliases, the `researchers` bucket, and the
`research` tribe display config, all of which also live in that chezmoi config.
`#old_research_swarm` is a legacy alias and is dropped, not ported.

## 3. Design

### 3.1 What a ref is

A ref is a typed citation in an agent prompt:

```text
@<kind>:<argument>
@<kind>:"<argument containing spaces>"
```

Every ref kind configures exactly four behaviors:

| Behavior        | Question it answers                                                            |
| --------------- | ------------------------------------------------------------------------------ |
| **Expansion**   | What text replaces `@<kind>:<arg>` in the launch prompt?                       |
| **Inventory**   | Which files or entries exist, for completion, validation, and ACE?             |
| **Properties**  | Which typed fields are parsed, for filtering and detail rendering?             |
| **Publication** | What does the published `[@kind:arg][N]` link point at, and does it back-link? |

Three principles keep this intuitive:

1. **One filter vocabulary.** `filter_artifact_ref_path_payloads` semantics — normalized
   repo-relative POSIX, case-sensitive, `**/` matches zero directories, OR-ed positives,
   `!` vetoes, a negative-only list starts allow-all — govern sidecar inventories,
   `@file` roots, and `file_hooks` alike. One language everywhere beats per-feature
   ergonomics.
2. **`use:` and inline are the same thing.** `use: <provider>` means "start from this
   registered spec." Both spellings normalize to one spec and there is a golden parity
   test. There is no privileged builtin path: the in-tree `plan` provider registers on
   the same group as third-party providers.
3. **Declarative specs, never per-use callbacks.** Plugins return immutable specs once,
   at registry assembly. The shared core executes a small closed set of strategies.
   Completion and the ACE inventory call resolution at high frequency; running
   third-party Python there would make latency and failure modes unbounded, would make
   config validation impossible without importing plugin code, and would guarantee that
   native consumers (the LSP, any future frontend) diverge. A provider that genuinely
   needs new behavior motivates a new _declared strategy_ in core — a reviewable, shared
   change.

### 3.2 The kinds

| Kind        | Argument                                                | Strategy             |
| ----------- | ------------------------------------------------------- | -------------------- |
| `@stitch`   | `<short_hash>` or `<repo>@<short_hash>`                 | builtin entry        |
| `@patch`    | Patch name                                              | builtin entry        |
| `@bead`     | `<short_id>` or `<full_bead_id>`                        | builtin entry        |
| `@agent`    | agent name                                              | builtin entry        |
| `@file`     | allow-listed local path, or `explicit\|default:<hex24>` | special local-file   |
| `@plan`     | repo-relative path in the plans artifact repo           | declarative document |
| `@research` | repo-relative path in the research artifact repo        | declarative document |
| _(custom)_  | repo-relative path                                      | declarative document |

**Removed from live authoring:** `@chat` (an `@agent` ref already covers that agent's
chat transcript alongside everything else useful about the agent) and `@bug` (every bug
now has a bead, and the Bugs sub-tab was merged into Beads by `sase-jd`). Both remain
**parse-only historical readers** — their strings are persisted in bead `refs:` lists,
`artifact_consumption.jsonl`, prompt manifests, and Patch files, so archives must keep
rendering. They are absent from completion, validation of new prompts, and the ACE
inventory.

**Renames.** `commit:` becomes a **permanent** parse alias canonicalizing to `stitch:` —
never a hard rename, for the same persistence reason. `plans:` becomes a deprecated
parse alias for `plan:` with an actionable diagnostic for one release; completion offers
only `@plan` from day one.

**`@file` keeps both payload shapes.** A payload whose first segment is `explicit` or
`default` is a digest reference (what `sase artifact create` emits today); anything else
is a path. `scanner.rs` already special-cases the two-colon `file:` payload, so
disambiguation is total. Splitting off a separate `@artifact` kind would break every
persisted `file:` string for no user-visible gain.

**`@file:<path>` is not the same as `@<path>`.** The bare `@/abs/path` form handled by
`src/sase/file_references.py` stays exactly as it is: an untracked convenience that
inlines or attaches a file. `@file:<path>` is the tracked form — allow-listed, captured,
versioned, published, and visible in the Files sub-tab. Say so in the docs; users will
otherwise assume they are aliases.

### 3.3 The runtime contract

Rust owns it, per `CLAUDE.md`'s Rust core backend boundary: grammar, resolution,
filtering, link numbering, and footer rendering are all cross-frontend behavior.
Provider discovery, config merge, pluggy dispatch, and ACE presentation stay in Python.

```text
descriptor()              stable kind, labels, grammar, capabilities, schema version
inventory(context)        files or generic entries for completion and ACE
resolve(argument, ctx)    one normalized entry, or a structured diagnostic
prompt_text(resolved)     the text substituted into the launch prompt
properties(resolved)      typed values for filtering and detail rendering
publication_target(use)   durable link target for the captured version
```

Four strategy families implement it, with no per-provider dynamic dispatch: builtin
entries, the special local-file provider, declarative artifact-repo documents, and
parse-only historical readers.

Normalized results:

```text
ArtifactEntry
  stable_id           provider-scoped logical identity
  ref_kind            stitch | patch | bead | agent | file | plan | research | ...
  canonical_argument, display_label, project_display_name
  repository, repo_relative_path
  captured_revision   full VCS commit, when applicable
  captured_digest     full SHA-256, when bytes were observed
  logical_path        portable configured-root identity (for @file)
  properties          typed map constrained by the provider schema
  origin              prompt_ref | agent_artifact | both

ResolvedArtifactRef
  raw_ref, canonical_ref, occurrence_span
  entry, prompt_text, publication_target, captured_file, diagnostics
```

`stable_id` is _logical_ identity; `captured_revision` / `captured_digest` is _version_
identity. That split is what lets the Files pane show one row for `~/bob/gtd.md` while
exposing every captured version of it.

### 3.4 The declarative spec

```yaml
schema_version: 1
provider: research
ref:
  kind: research
  display_name: Research
  tab_label: Research
  description: Durable research reports and generated media
  argument: { type: repo_path, quoting: shell }
  expansion:
    format: "the {checkout_path} file in the {sidecar_role} artifact repo"
  inventory:
    path_globs: ["20*/**/*.md"]
  identity: { property: id, fallback: repo_path }
  properties:
    source: markdown_frontmatter
    fields:
      create_time: { type: datetime, label: Created }
      updated_time: { type: datetime, label: Updated }
      status: { type: string, label: Status, facet: true }
      tags: { type: string_list, label: Tags }
  detail:
    title: title
    fields: [status, create_time, updated_time, tags]
    body: markdown
  publication:
    link: vcs_permalink
    referenced_by: markdown_table
```

`expansion.format` is a **small closed formatter**, not Jinja and not an xprompt. Its
valid placeholders are declared by the contract and validated in Rust; it cannot run
commands, expand directives recursively, or touch the filesystem. Property types stay
modest — `string`, `enum`, `boolean`, `integer`, `number`, `date`, `datetime`,
`string_list`. Unknown frontmatter is preserved in raw detail data but is not
filterable; otherwise every arbitrary YAML value becomes a TUI query language.

Presentation entries are **hints, not widgets**. A generic pane renders `display_name`,
a title property, ordered detail fields, and a Markdown body. Provider packages must
never ship Python TUI classes.

For entry-backed builtins, `properties.source` is `provider` and the builtin supplies
values from the domain model: **stitch** — repo, sha, subject, author, date, Patch,
stitch number; **patch** — project, status, parent, PR, mentors, stitch count; **bead**
— project, id, title, type, tier, status, priority, size, parent, assignee; **agent** —
project, lane, clan, tribe, model, state, started, finished. That is what makes
filtering uniform across path-backed and entry-backed refs without frontmatter existing
everywhere.

### 3.5 Config surface

```yaml
# Provider-backed — one field
repos: { sidecar: { custom: { research: { ref: { use: research } } } } }

# Provider-backed with a field-level override
repos:
  sidecar:
    custom:
      research:
        ref:
          use: research
          inventory: { path_globs: ["20*/**/*.md", "!20*/scratch/**"] }

# Fully inline, no plugin installed — the §3.4 block verbatim under `ref:`
```

Merge rules: scalars replace, mappings deep-merge, **lists replace rather than
concatenate**. `schema_version`, provider identity, and strategy type cannot be
overridden. Diagnostics name the supplying distribution and version plus the config
layer for each override — that is what makes the retired seven-level precedence table
unnecessary.

The same machinery serves file hooks:

```yaml
file_hooks:
  - use: research-highlights
    command: bob highlights create --include-id
```

The `research-highlights` provider supplies name, description, `sidecars: [research]`,
path globs, `agent_name_globs`, `ops: [ADD]`, and `timeout: 120s`, and marks `command`
as a **required user override** — the policy is portable, the executable is local.

And `@file` roots:

```yaml
artifact_refs:
  file:
    roots:
      - name: bob
        path: ~/bob
        path_globs: ["**/*.md"]
```

The **named root** gives a portable logical identity (`bob:gtd.md`) while the UI
displays the friendly `~/bob/gtd.md`. File-type filtering falls out of `path_globs` for
free, and multiple roots express different directory and file-type policies.

**Fail soft, always.** An unresolvable `use:` warns and degrades — never raises on the
launch path — exactly as `config/file_hooks.py` already does per entry. Pair that with a
hard `sase doctor` finding that names the config path, the missing provider, and the
install command. A project whose `sase.yml` names an uninstalled provider must still
launch agents.

### 3.6 Project context comes from the prompt, never from `cwd`

`@stitch:<short_hash>`, short `@bead`, and `@patch` all need to know which project is in
context. That answer comes from the VCS xprompt workflow in the prompt segment
(`#git:<name>` / `#gh:<org>/<repo>`), promoted into an explicit `PromptRefContext`:

```text
raw prompt segment
  -> identify the #git/#gh VCS workflow and project
  -> expand ordinary xprompts / workflow steps
  -> assemble the provider registry + PromptRefContext
  -> scan, resolve, capture, expand refs
  -> launch the agent
```

The context carries the project key and display name, the primary repo, every resolvable
SASE repo, the Patch store, the bead stores, the enabled artifact-repo providers, the
`@file` roots, and the VCS hosted-link capabilities.

Inferring from `cwd` happens to work for a simple single-segment launch and is wrong
everywhere else: xprompt swarms and `---`-separated multi-agent prompts can target
different projects in one process, and `cwd` is unavailable in home mode, in validation,
and in editor completion. When a segment has no VCS project and a short form needs one,
the error asks for a qualified form or a VCS workflow — it never searches whichever
workspace happens to be current.

### 3.7 Capture, publication, linking

**One immutable use row per occurrence**, in a per-agent `ref-uses` manifest:

```json
{
  "schema_version": 1,
  "use_id": "...",
  "agent": "research.09.cld",
  "project": "sase",
  "provider": "research",
  "raw_ref": "@research:202608/x.md",
  "canonical_ref": "research:202608/x.md",
  "span": { "start": 120, "end": 156 },
  "entry_id": "research:202608/x.md",
  "logical_path": null,
  "captured_revision": "<full sha>",
  "captured_digest": "<sha256>",
  "origin": "prompt_ref",
  "properties": { "status": "research" },
  "captured_at": "2026-08-11T...Z"
}
```

The existing global `artifact_consumption.jsonl` index stays as a derived query
accelerator, not the source of truth. Repeated refs in one prompt produce repeated rows
but share one captured object and one back-reference row carrying an occurrence count.

**The revision is pinned at resolution time, never at publication.** A `main`-branch
link rots as soon as the artifact is edited, and the whole point of the citation is to
name what the agent actually read. For a clean tracked file, capture both revision and
content digest. For a **dirty or untracked** artifact file, do not manufacture a
permalink to HEAD: snapshot the exact bytes into the agents object store, record the
provenance as local, and surface that no artifact-repo permalink exists.

**A true content-addressed store**, in the agents sidecar, killing both duplication
vectors from §2:

```text
files/objects/sha256/ab/abcdef...<64 hex>      # one byte sequence -> exactly one path
agents/<agent>/ref-uses.json
```

Write atomically to a temp file, verify the digest, rename into place only if absent; an
existing path at that digest requires byte verification, never a blind overwrite.
Metadata and logical names (MIME, original basename, labels) live in manifests, not in
the path. One store serves both `@file` captures and `sase artifact create` outputs;
they differ in provenance records, not in byte storage. Existing artifact ids become
compatibility aliases to the new object/version records.

**Reference-style links** in the published prompt document:

```markdown
Read [@research:202608/x.md][2] and [@file:~/bob/gtd.md][4].

[2]: https://github.com/sase-org/sase--research/blob/<captured-sha>/202608/x.md
[4]: ../../files/objects/sha256/ab/abcdef...
```

| Ref                               | Destination                                                           |
| --------------------------------- | --------------------------------------------------------------------- |
| `@stitch`                         | hosted commit URL for the captured full SHA                           |
| `@patch`                          | PR URL if present; else the published Patch/agent page; else unlinked |
| `@bead`                           | hosted bead page (`bead_links.py` already resolves these)             |
| `@agent`                          | agent page (`agent_lanes.lane_page_path`)                             |
| clean artifact-repo ref           | `blob/<captured-full-SHA>/<repo-relative-path>`                       |
| dirty/untracked artifact-repo ref | agents-sidecar SHA-256 snapshot                                       |
| `@file` (both payloads)           | relative link to the agents-sidecar SHA-256 object                    |

**Tracking must never depend on linkability.** An unlinkable ref still gets a use row,
an `ARTIFACTS` entry, and a code span in the published prompt.

The allocator picks **the lowest positive integer not used by a different link or
definition in that document**, reusing an existing label whose destination already
matches. A relative link from the prompt to the digest object (as
`relative_artifact_link` already does) avoids a commit-hash circular dependency and
resolves both at HEAD and when browsing the prompt at an older commit.

### 3.8 `Referenced By`

At the **bottom** of a cited artifact file, only when citations exist, inside managed
markers:

```markdown
<!-- sase:referenced-by:start -->

## Referenced By

| Agent                | Project | Reference               | Published  | Uses |
| -------------------- | ------- | ----------------------- | ---------- | ---: |
| [research.09.cld][7] | sase    | `@research:202608/x.md` | 2026-08-11 |    2 |

<!-- sase:referenced-by:end -->
```

Columns come from the provider spec; rows from a default row builder. Reuse the plan-
header block's proven properties — Rust-owned rendering, deterministic sort, a hard cap
(50) with an `omitted: N` line, full idempotency — as a new _footer_ block module, since
`plan_header_block` is anchored to the top. Back the table with a machine-readable index
in the artifact repo (`.sase/referenced-by/<artifact-id>.json`) holding exact use ids,
destinations, and timestamps, so SASE never reverse-engineers its state from Markdown.
The table is a projection.

Two invariants that are easy to miss:

- **Back-reference metadata must not redefine an artifact's semantic version.** Content
  digests and change detection must ignore the managed block, or every citation creates
  a "new version" whose only change is being cited, and citations form a feedback loop.
- **Back-reference commits must not run ordinary user `file_hooks`.** Add a
  `system_projection: referenced_by` cause and exclude it by default. Bryan's hook
  happens to filter `ops: [ADD]` while back-references are modifications, but relying on
  that incidental filter would make the generic feature unsafe for the next hook someone
  writes.

`Referenced By` is a _different relation_ from the plans sidecar's existing `AGENTS`
section: agents that _worked on_ a plan versus agents that _cited_ it. Keep both,
document the distinction, and never let a `@plan:` citation add an `AGENTS` entry.

Write-back generalizes `refresh_plan_links`, which already resolves the sidecar root
from the `SddStore`, takes `store_git_write_lock(..., mutates_worktree=True)`,
re-renders each document, and writes one batched commit with a structured report. Route
it through `publication_outbox*` rather than calling it inline: a locked, offline, or
contended artifact repo must never fail the agents-sidecar publication that is the
actual deliverable. Fix a **lock order — artifact repos first, `agents` last** — so a
publication writing back to two artifact repos can never deadlock against a concurrent
`sase plan links --write`. One commit per affected artifact repo per agent publication,
batching all of that agent's refs. There is no cross-repo transaction and no need to
simulate one.

### 3.9 ACE "Artifacts" tab

Target order, all dynamic pieces resolved from **configuration of enabled projects**,
not from what happens to be installed:

```text
Stitches | Patches | Beads | Plans | Research | ... | Files
```

`Plans` appears only if some enabled project configures the plans artifact-repo ref;
`Research` only if some enabled project sets `ref.use: research`.

- `ArtifactsSubTab` becomes `str` plus a runtime registry; `artifact_tabs.py` grows
  `resolve_artifacts_subtabs()` cached on `current_config_token()`, mirroring
  `get_all_workflow_metadata()` + `reset_workflow_metadata_caches()`.
- Selection and persistence use **stable ids, not display names** (`ref:plan`,
  `ref:research`).
- Number keys stay `1..N` for the fixed tabs; provider tabs take the numbers after them;
  `[` / `]` (`cycle_artifacts_subtab`) is promoted to the primary affordance in the help
  modal. Unstable number keys would be a worse regression than no number keys.
- `current_files_subtab` disappears. A persisted `current_artifacts_subtab` naming a
  provider that is no longer configured falls back to `DEFAULT_ARTIFACTS_SUBTAB` rather
  than erroring. `LEGACY_ARTIFACTS_SUBTABS` keeps mapping `prs`/`bugs`.
- `cycle_files_subtab` / `_reverse` (`(` / `)`) free up in `default_config.yml` and are
  repurposed for version toggling on the new Files tab.

**Collapse three parallel pane implementations into one.** `ace/tui/widgets/artifacts/`
currently holds 13 `plans_*`, 9 `files_*`, and 8 `chats_*` modules — three copies of
list + filter-bar + filter-session + detail + rendering + navigation. Collapse `plans_*`
into one `ArtifactsDocumentsPane(provider)` driven by the spec's `properties` (filter
tokens) and `detail` block (label, accent, grouping). `plan` becomes its first instance;
`research` then costs **zero new pane code**. Delete `chats_*` and the old `files_*`.
`plans_filtering.py::_PlanFilterRecord` is already a generic (project, status labels,
tier labels, kind labels, timestamp, haystack) record — it becomes
`(project, {property: labels}, timestamp, haystack)` driven by the declared schema. Net
ACE LOC should _drop_ despite gaining dynamic tabs.

**The new Files pane.**

- **One row per unique logical file.** The group key is the named-root logical path for
  `@file:<path>` and the artifact id (or a meaningful original source path) for
  `sase artifact create` rows. Physical `/home/...` paths never become identity.
- **Version toggling on the selected row**, bound to `(` / `)`. Versions are
  `(sha256, first_seen_at, agents[])` tuples. The detail header shows `version i/n`,
  digest, capture time, agent, project, origin, MIME, and size. A repeated capture with
  an unchanged digest does not create a version but does add provenance. Reuse the
  existing MIME-aware artifact viewer; do not build a second renderer.
- **Origin must be visible.** Three origins: `ref` (cited in a prompt), `created`
  (`sase artifact create`), `capture` (automatic staging). The index already carries an
  `explicit` boolean (`FilesSnapshot.explicit_count`); **widen it to an enum** rather
  than adding a parallel flag. Render it as a badge column and make it a filter facet.

**Two gaps to close.** Prompt-staged files land in the _workspace_ `.sase/artifacts`
pool and manifest, not the durable index that `query_artifact_files` reads — promote
`@file` versions into the durable index (or a sibling `ref_files` index) keyed by
`(logical_path, sha256)` **at publication time**, which keeps the launch path cheap and
gives the right semantics (only published runs contribute versions). And read
`sase/memory/tui_perf.md` before implementing: N artifact repos means N tree scans, and
dynamic tabs invite eager construction. Keep `ContentSwitcher` + `activate()`-on-visible
laziness, share one off-thread loader keyed by provider, and cache inventory by
(provider-spec digest, project config, repo HEAD). Do **not** rebuild the previously
removed global artifact graph; disposable revision-keyed indexes suffice. Expect
`tests/ace/tui/visual/snapshots/png/` goldens to need `--sase-update-visual-snapshots`.

## 4. Phases

### 4.1 Ref contract wire types in sase-core

Everything here lands in the linked `sase-core` repo (open it with `/sase_repo`) and
ships as one `sase-core-rs` release. `artifact_ref/filter.rs` is untouched.

- **Provider spec wire.** `ArtifactRefProviderSpecWire` covering §3.4: descriptor,
  argument grammar, expansion format, inventory filters, identity, typed property
  schema, detail hints, publication policy, plus a `schema_version` and a spec digest.
  Validation rejects unknown placeholders, unknown property types, reserved kinds
  (`stitch`/`patch`/`bead`/`agent`/`file`), and non-lowercase ids.
- **The closed expansion formatter.** Declared placeholder set, validated at spec load,
  evaluated without filesystem or process access.
- **Entry, resolution, and use wire types** per §3.3 and §3.7.
- **Kinds.** Add `Stitch { repo, sha }` and `Patch { project, name }`; extend `File`
  with a `Path { root, relpath }` payload variant alongside the existing digest payload;
  keep `Bead`, `Agent`, `Document`. Move `Bug` and `Chat` behind a parse-only historical
  reader that still round-trips persisted strings and still renders in archives.
- **Aliases.** `commit:` -> `stitch:` permanently; `plans:` -> `plan:` with a
  deprecation diagnostic.
- **Quoted-argument grammar** in `scanner.rs`: `@kind:"argument with spaces"`, with
  escaping, while keeping fenced-block/inline-code/literal-zone protection and the
  trailing `#fragment` split. Completion inserts quotes automatically when the argument
  requires them.
- **`markdown_link_refs.rs`.** Lift `assign_reference_id`, `allocate_numeric_id`, and
  `parse_reference_definition` out of `commit_footer.rs` into a shared module, then
  extend the scan to numeric _uses_ as well as definitions, outside literal zones,
  reusing the zone logic `rewrite_prompt_artifact_links` already has. Honor CommonMark's
  first- definition-wins rule (never emit a duplicate `[N]:` hoping the last wins) and
  keep footnote labels (`[^1]`) in their own namespace. Add a reference-style emitter.
- **`Referenced By` footer block.** A new module mirroring `plan_header_block` +
  `sections.rs`, anchored at the bottom, with managed markers, deterministic sort, a cap
  of 50 plus an `omitted: N` line, and byte-identical repeat renders.
- Bump the affected wire schema versions, regenerate the Python binding surface, and
  keep `tools/validate_sase_core_rs` green.

Tests: quoted/escaped/fragment/punctuation parsing; short-hash and `<repo>@<sha>`
disambiguation; alias round-trips; formatter placeholder validation and injection
refusal; numeric-link allocation with gaps, pre-existing definitions, same-target reuse,
code fences, footnotes, dangling uses, and repeat-run idempotence; footer block
idempotence, cap, and marker recovery.

### 4.2 Retire the ref xprompt surface

Land this deletion on its own so it reviews alone and `master` stays shippable. Builtins
fall back to the pre-`sase-ho` hardcoded rendering path still present in
`artifact_ref_prompt_rendering.py`.

Delete: `src/sase/xprompts/refs/*.md`; `src/sase/xprompt/loader_refs.py`; the `ref` /
`ref_kind` / `ref_sidecar_role` / `ref_path_globs` / `ref_shadowed_sources` fields on
`XPrompt` and their catalog wires; the `#ref/` rewriting in
`artifact_ref_prompt_parsing.py`; `sase/refs/` sources in `content_layout.py` and
`resolve_ref_file_sources`; `#ref/` completion in `ace/tui/prompt_catalog.py` and
`_startup_prompt_catalog.py`; the `xprompt_browser_helpers.py` synthetic-source special
cases; the contextual-ref catalog entries in `xprompt_catalog.rs` and
`sase_xprompt_lsp`; `reject_misplaced_ref()` namespace policing on ordinary xprompt
load; the synthetic source schemes `sidecar_ref_config:` and `generated_sidecar_ref:`;
the seven-level renderer precedence table; and the ref-renderer sections of
`docs/xprompt.md`, `docs/content_layout.md`, and `docs/plugins.md`. Retire the matching
tests (`test_xprompt_ref_sources.py`, `test_artifact_ref_xprompt_integration.py`, and
the ref portions of others).

Keep and re-point: `artifact_ref/filter.rs`, `filter_artifact_ref_path_payloads`, and
`ARTIFACT_REF_PATH_FILTER_WIRE_SCHEMA_VERSION`; the `filtered` resolution status and its
non-leaking diagnostic (it already refuses to leak absolute local paths into prompt
text, which `@file` needs); `ArtifactRefDocumentRootWire` and `ArtifactRefContextWire`;
and the shape of `effective_sidecar_ref_policies` — swap its `xprompt` field for the
provider-resolved spec and keep the role merge/disable logic.

Before deleting anything, capture golden fixtures for all eight current refs,
xprompt/literal-zone interactions, completion, staging, old manifest parsing, and prompt
publication, each marked _historical compatibility_ or _desired new behavior_. Those
fixtures are the regression net for every later phase.

### 4.3 Provider registry, plugin hooks, and config

- **New pluggy project `sase_artifact`** with two hookspecs, called once while
  assembling the config registry and never per ref, per keystroke, or per file event:

  ```python
  sase_artifact_ref_providers() -> Iterable[ArtifactRefProviderSpec]
  sase_file_hook_providers()    -> Iterable[FileHookProviderSpec]
  ```

  Two entry-point groups, `sase_artifact_refs` and `sase_file_hooks`, registered in
  `plugins/inventory.py::ENTRY_POINT_GROUPS` as provider groups, so
  `SASE_DISABLE_PLUGIN_ARTIFACT_REFS` and `SASE_DISABLE_PLUGIN_FILE_HOOKS` work
  independently and `sase plugin` inventory stays legible. One pluggy project covering
  two hook families matches `sase_workspace` precedent; ref providers and file-hook
  providers ship together in practice.

- **Registry rules.** Lowercase provider id and ref kind; a host-understood schema
  version; unique ids across distributions; reserved kinds `stitch`/`patch`/`bead`/
  `agent`/`file`; exactly one effective provider per kind per project; deterministic
  ordering independent of pluggy call order; and a cache token of (distribution name,
  version, spec digest).

- **`use:` and inline merge** per §3.5, with a golden parity test asserting that the two
  spellings produce byte-identical normalized specs.

- **Schema deltas** in `src/sase/config/sase.schema.json`: `sidecarRef` drops `xprompt`
  and gains `use`, `kind`, `display_name`, `tab_label`, `argument`, `expansion`,
  `inventory`, `identity`, `properties`, `detail`, and `publication`, keeping
  `filters.path_globs` as a deprecated alias for `inventory.path_globs` for one release;
  `fileHook` gains `use` and relaxes `required: ["name", "command"]` to
  `required: ["command"]` via `anyOf` when `use` is present. Mirror the `use:` handling
  in `config/file_hooks.py`, whose `_KNOWN_FILE_HOOK_KEYS` and `_parse_file_hook`
  currently reject it.

- **Builtin providers registered on the same group by the `sase` package itself**, so
  `use:` has no privileged path: `plan` (document strategy over the plans role) plus the
  entry-backed descriptors for `stitch`, `patch`, `bead`, `agent`, and `file`.

- **`sase init` writes the plan ref** idempotently through the existing `ruamel` writer
  in `main/_repo_init_config.py`:

  ```yaml
  repos:
    sidecar:
      builtin:
        plans:
          auto_clone: true
          ref:
            use: plan
  ```

  Skip when `use:` is already present; never clobber a user's inline `ref:`.

- **Fail-soft plus doctor.** Unresolvable `use:`, duplicate provider ids, unknown schema
  versions, and invalid specs warn and degrade on the launch path and surface as hard
  `sase doctor` findings naming the config path and the install command. **A linked-repo
  clone is not an installed Python distribution and does not make entry points visible**
  — this is the most likely deployment failure, and neither a `linked` repo entry nor
  `auto_clone: true` solves it. The doctor check must say so explicitly.

### 4.4 Builtin refs and prompt ref context

- **`PromptRefContext`** per §3.6: resolve the VCS project from each prompt segment's
  `#git`/`#gh` workflow and thread it into late ref processing. Never infer from `cwd`.
  Multi-segment prompts and xprompt swarms get one context per segment.
- **`@stitch`.** `@stitch:<short_hash>` against the in-context project's primary repo;
  `@stitch:<repo>@<short_hash>` against any repo in the SASE repo registry.
  `<repo>@<sha>` is already exactly how `commit` serializes and `scanner.rs` splits only
  on the first `:`, so a payload-internal `@` is already tolerated; `parse_payload`
  disambiguates on the presence of `@`. Accept 7-40 hex characters, resolve to a commit
  object, error on ambiguity, and store the **full** hash plus canonical repo identity.
  Attach the containing Patch name and stitch number when discoverable without requiring
  them. Expansion: `stitch <full-sha> in <repo> (checkout: <path>)`.
- **`@patch`.** Resolve in the in-context project's active and archive ProjectSpecs
  (`<key>.sase`, `<key>-archive.sase`). Never silently select a same-named Patch from
  another project; with no project context, accept a globally unique name but make
  ambiguity a structured error listing candidate projects. Expansion is concise, not an
  inlined ProjectSpec:
  ``the <name> Patch in project <project> (inspect with `sase patch show <name>`)``.
- **`@bead`.** Accept a short id or a full bead id. A short id searches the in-context
  project's store first; `ArtifactRefContext.bead_stores` is already a cross-project
  tuple, so a cross-project collision returns the existing `ambiguous` status with fully
  qualified candidates. Prefix ambiguity is an error, never first-match.
- **`@agent`.** Keep the existing resolution, restated as a builtin entry provider with
  the §3.4 property set, and make its expansion cover the agent's chat transcript so
  removing `@chat` loses nothing.
- **Use records.** Write the immutable per-agent `ref-uses` manifest of §3.7 at
  resolution time for **every** ref kind, including builtins. This is the phase that
  makes "which refs did which agent use" answerable; later phases only read it.
- **Aliases and deprecation.** `commit:` parses forever. `plans:` parses with an
  actionable diagnostic; completion offers only `@plan`. `chat:` and `bug:` parse
  historically and are absent from completion and new-prompt validation.

### 4.5 The @file ref and the content-addressed store

- **Allow-list config** `artifact_refs.file.roots` per §3.5, using the
  `filter_artifact_ref_path_payloads` vocabulary, with schema, layered merge, and doctor
  validation.
- **Resolver obligations, in order:** expand `~` and parse the possibly quoted argument;
  resolve the physical path and verify containment in exactly one configured root; apply
  path globs to the root-relative POSIX path, where a miss is `filtered`, not `missing`;
  reject directories, devices, sockets, symlink escapes, and unreadable files; enforce
  the configurable size ceiling (`capture_file_exceeds_size_limit` already exists); read
  the bytes **once** and hash _those_ bytes, never a later re-read of the source; record
  logical path, authored spelling, size, MIME, capture time, and full SHA-256; and
  expand the prompt to the **immutable captured copy**, so the agent reads the same
  bytes publication later records.
- **Security posture.** Using `@file` is explicit publication intent, but the allow-list
  is the exfiltration boundary — gitignore rules are irrelevant and must never be read
  as permission. Published metadata carries `bob:gtd.md` or `~/bob/gtd.md`; the physical
  `/home/bryan/...` path stays private runtime metadata. Surface the target sidecar's
  visibility in the launch preview.
- **The store** per §3.7: `files/objects/sha256/<ab>/<64hex>` in the agents sidecar, one
  path per byte sequence, atomic write-verify-rename, byte verification instead of blind
  overwrite, and metadata in manifests rather than in the path. Serve both `@file`
  captures and `sase artifact create` outputs from it, with existing artifact ids as
  compatibility aliases. Do **not** build on `artifact_pool_filename`: it keeps the
  basename duplication vector and therefore cannot satisfy "exactly one location per
  unique contents."
- **Durable version index.** Promote `(logical_path, sha256)` versions into the index
  that `query_artifact_files` reads, at publication time, so only published runs
  contribute versions and the launch path stays cheap.

Tests: symlink escape, traversal, special files, size ceiling, unreadable, changed-
during-capture, duplicate content; exactly one object per full digest across differing
names, agents, months, and origins; both `@file` payload shapes; root containment and
glob misses.

### 4.6 Reference links and Referenced By write-back

- **Reference-style rewriting.** Extend `rewrite_prompt_artifact_links` — which already
  guarantees idempotency, literal-zone skipping, and leaving pre-existing links alone —
  to emit `[@kind:arg][N]` using the shared allocator from §4.1. It must collect numeric
  definitions _and_ uses outside protected zones; reuse an existing label whose
  destination already matches; otherwise take the lowest positive integer free in that
  document; rewrite every live occurrence while preserving visible text exactly; append
  new definitions in numeric order at the bottom; return the linked use ids and labels
  so the `ARTIFACTS` header section stays in sync; and be byte-identical on a second
  run.
- **Destinations** per the §3.7 table, always at the revision pinned at resolution time.
  `prompt_archive/preparation.py` already threads `primary_revision` and a
  `HostedLinkResolver` into `_ArtifactTargetResolver`, so the plumbing exists. The VCS
  provider constructs the URL — the contract requests `vcs_permalink` and never
  hardcodes GitHub.
- **`Referenced By` write-back** per §3.8: generalize `refresh_plan_links` to operate on
  any artifact-repo role using the provider's row builder, back it with the structured
  `.sase/referenced-by/<artifact-id>.json` index, route it through
  `publication_outbox*`, and observe the artifact-repos-before-`agents` lock order.

  ```text
  1. Resolve refs and capture immutable use records at launch.
  2. Copy missing SHA-256 objects; publish the agent page, prompt, and manifests.
  3. Push the agents commit; obtain its permalink.
  4. Enqueue one back-reference update per affected artifact.
  5. Group by artifact repo; update all managed blocks and indexes in one commit per repo.
  6. Pull/rebase and retry on non-fast-forward.
  7. Mark outbox rows complete only after the artifact-repo push succeeds.
  ```

  Agent publication is the source of truth: a failed artifact-repo push stays a visible
  retryable diagnostic and never rolls back or hides the agent. If an artifact moved,
  locate it by the provider's identity property; if it was deleted, leave a visible
  outbox error rather than attaching the row to a guessed path.

- **Exclusions.** The managed block is excluded from content digests and change
  detection, and back-reference commits carry a `system_projection: referenced_by` cause
  that user `file_hooks` skip by default.

Tests: back-reference idempotence; renamed and deleted artifacts; concurrent update;
push failure and retry; "does not run user file hooks"; captured-revision correctness
when the artifact repo's HEAD advances between resolution and publication; the
dirty/untracked fallback; and archive rendering of historical `chat:`/`bug:`/`commit:`
strings.

### 4.7 Generated Artifacts sub-tabs and the new Files pane

Implement §3.9 against the tree described in §2, not the research report's older tree.
Land this last among the sase-repo phases — it is the only one whose blast radius
includes keymaps and PNG visual snapshots. Read `sase/memory/tui_perf.md` with
`/sase_memory_read` before starting.

Tests: dynamic tab union across enabled projects; project enable/disable; provider
removal with a persisted selection; number-key stability; `[`/`]` cycling; Files origin
grouping, one row per logical path, and version navigation with `(`/`)`; documents-pane
parity with the retired plans pane for filtering, detail, and navigation; refreshed
visual snapshots.

### 4.8 The sase-research plugin repository

Build `sase-org/sase-research` (open it with `/sase_repo`; today it is one commit and a
0-byte README). Take structure from `sase-telegram` (hatchling, `src/` layout, ruff plus
strict mypy, pytest with `--strict-markers --strict-config`, `.github/`, `Justfile`,
release-please, `docs/`) and entry-point patterns from `sase-github`.

```toml
[project.entry-points."sase_artifact_refs"]
research = "sase_research.provider:RESEARCH_REF_PROVIDER"

[project.entry-points."sase_file_hooks"]
research-highlights = "sase_research.provider:RESEARCH_HIGHLIGHTS_HOOK"

[project.entry-points."sase_xprompts"]
sase_research = "sase_research"

[project.entry-points."sase_config"]
sase_research = "sase_research"
```

- `src/sase_research/provider.py` — one ref spec literal (kind `research`, inventory
  globs `["20*/**/*.md"]`, properties `create_time`/`updated_time`/`status`/`tags`,
  detail hints, `Referenced By` columns) and one file-hook spec literal
  (`research-highlights`, `sidecars: [research]`, path globs
  `["20*/**/*.md", "!20*/*/*__*.md"]`,
  `agent_name_globs: ["!research.*.cld", "!research.*.cdx"]`, `ops: [ADD]`,
  `timeout: 120s`), with **`command` deliberately unset and marked required**.

  The two glob sets differ on purpose, and that difference is the point of separating
  the ref provider from the hook provider. The hook excludes `__a`/`__b` swarm drafts
  because Bryan does not want a Highlights PDF per draft. The ref inventory keeps them,
  because citing a specific researcher's draft is legitimate. Document the divergence in
  the repo's README so it does not read as a porting bug.

- `src/sase_research/xprompts/` — the `#research*` xprompts. **Four of them
  (`#research`, `#research/image`, `#research/more`, `#research/prompt`) must be lifted
  out of chezmoi's `xprompts:` YAML block, not moved as files**; only
  `research_swarm.md` is already a file. Drop `#old_research_swarm` rather than porting
  it.
- `src/sase_research/default_config.yml` — the `research_a` / `research_b` /
  `research_lead` model aliases, the `researchers` bucket, and the `research` tribe
  display config, so `#research_swarm` works on a fresh install. It references
  `%model:@research_a` / `@research_b` / `@research_lead` and
  `$(sase repo path research --ensure)`, so without this the moved xprompt breaks.
  Bryan's chezmoi values still win by layer precedence.
- `README.md`, `docs/`, `tests/`, `.github/workflows/ci.yml`, `LICENSE`, `CHANGELOG.md`.

**Name disambiguation is a deliverable, not a nicety.** `sase-org/sase-research` (this
plugin) and `sase-org/sase--research` (the research _content_ sidecar) differ by one
hyphen, and the plugin repo's current GitHub description is literally
`sase--research Artifact Repo Plugin`. State the distinction in three places: the GitHub
description, the README's first paragraph, and — most importantly — the
`repos.linked[].description` this project's `sase/sase.yml` will carry, because that
string is what `sase memory init` renders into the Repositories section of every agent
instruction file. Add a `description:` to the research _sidecar_ entry pointing back the
other way.

**Quality bar — do not skimp.** CI must test the **wheel**, not just the source tree;
resource and entry-point packaging is this repo's most likely failure mode. Build sdist
plus wheel, install into a clean environment, enumerate entry points, and assert every
packaged xprompt resource is discoverable. Beyond that: unit tests for spec schema, the
`command` override, filter/glob semantics against real path fixtures, frontmatter
parsing, and duplicate/missing-provider diagnostics; an inline-versus-`use`
normalization golden test; `#research_swarm` segment-boundary and wait/fork directive
parsing tests; a matrix pinned against the minimum supported `sase` version; and
release-please plus trusted PyPI publication.

### 4.9 Adoption, glossary, and documentation

- **Link and install the plugin.** Add `sase-research` to `repos.linked` in this
  project's `sase/sase.yml` with the disambiguating description from §4.8, add a
  `description:` to the `research` sidecar entry, and set
  `repos.sidecar.custom.research.ref.use: research`. Install the distribution (editable
  for linked development) — a clone alone does not register entry points.
- **Move Bryan's chezmoi config.** In `home/dot_config/sase/sase.yml`, replace the
  `research-highlights` file-hook block with exactly:

  ```yaml
  file_hooks:
    - use: research-highlights
      command: bob highlights create --include-id
  ```

  Delete the four `research*` entries from the `xprompts:` block and delete
  `home/sase/xprompts/research_swarm.md` and `home/sase/xprompts/old_research_swarm.md`
  — but **only after** the clean-wheel resource-discovery smoke test passes and
  `sase xprompt list` shows the plugin-provided `#research*` entries. The `research_a` /
  `research_b` / `research_lead` aliases, the `researchers` bucket, and the `research`
  tribe config stay in chezmoi as user overrides.

- **Add the `@file` roots** to `home/dot_config/sase/sase.yml` (the home layer, since
  `~/bob` is machine-global rather than project-scoped):

  ```yaml
  artifact_refs:
    file:
      roots:
        - name: bob
          path: ~/bob
          path_globs: ["**/*.md"]
  ```

- **Glossary.** Add to `memory.glossary` in this project's `sase/sase.yml`, then run
  `sase memory init` to regenerate `AGENTS.md` and the provider shims:

  ```yaml
  Artifact Reference:
    aliases:
      - ref
    definition: >-
      An artifact reference (ref) is a typed `@<kind>:<argument>` citation in an agent
      prompt. Builtin kinds are `@stitch`, `@patch`, `@bead`, `@agent`, and the special
      `@file`; artifact repos add document kinds such as `@plan` and `@research` through
      a project's `ref:` config, written inline or with `use: <provider>` from an
      installed provider plugin. Every ref expands to prompt text, is recorded against
      the agent that used it, and publishes as a `[@kind:arg][N]` link.
  ```

  Update the `Sase Repo` glossary entry, which still says `<project>--plans` or
  `<project>--research` are "SDD sidecar" repos, to match the artifact-repo framing.

- **Docs.** Rewrite `docs/xprompt.md` (drop artifact-reference xprompts, point at the
  new page), `docs/plugins.md` (two new groups, the six-group table is now eight, and
  the "linked repo is not an installation" warning), `docs/configuration.md` (`ref.use`,
  `file_hooks[].use`, `artifact_refs.file.roots`), `docs/content_layout.md`,
  `docs/ace.md` (dynamic Artifacts sub-tabs, the new Files pane, `(`/`)`),
  `docs/agents_sidecar.md` (the object store and `ref-uses`), `docs/sdd.md`
  (`Referenced By` versus `AGENTS`), `docs/editor.md`, `docs/getting_started.md`, and
  `docs/llms.md`. Add one new page that is the single authoritative description of the
  ref contract, the eight kinds, the grammar, and the provider spec.
- **End-to-end verification.** Launch a real agent whose prompt cites one ref of each
  live kind, confirm expansion, the `ref-uses` rows, the published `[@kind:arg][N]`
  links, the CAS object, the `Referenced By` table in the research sidecar, and the
  `Research` sub-tab in ACE.

## 5. Migration and compatibility

| Current                      | New                                   | Compatibility                                               |
| ---------------------------- | ------------------------------------- | ----------------------------------------------------------- |
| `@commit:<repo>@<sha>`       | `@stitch:[<repo>@]<sha>`              | **permanent** parse alias                                   |
| `@plans:<path>`              | `@plan:<path>`                        | deprecated alias for one release; completion offers `@plan` |
| `@file:<src>:<hex24>`        | unchanged, plus `@file:<path>`        | both live; `explicit`/`default` reserved as first segment   |
| `@bead:<id>`                 | `@bead:<short-or-full-id>`            | extended in place                                           |
| `@research:<path>`           | provider-backed, same spelling        | no authoring change                                         |
| `@agent:<name>`              | builtin entry provider, same spelling | no authoring change                                         |
| `@chat:<path>`, `@bug:<p#n>` | removed from authoring                | parse-only historical readers, forever                      |
| `#ref/<kind>`                | removed                               | actionable deprecation diagnostic during the window         |

Deprecated kinds do **not** appear in completion during the warning release — that would
just encourage new use. Historical archive rendering uses the manifest schema recorded
with the run; new prompt validation uses the new grammar.

Data migration: keep existing prompt-artifact and consumption manifests readable as
version 1 and start writing version 2 with occurrence, provider, revision, stable id,
logical path, and origin; populate the SHA-256 object store lazily during publication,
verifying **full** hashes before deduplicating old 12-prefix-plus-basename objects;
build the Files-pane logical and version indexes from both the old artifact index and
the new use manifests. Do not rewrite historical agent pages merely to change link
style, and do not backfill historical artifact-repo back-references speculatively —
start with newly published agents and offer an explicit audited backfill later if it
proves wanted.

## 6. Risks

| Risk                                                                                                                                      | Severity | Mitigation                                                                                                                 |
| ----------------------------------------------------------------------------------------------------------------------------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------- |
| Persisted `commit:`/`file:`/`chat:`/`bug:`/`agent:` strings in bead refs, `artifact_consumption.jsonl`, prompt manifests, and Patch files | High     | Permanent parse aliases; never a hard rename; add a `sase artifact doctor` normalization report                            |
| Wire schema churn plus a `sase-core-rs` release and roll                                                                                  | High     | One Rust phase, one release; `tools/validate_sase_core_rs` already gates the binding                                       |
| `use:` naming an uninstalled provider                                                                                                     | High     | Fail soft plus a doctor finding, exactly like `config/file_hooks.py`; never raise on launch                                |
| A linked-repo clone mistaken for a plugin installation                                                                                    | High     | Explicit install step and a doctor check that says so                                                                      |
| Publication linking bytes the agent never read (HEAD drift)                                                                               | High     | Pin the revision at resolution time; never resolve HEAD at publication                                                     |
| Cross-repo write-back deadlock or publication failure                                                                                     | Medium   | Fixed lock order (artifact repos, then `agents`); route back-references through the outbox                                 |
| Back-reference commits triggering user `file_hooks` or churning artifact versions                                                         | Medium   | `system_projection` cause excluded by default; managed block excluded from digests                                         |
| ACE regression surface (keymaps, PNG snapshots, persisted sub-tab)                                                                        | Medium   | Land it late; stable number keys for fixed tabs; fall back on an unknown persisted sub-tab                                 |
| ACE latency from N artifact repos                                                                                                         | Medium   | Lazy activation, a shared off-thread loader, config-token-cached specs; read `tui_perf.md`                                 |
| `@file` allow-list leaking private files into a published sidecar                                                                         | Medium   | Opt-in root-scoped allow-list; size cap; target visibility in the launch preview; logical paths only in published metadata |
| Numeric-label collision from a dangling `[x][3]` use                                                                                      | Low      | Scan uses as well as definitions, outside literal zones                                                                    |
| `sase-research` versus `sase--research` confusion                                                                                         | Low      | Three-place disambiguation, with `repos.linked[].description` as the load-bearing one                                      |

## 7. Alternatives considered and rejected

**Keep refs as xprompts and add frontmatter.** Preserves the category mistake; rendering
is the smallest part of the contract and the only part the framing could extend. It
still needs synthetic sources, and direct `#ref/*` invocation bypasses usage tracking.

**Layer providers on top of ref xprompts.** Two overlapping definition systems for one
concept, and the precedence table is already seven levels deep.

**`git revert` the `sase-ho` commits.** Throws away the filter contract and
document-root plumbing this design depends on, and downgrades an already-released wire
schema.

**Plugins implement arbitrary resolvers or renderers.** Unbounded latency and failure
modes on the completion and TUI paths, impossible config validation without importing
plugin code, and guaranteed divergence for native consumers.

**Resource-only plugins (`ref.yml` package data, like `sase_xprompts`).** Attractive —
no code import, trivially cacheable — but cannot express provider-supplied properties
for entry-backed refs.

**Pure config with no plugin hook.** Fails the stated goal of users defining and
_sharing_ ref types, and the research file hook plus `#research*` xprompts need a
distributable home regardless.

**Split `@file` into `@file` (paths) and `@artifact` (digests).** Breaks every persisted
`file:` string and `sase artifact create`'s printed `ref:` line for no user-visible
gain.

**Reuse `artifact_pool_filename` for the new store.** Keeps the basename duplication
vector and so cannot satisfy "exactly one location per unique contents."

**Store back-references only in a central index or Git notes.** Avoids touching artifact
Markdown but does not deliver the portable visible table. The structured index _plus_
the generated managed block gets both.

**Synchronously atomic cross-repo publication.** Git offers no shared transaction;
simulating one makes agent publication fragile and still leaves partial pushes.

**A new global artifact database or graph.** Repeats a large subsystem SASE already
built and removed. Provider inventories, per-agent manifests, a digest object store, and
disposable revision-keyed indexes suffice until measured scale says otherwise.

**Put the ref registry in Python only.** Violates the Rust core backend boundary:
grammar, resolution, filtering, link numbering, and footer rendering are cross-frontend
behavior.

## 8. Out of scope

- Renaming "sidecar repo" to "artifact repo" across the codebase, config, and docs. This
  plan uses "artifact repo" in prose where it aids comprehension but changes no
  identifiers, config keys, or role names. That rename is its own migration-shaped
  change.
- Backfilling `Referenced By` tables for historically published agents.
- Rewriting historical agent pages to use reference-style links.
- Consolidating any further ACE sub-tabs beyond deleting the `plans`/`chats`/`other`
  sub-sub-tabs.
- A public plugin-registry entry for `sase-research` beyond what `sase plugin` discovers
  from the `sase-org` GitHub org automatically.
