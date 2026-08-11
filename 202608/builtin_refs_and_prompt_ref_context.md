---
tier: tale
title: Builtin refs and prompt ref context
goal:
  "`@stitch`, `@patch`, short-id `@bead`, and entry-backed `@agent` resolve against an
  explicit per-segment `PromptRefContext` derived from the prompt's `#git`/`#gh`
  workflow rather than `cwd`; every ref occurrence writes one immutable `ref-uses` row;
  and the `commit:`/`plans:`/`chat:`/`bug:` parse aliases behave per the epic's
  migration table."
size: medium
proposed_by: bbugyi200.athena.sase-js.4
bead: sase-js.4
create_time: 2026-08-11 16:49:02
status: wip
---

- **PARENT:**
  [202608/artifact_ref_contract.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifact_ref_contract.md)
- **BEAD:**
  [sase-js.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-js/sase-js.4.md)

# Plan: Builtin refs and prompt ref context

Implements phase `builtins` (bead `sase-js.4`) of the epic
`plans:202608/artifact_ref_contract.md` (§3.6, §3.7, §4.4). Read §3.2–§3.4 and §4.4 of
that plan before starting; this document does not restate the epic's rationale, only the
decisions and steps needed to land the phase.

## 1. Scope

Seven deliverables, all from §4.4:

1. An explicit per-segment `PromptRefContext` threaded through late prompt processing,
   never inferred from `cwd`.
2. `@stitch` resolution (`<sha>` and `<repo>@<sha>`).
3. `@patch` resolution against the in-context project's active and archive ProjectSpecs.
4. `@bead` short-id resolution.
5. `@agent` restated as a builtin entry provider whose expansion covers the agent's chat
   transcript.
6. One immutable use row per ref occurrence, for **every** kind, written at resolution
   time.
7. The legacy parse aliases: `commit:` permanent, `plans:` deprecated-with-diagnostic,
   `chat:`/`bug:` historical and absent from completion.

## 2. Hard constraints

### 2.1 No `sase-core` changes

Phase 4 is a Python-only phase. Do **not** touch `../sase-core`, do not bump a wire
schema, and do not add a binding function.

Reason: `sase-js.5`, `.6`, and `.7` are `in_progress` in parallel workspaces against the
current binding. `_require_artifact_ref_schema` hard-fails on a version mismatch, so a
wire bump would break every concurrent phase worker. The epic's own risk table commits
to "one Rust phase, one release", and that release already shipped.

Everything Rust must own for this phase is already in the installed binding
(`sase_core_rs` built from the `sase-core` checkout, artifact-ref wire schema 5). Verify
once at the start:

```bash
just install
.venv/bin/python -c "import sase_core_rs as r; print(r.artifact_ref_wire_schema_version())"
```

Available and required:

| Binding                                                              | Used for                                       |
| -------------------------------------------------------------------- | ---------------------------------------------- |
| `artifact_ref_kind_catalog`                                          | live/alias/deprecated/historical kind metadata |
| `artifact_ref_kind_canonicalize`                                     | one label -> canonical + diagnostic            |
| `artifact_ref_parse_canonical`                                       | canonicalizing parse (`commit:`, `plans:`)     |
| `artifact_ref_expansion_validate` / `artifact_ref_expansion_render`  | the closed expansion formatter                 |
| `artifact_ref_entry_validate`                                        | `ArtifactEntry` validation                     |
| `artifact_ref_use_record_render` / `artifact_ref_use_manifest_parse` | the `ref-uses` JSONL manifest                  |
| `artifact_ref_resolve`                                               | document/chat/file/bug/bead/agent resolution   |

`artifact_ref_resolve` deliberately returns `unknown_kind` with the diagnostic "`<kind>`
references resolve through the provider registry, not this crate" for `Stitch`, `Patch`,
and the `FilePath` payload. That is phase 1's contract, not an oversight: `stitch` and
`patch` resolution is this phase's job and lives in Python. Grammar, canonicalization,
the expansion formatter, entry validation, and manifest rendering stay Rust-owned, which
is what the `CLAUDE.md` core-backend boundary requires of shared cross-frontend
behavior. Record a follow-up (§8) proposing that entry resolution migrate into Rust in a
future dedicated Rust phase.

### 2.2 The launch path must never crash and must stay fast

Every new lookup is best-effort: wrap store, ProjectSpec, VCS, and sidecar reads so a
failure degrades a _property_ or a _diagnostic_, never the launch. Resolution failure of
the ref itself still exits non-zero through the existing `_print_artifact_ref_failures`
path — that behavior is unchanged.

