---
tier: epic
title: Give the research lead report paths through a shared wait namespace
goal:
  Research leads receive their two predecessors' registered report paths without
  transcript discovery, using a generic wait.artifacts context alongside wait.chats with
  no additional artifact discovery for prompts that do not use it.
phases:
  - id: artifact-query
    title: Add a batched Rust query for waited producers' artifact metadata
    depends_on: []
    size: medium
    description:
      "artifact-query: implement the exact-producer metadata contract in Rust, its
      Python binding and thin adapter, and isolation tests."
  - id: wait-context
    title: Expose the wait namespace at the runtime rendering boundary
    depends_on:
      - artifact-query
    size: medium
    description:
      "wait-context: share waited-producer resolution, inject wait.chats and lazy
      wait.artifacts without serializing the resolver, and update documentation and
      static prompt tooling."
  - id: research-handoff
    title: Register research reports and pass their paths to the lead
    depends_on:
      - wait-context
    size: medium
    description:
      "research-handoff: register reports through the existing artifact command, replace
      the lead's transcript input with report metadata, and verify the complete
      producer-to-consumer flow."
proposed_by: bbugyi200.athena.0gj.f0.f0
bead_id: sase-x8
create_time: 2026-09-09 19:52:55
status: wip
---

- **PROMPT:**
  [prompts/202609/wait_artifacts.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/wait_artifacts.md)
- **BEAD:**
  [sase-x8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-x8/README.md)

# Outcome and scope

Today the lead receives `{{ wait_chats }}` and must read two conversations to find the
reports. After this change, it receives the actual report filenames, their producer
identities, and durable artifact references. It reads the reports through
`sase artifact read`, researches the gaps, and preserves the existing consolidated
directory layout. Neither transcript text nor report contents are automatically inserted
into the lead's prompt.

This is an epic because the work crosses the Rust domain API, Python execution and Jinja
rendering, and the separately packaged research plugin. The three phases have explicit
sequential dependencies and can each be implemented directly. Authoring this epic is
xlarge work; each implementation phase is medium.

Use `/sase_repo` to open `gh:sase-org/sase-core` and `sase-research-artifacts` from the
implementing agent's own checkout. Use only the paths those commands return. Read each
repo's applicable instructions. No research sidecar content needs to be read or modified
to implement this feature; use temporary repositories and fixtures for report tests. Do
not launch actual research agents for verification.

Do not change wait eligibility, queue priority, fork semantics, research suffix
assignment, the independent-research instructions, or the image agent's dependency. Do
not introduce a directory crawler, parse transcripts to recover outputs, change global
automatic capture policy, or materialize artifacts while rendering a prompt.

# Findings from the current source

- `src/sase/axe/run_agent_runner_launch.py` calls `resolve_wait_chat_paths` after
  dependency waiting. `src/sase/axe/run_agent_refs.py` resolves each name through
  `resolve_resume_agent_name` and reads `done.json.response_path`. It preserves name
  order and duplicates and skips missing results with warnings.
- `AgentExecContext.wait_chats` carries that list into `_build_named_args` in
  `run_agent_exec.py`. Empty lists are not injected under the old name. There is no
  corresponding runtime artifact variable.
- The generic file index already has `ArtifactFile` metadata, including stored path,
  original `source_path`, kind, explicit status, producer artifact directory, and
  optional VCS provenance. `artifact_file_query_facade.py` calls Rust's
  `artifact_file.rs`; each query reads the JSONL index, and its existing `agent` filter
  matches a name rather than an exact producer generation.
- Automatic persistence in `artifact_file_defaults.py` captures media, not every
  Markdown report. Generated PDF metadata is an unreliable substitute for the original
  report. `sase artifact create` already creates an explicit immutable snapshot, records
  its source path and producer, and leaves the original in place.
- `WorkflowExecutor._save_state` JSON-serializes its complete context. Putting an
  arbitrary lazy resolver into ordinary workflow arguments would break this or
  accidentally force expensive work during persistence.
- Both workflow `render_template` and xprompt rendering already merge global template
  values at rendering time. This provides a small boundary for a temporary runtime
  overlay without rewriting workflow state storage.
- The plugin's swarm declares an input named `wait`. Its lead's current runtime
  interpolation is protected with `{% raw %}` during swarm expansion. Preserve that
  two-stage behavior so the input and the runtime namespace can coexist.

# Public contract

`wait` is available during an executing agent's prompt rendering. It is not a promise
that predecessor context exists during early swarm expansion or editor inspection. With
no named agent dependencies, its two members yield empty lists. Declared
xprompt/workflow inputs retain their existing precedence over runtime globals; runtime
expressions in swarm templates must remain deferred until the appropriate segment
executes.

