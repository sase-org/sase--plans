---
tier: tale
title: Reference links and Referenced By write-back
goal:
  Published prompts cite artifacts as numbered `[@kind:arg][N]` reference links whose
  destinations are pinned to the revision the agent actually read, and every cited
  artifact-repo document gains a managed `Referenced By` table plus a structured
  `.sase/referenced-by/` index, reconciled through a durable outbox that never fails or
  delays the agents publication it follows.
size: medium
proposed_by: bbugyi200.athena.sase-js.6
bead: sase-js.6
create_time: 2026-08-12 07:54:30
status: done
---

- **PROMPT:**
  [prompts/202608/reference_links_and_referenced_by.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/reference_links_and_referenced_by.md)
- **PARENT:**
  [202608/artifact_ref_contract.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifact_ref_contract.md)
- **BEAD:**
  [sase-js.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-js/sase-js.6.md)
- **AGENTS:**
  - bbugyi200.athena.sase-js.6
- **COMMITS:**
  - [9907b1d](https://github.com/sase-org/sase/commit/9907b1d1611bb397d21237367c95acd4b5578f00)
    — feat(agents-sync): write referenced-by links for prompt artifacts

# Plan: Reference links and `Referenced By` write-back

Phase `linking` of epic `sase-js` (bead `sase-js.6`), whose plan is
`plans:202608/artifact_ref_contract.md` — §3.7, §3.8 and §4.6. Depends on `builtins`
(`sase-js.4`) and `files` (`sase-js.5`), both landed. Siblings `ace` (`sase-js.7`) and
`adopt` (`sase-js.9`) are in flight in parallel workspaces; this plan touches no ACE
module and no `docs/` page, so the land agent has nothing to reconcile with them.

## 1. Goal

Three things, in the order publication performs them:

1. **Reference-style links.** `rewrite_prompt_artifact_links` stops emitting inline
   `[@kind:arg](url)` and emits `[@kind:arg][N]`, allocating `N` through the shared
   `markdown_link_refs` allocator that phase `core` landed, and appending the matching
   `[N]: dest` definitions at the bottom of the published prompt.
2. **Correct destinations.** Every kind in epic §3.7's table gets its destination, and
   the VCS revision in a destination is the one pinned when the ref resolved, never HEAD
   at publication.
3. **`Referenced By` write-back.** Each cited artifact-repo document gains the managed
   bottom block that phase `core` landed in `referenced_by.rs`, backed by a structured
   `.sase/referenced-by/` index in the same repo, reconciled by a durable per-project
   outbox drained _after_ the agents lock is released, with back-reference commits
   carrying a `referenced_by` cause that user `file_hooks` skip by default.

**Non-goals.** The Files pane and dynamic Artifacts sub-tabs (phase `ace`); config,
glossary, and the documentation rewrite (phase `adopt`, which owns `docs/sdd.md`'s
`Referenced By`-versus-`AGENTS` section and `docs/agents_sidecar.md`); backfilling
back-references for historically published agents and rewriting historical agent pages
to reference style, both explicitly out of scope in epic §5 and §8; provider-declared
`Referenced By` columns (see §6).

## 2. Verified current state

Read in this workspace at `62951abcb`, in the linked `sase-core` checkout at `a7d5c9e`,
and in the external `gh:sase-org/sase-research` checkout. Open both with `/sase_repo`
(`sase repo open sase-core`, `sase repo open gh:sase-org/sase-research`). Dev installs
build the extension from `sase/repos/linked/sase-core`, so unreleased core commits are
expected and the published window in `pyproject.toml` is ratcheted separately at release
time.

**Every Rust primitive this phase needs already exists and none of it is called from
Python.** `crates/sase_core/src/markdown_link_refs.rs` has
`scan_markdown_reference_links` (definitions first-wins, plus numeric uses in full,
collapsed, and shortcut forms, all outside `prompt_literal_zone_ranges`),
`allocate_markdown_reference_label` (reuse-by-destination, otherwise lowest free
integer, counting definitions _and_ uses _and_ labels assigned this run), and
`append_markdown_reference_definitions` (numeric order, one blank line, idempotent,
trailing-newline shape preserved). `crates/sase_core/src/referenced_by.rs` has
`render_referenced_by_block`, `parse_referenced_by_block`, `upsert_referenced_by_block`,
`remove_referenced_by_block`, and `strip_referenced_by_block`, with markers
`<!-- sase:referenced-by:start -->` / `:end`, heading `## Referenced By`, deterministic
row sort over the declared column order, a 50-row cap with an `_… and N more_` line, and
marker recovery. All ten are exported from `lib.rs` and bound in
`crates/sase_core_py/src/lib.rs` as `markdown_reference_links_scan`,
`markdown_reference_label_allocate`, `markdown_reference_definitions_append`,
`referenced_by_block_{parse,render,upsert,remove,strip}` plus the two
`*_wire_schema_version` accessors.
`grep -rn "referenced_by_block\|markdown_reference" --include=*.py src/` returns
nothing: this phase is their first caller.

**The rewrite is inline-style and its shared helper has no notion of a label.**
`prompt_artifact.rs::rewrite_prompt_artifact_links` builds one `PromptLinkCandidate` per
`(record, raw_ref | expanded_ref)` and hands them to
`prompt_rewrite.rs::rewrite_prompt_links`, whose only emission is `[{token}]({target})`.
`rewrite_prompt_links` is `pub(crate)` and `rewrite_prompt_artifact_links` is its only
caller. `markdown_link_ranges` inside it already protects both `[x](y)` **and**
`[x][y]`, which is what makes a reference-style second pass a no-op.

**Publication resolves some destinations at publication time, which is the HEAD-drift
bug.** `agents_sync/prompt_archive/preparation.py::_ArtifactTargetResolver.__call__`
uses `self.primary_revision` (the commit being published, not the one the agent read)
for a record whose repo root is the primary checkout, and `_git_revision(root, ...)` — a
live `git rev-parse HEAD` — for every other repo. `PromptArtifactRecord` has no revision
field to pin instead.

**Three of the seven destination rows are missing.** The resolver handles pooled digests
(`artifact_object_prompt_link`), clean VCS blobs, `http(s)` locators,
`ref_kind == "commit"`, `ref_kind == "bug"`, and `ref_kind == "agent"`. There is no
`stitch`, no `patch`, and no `bead` branch. Since phase `builtins`,
`prompt_artifact_staging._NON_FILE_REF_KINDS` is
`{"agent", "bug", "commit", "patch", "stitch"}`, so a `@stitch` or `@patch` ref reaches
publication as a locator-only row and silently resolves to `None` — tracked, but
unlinked. Locators are `<repo>@<full_sha>` for stitch (`builtin_entry_stitch.py:98`),
`<project>/<name>` for patch (`builtin_entry_patch.py:70`), and `<project>/<bead_id>`
for bead (Rust `resolve_bead`, `artifact_ref/mod.rs:985`). `bead` is _not_ in
`_NON_FILE_REF_KINDS`, so a bead ref stages as a file in the beads sidecar and currently
links as a blob URL.

**`HostedLinkResolver` already resolves every URL shape needed.** `sdd/hosted_links.py`
exposes `plan_url`, `prompt_url`, `agent_url`, `bead_url`, `commit_url`,
`commit_url_for_repository`, and `blob_url_for_repository`, all best-effort and
`None`-returning rather than guessing.

**The use manifest exists, is per-occurrence, and is unpublished.**
`core/artifact_ref_uses.py` appends one `ref-uses.jsonl` row per ref occurrence into
`$SASE_ARTIFACTS_DIR` under `fcntl`, written by
`artifact_ref_prompt.py::_record_artifact_ref_uses`. Its module docstring says outright
that "the publication step that later reads it into `agents/<agent>/ref-uses.json`
belongs to a future phase". The wire (`artifact_ref/uses.rs`, schema 1) carries
`recorded_at`, `agent_name`, `raw_ref`, `canonical_ref`, `ref_kind`, `stable_id`,
`prompt_text`, and the always-`None`-today `publication_target` and `captured_file`.
`raw_ref` on a use row is the same scanned token that `stage_prompt_artifact` records,
so the two manifests join on `raw_ref`.

**The outbox machinery is a clean template, and it is single-purpose.**
`agents_sync/publication_outbox{,_models,_operations,_serialization,_store}.py` is a
per-project `~/.sase/projects/<key>/agents-publication-outbox.json` with an `fcntl`
lock, an atomic whole-document write, a logical key, attempts, quarantine, and a
terminal state. Its item type is entirely about `(global_agent, primary_revision)` hood
publication; `validate_publication_item` and `publication_sort_key` hardcode those
fields.

**Both lock holders are already disjoint.** `prompt_archive/publish.py` takes only
`git_sync.bounded_agents_lock` on the agents sidecar;
`sdd/plan_links_refresh.py::refresh_plan_links` takes only
`store_git_write_lock(store.repo_root_for_kind("plans"), mutates_worktree=True)`.
Nothing takes both. `git_sync.py::sync_projects` drains the publication outbox around
`_sync_project`, which is where a second drain belongs.

**`refresh_plan_links` is the shape to copy, not the function to extend.** It resolves
the role root from the `SddStore`, locks, re-renders each document, writes, formats with
prettier, and emits one batched `commit_sdd_store_files(..., already_locked=True)` with
a structured report of typed actions and issues. What it reconciles — the `AGENTS`
section — comes from `sdd/associations/_artifacts.py`, i.e. agents whose artifact
records name the plan as the plan they _worked on_. Nothing in that path reads artifact
consumption or ref uses, so a `@plan:` citation already cannot add an `AGENTS` entry;
that invariant needs a regression test, not a fix.

**File hooks have no cause.** `config/file_hooks.py::FileHookEvent` carries `project`,
`repo_kind`, `sidecar_role`, `rel_path`, `op`, `agent_name`; `FileHookFilters` carries
`projects`, `sidecars`, `path_globs`, `agent_name_globs`, `ops`.
`_FILE_HOOK_FILTER_KEYS` rejects anything else, and `hook_matches_event` has no cause
test. `sdd/_commit_store.py::_emit_sdd_file_hooks` calls `emit_commit_file_hook_events`
unconditionally after every store commit, which is exactly the path a back-reference
commit would take.

**The shipped research provider declares no `Referenced By` columns.**
`gh:sase-org/sase-research`'s `RESEARCH_REF_PROVIDER_SPEC` has
`"publication": {"link": "vcs_permalink", "referenced_by": "markdown_table"}` and
`"identity": {}`. `provider_spec.rs` validates `publication.referenced_by` against
`["markdown_table", "none"]` and has no column vocabulary at all.

**Binding gates.** `tools/validate_sase_core_rs` has a hand-maintained
`REQUIRED_BINDINGS` tuple that phase `files` extended (`artifact_object_relpath`,
`artifact_ref_file_row_{render,validate}`); no `referenced_by_*` or
`markdown_reference_*` name is in it yet. `tools/check_sase_core_rs_bindings` derives
its list statically from `require_rust_binding` literals, so new bindings must be called
with literal names.

## 3. Design decisions

### 3.1 Labels are allocated at match time, not up front

Allocating a label for every resolved record before rewriting would burn numbers on
records whose token never appears in the prompt (a ref inside a literal zone, or one
whose expansion replaced the token), leaving gaps that violate "the lowest positive
integer not used by a different link or definition in that document". So
`rewrite_prompt_links` gains a replacement _builder_ callback invoked only when a
candidate actually matches, and `rewrite_prompt_artifact_links` allocates inside it. Two
records sharing one destination therefore share one label, which is exactly what
`allocate_markdown_reference_label` already does.

### 3.2 The revision is a manifest field, captured at staging

Pinning has to survive the workspace, the process, and a deferred republication, so it
belongs in the append-only prompt-artifact manifest rather than being recomputed.
`PromptArtifactRecord` gains `vcs_revision: Option<String>`, filled by
`_clean_vcs_provenance` with one extra `git rev-parse HEAD` at staging time — the same
place that already runs four git probes per candidate. Publication prefers it and falls
back to today's behavior only for historical rows with no revision, so old archives keep
rendering. This is additive and `#[serde(default)]`, so the manifest schema stays at 2.

### 3.3 The back-reference queue is a sibling outbox, not a widened one

`AgentPublicationOutboxItem`'s identity, validation, and sort key are all about
`(global_agent, primary_revision)` hood publication; a back-reference request is per
`(agent, publication, artifact-repo role, artifact id)` and carries a table row. Adding
a second shape to that item would make `validate_publication_item` conditional and the
logical key polymorphic. A sibling `~/.sase/projects/<key>/referenced-by-outbox.json`
reuses the proven store, lock, retry, quarantine, and terminal-retirement _pattern_
verbatim while keeping each queue's invariants exact. That is what epic §4.6's "route it
through `publication_outbox*`" buys: durability and quarantine, not a shared row type.

### 3.4 The drain runs outside the agents lock, always

Epic §4.6 fixes the lock order as artifact repos first, `agents` last. This plan
achieves it structurally rather than by discipline: the enqueue happens inside the
agents lock (it only writes the project outbox JSON), and the drain runs after the
`with bounded_agents_lock(...)` block has exited, so no code path ever holds the agents
lock while asking for an artifact-repo lock. A concurrent `sase plan links --write`
therefore cannot deadlock against a publication.

### 3.5 Columns are a v1 default, and the artifact id is the canonical ref

Epic §3.8 says columns come from the provider spec, but the spec wire shipped in phase
`core` has no column vocabulary and the already-released `sase-research` spec declares
none. Rather than change a validated wire and a published plugin from inside this phase,
the default row builder supplies epic §3.8's exact column set —
`Agent | Project | Reference | Published | Uses`, `Uses` numeric, `Agent` linked — for
every provider whose `publication.referenced_by` is `markdown_table`. Provider-declared
columns become a recorded follow-up (§6).

Artifact identity is `<provider>:<repo-relative-posix-path>`, which is also the
`canonical_ref` the prompt cites, and the index file mirrors it as
`.sase/referenced-by/<provider>/<repo-relative-path>.json`. That is filesystem-safe by
construction (the path is already a valid repo path) and human-greppable. When the
provider declares an `identity.property` the index file also records the identity value,
so a future rename can be resolved by scanning; with no identity property a document
that moved is indistinguishable from one that was deleted, and epic §4.6's rule applies
— a visible retryable outbox error, never a guessed path.

### 3.6 The managed block is excluded from the _clean-VCS_ digest only

Epic §3.8 forbids a citation from redefining an artifact's semantic version. The
concrete leak is `stage_prompt_artifact`, which records `sha256 = hash_file(source)` for
every file-backed ref including artifact-repo documents. For a **clean tracked**
document the record has no `pool_relpath`, nothing is copied, and the digest is pure
metadata — so it is computed over `strip_referenced_by_block(text)` and a citation
therefore does not change it. For a **dirty or untracked** document the bytes _are_
copied into the content-addressed store and the digest must match those bytes exactly or
`_copy_content_addressed` will refuse the object; there the raw digest is kept and the
snapshot legitimately includes whatever block was on disk. Reconciliation itself uses
`strip_referenced_by_block` for its "did this document actually change" comparison, so a
re-render that only reorders nothing is a no-op.

### 3.7 A cause, not an incidental filter

`FileHookEvent` gains `cause: str = "user"` and `FileHookFilters` gains
`causes: tuple[str, ...] | None`. `hook_matches_event` rejects any event whose cause is
not `"user"` unless the hook lists it. Excluding by default is the point: Bryan's
`research-highlights` hook happens to filter `ops: [ADD]` while back-references are
`MODIFY`, and epic §3.8 says relying on that coincidence would make the feature unsafe
for the next hook anyone writes.

## 4. Implementation

### 4.1 sase-core: reference-style rewriting and a pinned revision

Open the repo with `/sase_repo` first. All Rust below lands in `crates/sase_core`.

**`prompt_rewrite.rs`** — replace `PromptLinkCandidate`'s `target: &'a str` use at the
emission site with a caller-supplied builder:

```rust
pub(crate) fn rewrite_prompt_links<F, R>(
    prompt: &str,
    record_count: usize,
    candidates: &[PromptLinkCandidate<'_>],
    valid_start: F,
    mut replacement: R,      // (record_index, token, target) -> String
) -> (String, Vec<bool>)
```

The builder is called only for a candidate that actually matched, at the moment it
matches, in document order. Everything else — protected ranges, longest-token
preference, `valid_start` — is unchanged.

**`prompt_artifact.rs`**

- `PromptArtifactRecord` gains `#[serde(default)] pub vcs_revision: Option<String>`.
  `PROMPT_ARTIFACT_MANIFEST_SCHEMA_VERSION` stays `2`.
- `PromptArtifactRewrite` gains
  `#[serde(default)] pub reference_definitions: Vec<MarkdownReferenceDefinitionWire>`
  and `#[serde(default)] pub reference_labels: Vec<PromptArtifactReferenceLabelWire>`,
  where the new struct is `{ raw_ref: String, label: String, destination: String }`.
- `rewrite_prompt_artifact_links` now: scans the incoming prompt once with
  `scan_markdown_reference_links`; keeps a `BTreeMap<String, String>` of labels assigned
  this run; in the replacement builder calls
  `allocate_markdown_reference_label(&scan, target, &assigned)`, inserts the pair, and
  returns `format!("[{token}][{label}]")`; and after the rewrite appends the definitions
  for exactly the assigned labels through `append_markdown_reference_definitions`.
  `reference_labels` is emitted in linked-record order.

**`crates/sase_core_py/src/lib.rs`** — `prompt_artifact_rewrite_links` gains the two new
result keys; `prompt_artifact_records_from_py_list` and the record serializer carry
`vcs_revision`. Update the module doc-comment binding list.

**Rust tests** (in-module): first run emits `[@kind:arg][1]` plus one bottom definition;
a second run over the output is byte-identical and reports no linked records; two
records sharing a destination share one label and emit one definition; a document with a
pre-existing `[1]: …` definition and a dangling `[x][3]` use allocates `2`; a
pre-existing definition whose destination already matches is reused rather than
duplicated (CommonMark first-definition-wins); tokens inside code fences, inline code,
and existing `[x](y)` / `[x][y]` links are untouched; a record whose token never appears
consumes no label; the trailing-newline shape of the prompt is preserved.

### 4.2 sase: destinations and revision pinning

**`src/sase/core/prompt_artifact_staging.py`**

- `PromptArtifactRecord` (TypedDict) gains `vcs_revision: str | None`; `_base_record`
  initializes it to `None`.
- `_clean_vcs_provenance` returns `(repo_name, relpath, revision)`, adding one
  `git -C <root> rev-parse HEAD` probe alongside the existing tracked/clean probes.
  `stage_prompt_artifact` stores it in `record["vcs_revision"]`.
- Per §3.6, when a clean-VCS record's source is a Markdown document, record `sha256`
  over `referenced_by_block_strip(text)` rather than `hash_file(source)`. The
  dirty/untracked and pooled paths are unchanged.

**`src/sase/agents_sync/prompt_archive/preparation.py`** —
`_ArtifactTargetResolver.__call__` gains, in this order before the existing VCS branch:

| `ref_kind`            | Destination                                                                             |
| --------------------- | --------------------------------------------------------------------------------------- |
| `stitch` (+ `commit`) | split `locator` on the last `@`; `hosted.commit_url_for_repository(repo_root, sha)`     |
| `bead`                | split `locator` on the last `/`; `hosted.bead_url(bead_id)`                             |
| `patch`               | the Patch's `PR:` URL when its ProjectSpec records one, else `None` (tracked, unlinked) |

and, in the VCS branch, prefers `record["vcs_revision"]` over both `primary_revision`
and `_git_revision(root, ...)`, keeping today's behavior only when the field is absent.
`bead` is placed before the VCS branch deliberately: epic §3.7's table names the hosted
bead page, and a bead's page is a live projection of mutable state, so pinning a blob
would name bytes that no longer describe the bead.

`patch` resolution reads the in-context project's `<key>.sase` / `<key>-archive.sase`
through the existing Patch store facade and never searches other projects. There is no
published Patch page today, so "else the published Patch/agent page" in epic §3.7
degrades to unlinked; the ref still gets a use row and an `ARTIFACTS` entry, per epic
§3.7's "tracking must never depend on linkability".

**`src/sase/agents_sync/prompt_archive/render.py`** — `RenderedPromptArchive` gains
`reference_labels: tuple[Mapping[str, str], ...]` taken straight from the rewrite
payload, so `preparation.py` can hand the outbox the exact destination each ref was
published under. The `ARTIFACTS` header section keeps its existing inline-link shape and
its existing `resolve_target` source, which is what keeps it in sync with the numbered
body links.

### 4.3 sase: the referenced-by outbox

**`src/sase/agents_sync/referenced_by_outbox_models.py`** — mirrors
`publication_outbox_models.py`:

```python
REFERENCED_BY_OUTBOX_SCHEMA_VERSION = 1

@dataclass(frozen=True, slots=True)
class ReferencedByOutboxItem:
    project_key: str
    project: str
    global_agent: str
    agent_url: str | None
    primary_revision: str
    sidecar_role: str
    provider: str
    artifact_id: str          # "<provider>:<repo-relative-path>"
    repo_relpath: str
    identity_value: str | None
    canonical_ref: str
    destination: str | None
    uses: int
    published_date: str       # YYYY-MM-DD, UTC
    attempts / last_error / quarantined / quarantined_at / terminal / terminal_reason
    created_at / updated_at
```

`logical_key` is `(global_agent, primary_revision, sidecar_role, artifact_id)`; `id` is
a sha256 over that tuple plus `project_key`. `validate_referenced_by_item` requires
every identity field non-empty and `uses >= 1`.

**`..._store.py` / `..._operations.py`** — byte-for-byte the same pattern as their
publication counterparts, against `referenced-by-outbox.json`:
`list_referenced_by_requests`, `mutate_referenced_by_outbox`,
`enqueue_referenced_by_request`, `update_referenced_by_requests`,
`acknowledge_referenced_by_requests`, `clear_quarantined_referenced_by_requests`,
`drop_terminal_referenced_by_requests`, `referenced_by_quarantine_diagnostics`.
`src/sase/agents_sync/referenced_by_outbox.py` is the façade, matching
`publication_outbox.py`.

**`src/sase/agents_sync/referenced_by_planning.py`** — turns one prepared archive into
requests:

- Read `<agent_artifacts_dir>/ref-uses.jsonl` with `read_artifact_ref_uses` and count
  occurrences per `raw_ref` for this agent; a missing or unreadable manifest degrades to
  a count of 1 per linked record rather than skipping the write-back.
- For each `linked_record` with a `vcs_repo`, resolve that repo's root and match it
  against `SddStore.kind_root(role)` for every role in
  `effective_sidecar_ref_policies(...)` whose policy is a document provider with
  `publication.referenced_by == "markdown_table"`. A record that matches no such role is
  skipped — `@bead` in the beads sidecar and `@file` objects in the agents sidecar are
  not artifact-repo documents.
- Emit one `ReferencedByOutboxItem` per `(role, repo_relpath)`, with `uses` summed over
  the record's occurrences, `destination` from `reference_labels`, `agent_url` from
  `hosted.agent_url(global_agent)`, and `published_date` from the UTC date.

### 4.4 sase: the reconciler

**`src/sase/sdd/referenced_by_index.py`** — the structured index, Rust-free and small:
`referenced_by_index_path(repo_root, artifact_id)`, `read_referenced_by_index(path)`,
and `merge_referenced_by_rows(existing, requests)`. The document is

```json
{
  "schema_version": 1,
  "artifact_id": "research:202608/x.md",
  "repo_relpath": "202608/x.md",
  "identity_value": null,
  "rows": [
    {
      "agent": "research.09.cld",
      "agent_url": "https://…",
      "project": "sase",
      "canonical_ref": "research:202608/x.md",
      "destination": "https://…/blob/<sha>/202608/x.md",
      "uses": 2,
      "published": "2026-08-11",
      "use_ids": ["…"]
    }
  ]
}
```

Rows are keyed by `(agent, canonical_ref)`; a repeat publication replaces the row rather
than appending. The Markdown table is a projection of `rows`, never the other way round.

**`src/sase/sdd/referenced_by_refresh.py`** — the writer, shaped after
`plan_links_refresh.py` and deliberately separate from it so a citation can never touch
the `AGENTS` relation:

```python
def refresh_referenced_by(
    store, *, role: str, requests: Sequence[ReferencedByOutboxItem], write: bool = False
) -> ReferencedByRefreshReport
```

1. `store_git_write_lock(store.repo_root_for_kind(role), op="sdd.referenced_by.refresh", mutates_worktree=True)`;
   a busy lock is a retryable report, not an exception.
2. `git pull --rebase` through the store's existing sync helper; a failure is retryable.
3. For each request: resolve `repo_relpath`; if it is gone, try the recorded
   `identity_value` and otherwise record an `artifact-missing` issue and leave the
   request queued as a visible error.
4. Merge into the index, render the table, and
   `referenced_by_block_upsert(document, table)`. Write only when the raw text differs
   **and** `referenced_by_block_strip(new) == referenced_by_block_strip(old)` — the
   first condition skips no-op rewrites, the second asserts that nothing outside the
   managed block was disturbed.
5. One batched
   `commit_sdd_store_files(store, "Update Referenced By projections", paths=changed, already_locked=True, cause="referenced_by")`
   per repo, then push.
6. Report typed actions and issues, `plan_links_refresh_to_json`-style.

**`src/sase/agents_sync/referenced_by_publication.py`** — the drain:
`drain_referenced_by_requests(target, *, git_runner)` groups the project's queued
requests by `sidecar_role`, resolves the `SddStore` once, calls the writer per role, and
acknowledges only roles whose push succeeded; every other role gets
`update_referenced_by_requests(..., increment_attempts=True, quarantine_threshold=configured_publication_max_attempts())`.
A locked, offline, or contended artifact repo therefore leaves a retryable row and
nothing else.

**Call sites.**

- `prompt_archive/publish.py::_publish_prompt_archive` — enqueue inside the agents lock
  right after the push succeeds; call the drain _after_ the `with bounded_agents_lock`
  block exits, wrapped so any failure only decorates
  `PromptArchivePublicationOutcome.error` and never changes `published`.
- `agents_sync/git_sync.py::sync_projects` — drain after `_sync_project` returns (again
  outside the agents lock) and fold `referenced_by_quarantine_diagnostics(...)` into the
  outcome diagnostics beside `publication_quarantine_diagnostics(...)`.
  `retry_quarantined` and `drop_retired` clear and drop this queue too.

### 4.5 sase: the file-hook cause

- `config/file_hooks.py`: `FileHookEvent` gains `cause: str = "user"`; `FileHookFilters`
  gains `causes: tuple[str, ...] | None`; `_FILE_HOOK_FILTER_KEYS` gains `causes`;
  `_parse_file_hook_filters` parses it with the existing `_optional_string_tuple`;
  `hook_matches_event` returns `False` when `event.cause != "user"` and
  `event.cause not in (filters.causes or ())`.
- `config/sase.schema.json`: a `causes` array of strings on the existing
  `fileHookFilters` definition (which is `additionalProperties: false`).
- `file_hooks/engine.py`: `CapturedFileEvent` gains `cause`, `matching_event()` passes
  it through, `_batch_payload` records it per run, and
  `emit_commit_file_hook_events(..., cause: str = "user")` threads it into
  `_derive_commit_file_events`.
- `sdd/_commit_store.py`: `commit_sdd_files` and `commit_sdd_store_files` gain
  `cause: str = "user"`, forwarded to `_emit_sdd_file_hooks` and on to
  `emit_commit_file_hook_events`.

### 4.6 Binding validation

Add `markdown_reference_links_scan`, `markdown_reference_label_allocate`,
`markdown_reference_definitions_append`, `referenced_by_block_upsert`,
`referenced_by_block_render`, `referenced_by_block_parse`, `referenced_by_block_strip`,
and `referenced_by_wire_schema_version` to `REQUIRED_BINDINGS` in
`tools/validate_sase_core_rs`, and call each through `require_rust_binding` with a
literal name so `tools/check_sase_core_rs_bindings` resolves it statically.

## 5. Tests

Rust tests are listed in §4.1. Python tests:

- `tests/agents_sync/test_prompt_archive_reference_links.py` — a published prompt cites
  `[@research:…][1]` with a matching bottom definition; re-rendering the same raw prompt
  is byte-identical, including after `format_with_prettier`; two refs to one destination
  share a label; a prompt carrying a pre-existing `[1]: …` definition and a dangling
  `[x][3]` use gets `2`; refs in fences, inline code, and existing links are untouched;
  a historical archive containing inline `[x](y)` links re-renders unchanged.
- `tests/agents_sync/test_prompt_archive_destinations.py` — one case per epic §3.7 row:
  stitch, patch with and without a PR, bead, agent, clean artifact-repo document, dirty
  document falling back to the CAS object, and `@file`; plus the pinning case — the
  artifact repo's HEAD advances between staging and publication and the published link
  still names the staged revision; plus a legacy row with no `vcs_revision` keeping
  today's behavior; plus an unlinkable `@patch` still producing a use row, an
  `ARTIFACTS` entry, and a plain code-span occurrence.
- `tests/agents_sync/test_referenced_by_outbox.py` — enqueue is idempotent on the
  logical key; attempts, quarantine at the configured threshold, terminal retirement,
  acknowledge, and the diagnostics strings.
- `tests/sdd/test_referenced_by_refresh.py` — first write inserts the block at the
  bottom with the §3.8 columns; a second run is byte-identical; a second agent adds a
  row and the sort stays deterministic; 60 rows render 50 plus `_… and 15 more_`; a
  renamed document with no identity property produces an `artifact-missing` issue and no
  guessed write; a deleted document does the same; a busy store lock is a retryable
  report; the block round-trips through `parse_referenced_by_block`.
- `tests/sdd/test_referenced_by_isolation.py` — a `@plan:` citation writes a
  `Referenced By` row and adds **no** `AGENTS` entry, and `refresh_plan_links` over the
  same tree leaves the managed block untouched.
- `tests/file_hooks/test_file_hook_causes.py` — a hook with no `causes` skips a
  `referenced_by` event and still matches a `user` event; a hook listing
  `causes: [referenced_by]` matches; the batch payload records the cause; an unknown
  filter key is still rejected.
- `tests/agents_sync/test_referenced_by_publication.py` — a failed artifact-repo push
  leaves the request queued and the agent publication reported as published; the drain
  never runs while the agents lock is held (assert via a lock-order probe); one commit
  per repo per publication batching all of that agent's refs.
- `tests/core/test_prompt_artifact_staging.py` (extend) — `vcs_revision` is recorded for
  a clean tracked file; a clean Markdown document's digest ignores a managed
  `Referenced By` block while a dirty one's digest does not.
- `tests/test_config_schema.py` (extend) — `filters.causes` validates; a misspelling is
  rejected.

Run `just install` first (ephemeral workspaces), then `just check`. Because this change
touches the Rust binding, the publication path, and config schema, finish with
`just check-full`.

## 6. Risks and open items

| Risk                                                                          | Mitigation                                                                                                                                                        |
| ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Prettier reflows the appended reference definitions and breaks idempotency    | `format_with_prettier` is asserted in the round-trip test, not assumed; definitions are a plain bottom block prettier already leaves alone                        |
| Changing `rewrite_prompt_artifact_links` in place alters historical rendering | Existing inline links are inside `markdown_link_ranges` and are never rewritten; only newly linked tokens change style, per epic §5                               |
| A back-reference drain slowing the commit path                                | The drain is best-effort, bounded by the store lock's existing timeout, and every failure is a queued retry; the agents publication has already been pushed       |
| Two agents citing the same artifact concurrently                              | The store write lock plus `pull --rebase` and retry; the index is merged by `(agent, canonical_ref)` so neither row is lost                                       |
| `@patch` unlinkable today                                                     | Explicitly allowed by epic §3.7 ("tracking must never depend on linkability") and covered by a test asserting the tracked-but-unlinked shape                      |
| Overlap with parallel phases `ace` and `adopt`                                | No ACE module and no `docs/` page is touched; the only shared file is `config/sase.schema.json`, where this phase adds one leaf key to `fileHookFilters`          |
| The core floor in `pyproject.toml` still names `<0.25.0`                      | Unchanged by this phase; dev installs build from the checkout and the release-branch reconciler ratchets the window, exactly as phases `core` and `files` assumed |

**Deferred, to be recorded as `PROPOSED FOLLOW-UP:` notes on `sase-js.6` rather than
silently dropped:**

- **Provider-declared `Referenced By` columns.** Epic §3.8 wants columns from the
  provider spec; the phase-`core` spec wire has no column vocabulary and the released
  `sase-research` spec declares none. Adding one means a spec-wire change plus a
  coordinated `sase-research` release, which is a poor fit inside this phase. The
  default column set here is epic §3.8's own example, so adopting spec columns later is
  purely additive.
- **Publishing `agents/<agent>/ref-uses.json`.** Epic §3.7's store layout names it and
  `core/artifact_ref_uses.py` says a later phase publishes it, but `agents/<name>/` is
  owned by the v2 publication payload (`rendering.py`, `publication_validation.py`,
  `v2_import_package.py` all enumerate its exact file set), not by the prompt archive.
  Publishing it means threading ref-use rows through the inventory into the v2 payload
  and widening the expected-file contract — a self-contained change in a different
  subsystem. This phase reads the local manifest instead, and the outbox row carries
  everything the drain needs, so nothing here depends on it. </content> </invoke>