Per-ref cost budget: at most one VCS `revision_id` + one `log(limit=1)` for `@stitch`,
at most one cached ProjectSpec parse per project file for `@patch`, at most one
bead-store `resolve_id` per candidate store for `@bead`, and at most two small JSON
reads for `@agent`. Never build a hood inventory, never walk an artifact index, never
call `Path.cwd()`.

## 3. Design

### 3.1 `PromptRefContext`

New module `src/sase/artifact_ref_prompt_context.py`.

```python
@dataclass(frozen=True, slots=True)
class PromptRefProject:
    key: str                 # ProjectSpec directory key, e.g. gh_sase-org__sase
    display_name: str        # PROJECT_NAME, e.g. sase
    active_spec: Path        # <key>.sase
    archive_spec: Path       # <key>-archive.sase

@dataclass(frozen=True, slots=True)
class PromptRefContext:
    artifact_context: ArtifactRefContext
    project: PromptRefProject | None
    primary_repo: str | None      # repository name for an unqualified @stitch
    workspace_dir: Path | None
    workspace_num: int | None
    origin: Literal["vcs_workflow", "launch_identity", "explicit", "none"]
    vcs_ref: str | None           # raw ref text, for diagnostics only
```

Three builders, in resolution priority order:

1. **`prompt_ref_context_for_vcs_ref(vcs_tag, *, is_home_mode, workspace_num=None)`** —
   the §3.6 path. Normalize with `normalize_vcs_underscore_refs` and take the ref with
   `extract_project_from_vcs_tag`, then resolve project identity **through the project
   registry, which is plugin-independent**:

   ```python
   snapshot = load_project_ref_display_snapshot()   # sase.project_display_names
   key = snapshot.project_key_for_ref(ref)          # "sase" -> "gh_sase-org__sase"
   ```

   Verified behavior in this workspace: `sase` and `gh_sase-org__sase` both resolve to
   `gh_sase-org__sase`, `home` resolves to `home`, and an unknown ref resolves to
   `None`. Derive the display name from `list_project_records` +
   `effective_project_name` (the two functions `artifact_ref_context` already uses) and
   both ProjectSpec paths from `sase.ace.patch.project_spec_path` under
   `sase_projects_dir() / key`.

   Do **not** make `sase.workspace_provider.peek_ref` the primary mechanism. It is the
   right _read-only_ lookup (never `resolve_ref`, which claims a workspace), but it is
   provider-plugin dispatch and legitimately returns `None`: with only `bare_git`
   installed, `peek_ref("sase", "git")` and every `gh` form return `None` because no
   plugin claims them. Use it purely as optional enrichment for `primary_workspace_dir`
   / `checkout_target`, wrapped so `None` and exceptions both degrade.

   Resolve `workspace_dir` in this order: the launch identity's
   `SASE_ACTIVE_PROJECT_DIR` when it belongs to the same project key; else
   `peek_ref(...).primary_workspace_dir`; else the `kind == "primary"` record's path
   from `collect_repo_inventory(project=...)`. All three are explicit registry or
   launcher values, never `Path.cwd()`.

   `key is None` (unknown ref, or the `#gh:<org>/<repo>` spelling the snapshot does not
   index) and `key == "home"` both yield a no-project context rather than a guess.
   `origin="vcs_workflow"`.

2. **`prompt_ref_context_from_launch_identity(*, is_home_mode)`** — the launcher already
   records the segment's identity explicitly in the environment:
   `SASE_AGENT_PROJECT_FILE` (set by `axe/runner_artifacts.py`) and
   `SASE_ACTIVE_PROJECT_DIR` plus the workspace-number variables that
   `artifact_ref_launch_context._workspace_num_from_environment` already reads. Those
   values were derived from the segment's `#git`/`#gh` ref at launch, so using them is
   explicit threading, not `cwd` inference. `origin="launch_identity"`.
3. **`empty_prompt_ref_context(*, is_home_mode)`** — no project. Short forms produce the
   actionable errors specified in §3.3 (`@stitch`), §3.4 (`@patch`), and §3.5 (`@bead`);
   qualified forms still resolve. `origin="none"`.

`prompt_ref_contexts_for_prompt(prompt, *, is_home_mode, fallback)` returns
`tuple[tuple[tuple[int, int], PromptRefContext], ...]` — one `(character_span, context)`
pair per top-level prompt segment:

- Split on `^---\s*$` outside fenced blocks, reusing `protect_fenced_blocks` exactly the
  way `xprompt/_parsing_vcs_tags.normalize_default_vcs_workflow` already does. Extract
  the shared splitter into a small helper in `artifact_ref_prompt_context.py` rather
  than importing a private name across modules.