| Expression       | Value                                                                                                                              |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `wait.chats`     | List of transcript-path strings, with the same ordering, duplicate handling, and missing-result behavior as existing `wait_chats`. |
| `wait.artifacts` | List of metadata dictionaries for non-chat indexed files produced by the resolved named dependencies. No file contents.            |
| `wait_chats`     | Supported exact alias for the same chats value, retaining its existing absence when there are no chat paths.                       |

Use `wait.chats` in new documentation and examples. Keep `wait_chats` supported in this
change; the user requested a unified namespace, not deletion of existing templates. It
must share the same resolved data and implementation, not perform another lookup. This
plan does not deprecate the alias or introduce a temporary compatibility branch. Any
future removal must follow `sase_flags.md` separately.

Each artifact dictionary has these documented fields:

- `wait_name`: the requested named dependency responsible for this entry.
- `agent_name`: the concrete producing shell's recorded identity.
- `ref`: canonical `file:<id>` reference, without a leading `@`.
- `kind`, `label`, `explicit`: existing artifact metadata.
- `path`: stored absolute path, or null for a VCS-backed artifact.
- `source_path`: the original producing file's path, or null when unavailable.
- `vcs_repo`, `vcs_sha`, `vcs_relpath`: existing nullable provenance fields.

`path` identifies stored bytes; `source_path` is a provenance hint and may belong to
another checkout or no longer exist. Neither is permission to mutate the artifact store.
VCS-backed entries remain metadata-only: consumers use `sase artifact read` or `path`
when they actually need bytes. A nullable path must not trigger Git work at rendering
time.

Artifacts follow dependency order. Within each dependency, use deterministic producer
chronology followed by artifact creation time and ID. Deduplicate by artifact ID,
retaining the first requested dependency when waits overlap. Do not deduplicate distinct
IDs merely because their content hashes match. Exclude chat entries even if explicitly
indexed; `wait.chats` is their dedicated interface.

The scope is named agent dependencies supplied through the existing `wait_names`
channel. Time, priority, bead, and implicit fork-only waits do not fabricate agent
outputs. This does not extend tribe-wait membership semantics or reinterpret a clan as a
filesystem prefix search.

Resolve a concrete shell to its exact artifact directory. For a completed family or clan
reference, reuse the existing newest-generation membership helpers to include successful
producing members of that selected generation; this preserves reports registered before
a monitor/pipe continuation. `wait.chats` still selects the transcript it selects today.
Do not include failed members, earlier generations, similarly named agents, or unrelated
clan peers when waiting on one researcher. Share and memoize producer identity
information between the two context members; once a producer group is selected, do not
resolve it again to a newer generation halfway through the same consumer run. Prefer an
already-bound dependency identity where the runner has one. Do not redesign the
scheduler to implement this context.

Missing/unresolvable dependencies produce a bounded warning and no records. Supported
malformed-index-line tolerance remains in the Rust index reader. An unreadable index or
incompatible binding must produce a clear artifact-context error on access, not a
misleading successful empty result. Unknown `wait` members retain normal StrictUndefined
behavior.

# artifact-query: Artifact query

In `sase-core`, add a small, separately versioned batched artifact-context operation
next to `crates/sase_core/src/artifact_file.rs`, exposing it through
`crates/sase_core_py/src/lib.rs`. Inputs are the file-index location and an ordered list
of resolved producer groups: requested wait name and exact producer artifact
directories, with stable producer identities/order. Do not query one name at a time or
widen the existing public `agent` filter into prefix matching.

Reuse the existing tolerant reader and `ArtifactFileWire`. Rust owns matching, non-chat
selection, ordering, deduplication, and the result contract. Read the index at most once
for a nonempty batch; an empty batch must not open it. Do not stat, hash, open, copy, or
resolve the files named by matching rows. Do not call Git or walk project repositories.
A new operation avoids needlessly invalidating the existing query-wire version and all
its callers.

Add a thin Python facade in `src/sase/core/` that validates the new wire version and
returns the agreed data shape. Existing Python name/family/clan helpers may supply their
already-established membership models; do not add an independent Python implementation
of the new matching and projection rules. If a missing shared selection rule is
required, implement it in Rust and expose it through the facade.

Tests in both the Rust crate and its Python binding must cover empty batches, multiple
producers, repeated/overlapping dependencies, deterministic order, exact
artifact-directory matching despite reused names, different projects/generations,
malformed rows, missing index, unreadable index, non-chat filtering, explicit files, and
VCS-backed rows without stored paths. Verify the old query API remains intact.

Completion: the operation returns the specified records through the real binding,
including two registered reports, with one index read and no artifact-content I/O.

# wait-context: Wait context

Refactor `run_agent_refs.py` and the runner/context plumbing so existing chat resolution
produces reusable dependency information rather than discarding it. Retain the current
chat result and warnings. Keep the existing chat-resolution cost as the baseline; making
transcript lookup itself lazy is not needed for this task. Do not add artifact lookup to
that eager path.

