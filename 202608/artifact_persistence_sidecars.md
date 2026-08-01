---
tier: epic
title: Durable artifact persistence and canonical prompt archive in sidecar repos
goal: "Every artifact a prompt references is captured locally at launch, content-addressed so different bytes never
  overwrite each other, and published beside the prompt when the agent commits. The `<project>--agents` sidecar becomes
  the single canonical home for prompt Markdown files, each one linking inline to its own archived artifacts and
  cross-linking to its plan when it has one, with `sase validate` proving the whole graph.

  "
phases:
  - id: core
    title: Rust prompt-artifact contract and cross-repo header links
    depends_on: []
    size: medium
    description:
      "core: add the Rust-owned staging manifest, pool naming, prompt link rewriting, ARTIFACTS header section, and
      absolute PLAN/PROMPT targets, plus their pyo3 bindings."
  - id: stage
    title: Local .sase/artifacts staging at prompt launch
    depends_on:
      - core
    size: medium
    description:
      "stage: capture every prompt artifact reference into the content-addressed .sase/artifacts pool, move .sase/home
      under it, and record one manifest row per reference."
  - id: archive
    title: Agents sidecar prompt and artifact archive written by sase commit
    depends_on:
      - core
      - stage
    size: medium
    description:
      "archive: write prompts/<YYYYMM>/ and artifacts/<YYYYMM>/ into the agents sidecar at commit time, with inline
      artifact links, month indexes, locking, and outbox retry."
  - id: crosslink
    title: Plan and prompt cross-repo linkage
    depends_on:
      - core
      - archive
    size: medium
    description:
      "crosslink: point plan PROMPT bullets at the agents sidecar, point archived prompts back at their plans, and stop
      writing prompt snapshots into the plans sidecar."
  - id: validate
    title: Validation for the canonical prompt archive
    depends_on:
      - crosslink
    size: medium
    description:
      "validate: add sase agent prompts validate, teach plan links validate about cross-repo prompt links, and wire the
      new check into sase validate."
  - id: migrate
    title: Migrate historical prompts out of the plans sidecar
    depends_on:
      - validate
    size: medium
    description:
      "migrate: move every existing <YYYYMM>/prompts/*.md into the agents sidecar, repair both link directions, and
      leave the plans sidecar with zero prompt files."
  - id: docs
    title: Documentation, sidecar READMEs, and discoverability
    depends_on:
      - migrate
    size: small
    description:
      "docs: refresh sidecar README templates, user docs, and command help so the new artifact and prompt layout is
      discoverable."
proposed_by: bbugyi200.athena.rh
create_time: 2026-08-01 11:05:41
status: wip
---

- **PROMPT:** [202608/prompts/artifact_persistence_sidecars.md](prompts/artifact_persistence_sidecars.md)

# Plan: Durable artifact persistence and canonical prompt archive in sidecar repos

## Outcome

Today a prompt's artifacts are ephemeral. `@~/notes/design.md` is copied into the gitignored `.sase/home/` scratch
directory, where the next agent in that workspace silently overwrites it; `@doc:...`-style references expand to absolute
paths that only exist on one machine; and `@src/foo.py` references vanish entirely once the working tree moves on. The
prompt Markdown that survives lives in `<project>--plans` under `<YYYYMM>/prompts/`, and only for the minority of runs
that produced a plan. Reading a committed prompt six months later means reading dangling references.

After this epic:

- **Every** artifact reference a prompt makes — both `@<kind>:<payload>` artifact references and plain `@path` file
  references — is captured at launch into a content-addressed pool under `.sase/artifacts/`, so two prompts that
  reference the same path with different bytes keep both copies.
- `.sase/home/` moves to `.sase/artifacts/home/`, so one directory holds everything a prompt pulled in.
- When an agent runs `sase commit`, its prompt is written to `prompts/<YYYYMM>/<name>.md` in the `<project>--agents`
  sidecar, and its artifacts are copied to `artifacts/<YYYYMM>/` — literally the directory next to the `prompts/`
  directory holding that prompt.
- The archived prompt reads exactly as authored, except every artifact reference is now a clickable inline Markdown link
  to the archived bytes (or, for files git already stores, to the primary repo blob at the committed SHA).