- For each segment, prefer that segment's own VCS tag (`extract_vcs_workflow_tag`, then
  `find_vcs_workflow_tag` for a non-leading tag). Segments whose tag was already
  consumed by embedded-workflow expansion fall back to `fallback`.
- Memoize built contexts by `(workflow_type, ref, workspace_num)` inside the call so an
  N-segment prompt on one project builds one `ArtifactRefContext`.
- A prompt with no `---` yields exactly one pair spanning the whole prompt, so the
  common single-segment launch does no extra work.

**Retire the `cwd` path.**
`artifact_ref_launch_context.build_launch_artifact_ref_context` keeps `Path.cwd()` +
`find_marker_from_cwd`; it stays for the _CLI and ACE_ callers
(`artifact_cli/references.py`, `ace/tui/widgets/artifacts/files_detail.py`,
`ace/tui/models/artifact_file_clipboard.py`, `core/artifact_file_doctor.py`), where the
user's working directory is intentional. The prompt path must stop calling it: in
`artifact_ref_prompt._expand_artifact_references`, replace the
`launch_artifact_ref_context(is_home_mode=...)` fallback with
`prompt_ref_contexts_for_prompt(...)`. Add a comment on
`build_launch_artifact_ref_context` naming the prompt path as the one caller that must
not use it.

### 3.2 The builtin entry layer

New modules under `src/sase/artifact_providers/`:

- `builtin_entries.py` — `BuiltinEntryOutcome`, the `resolve_builtin_entry` dispatcher,
  and `BUILTIN_ENTRY_KIND_TYPES = ("stitch", "patch", "bead", "agent")`.
- `builtin_entry_stitch.py`, `builtin_entry_patch.py`, `builtin_entry_bead.py`,
  `builtin_entry_agent.py` — one resolver each, public names inside their modules (the
  existing `_builtin.py` / `_hookspec.py` pattern shows private module names with public
  symbols is fine; private _symbols_ must not cross files, per `symvision`).

```python
@dataclass(frozen=True, slots=True)
class BuiltinEntryOutcome:
    status: ArtifactRefResolutionStatus
    entry: ArtifactEntry | None = None
    prompt_text: str | None = None
    locator: str | None = None
    resolved_path: Path | None = None
    candidates: tuple[str, ...] = ()
    diagnostic: str | None = None
    canonical_reference: str | None = None   # set when the payload was rewritten
```

`ArtifactEntry` is a new frozen dataclass in `artifact_ref_models.py` mirroring
`ArtifactEntryWire` (`stable_id`, `ref_kind`, `canonical_argument`, `display_label`,
`project_display_name`, `repository`, `repo_relative_path`, `captured_revision`,
`captured_digest`, `logical_path`, `properties: Mapping[str, str]`, `origin`) with a
`to_wire()` that stamps `ARTIFACT_REF_ENTRY_WIRE_SCHEMA_VERSION` (read from
`artifact_ref_entry_wire_schema_version()`). Every constructed entry is passed through
`artifact_ref_entry_validate` before use; a validation error is logged and downgrades
the entry to `None` rather than failing the launch. Property keys must match
`[a-z][a-z0-9_]*` and every value is a string — coerce at construction.

`resolve_builtin_entry(reference, *, context, ref_context)` dispatches on
`reference.kind_type` and returns `None` for kinds it does not own, so the caller falls
through to the Rust resolver unchanged.

### 3.3 `@stitch`

`ArtifactRefPayloadWire::Stitch { repo: Option<String>, sha }` already parses both
spellings and `validate_sha(sha, full=false)` already enforces 7–40 lowercase hex, so no
grammar work is needed.

Algorithm:

1. Pick the repository. With `payload.repo` set, match `context.repositories` by name or
   alias; no match -> `unknown_repo` with a diagnostic listing the known names. With
   `payload.repo` unset, use `ref_context.primary_repo`, which is the repository whose
   `ArtifactRefRepository.kind == "primary"`; with no project context -> `unknown_repo`
   with the diagnostic
   `"@stitch:<sha> needs a project; write @stitch:<repo>@<sha> or add a #git/#gh workflow"`.
2. Resolve the full hash. For the first existing entry of `repository.checkout_paths`,
   call the existing `artifact_ref_prompt_resolution.resolve_checkout_commit`, which
   already validates a 40-char lowercase result through the VCS provider.
3. No result -> `missing` with the diagnostic
   `"no commit matching <sha> in <repo>; git reports it as unknown or ambiguous — use a longer prefix"`.
   §4.4 asks to "error on ambiguity"; git already refuses an ambiguous short hash, so a
   `missing` status carrying an ambiguity-aware, actionable message satisfies that
   requirement without adding a git-specific disambiguation probe that would break VCS
   neutrality. State this in the module docstring.
