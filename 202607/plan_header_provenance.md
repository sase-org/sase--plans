---
tier: epic
title: Plan-file provenance header block
goal: 'Every committed plan file opens with one beautiful, self-healing header block
  whose bullets link the plan to its prompt, its parent plan, every agent that worked
  it, and every commit it produced, with GitHub URLs that render as hyperlinks on
  github.com.

  '
phases:
- id: block-contract
  title: Rust-owned plan header block grammar
  depends_on: []
  size: medium
  description: 'block-contract: replace the single-bullet SDD artifact-link contract
    in sase-core with a multi-section, nestable, wrap-tolerant header block, expose
    it through PyO3, and reimplement the existing Python adapter on top of it without
    changing current behavior.'
- id: hosted-links
  title: Hosted URL resolution for plans, agents, and commits
  depends_on: []
  size: small
  description: 'hosted-links: add one local-only module that resolves a plan reference,
    an agent name, and a commit SHA to absolute GitHub URLs, degrading to unlinked
    labels instead of guessing.'
- id: associations
  title: Derived plan association index
  depends_on:
  - hosted-links
  size: medium
  description: 'associations: derive each plan''s agents and commits from commit footers
    and agent artifact metadata, roll descendant associations up into epic plans,
    and return ready-to-render sections.'
- id: write-points
  title: Header writes at propose, commit, and post-commit
  depends_on:
  - block-contract
  - hosted-links
  - associations
  size: medium
  description: 'write-points: seed PROMPT and PARENT when a plan is proposed and committed,
    stop stamping the parent frontmatter property, and refresh the named plan''s header
    best-effort after each primary commit.'
- id: reconcile
  title: Tree-wide refresh and parent-property migration
  depends_on:
  - write-points
  size: medium
  description: 'reconcile: add sase plan links refresh for bulk, idempotent header
    reconciliation, migrate existing parent frontmatter into PARENT bullets, and deprecate
    the frontmatter property in the plan schema.'
- id: surfaces
  title: Display, documentation, and validation surfaces
  depends_on:
  - block-contract
  - reconcile
  size: small
  description: 'surfaces: teach the ACE plan viewer, the plans sidecar README, and
    the CLI help/output about the header block so the new bullets read well everywhere
    they are shown.'
create_time: 2026-07-28 09:48:54
status: done
bead_id: sase-ag
---