- `<project>--agents` becomes the canonical and only home for prompt Markdown. Plans link out to their prompt; prompts
  link back to their plan. Runs without a plan still get a prompt — that asymmetry is the whole reason for the move.
- `sase validate` proves the graph: every prompt parses, every artifact link resolves, every digest matches its bytes,
  and every plan/prompt pair agrees in both directions.

## Design

### The three storage tiers

| Tier              | Location                                                  | Lifetime                       | Purpose                                                                   |
| ----------------- | --------------------------------------------------------- | ------------------------------ | ------------------------------------------------------------------------- |
| Working copy      | `.sase/artifacts/home/<path-relative-to-$HOME>`           | Workspace scratch (gitignored) | The readable path the agent actually opens                                |
| Local pool        | `.sase/artifacts/pool/<sha12>-<basename>`                 | Workspace scratch, GC'd        | Immutable content-addressed capture, one entry per distinct byte sequence |
| Published archive | `<project>--agents/artifacts/<YYYYMM>/<sha12>-<basename>` | Forever, in git                | What the committed prompt links to                                        |

`<sha12>` is the first 12 hex characters of the artifact's SHA-256. The digest is the join key across all three tiers,
so publication is a copy with a name that is already correct, and re-publishing the same bytes is a no-op.

### What gets pooled, and what does not

Staging classifies each resolved reference:

1. **VCS-backed.** The resolved path is inside a git checkout SASE knows about (the primary workspace, a linked repo, a
   sidecar) _and_ the file is tracked with clean content. No bytes are copied — git already stores them durably. The
   manifest records `vcs_repo`, `vcs_relpath`, and `sha256`, and the published prompt links to the hosted blob URL at
   the commit being made. This reuses exactly the provenance model `ArtifactFile.is_vcs_backed` already encodes.
2. **External.** Everything else — home-directory files, `~/.sase/artifacts/...` rows, absolute paths outside any
   checkout, materialized VCS cache entries. The bytes are copied into the local pool and later published into the
   sidecar.
3. **Non-file.** `@bug:`, `@commit:`, and `@agent:` references resolve to locators rather than bytes. They are recorded
   in the manifest with `ref_kind` and their resolved locator so the published prompt can still link them, but nothing
   is copied.

Refusing to duplicate what git already stores is what keeps the archive small enough to live in a repo forever.

### Local manifest

`.sase/artifacts/prompt-artifacts.jsonl` is append-only, one JSON object per staged reference:

```json
{
  "schema_version": 1,
  "recorded_at": "2026-08-01T14:22:03Z",
  "agent_artifacts_dir": "/home/bryan/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/01/20260801105157",
  "raw_ref": "@~/Downloads/diagram.png",
  "expanded_ref": "@.sase/artifacts/home/Downloads/diagram.png",
  "ref_kind": "file",
  "label": "diagram.png",
  "source_path": "/home/bryan/Downloads/diagram.png",
  "sha256": "ab12cd34ef56...",
  "size_bytes": 40213,
  "mime_type": "image/png",
  "pool_relpath": "pool/ab12cd34ef56-diagram.png",
  "vcs_repo": null,
  "vcs_relpath": null,
  "locator": null
}
```

Recording **both** `raw_ref` (the token as the user authored it) and `expanded_ref` (the token after preprocessing
rewrote it) is what makes link rewriting a literal, unambiguous string substitution rather than a re-parse. The
canonical published prompt is the _authored_ prompt, so `raw_ref` is the one that normally matches; `expanded_ref` keeps
the rewrite available for flows that archive the submitted prompt instead.

Rows are keyed by `agent_artifacts_dir`, so a workspace reused by many agents keeps every run's provenance distinct even
though they share one pool.

### Published layout

```
<project>--agents/
  agents/<global-name>/…          # unchanged: owned by `sase agent sync`
  users/  families/  assets/      # unchanged
  prompts/<YYYYMM>/<name>.md      # NEW: canonical prompt Markdown
  prompts/<YYYYMM>/README.md      # NEW: generated month index
  artifacts/<YYYYMM>/<sha12>-<basename>   # NEW: published artifact pool
```

