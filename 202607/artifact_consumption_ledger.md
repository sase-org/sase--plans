---
tier: epic
title: Record artifact consumption at `@`-ref expansion
goal: 'Every artifact reference an agent actually consumes at launch is recorded as
  a typed, attributable edge in a durable ledger, so `sase artifact show` reports
  who used a reference and `sase artifact list --unused` can name what nobody ever
  did.

  '
phases:
- id: core-ledger
  title: 'Rust core: consumption ledger reading, aggregation, and the unused query
    filter'
  depends_on: []
  size: medium
  description: 'core-ledger: add a tolerant consumption-log reader, a per-reference
    aggregation summary, an `unused_only` artifact-file query filter applied before
    the row limit, and a fragment-free reference renderer, all exposed through `sase_core_rs`.

    '
- id: py-ledger
  title: Python ledger module and the expansion call site
  depends_on:
  - core-ledger
  size: medium
  description: 'py-ledger: raise the core pin, add the event record with its role
    vocabulary and locked append, and record one edge per successfully expanded reference
    at the launch rewrite path without ever being able to fail a launch.

    '
- id: read-surfaces
  title: Consumption on `sase artifact show` and `--unused` on `sase artifact list`
  depends_on:
  - py-ledger
  size: medium
  description: 'read-surfaces: add the Rust-backed summary facade, report consumption
    counts and consuming agents for any reference, and add the alphabetized `-u/--unused`
    list filter that reaches the query rather than post-filtering it.

    '
- id: docs-and-ledger-reference
  title: Docs, skill, and ledger reference
  depends_on:
  - read-surfaces
  size: small
  description: 'docs-and-ledger-reference: document the ledger file, its record shape,
    the role vocabulary, and the new read surfaces in the artifact documentation and
    the `sase_artifact_file` skill source, and regenerate the deployed skill.

    '
create_time: 2026-07-30 10:36:33
status: done
bead_id: sase-b9
---

- **PROMPT:** [prompts/202607/artifact_consumption_ledger.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/artifact_consumption_ledger.md)
- **BEAD:** [sase-b9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-b9/README.md)

# Plan: Record artifact consumption at `@`-ref expansion

This implements recommendation 5 of `research:202607/artifact_capture_and_retention/artifact_capture_and_retention.md`
("Record artifact consumption at `@`-ref expansion"), including §3.4's resolution to take `__b`'s
append-at-the-existing-call-site mechanism with `__a`'s typed-role vocabulary on the edge.

The research frames the value precisely: production provenance is already recorded richly on every artifact record
(`agent_name`, `workflow`, `raw_timestamp`, `agent_artifacts_dir`, `workspace_dir`, `project`), so the one half of the
graph that does not exist is **consumption**. Recording it "is what makes item 3's pruning defensible rather than merely
aggressive."

## 1. Context, and what the code already gives us for free

All of the following was verified in this workspace on 2026-07-30.

**The call site exists and is singular.** `_expand_artifact_references()` in `src/sase/artifact_ref_prompt.py:67` is the
only place `@kind:payload` references become concrete paths. It is reached from `preprocess_prompt_late()`
(`src/sase/llm_provider/preprocessing.py:166`) in exactly two rewriting callers — the workflow prompt step
(`src/sase/xprompt/workflow_executor_steps_prompt.py:242`) and `invoke_agent` via `preprocess_prompt()`
(`src/sase/llm_provider/_invoke.py:144`). A third caller, `src/sase/main/xprompt_handler.py:55`, passes
`file_ref_mode="validate"`, which routes to `validate_artifact_references()` and rewrites nothing.

**The consuming agent is already ambient — no threading is required.** For a real SASE agent run,
`run_agent_directive_identity.py:301` sets `SASE_AGENT_NAME` during bootstrap and `publish_phase_env()` sets
`SASE_ARTIFACTS_DIR` at `src/sase/axe/run_agent_exec.py:216`, both _before_ `execute_workflow()` reaches the prompt
step. So `discover_agent_identity()` (`src/sase/agent/identity.py:28`) resolves the consuming agent at expansion time,
and `artifact_file_association_from_dir()` (`src/sase/core/artifact_file_types.py:138`) derives `project` from the
artifacts-directory path with no filesystem I/O.