- **PROMPT:** [prompts/202607/plan_header_provenance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/plan_header_provenance.md)
- **BEAD:** [sase-ag](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ag/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ag.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ag.land.md#member-code)
- **COMMITS:**
  - [702f1ae](https://github.com/sase-org/sase/commit/702f1aece2375113427d437497924e960d5ca735) — build(deps): require sase-core-rs 0.12.4 (sase-ag)

# Plan: Plan-file provenance header block

## Goal

A committed plan file today opens with exactly one bullet linking it to its prompt snapshot, and records its parent epic
plan in a `parent:` YAML frontmatter property that GitHub renders as inert text. Neither the agents that worked the plan
nor the commits they produced are discoverable from the plan file at all, even though both associations already exist in
durable SASE state.

Replace that single bullet with an ordered **plan header block**: a contiguous run of top-of-body Markdown bullets, some
with nested sub-bullets, that carries the plan's full provenance and renders as real hyperlinks on github.com. Retire
the `parent:` frontmatter property in favor of a `PARENT` bullet.

The header block is a **projection of durable state, never an accumulator**. Nothing new is persisted: agents and
commits are re-derived from commit footers and agent artifact metadata on every refresh. A stale, duplicated, or wrong
entry disappears the moment the underlying source is corrected, and any write point can be best-effort because a later
reconcile repairs it.

## Rendered format

Section order is fixed: `PROMPT`, `PARENT`, `AGENTS`, `COMMITS`. Prompt snapshots keep their single `PLAN` section, so
one grammar serves both artifact kinds.

```markdown
---
tier: tale
title: Agents sync engine and CLI
goal: Completed commit-associated agents synchronize safely through each project's hidden agents sidecar.
bead: sase-8k.6
create_time: 2026-07-22 15:14:56
status: done
---

* **PROMPT:** [202607/prompts/agents_sync_engine.md](prompts/agents_sync_engine.md)
- **PARENT:**
  [202607/agents_sidecar_repo.md](https://github.com/sase-org/sase--plans/blob/main/202607/agents_sidecar_repo.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-8k.6](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-8k.6/README.md)
  - [bbugyi200.athena.sase-8k.6--fix](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-8k.6--fix/README.md)
- **COMMITS:**
  - [699456a](https://github.com/sase-org/sase/commit/699456a521e25e0aaa38f4e289db38e71a6488a6) — fix(xprompt):
    canonicalize workflow project identity

# Plan: Agents sync engine and CLI
```

Rules that make this beautiful and stable:

- The block is the first Markdown body element, starts at the first body byte, and is followed by exactly one blank line
  — the invariant the current single-bullet contract already enforces via `canonical_layout`.
- `PROMPT` stays first and keeps its existing file-relative href. It is unchanged from today's contract, so more than
  3,200 already-committed plan files take no diff from this feature.
- Sub-bullets are indented exactly two spaces and are deterministically ordered: agents by global name, commits by
  committed timestamp then SHA.
- A section with nothing to show is **omitted entirely**. An empty `- **AGENTS:**` header is never rendered.
- Commit sub-bullets show the seven-character short SHA as link text, link to the full SHA, and append `— <subject>`
  with the subject's Markdown metacharacters escaped and any trailing bead-id parenthetical left intact.
- Agent sub-bullets show the globally unique agent name, matching the `SASE_AGENT=` commit footer and the agents sidecar
  page title, so a name in a commit message and a name in a plan file are always the same string.

## Link policy

Every bullet or sub-bullet that references a plan, an agent, or a commit links to its GitHub URL:

| Section   | Destination                                                                          |
| --------- | ------------------------------------------------------------------------------------ |
| `PROMPT`  | file-relative href inside the plans sidecar (unchanged)                              |
| `PARENT`  | `https://<host>/<org>/<plans-sidecar>/blob/<branch>/<YYYYMM>/<name>.md`              |
| `AGENTS`  | `https://<host>/<org>/<agents-sidecar>/blob/<branch>/agents/<global-name>/README.md` |
| `COMMITS` | `https://<host>/<org>/<primary-repo>/commit/<full-sha>`                              |

Link generation is local-only and best-effort, exactly like the existing `format_sase_plan_link` and
`resolve_agent_commit_tag` paths. When a store has no hosted remote, no resolvable branch, or a non-GitHub host without
authoritative provider metadata, the entry degrades rather than guessing:

- `PARENT` falls back to a file-relative href, matching `PROMPT`.
- `AGENTS` and `COMMITS` render their label unlinked. A broken or invented URL is never written.

Visible labels stay stable across those fallbacks so a store that gains a remote later produces a link-only diff.

## Prettier interaction (do not skip)

`format_with_prettier` runs over plan content in `write_sdd_files`, `handle_sase_plan`, and
`handle_plan_propose_command` with `--prose-wrap=always --print-width=120`. A commit sub-bullet carrying a 79-character
commit URL plus a subject exceeds 120 columns, so **prettier will wrap it onto continuation lines**. The current
line-anchored `artifact_link_regex` in `sase-core` cannot survive that.

Two requirements follow, and both belong to `block-contract`:

1. **Parsing is wrap-tolerant.** Before matching, a bullet's continuation lines — lines indented deeper than their
   bullet marker that do not themselves start a new bullet — are joined into one logical line.
2. **Change detection compares parsed logical content, not bytes.** An upsert that produces the same logical sections
   must return the document unmodified. Without this the file oscillates between prettier's wrapped form and the
   renderer's unwrapped form, producing an endless stream of empty commits to the plans sidecar.

The renderer emits one logical bullet per physical line and lets prettier own the wrapping. That keeps Rust out of the
business of reimplementing prettier's line-breaking algorithm.

## Sources of truth

Nothing here introduces a new store.

- **Parent** — the proposing agent's `epic_plan_ref`, already resolved in `src/sase/main/plan_propose_handler.py` from
  `SASE_EPIC_PLAN_REF` or the durable `agent_meta.json` marker, canonicalized through `sase.sdd.plan_refs`.
- **Commits** — the primary repository's `git log`, parsed with `sase.core.commit_footer_facade.parse_commit_footer`.
  Commits already carry a `SASE_PLAN=` trailing tag written by `handle_sase_plan`, so history predating this feature
  backfills correctly. `src/sase/agents_sync/inventory.py::_historical_associations` is the existing precedent for this
  exact walk, grouped by `SASE_AGENT` instead of by plan.
- **Agents** — the union of agents named by `SASE_AGENT=` on commits that carry a matching `SASE_PLAN=` tag, and agents
  whose `agent_meta.json` associates them with the plan through `sdd_plan_path`, `plan_path`, `archived_plan_path`, or
  `epic_plan_ref`. The second source is what makes planner agents, reviewers, and land agents appear even when they
  produced no commit of their own. Agents marked `hidden` are excluded.

**Epic roll-up.** An epic plan's `AGENTS` and `COMMITS` sections are the union of its own direct associations and the
associations of every descendant plan reachable through the parent graph. A phase's tale plan lists only its own work;
the epic above it shows the whole story. Cycles in the parent graph are ignored rather than followed.

## Implementation

### 1. block-contract: Rust-owned plan header block grammar

Work in `../sase-core` (open it with `/sase_repo`), in `crates/sase_core/src/plan/artifact_link.rs` and its PyO3 surface
in `crates/sase_core_py/src/lib.rs`. This phase changes no rendered plan file; it only replaces the contract underneath.

Today `artifact_link.rs` models exactly one bullet: `SddArtifactLinkTypeWire` has two variants, `artifact_candidates`
rejects a document containing more than one artifact bullet, and `upsert_sdd_artifact_link` rebuilds the document as
`{prefix}\n{bullet}\n\n{body}`. Generalize that to a block:

- Add a section-kind wire enum covering `PLAN`, `PROMPT`, `PARENT`, `AGENTS`, and `COMMITS`, with `PLAN`/`PROMPT`
  retaining their existing legacy YAML-property mapping and the three new kinds having none.
- Model a section as a label/target pair for the link-shaped kinds and an ordered list of entries — each an optional
  link plus optional trailing text — for the list-shaped kinds. Give the whole block a wire schema version constant
  following the `PLAN_WIRE_SCHEMA_VERSION` and `PLAN_REFERENCE_RESOLUTION_WIRE_SCHEMA_VERSION` precedent, and assert it
  from the Python adapter the way `plan_refs.py` already does.
- Parse the leading run of bullets into an ordered section map. Stop at the first line that is neither a block bullet, a
  sub-bullet, nor a continuation line. Joining continuation lines before matching is what delivers the wrap tolerance
  described above. Preserve the existing `Invalid` disposition with a `reason` for duplicated sections, malformed
  Markdown, unknown `**KEY:**` bullets inside the block, and a missing frontmatter terminator.
- Render sections in the fixed order, escaping Markdown metacharacters in labels and trailing text, and cap list
  sections at a shared constant (reuse the `MAX_RENDERED_COMMITS = 50` value already used by
  `src/sase/agents_sync/rendering_commits.py`) with a trailing `… and N more` sub-bullet. The cap must be visible in the
  document, never a silent truncation.
- Provide upsert-one-section, replace-whole-block, and remove-section entry points. All must be idempotent on logical
  content, must preserve `canonical_layout`, and must keep the `remove_legacy` frontmatter-key removal behavior for
  `PLAN`/`PROMPT` without reserializing unrelated frontmatter.
- Keep back-compat total: a document with today's single `- **PROMPT:**` bullet, a document with only a legacy `prompt:`
  YAML property, and a mixed document must all parse to the same dispositions they do now, including the
  `Mixed`-agreement logic that `SddArtifactDocumentWire::mixed_representations_agree` implements.

Then reimplement the existing `sdd_artifact_link_parse` / `_render` / `_upsert` bindings as thin wrappers over the block
API so every current Python caller — `src/sase/sdd/artifact_links.py`, `_write.py`, `_link_repair.py`,
`_link_validation.py`, and `workflows/commit/commit_hooks.py` — keeps working untouched. Add the new block bindings
alongside. Add a Python adapter for the block API in `src/sase/sdd/` next to `artifact_links.py`, typed the same way
(frozen dataclasses, `require_rust_binding`, no logic outside the binding).

Rust tests must cover: multi-section round-trip; wrap-tolerant parsing of a prettier-wrapped commit sub-bullet;
byte-identical output for an unchanged upsert; section omission when a list is empty; the `… and N more` cap; escaping;
and every back-compat case above.

### 2. hosted-links: hosted URL resolution

Add one module under `src/sase/sdd/` that turns identities into absolute GitHub URLs. It composes existing helpers
rather than re-deriving anything:

- `sase._git_remote.github_blob_url` and `github_commit_url` already sanitize the remote, require either `github.com` or
  authoritative `provider == "github"` metadata, and validate the SHA shape.
- `SddStore` already carries `remote_url`, `provider`, and `repo_root`; `format_sase_plan_link` in
  `src/sase/workflows/commit/plan_paths.py` already shows the store-branch resolution and its `is_in_tree` bail-out.
- `sase.core.agent_identity_facade.agent_link_target` already produces the agents-sidecar path and anchor, and
  `src/sase/agents_sync/links.py::hosted_provider` already resolves GitHub Enterprise hosts from configured hosts.

Expose three resolvers — plan reference, agent name, commit SHA — each returning an optional URL, never raising, and
each usable without network access. Reuse `resolve_sync_targets` for the agents sidecar remote and the primary
checkout's remote for commits. Factor the branch resolution shared with `plan_paths._store_commit_branch` and
`agents_sync.links._sidecar_branch` instead of adding a third copy.

Cache resolution per store within a single process run: the tree-wide reconcile in `reconcile` resolves the same three
remotes for thousands of plans and must not shell out to `git symbolic-ref` once per file.

### 3. associations: derived plan association index

Add a package under `src/sase/sdd/` that builds, in one pass, a map from canonical plan reference to its agents and
commits.

- Walk the primary repository once with `git log --format=%H%x00%ct%x00%s%x00%B%x00`, mirroring
  `agents_sync/inventory.py::_historical_associations`. For each commit, read the trailing tags through
  `parse_commit_footer` and index it under its `SASE_PLAN` tag value when present, recording the `SASE_AGENT` tag as an
  agent association for the same plan. Note that `SASE_PLAN` tag values may be `LinkedCommitTagValue` — take the tag's
  path value, not its rendered Markdown.
- Normalize every plan reference on both sides through `sase.sdd.plan_refs` so a store-relative tag value, a legacy
  path, and an absolute workspace path collapse to one key.
- Scan agent artifact metadata for `sdd_plan_path`, `plan_path`, `archived_plan_path`, and `epic_plan_ref`, reusing the
  existing artifact index rather than globbing `agent_meta.json` files directly where an indexed query is available.
  Skip agents whose metadata marks them `hidden`.
- Build the parent graph from `PARENT` bullets and, during migration, from surviving `parent:` frontmatter, then roll
  descendant associations up into each epic plan. Guard against cycles.
- Return typed frozen records carrying the display label, the resolved URL from `hosted-links`, and the sort key, so the
  caller hands them straight to the block renderer with no further formatting decisions.

Performance is a first-class requirement: one `git log` walk and one artifact scan per invocation, results reusable
across every plan in the tree. Respect `sase/memory/tui_perf.md` if any of this becomes reachable from ACE.

### 4. write-points: header writes at propose, commit, and post-commit

- In `src/sase/main/plan_propose_handler.py`, stop writing `stamps["parent"]`. Keep the `bead` and `parent_bead` stamps,
  which are bead ids rather than file references and stay in frontmatter. Record the resolved parent plan reference so
  the downstream write can render a `PARENT` bullet instead.
- In `src/sase/sdd/_write.py::write_sdd_files` and `src/sase/workflows/commit/commit_hooks.py::handle_sase_plan`, write
  `PROMPT` and `PARENT` through the block API in place of the current single-section upsert. `handle_sase_plan` already
  reads, normalizes, validates, writes, and conditionally commits the plan through `commit_sdd_store_files`; extend that
  existing path rather than adding a second writer.
- Refresh `AGENTS` and `COMMITS` for the one plan named by `SASE_PLAN` after the primary commit exists. The natural seam
  is alongside `publish_committed_agent_hood` in `src/sase/agents_sync/commit_publication.py`, which already runs
  post-commit with the primary revision in hand and already owns sidecar locking and retry.
- That refresh is strictly best-effort. **A plans-sidecar failure must never fail, block, or roll back the code
  commit.** Swallow and log; `reconcile` is the repair path. Write and commit only when the parsed logical content
  actually changed, so a re-run produces no commit.

### 5. reconcile: tree-wide refresh and parent-property migration

- Add `sase plan links refresh` beside the existing `list`, `validate`, and `repair` subcommands in the
  `sase plan links` group. Dry-run by default with a summary of what would change, `--write` to apply, `--plan <ref>` to
  scope to one plan. Follow `sase/memory/cli_rules.md` for flags, help text, and output shape, and offer `--json`
  consistent with the neighboring subcommands.
- Refresh rebuilds each plan's `AGENTS` and `COMMITS` from the `associations` index and rewrites only files whose
  logical content changed. Commit the changed files to the plans sidecar in one batch through `commit_sdd_store_files`.
- Migrate the surviving `parent:` frontmatter. 55 of the 3,235 plan files in the `sase--plans` sidecar currently carry
  it. For each, render the `PARENT` bullet from the property's value and remove the property, reusing the same
  remove-top-level-YAML-key behavior that `remove_legacy` already applies to `plan:` and `prompt:`. Report the plans
  whose `parent:` value cannot be resolved to a real plan file instead of dropping them silently.
- In `../sase-core`, keep `parent` accepted by the plan frontmatter schema so already-committed plans still validate,
  but downgrade it to a deprecated field that emits a warning diagnostic pointing at the `PARENT` bullet, and update its
  schema description. Removing it outright would break `just validate-committed-plans` against history.
- Confirm `sase plan links validate` reports the new sections: an unresolvable `PARENT` target, a duplicated section,
  and a malformed block are errors; a plan with no associations is not.

### 6. surfaces: display, documentation, and validation surfaces

- Teach the ACE plan viewer about the block. `src/sase/sdd/plan_display.py` renders plan documents for the TUI and
  currently only understands the frontmatter properties plus body text; the new bullets should read as a compact
  provenance header there rather than as raw Markdown. Presentation-only Textual work stays in this repo per the Rust
  core backend boundary.
- Update `src/sase/sdd/templates/sidecar-plans-README.md`, whose "Directory Layout" section documents the exact
  one-bullet contract and its blank-line rule. Describe the full block, the section order, the link policy, and
  `sase plan links refresh`. `sase repo init` regenerates the sidecar README from this template.
- Update `src/sase/main/plan_explain.py`, which prints the authoring example shown by `sase plan validate --explain`,
  and any `docs/` page describing plan frontmatter, so `parent:` is no longer presented as the way to name a parent.
- Leave `src/sase/xprompts/skills/sase_plan.md` alone unless a check fails: the skill never instructs an agent to author
  `parent:`, because SASE writes it. If the generated skills do change, follow `sase/memory/generated_skills.md`.

## Constraints and verification

- Run `just install` before `just check` in this workspace; ephemeral workspace clones may have stale dependencies.
  `just check` is mandatory for every phase that touches files in this repo.
- Changes under `../sase-core` must be opened through `/sase_repo` and verified with that repo's own test command.
  Because `just install` builds `sase_core_rs` from the local checkout, a phase touching both repos can validate the
  full stack in one workspace.
- Respect `_lint-toobig` and `_lint-symvision`. The new Rust grammar and the association index are both large enough to
  warrant splitting across focused modules from the start; see `sase/memory/symvision.md` before suppressing anything.
- Read `sase/memory/xprompts.md` before touching any prompt-directive or VCS-workflow behavior, and
  `sase/memory/cli_rules.md` before adding the new subcommand.
- Every phase that changes rendered output must add regression coverage. The highest-value tests are: a prettier
  round-trip that proves a rendered block survives `format_with_prettier` and re-parses to identical logical content; an
  idempotence test proving a second refresh writes nothing; a fallback test proving a store without a hosted remote
  produces unlinked labels rather than broken URLs; and an epic roll-up test proving a parent plan lists its children's
  agents and commits.
- Do not commit anything to the `sase--plans` sidecar as part of implementing `block-contract` through `write-points`.
  The tree-wide rewrite belongs to `reconcile`, where it is a single reviewable batch.