`artifacts/` sits next to `prompts/` at the sidecar root, and both are month-sharded exactly like the plans sidecar, so
the two sidecars read as one system. A prompt links its artifacts with `../../artifacts/<YYYYMM>/<file>`.

The two new trees are **not** part of `_AGENTS_PAYLOAD_PATHS`. `sase agent sync` rebuilds `agents/`, `users/`,
`families/`, `README.md`, and `schema.json` as a complete deterministic payload and prunes anything else it owns; the
prompt archive is written incrementally by `sase commit` instead. Disjoint path sets, one shared repo lock.

### Prompt document shape

```markdown
- **PLAN:**
  [202608/artifact_persistence_sidecars.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifact_persistence_sidecars.md)
- **AGENTS:**
  - [bbugyi200.athena.k7--plan](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.k7--plan/README.md)
- **ARTIFACTS:**
  - [diagram.png](../../artifacts/202608/ab12cd34ef56-diagram.png)
  - [src/sase/file_references.py](https://github.com/sase-org/sase/blob/605fbeb.../src/sase/file_references.py)

Can you help me add much better support for artifact persistence…

- Take a look at [@~/Downloads/diagram.png](../../artifacts/202608/ab12cd34ef56-diagram.png) and at
  [@src/sase/file_references.py](https://github.com/sase-org/sase/blob/605fbeb.../src/sase/file_references.py).
```

Two deliberate choices:

- The header block reuses the existing plan-header machinery. `PLAN` and `AGENTS` already exist; only `ARTIFACTS` is
  new. That gives the prompt archive the same parser, renderer, fixed section order, and validation the plans sidecar
  already trusts.
- Inline links keep the original reference token verbatim **as the link text**, `@` included. The prompt still reads
  exactly as the user wrote it and stays greppable, while every reference becomes clickable. Nothing is lost.

### Prompt file naming

`<name>` is the plan slug when the run produced or executed a plan (preserving today's names, so migrated history keeps
its URLs meaningful), otherwise the committing agent's global lane name — for example `bbugyi200.athena.sase-b3.1.md`.
Both are already unique within a month, and the two shapes cannot collide because agent lane names always contain `.`
separators that plan slugs never use. When a name is taken by a _different_ run, the existing `_1`, `_2` disambiguation
convention applies.

One agent has one prompt file no matter how many commits it makes. One plan can therefore have several prompts (planner
plus implementers): the plan's `PROMPT` bullet names the prompt that produced it, and the rest are reachable through the
plan's `AGENTS` bullets.

### Failure policy

Publication is auxiliary and must never fail a primary commit. Every step is wrapped the way
`refresh_committed_plan_header` already is: warn, record, and enqueue into the existing agent publication outbox for
retry on the next commit or sync.

## Phase 1 — Rust prompt-artifact contract and cross-repo header links

Work in `../sase-core` (`crates/sase_core`, `crates/sase_core_py`), because all of this is shared domain behavior any
frontend must agree on.

**New `prompt_artifact` module** in `crates/sase_core/src/`:

- `PromptArtifactRecord` wire type matching the manifest schema above, with `schema_version` gating and a
  `PROMPT_ARTIFACT_MANIFEST_SCHEMA_VERSION` constant.
- `artifact_pool_filename(sha256, original_name) -> String` — `<sha12>-<sanitized-basename>`. Sanitization strips
  directory separators and any character outside `[A-Za-z0-9._-]`, collapses runs, caps the basename length, and
  preserves the extension. Deterministic and injective enough that two different names never collide after sanitization
  within one digest.
- `parse_prompt_artifact_manifest(bytes) -> Vec<PromptArtifactRecord>` and `render_prompt_artifact_record(record)`,
  tolerating unknown forward-compatible fields and skipping malformed lines with a diagnostic rather than failing the
  whole file.
- `select_manifest_records(records, agent_artifacts_dir) -> Vec<PromptArtifactRecord>` — dedupes by `(raw_ref, sha256)`,
  keeps the newest row per reference, and returns them in first-appearance order.