**The artifact id is already in the reference payload.** A `file:` reference carries `payload.source` and
`payload.digest`, and the artifact-file id is `f"{source}:{digest}"` — the same construction
`_materialize_vcs_file_reference()` uses at `src/sase/artifact_ref_prompt.py:217`. Recording the edge therefore needs
**no index read** on the launch path.

**There is a precedent for exactly this shape of ledger.** `src/sase/repo_open_log.py` is an append-only, schema-
versioned, lock-protected JSONL audit trail whose events are attributed through `discover_agent_identity()` with an
`interactive` fallback, and which is surfaced by `sase repo log`. This plan reuses that structure verbatim for the write
side, and the artifact index's own locking and atomic-write helpers (`src/sase/core/artifact_file_explicit.py:375`) for
the file mechanics.

Two corrections to the research's assumptions, both of which change the design:

**Correction 1 — a canonical reference string cannot be split on `#`, so rendering must stay in Rust.**
`render_artifact_ref()` appends the fragment to the rendered reference at
`crates/sase_core/src/artifact_ref/mod.rs:144-155`, so `ArtifactRefResolution.rendered` for `@file:default:abc#L1-L5` is
`file:default:abc#L1-L5`. The obvious `rendered.split("#")[0]` is wrong, because `bug:` payloads legitimately contain
`#` (`bug:sase#123`, rendered at `mod.rs:110`). The ledger's join key must be the fragment-free canonical reference, so
this plan exposes the already-`pub` `render_artifact_ref` through a binding rather than re-deriving fragment syntax in
Python. That is what puts a Rust phase ahead of the Python one.

**Correction 2 — `--unused` cannot be a Python post-filter.** `query_artifact_files()` truncates to `limit` inside Rust
(`crates/sase_core/src/artifact_file.rs:250-254`). Filtering unused rows in Python after that returns "however many of
the newest 50 happen to be unused", not "50 unused artifacts". Making the caller pass `limit=None` instead would read
and marshal the entire 4,300-row index for every `list --unused`, which is the exact cost the Rust query exists to
avoid. The filter must run inside the query, which settles the `rust_core_backend_boundary` question: ledger reading and
aggregation are core backend logic, the mobile gateway (research item 4) and retention (research item 3, which is
explicitly told to "never prune anything with recorded consumption") are its next two consumers, and the write side
stays in Python beside the artifact-index writer it mirrors.

## 2. The ledger

One append-only file beside the artifact index: `~/.sase/artifacts/consumption.jsonl`, resolved as
`default_artifact_files_root() / "consumption.jsonl"`, with its own `consumption.lock` sibling. Each line is an envelope
mirroring the index's `{schema_version, artifact}` shape:

```json
{
  "schema_version": 1,
  "consumption": {
    "id": "3f0a91c2d4e5",
    "timestamp": "2026-07-30T14:02:11.481293+00:00",
    "ref": "file:default:52895d68931185056fd0e49f",
    "ref_kind": "file",
    "fragment": null,
    "role": "image",
    "artifact_id": "default:52895d68931185056fd0e49f",
    "resolved_path": "/home/bryan/.sase/artifacts/agents/gh_sase-org__sase/20260706061845/x-a1f58dfe27c5.png",
    "resolution_status": "exact",
    "agent_name": "sase-b8.2",
    "agent_source": "SASE_AGENT_NAME",
    "artifacts_dir": "/home/bryan/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202607/30/20260730134501",
    "project": "gh_sase-org__sase"
  }
}
```