4. Build the entry: `stable_id = f"stitch:{repo}@{full_sha}"`,
   `canonical_argument = f"{repo}@{full_sha}"`, `display_label = full_sha[:12]`,
   `repository = repo`, `captured_revision = full_sha`, `origin = "prompt_ref"`.
   Properties, best-effort from one
   `get_vcs_provider(checkout).log(checkout, 1, revs=(full_sha,), merges="show")` call:
   `repo`, `sha`, `subject`, `author`, `authored_at`. `patch` and `stitch_number` are
   **omitted** — §4.4 asks for them only "when discoverable without requiring them", and
   no cheap sha-to-Patch mapping exists (`Stitch` in `ace/patch/models/stitches.py`
   records no hash). File a follow-up (§8).
5. `locator = f"{repo}@{full_sha}"`, `resolved_path = checkout_path`.
6. `prompt_text` renders the §4.4 format through the Rust closed formatter:
   `"stitch {captured_revision} in {repository} (checkout: {checkout_path})"`. Validate
   the format string once with `artifact_ref_expansion_validate` in a module-level
   constant test (§6) so an unknown placeholder cannot ship.

**`commit:` uses this same resolver.** Canonicalize the label first (§3.6), so
`@commit:sase@<sha>` and `@stitch:sase@<sha>` produce byte-identical expansions and one
code path. The Rust `resolve_commit` arm stays untouched for the CLI/list surfaces that
still call `resolve_artifact_ref` with a `Commit` kind directly.

### 3.4 `@patch`

1. With a project context, parse both ProjectSpecs through
   `sase.ace.patch.cache.get_global_snapshot_cache().get_file_specs(path, display_name)`
   — it is already mtime/size-cached — and match `Patch.name == payload.name` exactly.
   Prefer an active-file hit over an archive hit and note the preference in a comment.
   No hit -> `missing`, with both spec paths as `candidates`.
2. With no project context, fall back to
   `find_all_patches_cached(include_states=("enabled",))`. Exactly one name match ->
   accept it. More than one -> `ambiguous` with candidates
   `f"{patch.project_display_name}: {patch.name}"` and the diagnostic
   `"@patch:<name> is ambiguous across projects; add a #git/#gh workflow to the prompt segment"`.
   Zero -> `missing`. Never silently pick a same-named Patch from another project.
3. Entry: `stable_id = f"patch:{project}/{name}"`, `canonical_argument = name`,
   `display_label = name`, `project_display_name = project`, `origin = "prompt_ref"`.
   Properties: `project`, `status`, `parent`, `pr` (from `pr_url`), `mentors` (count as
   a string), `stitch_count`.
4. `locator = f"{project}/{name}"`, `resolved_path = None`.
5. `prompt_text` through the closed formatter:
   ``"the {display_label} Patch in project {project} (inspect with `sase patch show {display_label}`)"``.
   Never inline a ProjectSpec block.

Quoted arguments already work: `scanner.rs` scans `@patch:"My Patch Name"` and
`validate_patch_name` permits interior spaces. Add the `quoted: bool` field the scan
wire already emits to `ArtifactRefPromptCandidate` (default `False`) so the use record
can keep `raw_ref` byte-faithful.

### 3.5 `@bead` short ids

Per `sase/memory/sase_beads.md`, a short bead id is the suffix after the final dash
(`js.4` for `sase-js.4`), and ambiguous shorthand must fail with candidates.

1. Order candidate stores: the in-context project's store first (match
   `ArtifactRefBeadStore.project` against `ref_context.project.display_name`), then the
   remaining `context.bead_stores` in their existing order.
2. For each store call `sase.core.bead_read_facade.resolve_id(store.root, payload.id)`
   inside `try/except`. A raise means "not resolvable in this store" — except that an
   ambiguity message from a single store is surfaced verbatim as `ambiguous` with that
   store's candidates.
3. Exactly one store resolves -> rewrite the payload to the full id
   (`replace(reference, payload=replace(payload, id=full_id))`), set
   `canonical_reference` on the outcome, and delegate to `resolve_artifact_ref` so the
   existing page-address logic, `missing` candidates, and `artifact_ref_resolution_hint`
   behavior are all preserved unchanged.
4. More than one store resolves -> `ambiguous`, candidates fully qualified as
   `f"{store.project}: {full_id}"`.
5. No store resolves -> return `None` and let the Rust resolver produce today's
   `missing` / `unknown_project` result, so a full id keeps its current behavior
   exactly.
6. Entry properties, best-effort from one `bead_read_facade.show(store.root, full_id)`:
   `project`, `id`, `title`, `type`, `tier`, `status`, `priority`, `size`, `parent`,
   `assignee`. Expansion is unchanged (`@<page path>`), so the bead page keeps being
   attached.