- `rewrite_prompt_artifact_links(prompt, records, resolver) -> PromptArtifactRewrite` — replaces each record's reference
  token with `[<token>](<target>)`. Must skip tokens inside fenced code blocks and existing Markdown links (reuse the
  existing `prompt_literals` / literal-zone helpers), must be idempotent (running it twice produces identical output),
  and returns which records were actually linked so the caller can build the `ARTIFACTS` section from exactly those.

**Plan header block changes** in `crates/sase_core/src/plan/artifact_link.rs`:

- `validate_section_target` currently forces `Plan` and `Prompt` targets to be relative. Both must accept
  `validate_absolute_or_relative_target` instead, so a plan can link a prompt in a different repository and vice versa.
  Relative targets keep working unchanged.
- Add `Artifacts` to `SddPlanHeaderSectionKindWire` as a list-shaped section (alongside `Agents` and `Commits`), with
  its fixed position after `AGENTS` and before `COMMITS` in the canonical order, and `link_shaped() == false`.
- Extend the mutual-exclusion rule check: `PLAN` and `PROMPT` remain mutually exclusive; `ARTIFACTS` is compatible with
  either.

**Bindings** in `crates/sase_core_py/src/lib.rs`: `prompt_artifact_pool_filename`, `prompt_artifact_manifest_parse`,
`prompt_artifact_manifest_render_record`, `prompt_artifact_manifest_select`, `prompt_artifact_rewrite_links`, and a
`prompt_artifact_wire_schema_version`. Follow the existing `py_artifact_ref_*` naming and error-mapping style.

**Tests:** Rust unit tests for pool naming (unicode, no extension, very long names, path traversal attempts), manifest
round-trips, rewrite idempotency, fenced-block and existing-link skipping, and header-block tests covering absolute
`PLAN`/`PROMPT` targets plus `ARTIFACTS` parse/render/upsert ordering.

Bump the crate version and publish/point the Python binding at it per the repo's normal `sase-core` release flow, then
update `pyproject.toml`'s `sase-core-rs` pin in this repo so later phases can call the new bindings.

## Phase 2 — Local `.sase/artifacts` staging at prompt launch

**New module `src/sase/core/prompt_artifact_staging.py`** exposing:

```python
def stage_prompt_artifact(
    *, raw_ref, expanded_ref, resolved_path, ref_kind, label, locator=None,
    workspace_root=None, agent_artifacts_dir=None,
) -> PromptArtifactRecord | None
```

It classifies the reference (VCS-backed / external / non-file), hashes external bytes, copies them into
`.sase/artifacts/pool/<name-from-Rust>` when absent, and appends one manifest row under an `fcntl` lock — mirroring the
locking already used by `sase.core.artifact_file_explicit`. It returns `None` and logs at debug level for anything it
cannot classify; staging must never break a launch.

Reuse rather than reinvent: `sase.core.artifact_file_helpers.hash_file` / `artifact_file_mime_type`,
`sase.core.artifact_capture_policy` for the maximum capture size, and `sase.repo_inventory` for the checkout set used to
detect VCS-backed paths. Skip pooling (but still record a manifest row, flagged `skipped_reason: "too-large"`) when a
file exceeds the configured cap.

**Wire into both reference processors:**

- `src/sase/artifact_ref_prompt.py`: `_expand_artifact_references` already computes
  `(parsed, resolution, resolved_path)` tuples for `_record_artifact_ref_consumption`. Stage from that same list, so
  artifact-reference capture and consumption telemetry cannot drift. Pass `render_artifact_ref(parsed)` as `raw_ref` and
  the rewritten text as `expanded_ref`.
- `src/sase/file_references.py`: `process_file_references` stages every entry in `parsed.absolute_paths` **and** every
  existing relative path (the VCS-backed case that is invisible today). Home-directory files keep their readable working
  copy, now at `.sase/artifacts/home/<rel>`.

**Move `.sase/home/` to `.sase/artifacts/home/`:** update the destination, the `print_status` text, and the docstrings
in `process_file_references`. Do not delete a pre-existing `.sase/home/` — a live agent in that workspace may still be
reading it; instead add a `sase doctor` check that reports and offers to remove stale `.sase/home/` directories. Update
`tests/test_file_references_invoke.py`, `tests/test_file_references_substitution.py`, and the `.sase/`-prefix
expectations in `tests/history/test_file_references.py`.