Create a small per-execution namespace whose `chats` property returns the shared list
and whose `artifacts` property calls the new facade only on first access. Expand family
membership only when needed for artifacts and reuse the resolved generation. Cache the
completed metadata result for that consumer execution. This is intentionally a
single-use cache, not a process-global cache or invalidation system.

Bind this namespace around the agent's rendering/execution lifetime using a scoped
runtime context (for example, a ContextVar with token reset in `finally`). Merge its
values only in the existing Jinja rendering helpers, including workflow, xprompt, and
late top-level rendering. Keep the namespace object out of persistent workflow
arguments, `workflow_state.json`, step outputs, and saved inputs. Plain chat lists may
continue to flow through the legacy serializable argument path. Audit nested execution,
exception cleanup, and follow-up entry points so one agent's resolver cannot leak into
the next. A resumed execution reconstructs its namespace from dependency data, rather
than deserializing a Python object.

Use a concrete list of plain dictionaries as the artifact property's result so Jinja
loops, `selectattr`, `map`, indexing, and `tojson` work normally. A property is
sufficient; do not implement a general lazy-dictionary framework or inspect raw prompt
strings for the substring `wait.artifacts`. Substring gating would miss expressions
supplied by nested xprompts and mishandle literal blocks.

Update `BUILTIN_RUNTIME_NAMES`, `docs/xprompt.md`, and static prompt-tooling tests. Keep
completion and diagnostics entirely static: they must never instantiate or inspect a
live namespace. Offer `wait` and its known `chats`/`artifacts` members without accessing
runtime data. If dotted completion needs a small token-boundary adjustment, keep it
confined to `jinja_inspect.py` and the Jinja completion widget; do not build arbitrary
runtime-object introspection. Preserve completion for the existing supported alias and
existing unknown-variable diagnostics.

Extend the resolver, execution-loop, preprocessing, and Jinja tests, especially:
`tests/test_axe_run_agent_phases_wait_chats.py`,
`tests/test_axe_run_agent_exec_repeat_env.py`,
`tests/test_preprocessing_jinja_context.py`, `tests/test_xprompt_jinja_inspect.py`, and
`tests/ace/tui/widgets/test_prompt_jinja.py`.

Verify real rendering across all relevant stages, with both namespace members, the
legacy alias, empty waits, missing transcripts but present artifacts, explicit
shell/family/template references, continuation producers, and two isolated runs.
Exercise raw blocks, fenced code, disabled xprompt regions, nested xprompts, and the
swarm input named `wait` without changing precedence. Preserve historical transcript
sanitization fixtures where they intentionally represent old prompts.

Performance acceptance is structural rather than a fragile timing threshold:

1. Rendering an ordinary prompt, a chats-only prompt, or a literal/inert mention of
   artifacts causes zero artifact-index queries and zero artifact-content reads.
2. Constructing the namespace, saving/reloading workflow state, and typing/completing
   Jinja expressions also cause zero artifact queries. State remains valid JSON.
3. First actual `wait.artifacts` access queries a nonempty producer batch once; repeated
   loops/filters/accesses in that execution reuse the result.
4. A zero-dependency artifact access returns `[]` without opening the index.
5. No new work reaches TUI startup, refresh, or the event loop.

Use call-count/forbidden-I/O assertions and a representative synthetic large index to
substantiate these limits. Record a small before/after sample of unused and used-context
rendering if useful, but do not add a benchmark framework or a new persistent index.
Requested artifact retrieval necessarily has a cost; the promise is no additional
artifact-discovery I/O on existing paths that do not request it.

# research-handoff: Research handoff

In `sase-research-artifacts`, update `src/sase_research_artifacts/xprompts/research.md`
to tell the agent to register its finished report, after its last edit, through:

```text
sase artifact create -p "<absolute-report-path>" -l "research:<repo-relative-report-path>"
```

The label is a canonical portable report reference, for example
`research:202609/topic__a.md`. Calculate it relative to the actual research repo,
including any subdirectories selected by `report_target`; do not reconstruct it from the
consumer's month or clock. Register only after a successful write and do not use
`--move`. The source stays in the research repo for the lead to reorganize, while the
snapshot gives the wait context an exact producer-associated file record. Report
registration failure must be visible, not falsely reported as full success.

Apply this shared instruction to all existing `#research` branches, preserving
`report_target` precedence, optional suffix behavior, and collision handling. This is
the missing producer contract; do not rely on PDF generation or on a researcher
mentioning its path in a final answer. The instruction should require just the report's
final registration, not repeated snapshots after every edit.