### 3.6 `@agent`

Resolution stays exactly as it is — delegate to `resolve_artifact_ref`, which already
globalizes the name and finds `agents/<name>/README.md`. Add two things.

**Entry properties**, read from the resolved page's own directory
(`resolved_path.parent`), which the publication snapshot writes alongside `README.md`:
`meta.json` via `agents_sync.v2_run_io.run_metadata_from_json` and `state.json` via
`run_state_from_json`. That yields `project`, `agent`, `model`, `llm_provider`, `state`,
`started_at`, `finished_at`, plus `tribe` when present in the metadata mapping, and
`lane` from `agent_lanes.lane_name`. Both reads are small, adjacent, and per-agent; wrap
them so a missing or malformed file simply drops those properties.

**Expansion covers the chat transcript.** The same directory contains `chat.md` and
`prompt.md`. Keep attaching `@<README path>` (so `process_file_references` behavior is
unchanged) and append a prose pointer naming the sibling transcript, e.g.

```
@<...>/agents/<name>/README.md (agent <name> in project <project>; its prompt and chat
transcript are prompt.md and chat.md beside that page)
```

Only name `chat.md` / `prompt.md` when those files actually exist next to the page. That
is what makes dropping `@chat` from authoring lossless.

### 3.7 Aliases, kind catalog, completion

New module `src/sase/artifact_ref_kinds.py` wrapping the Rust catalog with a
process-level memo (one binding call, invalidated never — the catalog is compiled in):

```python
def artifact_ref_kind_descriptors() -> tuple[ArtifactRefKindDescriptor, ...]
def parsable_artifact_ref_kinds() -> tuple[str, ...]     # every registered kind label
def completion_artifact_ref_kinds() -> tuple[str, ...]   # offered_in_completion is True
def canonical_artifact_ref_kind(label) -> ArtifactRefKindAlias
def parse_artifact_ref_canonical(value) -> CanonicalArtifactRef
```

The binding wrappers themselves go in `artifact_ref_operations.py` alongside the
existing ones; `artifact_ref_kinds.py` owns the memo and the small dataclasses.

Then:

- `ArtifactRefContext.known_kinds` becomes
  `dict.fromkeys((*parsable_artifact_ref_kinds(), *document_root_kinds))`. This is a
  **required** fix, not a nicety. Measured in this workspace today:

  ```
  document roots: [('plan', .../sase/repos/plans), ('plan', ~/.sase/plans), ('research', ...)]
  known kinds:    ('commit', 'chat', 'bug', 'file', 'bead', 'agent', 'plan', 'research')
  ```

  `stitch`, `patch`, and `plans` are all absent, and `_expand_artifact_references` does
  `if candidate.kind not in known_kinds: continue` — so `@stitch:`, `@patch:`, and
  `@plans:` are today **silently dropped from prompts with no error at all**. Phase 3
  renamed the plans role's ref kind to `plan` (`_BUILTIN_SIDECAR_REF_KIND` in
  `sidecar_ref_config.py`), which is what broke `@plans:`. The Rust catalog supplies
  `plans`, `commit`, `chat`, and `bug` alongside `stitch` and `patch`, restoring parsing
  for all six. Add an explicit regression test for the silent-drop behavior.

- Delete the hardcoded `BUILTIN_ARTIFACT_REF_KINDS` tuple and update its consumers
  (`artifact_refs.py`, `ace/tui/widgets/_artifact_ref_completion_menu.py`,
  `ace/tui/widgets/_artifact_ref_highlight.py`) to `completion_artifact_ref_kinds()` for
  menus and `parsable_artifact_ref_kinds()` for highlighting. Highlighting must stay
  cheap — the memo makes it a dict lookup.
- In `_expand_artifact_references`, parse with `parse_artifact_ref_canonical` instead of
  `parse_artifact_ref`. Use the returned `reference` for resolution and keep the
  original candidate text as `raw_ref`.
- Diagnostics, printed as warnings on stderr without failing the launch:
  - `plans:` -> the canonical diagnostic the Rust registry already supplies
    (`"@plans: is deprecated; write @plan:<path>"`).
  - `chat:` -> `"@chat: is a historical reader; cite the agent with @agent:<name>"`.
  - `bug:` -> `"@bug: is a historical reader; cite the bead with @bead:<id>"`.
  - `commit:` -> silent. It is a permanent alias, not a deprecation. Emit each distinct
    diagnostic at most once per prompt.
- Completion offers only `offered_in_completion` kinds, which already excludes `commit`,
  `plans`, `chat`, and `bug` and already includes `stitch` and `patch`.

### 3.8 Use records