**Local pool GC:** add `prune_prompt_artifact_pool(workspace_root)` that drops pooled files whose only manifest rows
belong to runs that are terminal and already published, bounded by a config knob next to the existing artifact-capture
settings. Call it opportunistically from staging when the pool exceeds its budget, never on the hot path of a single
reference.

**Tests:** same bytes referenced twice yields one pool entry and two manifest rows; different bytes at the same path
yield two pool entries; a tracked in-repo file produces a VCS-backed row with no pool copy; an untracked or dirty
in-repo file falls back to a pool copy; `is_home_mode=True` still stages nothing; oversized files are recorded but not
copied; concurrent staging from two processes produces a well-formed manifest.

## Phase 3 — Agents sidecar prompt and artifact archive written by `sase commit`

**New package `src/sase/agents_sync/prompt_archive/`:**

- `paths.py` — `prompts_month_dir(repo, yyyymm)`, `artifacts_month_dir(repo, yyyymm)`,
  `prompt_document_path(repo, yyyymm, name)`, and the relative-link helper that yields
  `../../artifacts/<YYYYMM>/<file>`.
- `naming.py` — the plan-slug-or-agent-lane rule and `_1`/`_2` disambiguation, with a pure `resolve_prompt_name(...)`
  that takes the existing directory listing so it is testable without a repo.
- `render.py` — builds the document: reads `raw_xprompt.md` from the agent's artifact dir, calls the Rust rewrite with
  the run's manifest records, then upserts `PLAN`, `AGENTS`, and `ARTIFACTS` header sections via
  `sase.sdd.plan_header_block`. Formats through `format_with_prettier` so the archive matches every other saved Markdown
  artifact.
- `publish.py` — `publish_prompt_archive(agent_name, primary_revision, *, project, commit_cwd)`: resolves the agents
  target through `resolve_sync_targets`, copies pooled artifacts into `artifacts/<YYYYMM>/` (skipping any that already
  exist with a matching digest), writes the prompt document, regenerates the month `README.md` index, and commits with
  `git add -- prompts artifacts` under the existing agents-sidecar write lock, pushing asynchronously.
- `index.py` — the generated `prompts/<YYYYMM>/README.md`: a table of prompt name, title/first line, plan link, agent
  link, and artifact count. Deterministic ordering so it never churns.

Hosted URLs for the primary-repo blob links and the plans-sidecar plan link come from
`sase.sdd.hosted_links.HostedLinkResolver`, which already memoizes remote/branch lookups; degrade to an unlinked label
rather than a guessed URL when a remote is unresolvable, exactly as that module already does.

**Integration:** call `publish_prompt_archive` from `run_agent_publication_step` in
`src/sase/workflows/commit/workflow_publication.py`, guarded by its own `completed_steps` marker
(`publish_prompt_archive`) so a resumed commit does not redo it. Order it before `refresh_committed_plan_header` so the
plan header refresh in Phase 4 can point at a prompt that already exists. Wrap it the same way bead-page publication is
wrapped: catch, `print_status(..., "warning")`, continue.

**Tests:** a commit with no artifacts still publishes a prompt; a commit with home, VCS-backed, and non-file references
publishes exactly the right files and links; publishing twice is byte-identical; a missing agents checkout degrades to a
warning and leaves the primary commit successful; the month README is stable across reruns.

## Phase 4 — Plan and prompt cross-repo linkage

**Plans point at the agents sidecar.** `sase.sdd.plan_header_writes.project_plan_header_sections` and
`archived_prompt_path_for_plan` currently assume the prompt is a sibling file under `plans/<YYYYMM>/prompts/`. Replace
that with a hosted `PROMPT` target resolved through `HostedLinkResolver`, with a label of the form
`prompts/<YYYYMM>/<name>.md`. `sase.sdd.plan_header_refresh.refresh_committed_plan_header` gains the same projection so
the link appears on the first commit after a prompt is archived, and `sase.sdd.plan_links_refresh` reconciles it
tree-wide.