- **`ref` is the join key** and is the _fragment-free_ canonical rendering, so `file:default:abc` and
  `file:default:abc#L1-L5` aggregate together. `fragment` keeps the discarded anchor (`L1-L5`, `page=3`, `t=42`) as its
  rendered form without the leading `#`.
- **`ref_kind`** is `parsed.kind` — one of `file`, `chat`, `bead`, `agent`, `commit`, `bug`, or a document role such as
  `plans` / `research`.
- **`artifact_id`** is populated only for `file:` references and is what `list --unused` joins against; every other kind
  leaves it null.
- **`resolution_status`** is the status at expansion: `exact`, `drifted`, or `vcs_backed`. For `vcs_backed`,
  `resolved_path` is the materialized `vcs-cache` path the agent was actually handed, which is why the recorder must
  observe the path the replacement used rather than `resolution.resolved_path` (which is `None` for that status).
- **`agent_source`** is `SASE_AGENT_NAME`, `SASE_AGENT`, `agent_meta`, or `interactive`, mirroring `RepoOpenEvent`.

### Role vocabulary

The edge carries `__a`'s typed roles so later lineage work is additive rather than a rewrite (§3.4). The v1 vocabulary
is exactly those four values, derived by a pure function of `(ref kind, resolved path suffix)` — never by reading the
artifact index, because this runs on the launch path:

| Role          | v1 derivation                                                                                                                      |
| :------------ | :--------------------------------------------------------------------------------------------------------------------------------- |
| `report`      | `chat:` and document-role references; `file:` references whose path suffix is a document type (`.md`, `.markdown`, `.txt`, `.pdf`) |
| `image`       | visual media by suffix — images _and_ video (`.png`, `.jpg`, `.gif`, `.svg`, `.webp`, `.mp4`, `.mov`, `.webm`, …)                  |
| `source`      | everything else: `bead:`, `agent:`, `commit:`, `bug:`, code and data files, and any unrecognized suffix                            |
| `test-result` | reserved; nothing emits it in v1                                                                                                   |

Reuse `artifact_file_mime_type()` (`src/sase/core/artifact_file_helpers.py:50`) for the suffix classification rather
than a second extension table; it is already deterministic and covers every suffix in `_KNOWN_MIME_TYPES`. Grouping
video under `image` is a deliberate simplification: the role means "visual media", and holding v1 to `__a`'s exact four
values keeps the later lineage increment additive. Document it as such.

### Scope of what is logged

Log **every reference that successfully expands in rewriting mode**, of every kind — not just `file:` references. The
cost is identical (one row), the call site is the same, and a `research:`/`bead:`/`chat:` edge is precisely the lineage
data the later increment wants. `--unused` remains an artifact-file filter simply because `sase artifact list` lists
artifact files; `show` reports consumption for any kind because it resolves any kind.

Nothing is logged when `validate_artifact_references()` runs, and nothing is logged when the expansion pass ends in
failure — a launch that never happened consumed nothing.

### Duplicate events

Within one expansion call, deduplicate by `ref` so a prompt mentioning the same artifact three times yields one edge.
Across calls, do _not_ deduplicate: an agent-run retry re-expands the original prompt
(`src/sase/axe/run_agent_exec.py:214-230`) and a multi-step workflow expands once per step, and both are honestly
separate consumptions. Aggregation counts distinct agents separately from raw events, so repeated rows never inflate
"used by N agents".

### Growth and retention

At roughly 400 bytes per row and a handful of references per launch, the ledger grows on the order of a hundred
kilobytes per month — four orders of magnitude below the 662 MB the artifact-file store already holds. Ledger retention
is explicitly research item 3's job and is out of scope here; item 3 inherits a file it can prune with the same
predicates it applies to the store.

## 3. Rust core: consumption ledger reading, aggregation, and the unused query filter

Open the core repo the sanctioned way and use the path it prints:
`sase repo open sase-core -r "Add artifact-consumption ledger reading and the unused query filter"`. release-plz owns
the version; do not hand-edit `Cargo.toml` versions. The query-wire bump below is breaking for older `sase` installs, so
use a `feat!:` commit so release-plz computes a breaking release.