New module `src/sase/core/artifact_ref_uses.py`, modelled directly on
`core/prompt_artifact_staging.py` (its `locked_file` + fsync + Rust render/parse shape
is the house pattern).

```python
ARTIFACT_REF_USE_MANIFEST_NAME = "ref-uses.jsonl"
ARTIFACT_REF_USE_MANIFEST_LOCK_NAME = "ref-uses.lock"

def record_artifact_ref_use(...) -> ArtifactRefUseRecord | None
def read_artifact_ref_uses(path) -> list[ArtifactRefUseRecord]
```

- **Location:** `<SASE_ARTIFACTS_DIR>/ref-uses.jsonl`. Per-agent by construction (one
  artifacts dir per run), append-only, immutable, and no cross-agent locking. §3.7's
  `agents/<agent>/ref-uses.json` is the _published_ location, which phase 6 owns; this
  is the resolution-time manifest it will publish. JSONL, not JSON, because
  `artifact_ref_use_manifest_parse` is line-oriented.
- **One row per occurrence.** This is the explicit contrast with
  `_record_artifact_ref_consumption`, which dedupes by rendered ref; leave that
  function's dedupe alone — §3.7 keeps `artifact_consumption.jsonl` as a derived
  accelerator.
- **Fields** are exactly what `ArtifactRefUseRecordWire` accepts: `schema_version` (from
  `artifact_ref_use_wire_schema_version()`), `recorded_at`, `agent_name`, `raw_ref`,
  `canonical_ref`, `ref_kind`, `stable_id`, `prompt_text`, `publication_target` (always
  `None` this phase — destinations are phase 6), `captured_file` (always `None` —
  captures are phase 5). The richer §3.7 field set (`span`, `project`, `provider`,
  `origin`, `properties`, `captured_revision`, `captured_digest`) is **not** in the
  released wire and `render_artifact_ref_use_record` would drop it silently, so encode
  what fits: `stable_id` carries the pinned `stitch:<repo>@<full-sha>` identity, and
  `canonical_ref` carries the canonical spelling. File a follow-up (§8) for a use-wire
  v2 with the full field set.
- `agent_name` from `sase.agent.identity.resolve_local_agent_name()`. No agent identity
  (a bare CLI invocation) -> skip writing and log at debug. Never raise.
- Render through `artifact_ref_use_record_render` and validate nothing locally — the
  binding enforces the schema. Wrap the whole call in `try/except` at the
  `artifact_ref_prompt` seam, matching `_record_artifact_ref_consumption`'s posture.
- Rows are written for **every** successfully expanded ref, builtins included, in
  occurrence order.

### 3.9 Threading

`llm_provider/preprocessing.py`:

- `preprocess_prompt_late(prompt, *, file_ref_mode, is_home_mode, ref_contexts=None)`
  where `ref_contexts: Sequence[PromptRefContext] | None`. Forward it to
  `process_artifact_references` / `validate_artifact_references`.
- `PreprocessResult` gains `segment_vcs_refs: tuple[str | None, ...]`, filled by
  `preprocess_prompt_early` from the prompt it returns — the VCS tags are still present
  there, before embedded-workflow expansion consumes them.
- `preprocess_prompt` builds contexts from `early.segment_vcs_refs` and passes them on.

`main/xprompt_handler._handle_expand` and
`xprompt/workflow_executor_steps_prompt._execute_prompt_step` both call
`expand_embedded_workflows_in_query` / `_expand_embedded_workflows_in_prompt` _between_
early and late, which is exactly where the tags are consumed. Both must capture
`early.segment_vcs_refs`, build contexts from them, and pass `ref_contexts` to
`preprocess_prompt_late`.

`process_artifact_references(prompt, *, ref_contexts=None, ...)` resolution order for a
candidate at character offset `n`:

1. Split the prompt into top-level segments and locate `n`'s segment index `i`.
2. If that segment still carries its own VCS tag, use the context built from it.
3. Else if `ref_contexts` has an entry at index `i`, use it.
4. Else if `ref_contexts` has exactly one entry, use it (the overwhelmingly common
   single-segment launch, and the safe answer when embedded expansion changed the
   segment count).
5. Else fall back to `prompt_ref_context_from_launch_identity`, then to
   `empty_prompt_ref_context`.

Log at debug when a segment count mismatch forces step 4 so the situation is
diagnosable.

### 3.10 Consistency for non-prompt callers

`artifact_cli/references.resolve_cli_reference` must resolve the new kinds too, or
`sase artifact show stitch:<sha>` reports `unknown_kind` while a prompt resolves it.