In `research_swarm.md`, replace only the lead's transcript listing and transcript
discovery step. Use a raw-protected runtime loop over `wait.artifacts` to list the
Markdown report entries with a canonical research label, their `wait_name`, their
original `source_path`, stored `path`, and `ref`. Other media and generic Markdown
outputs are not research inputs. Render ordinary code-quoted paths/references, without a
leading `@` that would cause automatic file/transcript expansion.

The lead's instructions must:

- Identify one distinct A report and one distinct B report, belonging to this dispatch's
  `research.<marker>.cdx` and `.cld` dependencies. Match canonical research labels and
  the existing `__a.md`/`__b.md` suffixes, never list order.
- Read each report through its canonical research reference using `sase artifact read`,
  after opening the research repo through `/sase_repo` when manipulating its files. The
  immutable `file:<id>` reference is also available for an audited read if the original
  has moved. Do not read predecessor chats.
- Treat a producer's absolute source path as provenance. Resolve its canonical
  repo-relative research path inside the lead's opened research checkout before moving
  it. Never modify another agent's checkout or the stored snapshot.
- Preserve the existing descriptive final stem, no-overwrite behavior, authorship
  suffixes, and `<name>/<name>__a.md`, `<name>/<name>__b.md`, `<name>/<name>.md` layout.
  If a source cannot be located through its canonical reference, or the supplied records
  do not identify exactly the expected pair, report the missing or ambiguous input
  before moving anything. Do not search transcripts or guess from unrelated files.
  Existing naming/layout rules remain authoritative.

Keep the initial researchers' no-read prohibition and suffix assignments verbatim apart
from the necessary shared registration instruction. Preserve topic injection, optional
`wait`/`priority` handling including priority zero, all four segments, and the image
segment's wait/fork behavior. Update `docs/xprompts.md` to explain the registered-file
handoff and the lazy runtime namespace.

Extend `tests/test_xprompt_loading.py` and add a bounded integration fixture that uses
the public registration API/CLI and real Rust query in temporary stores:

1. Produce two differently named reports ending in `__a.md` and `__b.md`, under separate
   producer directories and a research repository fixture. Register them with canonical
   labels. Create completion metadata without usable transcripts.
2. Expand and execute the lead's rendering path using those dependencies. Assert that
   both exact paths, labels, and file refs appear and no chat path/content is accessed.
   Exercise reports created by an earlier successful continuation shell.
3. Include an unrelated swarm, an old producer generation, PDFs/images, and an extra
   Markdown artifact with a non-research label; none may become a report.
4. Test missing/ambiguous research entries, distinct report stems, spaces in paths, and
   a month boundary. The prompt must provide the evidence needed to identify missing
   inputs rather than inventing an A/B pairing.
5. Test both omitted and supplied `wait`/`priority`, two identical swarm invocations
   with distinct dispatch markers, and serialization without artifact access before the
   lead's actual runtime render.

Test producer instructions via rendered public xprompt APIs rather than duplicating
every sentence. Test the real create-to-query-to-render behavior for the data contract.
Do not test prose by launching models.

# Verification and landing

For changed SASE files, read `lint_and_test.md`, run `just install` if this checkout's
environment requires it, and run `just check`. For Rust changes run `just check` from
the opened core repo; `cargo test -p sase_core` alone excludes required binding tests
and is insufficient. Use `/sase_monitor` for long-running checks. For the plugin run
`just check` with both `SASE_RESEARCH_ARTIFACTS_SASE_SOURCE_DIR` and
`SASE_RESEARCH_ARTIFACTS_SASE_CORE_SOURCE_DIR` set to the opened coordinated sources. Do
not let a registry-installed binding conceal a missing new operation.

The land agent must verify the combined three-repo result and run SASE's
`just check-full` exclusively through `/sase_monitor` using its TESTING/TESTED
convention. Run the plugin's wheel/source-coordination smoke lane when validating its
dependency-floor changes. Update Python/plugin minimum dependency requirements to the
first releases containing the new binding/runtime API using actual release versions; do
not guess a future version or manually bump Rust workspace versions owned by
release-plz. Keep coordinated-source tests green until releases exist. Publish
dependency order is core, SASE, then the research plugin.

Do not publish an intermediate plugin that requires `wait.artifacts` before the runtime
is available. The core operation and runtime namespace are independently complete
additive APIs; the plugin adopts them after its dependency is ready. The existing
`wait_chats` alias keeps older plugin versions functional during that sequence. No new
feature flag or deprecation is needed for this additive rollout.

Before final declaration, check diffs and whitespace in every changed repository. Record
the focused correctness and no-extra-I/O evidence plus the required gate results. Use
host-owned finalizers and `/sase_final`; do not manually commit. Phase workers record
unrelated discovered work as `PROPOSED FOLLOW-UP:` notes on their phase bead instead of
creating new tasks.