**New `crates/sase_core/src/artifact_consumption.rs`**

- `ArtifactConsumptionEventWire` with every field in §2, each optional field `#[serde(default)]`, following
  `ArtifactFileWire`'s shape at `artifact_file.rs:33`.
- `ARTIFACT_CONSUMPTION_LOG_MIN_SCHEMA_VERSION: u64 = 1`, `ARTIFACT_CONSUMPTION_LOG_MAX_SCHEMA_VERSION: u64 = 1`, and
  `ARTIFACT_CONSUMPTION_WIRE_SCHEMA_VERSION: u64 = 1`.
- `read_artifact_consumption_log(path) -> Result<Vec<ArtifactConsumptionEventWire>, std::io::Error>` — tolerant in
  exactly the way `read_artifact_file_index()` is: a missing file is empty, and blank lines, malformed JSON, unsupported
  envelope versions, and rows missing a non-empty `ref` or `agent_name` are skipped rather than fatal.
- `summarize_artifact_consumption(events, refs: Option<&[String]>) -> BTreeMap<String, ArtifactConsumptionSummaryWire>`
  where the summary carries `consumption_count`, `distinct_agent_count`, `agent_names` (sorted, deduplicated), `roles`
  (sorted, deduplicated), `first_consumed_at`, and `last_consumed_at`. `refs = None` summarizes every reference; a
  supplied list restricts the result and omits references with no events, so an absent key means "never consumed".
- `consumed_artifact_file_refs(events) -> BTreeSet<String>` — the `ref` values whose `ref_kind` is `file`, used by the
  query filter.

**`crates/sase_core/src/artifact_file.rs`**

- `ArtifactFileQueryFiltersWire` gains `#[serde(default)] pub unused_only: bool` and
  `#[serde(default)] pub consumption_log_path: Option<String>`.
- When `unused_only` is set, read the ledger once, build the consumed set, and drop any row whose
  `format!("file:{}", row.id)` is in it. Apply this **with the other filters, before the `limit` truncation at
  `artifact_file.rs:250`** — that ordering is the whole reason the filter lives here. When `unused_only` is false, do
  not touch the ledger at all, so the default `list` path takes on no extra I/O. A missing or unreadable ledger yields
  an empty consumed set, which correctly reports everything as unused.
- Raise `ARTIFACT_FILE_QUERY_WIRE_SCHEMA_VERSION` from 2 to 3. The bump is required rather than cosmetic: the new filter
  fields are `#[serde(default)]`, so a stale wheel would silently ignore `unused_only` and return unfiltered rows. The
  Python facade asserts version equality, so the bump converts silent wrong answers into a loud startup failure.

**`crates/sase_core/src/lib.rs`** — register the new module and re-export its public items beside the `artifact_file`
exports.

**`crates/sase_core_py/src/lib.rs`** — expose and register three functions, and document them in the header comment list
beside `artifact_files_query`:

- `artifact_consumption_summary(log_path: str, refs: list[str] | None = None) -> dict`
- `artifact_consumption_wire_schema_version() -> int`
- `artifact_ref_render(reference: dict) -> str` — a thin wrapper over the already-`pub` `render_artifact_ref()`
  (`crates/sase_core/src/artifact_ref/mod.rs:82`), taking a parsed-reference wire mapping and returning its canonical
  rendering. This is what lets Python render a reference with `fragment` cleared to obtain the ledger's join key. It
  adds no new wire shape and therefore does **not** move `ARTIFACT_REF_PARSE_WIRE_SCHEMA_VERSION` or
  `ARTIFACT_REF_RESOLUTION_WIRE_SCHEMA_VERSION`, which stay at 3.