- Call `resolve_builtin_entry` there before falling through to `resolve_artifact_ref`,
  building the `PromptRefContext` from `launch_artifact_ref_context(...)` plus the
  detected project (the CLI _is_ the one caller for which the user's working directory
  is intentional; `origin="explicit"`).
- `ResolvedArtifactReference` gains `entry: ArtifactEntry | None`, and `to_json_dict()`
  gains an `"entry"` key. This is the phase's real consumer of entry properties, so
  nothing built here is dead.
- Extend `is_filesystem_backed` to keep excluding `stitch`/`patch` (no file payload).

### 3.11 Small required edits

- `core/prompt_artifact_staging.py`: `_NON_FILE_REF_KINDS` gains `"stitch"` and
  `"patch"`. Without this, a `@stitch` ref whose `resolved_path` is a checkout
  _directory_ is classified as an unresolvable file and silently gets no manifest row.
- `artifact_ref_prompt_resolution.artifact_resolved_path`: add `stitch` (returns the
  checkout path, mirroring the existing `commit` arm) and `patch` (returns `None`), so
  it stops raising `unsupported artifact reference kind`.
- `artifact_ref_prompt_rendering`: route `stitch`/`patch` to the entry's `prompt_text`,
  extend the `agent` arm with the transcript pointer, and keep every other arm
  unchanged.
- `artifact_refs.py`: re-export `ArtifactEntry`, `PromptRefContext`, the context
  builders, `completion_artifact_ref_kinds`, and `parsable_artifact_ref_kinds`.

## 4. Implementation order

Land in this order so each step is independently runnable:

1. `artifact_ref_kinds.py` + the new binding wrappers in `artifact_ref_operations.py` +
   `known_kinds` from the catalog + delete `BUILTIN_ARTIFACT_REF_KINDS` and update its
   consumers. Fixes the `@plans:` regression on its own.
2. `ArtifactEntry` and `ArtifactRefPromptCandidate.quoted` in `artifact_ref_models.py`.
3. `artifact_ref_prompt_context.py`.
4. `artifact_providers/builtin_entries.py` + the four resolver modules.
5. Wire the dispatcher into `artifact_ref_prompt_resolution` /
   `artifact_ref_prompt_rendering` / `artifact_ref_prompt`, including canonicalizing
   parse and the deprecation warnings.
6. `core/artifact_ref_uses.py` + the write call in `artifact_ref_prompt`.
7. Threading in `preprocessing.py`, `xprompt_handler.py`,
   `workflow_executor_steps_prompt.py`.
8. `artifact_cli/references.py` and `prompt_artifact_staging.py`.
9. Tests, then `Justfile` epic-symbol entries for anything only phases 5–7 will consume.

Keep every new file well under the `toobig` 700-line first tier; split a resolver module
if it grows past that.

## 5. Symvision

Public symbols that phase 4 itself does not consume need an
`--epic-symbol sase-js(<symbol>)` entry in the `_lint-symvision` recipe in the
`Justfile`, not a pragma. Expect this for `read_artifact_ref_uses` (phase 6 reads the
manifest) and possibly for entry accessors. Prefer making a symbol private, or finding a
real consumer, before whitelisting — `resolve_builtin_entry` and `ArtifactEntry` do have
real consumers via §3.10, so they need no entry. Remove any entry that turns out to be
unnecessary; Symvision reports stale ones.

## 6. Tests

Extend `tests/artifact_refs/` (reuse `helpers.context`, extend it with a `primary`-kind
repository and a second bead store) and add `tests/artifact_providers/` for the
resolvers.

**Prompt ref context**

- A `#gh:<project>` segment builds a context whose key, display name, and both
  ProjectSpec paths come from the project registry, with `origin="vcs_workflow"`.
- The same context builds correctly with `peek_ref` monkeypatched to return `None` and
  to raise, proving the registry — not provider dispatch — carries project identity.
- Two `---` segments naming different projects produce two different contexts, and refs
  in each segment resolve against their own segment's project.
- A prompt whose tag was consumed falls back to the launch-identity context built from
  `SASE_AGENT_PROJECT_FILE` / `SASE_ACTIVE_PROJECT_DIR`.
- Home mode and an unresolvable ref both produce a no-project context.
- A regression test asserting the prompt path never calls `Path.cwd()` or
  `find_marker_from_cwd` (monkeypatch both to raise).

**`@stitch`**

- `@stitch:<short>` against the in-context primary repo, and `@stitch:<repo>@<short>`
  against a non-primary repo; both expand to
  `stitch <full-sha> in <repo> (checkout: <path>)` and store the full 40-char hash.
- `@commit:<repo>@<sha>` expands byte-identically to the `@stitch:` spelling.
- Unknown repo, unresolvable hash, and an unqualified hash with no project context each
  produce their specific diagnostic.