**Prompts point back at plans.** `render.py` from Phase 3 writes the `PLAN` section using the hosted plans-sidecar URL
for the resolved plan, and omits the section entirely for runs with no plan. `sase.sdd.artifact_links` grows a
symmetrical `update_source_aware_artifact_link` path for cross-repo targets rather than a second, parallel
implementation.

**Stop writing prompts into the plans sidecar.** `sase.sdd._write.write_sdd_spec` and `write_sdd_files` no longer create
`plans/<YYYYMM>/prompts/<name>.md`. The planner's expanded prompt (`expand_prompt_for_spec`) is instead handed to the
prompt-archive writer, so plan approval publishes the planner's prompt to the agents sidecar. `write_sdd_files` keeps
returning a `(prompt_path, plan_path)` pair for its callers in `src/sase/axe/run_agent_exec_plan_accept.py`, but
`prompt_path` now names the agents-sidecar destination. `set_prompt_qa` / `update_prompt_with_qa` follow the prompt to
its new home, and the Q&A auto-commit path in `src/sase/llm_provider/commit_finalizer_git.py` — whose
`_EXTERNAL_SDD_PROMPT_PATTERN` hardcodes `\d{6}/prompts/[^/]+\.md` against SDD repos — is retargeted at
`prompts/\d{6}/[^/]+\.md` in the agents repo.

**Tests:** plan approval with and without a plan; a plan whose prompt link survives `plan links refresh`; Q&A updates
landing in the agents sidecar; `format_sase_plan_reference` unaffected.

## Phase 5 — Validation for the canonical prompt archive

**New `sase agent prompts` command group** (`list`, `migrate`, `show`, `validate`; bare invocation delegates to `list`
through the central `_default_list_subcommands()` wiring). Alphabetical placement between `names` and `retire-v1`, every
long option given a short alias, colored output, and worked examples in the epilog.

`sase agent prompts validate` walks the agents sidecar `prompts/` tree and reports:

| Code                 | Severity | Meaning                                                                            |
| -------------------- | -------- | ---------------------------------------------------------------------------------- |
| `prompt-parse`       | error    | Header block or frontmatter does not parse                                         |
| `artifact-missing`   | error    | An `ARTIFACTS` entry or inline link names a file absent from `artifacts/<YYYYMM>/` |
| `artifact-digest`    | error    | A published artifact's bytes do not hash to the digest in its filename             |
| `plan-unresolved`    | warning  | The `PLAN` target cannot be resolved in a locally available plans checkout         |
| `artifact-orphan`    | warning  | A file in `artifacts/<YYYYMM>/` is referenced by no prompt                         |
| `prompt-unpublished` | warning  | A local manifest names a run whose prompt was never published                      |

Warnings only when the counterpart repo is not cloned — a machine without the plans sidecar must still validate clean.
Support `--json`/`-j`, `--month`/`-m`, and `--show-warnings`/`-s`, matching `sase plan links validate`.

**Teach `sase plan links validate` about the move.** In `src/sase/sdd/_link_validation.py` and
`src/sase/sdd/_link_support.py`:

- A plan carrying a well-formed `PROMPT` section is paired, so it must not produce `unpaired-file`; that is what keeps
  the existing 537 warnings from exploding into thousands once prompts leave the tree.
- `resolve_link_path` must return `None` — not a bogus path — for absolute URL targets, and callers must treat "external
  target" as valid-but-unresolvable rather than broken.
- `PROMPT_KINDS` discovery of in-tree `prompts/` directories stays in place until Phase 6 completes, so validation
  passes at every intermediate commit; Phase 6 removes it.
- Add `prompt-in-plans-store` (warning before Phase 6, error after) for any prompt Markdown still living in the plans
  sidecar, which is what enforces "canonical and _only_ location".

**Wire into `sase validate`:** add `_ValidationCheck("agent prompts validate", ("agent", "prompts", "validate"))` to
`_CHECKS` in `src/sase/main/validate_handler.py`, after `plan links validate`.

**Tests:** each diagnostic code fires exactly once on a purpose-built fixture tree; a clean tree exits 0; a machine
without the plans checkout downgrades `plan-unresolved` to a warning; `sase validate` aggregates the new check's exit
code.