Tests, as Rust unit tests in the new module and additions to `artifact_file.rs`'s existing test module: tolerant parsing
(malformed line, unsupported envelope version, missing `ref`, missing `agent_name`, absent file); aggregation over
repeated events from one agent and from several (proving `consumption_count` and `distinct_agent_count` diverge);
`refs`-restricted summary omitting never-consumed references; and — the case that justifies the design — `unused_only`
combined with `limit`, asserting the limit applies to the filtered set. Also assert `unused_only` with no
`consumption_log_path` and with a nonexistent path both return every row.

## 4. Python ledger module and the expansion call site

Start by raising the `sase-core-rs` pin in `pyproject.toml` (currently `>=0.13.0,<0.14.0`) to the release the previous
phase produced, then `just install`; `sase core health` should pass before anything else.

**New `src/sase/core/artifact_consumption.py`**, modeled on `src/sase/repo_open_log.py`:

- `ARTIFACT_CONSUMPTION_LOG_SCHEMA_VERSION = 1` and
  `ArtifactConsumptionRole = Literal["source", "report", "image", "test-result"]`.
- `@dataclass(frozen=True) ArtifactConsumptionEvent` with the §2 fields.
- `default_artifact_consumption_log_path()` returning `default_artifact_files_root() / "consumption.jsonl"`.
- `artifact_consumption_role(kind_type, kind, resolved_path)` — the pure derivation from the §2 table.
- `build_artifact_consumption_event(...)` — resolves identity through `discover_agent_identity()` with the `interactive`
  fallback `repo_open_log.py:89` already implements, and derives `project` through
  `artifact_file_association_from_dir()` when an artifacts directory is known.
- `append_artifact_consumption_events(events, *, log_path=None)` — one exclusive-lock acquisition for the whole batch,
  appending each envelope on its own line and flushing. Reuse `locked_file()` from `sase.memory.locks` exactly as
  `append_repo_open_event()` does; do not invent a second locking idiom.

**`src/sase/artifact_ref_prompt.py`** — record at the rewrite path:

- `_artifact_ref_replacement()` returns `(text, resolved_path)` instead of `text`. It is private, so the signature
  change is free, and it is the only place that knows the materialized `vcs-cache` path for a `vcs_backed` file
  reference.
- `_expand_artifact_references()` accumulates a parallel list of `(parsed, resolution, resolved_path)` alongside
  `replacements`.
- After the failure gate and inside the `rewrite` branch only, call a single new private helper that builds the events,
  deduplicates them by `ref`, and appends them. The whole helper body is wrapped in `except Exception` and degrades to a
  one-line `log.debug` — **consumption recording must never be able to fail a launch**, exactly as
  `write_used_xprompts()` is guarded at `src/sase/axe/run_agent_runner_setup.py:273`. Add the module-level `log` that
  this file does not yet have.
- The fragment-free `ref` comes from the new `artifact_ref_render` binding applied to
  `replace(parsed, fragment=None).to_wire()`. When `parsed.fragment` is not `None`, derive the `fragment` field as
  `parsed.rendered[len(ref) + 1 :]`, which is exact because Rust rendered both strings.

Expose the binding through a small facade function in `src/sase/artifact_ref_operations.py`
(`render_artifact_ref(reference: ArtifactRef) -> str`), guarded by the existing `_require_artifact_ref_schema()`, and
export it from `src/sase/artifact_refs.py` alongside the other operations.

Every new public symbol in this phase lands with its consumer in the same phase, so no `--epic-symbol` entries are
needed in the `Justfile`'s `_lint-symvision` recipe. Read the `symvision` long-term memory before reaching for a pragma
if that turns out to be wrong.

Tests: a new `tests/artifact_consumption/` package covering role derivation as a table, event construction from an agent
environment and from a bare interactive environment, the `interactive` fallback naming, append/read round-trip including
the envelope shape, and batch append under one lock. Extend `tests/test_artifact_ref_preprocessing.py` with: one edge
per successfully expanded reference; a fragment-bearing reference recording the fragment-free `ref` plus its `fragment`;
a `bug:` reference recording `bug:<project>#<number>` intact; duplicate references in one prompt collapsing to one
event; `validate_artifact_references()` recording nothing; a failing expansion recording nothing; and a recorder that
raises leaving the expanded prompt byte-identical and the launch successful.