- 7-char and 40-char boundaries; a 6-char and a 41-char payload are malformed.
- The expansion format constant passes `artifact_ref_expansion_validate`.

**`@patch`**

- Resolves from the active spec and from the archive spec; active wins when both match.
- A same-named Patch in another project is not selected when a project context exists.
- With no project context: unique name resolves, duplicate name is `ambiguous` with
  project-qualified candidates and the add-a-workflow diagnostic.
- `@patch:"Name With Spaces"` resolves and the use row's `raw_ref` keeps the quotes.
- Expansion matches the §4.4 string exactly.

**`@bead`**

- `js.4`-style short id resolves against the in-context store; the same short id
  resolvable in two stores is `ambiguous` with fully qualified candidates.
- A full id behaves exactly as before (same status, path, and hint) for exact, missing,
  and unknown-project cases.
- Ambiguous prefix within one store errors instead of taking the first match.

**`@agent`**

- Expansion still attaches the README and now names `chat.md` / `prompt.md` when they
  exist beside the page, and omits the pointer when they do not.
- Properties come from `meta.json` + `state.json`; a missing or malformed file drops
  those properties without failing resolution.

**Use records**

- One row per occurrence: a prompt citing the same ref three times writes three rows,
  while `artifact_consumption.jsonl` still gets one deduped event.
- Rows are written for every kind including builtins, are round-trippable through
  `artifact_ref_use_manifest_parse`, and carry the pinned `stitch:<repo>@<full-sha>`
  `stable_id`.
- No agent identity -> no manifest, no exception.
- A write failure (unwritable artifacts dir) does not fail the launch.

**Aliases and kinds**

- `@plans:<path>` resolves _and_ warns once; `@plan:<path>` resolves silently.
- `@chat:` and `@bug:` still resolve (archive rendering) and warn with the replacement
  kind named.
- `@commit:` never warns.
- `completion_artifact_ref_kinds()` contains `stitch`/`patch` and excludes
  `commit`/`plans`/`chat`/`bug`; `parsable_artifact_ref_kinds()` contains all of them.

**Update, do not delete**, the existing expectations in
`tests/artifact_refs/test_preprocessing_expansion.py` and
`test_preprocessing_effects.py` that assert the old `commit:` expansion
(`sase@<sha> (checkout: ...)`) and the old agent expansion. Those changes are intended
new behavior; call that out in the commit message.

## 7. Verification

```bash
just install
just fmt
just check-full
```

`check-full` rather than `check`: this phase edits `llm_provider/preprocessing.py`,
`artifact_ref_models.py`, and the ACE completion/highlight call sites, all of which are
broadly imported. Also run the focused slices while iterating:

```bash
just test tests/artifact_refs tests/artifact_providers tests/test_artifact_file_e2e.py
just symvision
```

Finally, exercise the real path once end to end, confirming the expansion text and that
`<artifacts_dir>/ref-uses.jsonl` gained one row per occurrence:

```bash
sase xprompt expand '#gh:sase Read @stitch:<short-sha>, @patch:<name>, @bead:<short-id>, and @agent:<name>.'
```

## 8. Follow-ups to record (do not create beads)

Record each with
`sase bead note sase-js.4 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`:

- Migrate `stitch`/`patch` entry resolution from Python into `sase-core` in a dedicated
  Rust phase, replacing the `unknown_kind` placeholder arms, so the LSP and any future
  frontend share one resolver.
- Extend the artifact-ref use wire to schema 2 with the §3.7 field set (`span`,
  `project`, `provider`, `origin`, `properties`, `captured_revision`,
  `captured_digest`); phase 4 can only persist what schema 1 accepts.
- Add a cheap sha-to-Patch/stitch-number mapping so `@stitch` entries can carry the
  `patch` and `stitch_number` properties §3.4 lists.

## 9. Out of scope

- Any `sase-core` change (§2.1).
- `@file` paths, the `artifact_refs.file.roots` allow-list, and the content-addressed
  store — phase `files` (`sase-js.5`).
- Reference-style `[@kind:arg][N]` links, publication targets, and `Referenced By`
  write-back — phase `linking` (`sase-js.6`). `publication_target` stays `None`.
- Dynamic ACE Artifacts sub-tabs and the new Files pane — phase `ace` (`sase-js.7`).
- Docs and glossary rewrites, including documenting `@stitch`/`@patch` for users — phase
  `adopt` (`sase-js.9`) owns every doc page. Do not partially rewrite `docs/xprompt.md`
  or `docs/configuration.md` here.
- Changing which reference spelling ACE emits from `artifact_ref_entries.py` (`commit:`,
  `plans:`) — that surface belongs to the ACE phase.