## Phase 6 — Migrate historical prompts out of the plans sidecar

`sase agent prompts migrate` moves every `<YYYYMM>/prompts/*.md` from the plans sidecar into the agents sidecar. Follow
`sase plan links refresh`'s convention exactly: **read-only report by default**, `--write`/`-w` to apply, plus
`--json`/`-j`, `--month`/`-m`, and `--project`/`-p`.

Per prompt file, in one batched transaction per repository:

1. `git mv`-equivalent into `prompts/<YYYYMM>/<same-name>.md`, preserving the slug so existing links and muscle memory
   keep working.
2. Rewrite its `PLAN` bullet from the relative `../<name>.md` form to the hosted plans-sidecar URL.
3. Rewrite the paired plan's `PROMPT` bullet to the hosted agents-sidecar URL.
4. Leave `ARTIFACTS` absent — historical prompts have no captured artifacts, and inventing them would be a lie.

Requirements: idempotent (a second run reports zero changes), resumable (crash-safe per month), and it must take both
sidecar write locks in a fixed order (plans, then agents) to avoid deadlocking against a concurrent `sase commit`.
Report a summary table of prompts moved, plans relinked, and files skipped with reasons. Use the existing
`sase.sdd._git_contention.store_git_write_lock` and `sase.agents_sync.git_sync_ops` locking rather than new primitives.

After the migration lands, remove the now-dead in-tree prompt discovery: `PROMPT_KINDS` handling in
`src/sase/sdd/_link_files.py` and `_link_support.py`, the `prompts` branch of `list_sdd_files`, and flip
`prompt-in-plans-store` from warning to error.

**Tests:** a fixture plans tree with month-nested and legacy flat prompts migrates completely; re-running is a no-op; an
unpaired prompt (no plan) still migrates; a prompt whose plan is missing reports rather than crashes; `sase validate`
passes before, during (partial), and after.

## Phase 7 — Documentation, sidecar READMEs, and discoverability

- Update `src/sase/sdd/templates/sidecar-plans-README.md` and `src/sase/sdd/templates/README.md`, which currently
  document `<YYYYMM>/prompts/*.md` as the prompt home, to point at the agents sidecar instead.
- Add a prompt-and-artifact-archive section to the agents sidecar root README generated by
  `src/sase/agents_sync/rendering_index_pages.py`, describing `prompts/`, `artifacts/`, and the digest naming scheme.
- Document `.sase/artifacts/` (working copies, pool, manifest, GC) in the user docs under `docs/`, including the
  `.sase/home/` → `.sase/artifacts/home/` move and what to do with a leftover `.sase/home/`.
- Add a `CHANGELOG.md` entry.
- Review `sase artifact --help` and the `sase_artifact_file` skill text for statements the new layout invalidates.

Do **not** edit anything under `sase/memory/`, `AGENTS.md`, or the generated provider shims. If a memory update looks
warranted, file a `sase bead create -T task` proposing it and leave the decision to the project owner.

## Cross-cutting requirements

- **Rust boundary.** Digest naming, manifest schema, link rewriting, and header-block grammar are core backend behavior
  and belong in `../sase-core`. Python is an adapter. Do not reimplement any of them locally, and do not let a Python
  fallback shadow a missing binding.
- **Determinism.** Every generated file — prompt documents, month indexes, artifact filenames — must be byte-identical
  across reruns on the same inputs. No timestamps in generated bodies beyond what the manifest already records.
- **No silent data loss.** Staging, publication, and migration all refuse to overwrite differing bytes at a
  content-addressed path; a digest collision with different content is a hard error, not a clobber.
- **Never break the primary commit.** Every new step on the `sase commit` path is best-effort with a warning and outbox
  retry.
- **Privacy.** The agents sidecar is already consent-gated for publication. Artifact capture must honor the same gate:
  if a project or run is not consented for publication, stage locally but publish nothing. Verify against the existing
  consent checks in `sase.agents_sync` before writing the first byte to a sidecar.
- **Every phase leaves `sase validate` passing** and `just check` clean, including Symvision lint. Run `just install`
  before `just check` in a cold workspace.