## 5. Consumption on `sase artifact show` and `--unused` on `sase artifact list`

**New `src/sase/core/artifact_consumption_query.py`** — the Rust-backed read facade, mirroring
`src/sase/core/artifact_file_query_facade.py`: `ARTIFACT_CONSUMPTION_WIRE_SCHEMA_VERSION = 1`, a
`_require_consumption_schema()` equality guard over `artifact_consumption_wire_schema_version`, and
`summarize_artifact_consumption(refs=None, *, log_path=None) -> dict[str, ArtifactConsumptionSummary]` that validates
the returned wire rows the way `_artifact_file_from_wire()` does before constructing frozen dataclasses.

**`src/sase/core/artifact_file_query_facade.py`** — raise `ARTIFACT_FILE_QUERY_WIRE_SCHEMA_VERSION` to 3 to match Rust,
add `unused_only: bool = False` to `query_artifact_files()`, and pass `consumption_log_path` only when `unused_only` is
true so the default path stays free of ledger I/O.

**`src/sase/artifact_cli/show.py`** — both renderers gain consumption. Look the canonical reference up once through
`summarize_artifact_consumption([result.canonical_reference])` and add rows to `_print_file()` _and_
`_print_resolution()`, so a `research:` or `bead:` reference reports consumption too:

- `consumption_count` — total recorded expansions
- `consumed_by_agents` — distinct agent count, the research's "used by N agents"
- `consuming_agents` — sorted names, first five then `+N more`
- `last_consumed_at`
- All four render as `never consumed` / `-` when there is no summary.

`ResolvedArtifactReference.to_json_dict()` (`src/sase/artifact_cli/references.py:45`) gains an additive `"consumption"`
object carrying the full summary, or `null` when the reference has never been consumed. The key is additive, so the
documented stable schema is extended rather than changed.

**`src/sase/main/parser_artifact.py`** and **`src/sase/artifact_cli/listing.py`** — add the filter:

- `list_parser.add_argument("-u", "--unused", action="store_true", help="Only show artifacts no agent has ever referenced")`,
  placed after `-s/--since` to keep the alphabetical ordering the `cli_rules` long-term memory requires. Read that
  memory before touching the parser.
- `handle_list()` forwards `unused_only=bool(getattr(args, "unused", False))` into `query_artifact_files()`.
- The pretty panel title names the active filter (for example `Artifacts (12, unused)`) so a filtered empty result is
  not mistaken for an empty store.
- No new column and no per-row consumption count in `list`. Deferring that is deliberate: retention (research item 3) is
  the caller that will want per-row counts on the query row, and it can extend the row wire when it needs it rather than
  this plan speculatively widening it.

Tests: query-facade schema-guard and passthrough tests; `sase artifact show` against a reference with zero, one, and
several consuming agents, in both table and `-j` form; `sase artifact list --unused` proving an unreferenced row appears
and a referenced one does not; `--unused` combined with `--limit` returning a full page of unused rows; and an
end-to-end case in `tests/test_artifact_file_e2e.py` that captures an artifact, expands a `@file:` reference to it
through `process_artifact_references()`, and then observes it leaving the `--unused` set and gaining a consuming agent
in `show`.

## 6. Docs, skill, and ledger reference

- Add a "Consumption ledger" section to the artifact documentation page under `docs/`. Note that `sase-b7.5` is creating
  that page concurrently for VCS-backed artifact files; extend the page it lands rather than starting a competing one,
  and if it is still absent, create the page and register it in `mkdocs.yml`'s nav. Cover the file location, the record
  shape, the role vocabulary and its v1 derivation (including the deliberate video-under-`image` grouping and the
  reserved `test-result`), what is and is not logged, and the fact that retention for the ledger is not yet implemented.
- Update the `sase_artifact_file` skill source under `src/sase/xprompts/skills/`: consumption is recorded automatically
  when an agent's prompt references an artifact, `show` reports who has used a reference, and `list --unused` finds
  artifacts nobody ever referenced. Read the `generated_skills` long-term memory first and regenerate the deployed skill
  the way that note prescribes.
- Make `sase artifact list --help` and `sase artifact show --help` read well with the new option and fields, following
  the `cli_rules` long-term memory: alphabetical options, a short alias for every long option, scannable help.

## 7. Explicitly out of scope

- **Retention or pruning of the ledger itself, and any use of consumption as a pruning protection.** That is research
  item 3's job; this plan only supplies the signal it named as a hard protection ("never prune anything with recorded
  consumption").
- **A `sase artifact consumption` / `log` subcommand.** `show` already names the consuming agents, which is the
  interesting part; a dedicated browse command is worth revisiting once there is enough data to browse.
- **Per-row consumption counts on the artifact-file query wire and a `USED` column on `list`** — deferred to the caller
  that needs them, as above.
- **Any ACE TUI change.** The Files pane keeps its current detail rows; wiring consumption into it is additive later and
  would need the off-thread treatment the `tui_perf` long-term memory prescribes.
- **Any reference-grammar change.** The research is explicit that the grammar is finished for now; this plan adds a
  record and a renderer binding, not a kind or a fragment type.
- **Recording consumption from non-launch readers** (`sase artifact open`/`path`, the Files pane, `sase lsp`). Expansion
  at launch is the edge the research asked for and the only one that means "an agent was handed this content".

## 8. Risks

- **A launch broken by bookkeeping.** The single most important invariant. The recorder is wrapped in a blanket
  `except Exception` at the call site, logs at debug, and is called only after the expansion result is already computed,
  so no failure mode of the ledger can alter the expanded prompt or the launch's exit status. The tests assert this
  directly.
- **Launch-path latency.** One lock acquisition and one small append per expansion pass, with no index read: the
  artifact id comes from the reference payload and the role from the path suffix. The recorder deliberately never calls
  `query_artifact_files()`, which existing resolution paths do with `limit=None`.
- **Lock contention with concurrent agents.** The ledger takes its own `consumption.lock`, never the artifact index's
  `index.lock`, so a burst of parallel launches cannot serialize against artifact capture.
- **Wrong join key.** Mitigated structurally by rendering the fragment-free reference in Rust (§1, correction 1) rather
  than string-splitting, with a `bug:` regression test that would fail loudly under the naive implementation.
- **Cross-repo release coupling.** `ARTIFACT_FILE_QUERY_WIRE_SCHEMA_VERSION` moves 2→3 in lockstep across the two repos,
  and Python asserts equality at runtime, so a mismatched wheel fails fast and visibly. The core phase must be released
  before the Python phase raises the pin.
- **Effort versus the research's estimate.** The research sized this "S". It is larger, for the two reasons in §1: the
  join key cannot be computed in Python without duplicating Rust's rendering, and `--unused` cannot be a post-filter
  without either breaking `--limit` or reading the whole index. Both push work across the `sase-core` release boundary.
  The phase sizes here reflect that.

## 9. Verification

Each phase runs `just install` first (ephemeral workspaces drift), then `just check`. In `sase-core`, run that repo's
own check target before committing. Sanity checks after the read-surfaces phase, on a real run:

```bash
sase artifact list -l 5 -j                    # unchanged default output
sase artifact list -u -l 5                    # only never-referenced rows, five of them
sase artifact show file:default:<id>          # consumption rows present, or "never consumed"
sase artifact show research:202607/<doc>.md   # consumption reported for a non-file kind too
sase artifact show file:default:<id> -j       # additive "consumption" object
wc -l ~/.sase/artifacts/consumption.jsonl     # grows by one row per referenced artifact per launch
```
