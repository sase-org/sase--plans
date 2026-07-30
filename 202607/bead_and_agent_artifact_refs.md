---
tier: epic
title: Add `@bead:` and `@agent:` artifact reference kinds
goal: '`@bead:sase-9z` and `@agent:9w` are first-class artifact references everywhere
  the existing four builtin kinds already work — prompt expansion, `sase artifact
  show/path/open`, the ACE `@` menu and prompt highlighting, the LSP, and ACE copy
  mode — resolving through generated bead and agent pages with loud, actionable diagnostics
  when a page has not been published yet.

  '
phases:
- id: core_grammar
  title: Bead and agent reference grammar in sase-core
  depends_on: []
  size: small
  description: 'core_grammar: add the `Bead { id }` and `Agent { name }` kind/payload
    wire variants with lexical-only validation, reject fragments on both, bump the
    artifact-reference wire schema to 2, and make one shared builtin-kind constant
    the single source of truth.

    '
- id: core_resolve
  title: Local resolution and reverse canonicalization
  depends_on:
  - core_grammar
  size: medium
  description: 'core_resolve: extend `ArtifactRefContextWire` with bead stores, agent
    roots, and the local agent owner; port bead page addressing into Rust; implement
    `resolve_bead`/`resolve_agent` and the path-to-reference reverse mappings in `canonicalize_artifact_ref`.

    '
- id: core_editor
  title: Editor surfaces for the new kinds
  depends_on:
  - core_resolve
  size: small
  description: 'core_editor: teach the shared editor layer the two new kinds — diagnostics
    now resolve them, the `@` kind menu lists them, and bounded local payload enumeration
    lists bead ids and agent global names.

    '
- id: py_facade
  title: Python models and resolution context
  depends_on:
  - core_resolve
  size: medium
  description: 'py_facade: mirror the new kinds and wire schema in the Python artifact-reference
    models and build the bead-store, agent-root, and agent-owner context from local
    SASE storage without new hot-path I/O.

    '
- id: py_cli
  title: Prompt expansion and `sase artifact` support
  depends_on:
  - py_facade
  size: small
  description: 'py_cli: expand `@bead:`/`@agent:` to their page paths at launch, make
    `sase artifact show/path/open` accept the new kinds, and add publication-aware
    hints to unresolved-reference errors.

    '
- id: py_ace
  title: ACE `@` menu payload rows for beads and agents
  depends_on:
  - py_facade
  size: medium
  description: 'py_ace: add bounded, off-thread bead and agent payload catalogs to
    the grouped `@` menu, with their own row badges, panel titles, and durable insertions.

    '
- id: py_copy
  title: ACE copy mode yields bead and agent references
  depends_on:
  - py_ace
  size: medium
  description: 'py_copy: make `%@` on an epic/phase row copy `bead:<id>`, move the
    design plan reference to a new `%d` key, and add `%@` to the Agents tab.

    '
- id: docs
  title: Documentation sweep
  depends_on:
  - core_editor
  - py_cli
  - py_copy
  size: small
  description: 'docs: update the artifact-reference, ACE, editor, beads, and agents-sidecar
    documentation to describe the two new kinds, their page dependency, the no-fragment
    rule, and the single-project resolution scope.

    '
- id: pin
  title: Raise the published `sase-core-rs` floor
  depends_on:
  - docs
  size: small
  description: 'pin: after the sase-core release publishes, raise the `sase-core-rs`
    floor to the version that first ships the wire-schema-2 artifact-reference bindings,
    refresh the lock, and update the declared-minimum assertion.

    '
create_time: 2026-07-29 21:33:06
status: done
bead_id: sase-b2
---

- **PROMPT:** [202607/prompts/bead_and_agent_artifact_refs.md](prompts/bead_and_agent_artifact_refs.md)
- **BEAD:** [sase-b2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-b2/README.md)

# Plan: `@bead:` and `@agent:` artifact reference kinds

## Why now

`202607/sase_sites_hub_and_pages/sase_sites_hub_and_pages.md` §2.1 identifies this as the prerequisite that blocks the
whole sites model, and §5 Phase 0 recommends shipping it standalone: it is small, it unblocks the model, and it
independently improves ACE completion, LSP highlighting and diagnostics, `sase artifact show/path/open`, and prompt
reference expansion — with no sites, no HTML, and no server.

The gap is real on both sides of the binding. `src/sase/artifact_ref_models.py:12` declares
`BUILTIN_ARTIFACT_REF_KINDS = ("commit", "chat", "bug", "file")`, and `ArtifactRefKindWire` in
`crates/sase_core/src/artifact_ref/wire.rs` is exactly `Commit`, `Chat`, `Bug`, `File`, `Document { role }`. Today
`@bead:sase-9z` and `@agent:9w` are classified as _document roles_ named `bead`/`agent`, resolve `unknown_kind`, and
stay inert prose. `bug:` is a GitHub issue, not a bead.

## Design

Nine decisions carry this feature. Each one is load-bearing; implement them as stated rather than re-deriving them.

### 1. Singular kind names, appended to the builtin list

The kinds are `bead` and `agent`, not `beads`/`agents`. Singular matches the existing builtin convention (`commit`,
`chat`, `bug`, `file` all name one thing) and — decisively — avoids colliding with `RESERVED_SIDECAR_ROLES` in
`src/sase/sdd/_store_types.py:20-24`, which reserves the plural `beads` and `agents`. With singular names a builtin kind
can never shadow a configurable document role a user might legitimately want.

New kinds are **appended** to the builtin ordering, which drives `compare_kind_rows` ranking in the bare-`@` menu:

```
commit, chat, bug, file, bead, agent
```

Appending is deliberate: it keeps every existing row in its current position, so no muscle memory and no rendered output
changes for references that already work. Do not reorder the existing four.

### 2. Identity is the bead id and the agent global name — no new id space

```
bead:sase-9z          bead:sase-9z.1        bead:sase-ag.land
agent:bbugyi200.athena.9w                   agent:bbugyi200.athena.9w--code
```

A bead id already carries its own project qualifier (`config.json`'s `issue_prefix`), so `bead:<project>#<id>` would be
redundant and would break the parse↔render round trip that `every_kind_and_fragment_round_trips` enforces. The payload
is therefore `Bead { id }` — the project is _resolution output_ (the locator), never payload. This matches how `commit:`
carries a possibly-abbreviated sha and lets resolution report the canonical one.

`sase bead show 9z` fails today ("issue not found: 9z"), so `bead:` does **not** invent prefix-less shorthand either.
One identity spelling, no ambiguity.

Agent references accept both the local name (`agent:9w`) and the global name (`agent:bbugyi200.athena.9w`). Parsing is
permissive; resolution canonicalizes. This is exactly the established precedent: `resolve_commit` rewrites
`commit:core@0123456` to `commit:sase@<full sha>` and `resolve_bug` rewrites `bug:gh_sase-org__sase#123` to
`bug:sase#123`. So `agent:9w` resolves with `rendered = "agent:bbugyi200.athena.9w"`, which teaches the durable spelling
instead of merely tolerating the short one.

### 3. Both kinds resolve to a generated page path

`@bead:sase-9z` resolves to `<beads-sidecar>/pages/sase-9z/README.md`; `@agent:9w` resolves to
`<agents-sidecar>/agents/bbugyi200.athena.9w/README.md`. Both are self-contained Markdown renderings that already exist
and are exactly what an agent handed the reference needs to read. Resolving to a path also makes `sase artifact path`
and `sase artifact open` work with no per-kind viewer special-casing, and makes prompt expansion a plain `@<path>` token
that the existing file-reference machinery inlines — the same treatment `chat:`, `file:`, and document roles already
get.

Crucially, bead page addressing is already **lexical and offline-derivable**. `src/sase/bead_pages/paths.py` documents
this as an explicit property: _"Nothing here consults the bead store, which is what lets a `SASE_BEAD` commit tag link
to a page the same commit has not published yet."_ Resolution therefore costs one `is_file` stat per candidate root — no
reading of the 2.2 MB `issues.jsonl`, which matters because resolution runs on the ACE prompt highlight path and on
every LSP document change.

### 4. Fragments are rejected on both kinds

`bead` and `agent` join `Commit | Bug` in the fragment-rejection branches of both `parse_artifact_ref` and
`render_artifact_ref`. The payload names a logical entity whose Markdown rendering is regenerated by
`sase bead page refresh` / `sase agent sync`, so a `#L40` anchor would silently drift to the wrong content after any
refresh. A clear parse error ("bead references do not support fragments") beats a silently-wrong line number, and
`chat:` already covers the "point into a transcript" case. Adding fragment support later is backward compatible;
removing it would not be.

### 5. Resolution is scoped to the reference context's own project

`bead_stores` and `agent_roots` carry only the context's project, matching `document_roots`, which is also
single-project. `bug:` crosses projects only because it needs no filesystem work at all. A foreign prefix resolves
`unknown_project`, and the Python error path names the project when the prefix matches a known one, so the message is
actionable rather than merely negative. Widening to all enabled projects is a clean follow-up (it needs one
`resolve_sdd_store` per project) and is an explicit non-goal here.

### 6. Statuses reuse the existing vocabulary — no new resolution status

| Situation                                    | Status                            |
| -------------------------------------------- | --------------------------------- |
| Page found under exactly one root            | `exact`                           |
| Page found under more than one root          | `ambiguous` (candidates listed)   |
| No bead store / agent root in this context   | `missing`                         |
| Bead prefix matches no store in context      | `unknown_project`                 |
| Address is valid but the page file is absent | `missing` (candidate path listed) |

`drifted` is intentionally never produced: bead and agent page addresses are exact and lexical, so
`resolve_ordered_root_file`'s drift search does not apply. Do not route these kinds through it.

### 7. The wire schema version bump is the skew guard

Bump `ARTIFACT_REF_PARSE_WIRE_SCHEMA_VERSION` and `ARTIFACT_REF_RESOLUTION_WIRE_SCHEMA_VERSION` to `2` in Rust and
`ARTIFACT_REF_WIRE_SCHEMA_VERSION` to `2` in Python. Without this, a new Python against an old core would parse
`bead:sase-9z` as `Document { role: "bead" }` and report `unknown_kind` — silently wrong, which is the exact failure
class `tools/check_sase_core_rs_bindings` exists to prevent but cannot catch (no new binding _names_ are added; only the
wire changes). With the bump, `_require_artifact_ref_schema()` in `src/sase/artifact_refs.py:669-676` already fails
loudly: `"sase_core_rs artifact-reference wire is stale: expected 2, got 1"`.

Note `py_artifact_ref_wire_schema_version` in `crates/sase_core_py/src/lib.rs` `debug_assert_eq!`s the two Rust
constants, so they must move together.

### 8. Completion inserts the durable spelling and displays the readable one

The `@agent:` payload menu **inserts the global name** (`@agent:bbugyi200.athena.9w`) because the reference lands in
prompts, plans, and bead descriptions that get read on other machines, where a bare local name would silently mean a
different agent. The row **displays** `present_agent_name(...)` (`9w`) with the owner as row detail, so the menu stays
readable. Typing the short form still resolves, and `sase artifact show agent:9w` prints the global form.

### 9. `%@` on a bead row copies the bead, not its design plan

`reference_for_entry_target` currently answers epic/phase rows with the bead's `design` field — a `plans:` reference —
because `bead:` did not exist. Every other Artifacts sub-tab answers `%@` with _that row's own_ canonical reference
(commits → `commit:`, chats → `chat:`, bugs → `bug:`). Once `bead:` exists, the row's own reference is `bead:<id>`, so
`%@` returns that.

**This changes existing behavior.** The old capability is preserved, not dropped: a new `%d` key ("design") copies the
bead's design plan reference. Proposal and archive rows are genuinely plan rows and keep returning `plans:` from `%@`.

### Grammar reference

```text
bead reference    ::= "bead:" bead-id
bead-id           ::= segment ("." segment)*
segment           ::= [A-Za-z0-9_-]+                 # never empty, never "." or ".."

agent reference   ::= "agent:" agent-name             # any valid historical semantic agent name
```

Bead id validation is lexical and I/O-free: reject empty payloads, absolute paths, `/`, `\`, NUL, whitespace, control
characters, empty segments, and any character outside `[A-Za-z0-9_-]` within a segment. This is stricter than
`_INVALID_SEGMENT_RE` in `src/sase/bead_pages/paths.py` on purpose — a reference payload must be path-safe by
construction.

Agent name validation reuses `agent_identity`'s existing historical semantic-name rules (dotted base, at most one
terminal `--<role>`, no path separators, no control characters, 512-byte cap). A `YYMMDD.` archive prefix parses
successfully and resolves `missing`; do not add a special rejection rule for it.

The scanner in `crates/sase_core/src/artifact_ref/scanner.rs` needs **no changes**: agent names and bead ids use only
characters its candidate body already accepts, contain no `#`, and never end in trailing punctuation that
`trim_candidate_end` would strip.

### Non-goals

- No `family:`, `hood:`, `clan:`, or `tribe:` reference kinds. `agent:` resolves one agent's page only.
- No changes to `sase artifact list` or `sase artifact doctor`; both operate on the artifact-_file_ index
  (`src/sase/core/artifact_file_types.py`), a disjoint vocabulary.
- No new fragment types and no fragment support on the new kinds.
- No changes to how bead or agent pages are generated or published.
- No cross-project bead/agent resolution (decision 5).
- No sites, HTML rendering, retrieval index, or server work.
- No bare-id sugar (`sase artifact show sase-9z` stays an error; only `default:`/`explicit:` file-id sugar exists).

### Cross-repo workflow

Rust phases work in the sibling core repo. Every phase agent must open it through the skill first:

```bash
sase repo open sase-core -r "<phase-specific reason>"
```

Use the printed path as the only path for reads and writes. Per that repo's `AGENTS.md`, **release-plz owns versions** —
never hand-edit `[workspace.package].version`, crate versions, or path-dependency pins. Use Conventional Commits.

Every Python phase must, at the start of its run:

1. `sase repo open sase-core -r "..."` — this resets the linked checkout to `origin/master`, so the merged Rust changes
   are present.
2. `just install` — dev installs build `sase_core_rs` from that local checkout, so this is what makes the new wire
   available to Python. Workspace directories are ephemeral, so this step is mandatory, not optional.
3. `just check` before completing.

### File-size constraint

`just check` runs `toobig src 1000 850 700`. Two files this work touches are already over the info threshold:
`src/sase/artifact_refs.py` (747 lines) and `src/sase/ace/tui/widgets/_prompt_input_bar_completion_rows.py` (744 lines).
`src/sase/ace/tui/widgets/artifact_ref_completion.py` is at 670 and would cross 700 if the new catalog loaders were
appended to it. Extract new code into new sibling modules rather than growing these files; the phases below name the
modules to create.

---

## Phase: core_grammar — Bead and agent reference grammar in sase-core

Work in `crates/sase_core/src/artifact_ref/`.

`wire.rs`:

- Add `Bead` and `Agent` to `ArtifactRefKindWire`, and `Bead { id: String }` / `Agent { name: String }` to
  `ArtifactRefPayloadWire`. Both enums are `#[serde(tag = "type", rename_all = "snake_case")]`, so the wire forms are
  `{"type":"bead"}` / `{"type":"bead","id":"sase-9z"}` and `{"type":"agent"}` / `{"type":"agent","name":"…"}`.
- Extend `ArtifactRefKindWire::label` with `"bead"` and `"agent"`.
- Bump both `ARTIFACT_REF_PARSE_WIRE_SCHEMA_VERSION` and `ARTIFACT_REF_RESOLUTION_WIRE_SCHEMA_VERSION` to `2` (decision
  7).

`mod.rs`:

- `classify_kind`: `"bead" => Bead`, `"agent" => Agent`, before the `Document { role }` fallback.
- `parse_payload`: add both arms with the lexical validation from the grammar reference above. Add private
  `validate_bead_id` and `validate_agent_reference_name` helpers with precise, user-facing messages, e.g.
  `"bead id segment must contain only letters, digits, '-' and '_'"`.
- `render_artifact_ref`: add the two `(kind, payload)` arms returning `bead:<id>` and `agent:<name>`, re-validating the
  payload exactly as the existing arms do.
- Fragment rejection: add `Bead` and `Agent` to the `matches!(...)` guards in both `parse_artifact_ref` and
  `render_artifact_ref`, so `@bead:sase-9z#L1` fails with `"bead references do not support fragments"`.

`crates/sase_core/src/agent_identity/identity.rs`: expose the historical semantic-name validation used by
`validate_agent_reference_name`. Prefer a narrow new
`pub fn validate_agent_reference_name(name: &str) -> Result<(), AgentIdentityError>` that wraps the existing private
`validate_historical_semantic_name` over widening that function's visibility, so the reference contract has its own
named entry point.

Single source of truth for the builtin list: `crates/sase_core/src/editor/at_reference.rs` already exports
`BUILTIN_ARTIFACT_REF_KINDS`. Add `"bead"` and `"agent"` there (appended — decision 1) and delete the duplicate
hardcoded `["commit", "chat", "bug", "file"]` in `known_artifact_ref_kinds` in
`crates/sase_core/src/editor/diagnostics.rs`, making it consume the shared constant instead. There must be exactly one
list in Rust when this phase lands.

Tests (extend the existing `mod tests` in `artifact_ref/mod.rs`):

- `every_kind_and_fragment_round_trips` gains `("bead:sase-9z", …)`, `("bead:sase-9z.1", …)`,
  `("bead:sase-ag.land", …)`, `("agent:bbugyi200.athena.9w", …)`, `("agent:bbugyi200.athena.9w--code", …)`.
- `parse_rejects_invalid_shapes_and_illegal_fragments` gains `"bead:"`, `"bead:sase-9z."`, `"bead:.sase-9z"`,
  `"bead:sase-9z..1"`, `"bead:sase/9z"`, `"bead:sase 9z"`, `"bead:sase-9z#L1"`, `"agent:"`, `"agent:a/b"`,
  `"agent:.9w"`, `"agent:9w#L1"`.
- A new test asserting `parse_artifact_ref("bead:x").unwrap().schema_version == 2`.

## Phase: core_resolve — Local resolution and reverse canonicalization

Work in `crates/sase_core/src/artifact_ref/`.

### Context wire

Add to `ArtifactRefContextWire` in `wire.rs`, every field `#[serde(default)]` so old payloads still deserialize:

```rust
pub struct ArtifactRefBeadStoreWire { pub project: String, pub prefix: String, pub root: String }
pub struct ArtifactRefAgentRootWire { pub project: String, pub root: String }
pub struct ArtifactRefAgentOwnerWire { pub username: String, pub machine_name: String }

// on ArtifactRefContextWire:
pub bead_stores: Vec<ArtifactRefBeadStoreWire>,
pub agent_roots: Vec<ArtifactRefAgentRootWire>,
pub agent_owner: Option<ArtifactRefAgentOwnerWire>,
```

`bead_stores[].root` is the beads sidecar directory (the one containing `pages/`, `issues.jsonl`, `config.json`).
`agent_roots[].root` is the agents sidecar directory (the one containing `agents/`, `families/`, `users/`). `project` is
the display name, used only to build locators and messages. Export the three new records from `artifact_ref/mod.rs`.

### Bead page addressing in Rust

Port `bead_page_path` / `bead_lineage_root` from `src/sase/bead_pages/paths.py` into a private helper in the
`artifact_ref` module (or a small shared module if a better home is obvious), preserving its exact rules and its
docstring's reasoning:

- lineage root = the segment before the first `.`
- the lineage root's own page is `pages/<root>/README.md`
- every descendant is `pages/<root>/<full-id>.md`

Keep it lexical: no bead-store reads.

### `resolve_bead(id, rendered, context)`

1. `context.bead_stores` empty → `missing`.
2. Derive the prefix: take the first `.`-separated segment and `rsplit_once('-')`. `sase-9z` → `sase`; `my-repo-9z` →
   `my-repo`. If the first segment contains no `-`, no prefix can be derived.
3. Candidate stores: those whose `prefix` equals the derived prefix. When no prefix could be derived, try every store
   (tolerant — a hand-authored id should not hard-fail on a shape rule). When a prefix _was_ derived and no store
   matches, return `unknown_project`.
4. For each candidate store in order, stat `root/<bead_page_path(id)>`. Exactly one hit → `exact` with `resolved_path`
   set, `rendered = "bead:<id>"` unchanged, and `locator = "<project>/<id>"`. More than one → `ambiguous` with all hits
   as candidates. None → `missing`, with the derived candidate paths in `candidates` so the caller can show the exact
   path it looked for.

### `resolve_agent(name, rendered, context)`

1. `context.agent_roots` empty → `missing`.
2. Build ordered candidate directory names: the payload verbatim, then `globalize_agent_name(name, owner)` when
   `context.agent_owner` is present and differs from the verbatim form. Verbatim-first is what makes already-global
   names (`bbugyi200.athena.9w`) and legacy machine-qualified names (`athena.sase-7r.land--code`, which exists on disk)
   resolve without owner context.
3. For each root, for each candidate in order, stat `root/agents/<candidate>/README.md`; the first candidate that hits
   wins for that root. Exactly one root hit → `exact` with `resolved_path`, `rendered = "agent:<matched candidate>"`
   (the canonical global name — decision 2), and `locator = "<project>/<matched candidate>"`. More than one →
   `ambiguous`. None → `missing` with the attempted paths as candidates.

Wire both into `resolve_artifact_ref`'s match.

### Reverse mapping in `canonicalize_artifact_ref`

Insert after the chats-root check and **before** the artifact-index scan (which does file I/O), so the cheap prefix
strips run first:

- For each bead store: if the path is under `root/pages/`, derive the id from `<lineage>/README.md` → `<lineage>` or
  `<lineage>/<id>.md` → `<id>`, and return `bead:<id>` — but only when `bead_lineage_root(<id>) == <lineage>`. Rejecting
  the mismatch means a hand-moved page cannot mint a wrong reference.
- For each agent root: if the path is exactly `root/agents/<global>/README.md`, return `agent:<global>`. Sibling files
  (`chat.md`, `prompt.md`, `meta.json`) return no bead/agent reference and fall through to the remaining checks.

Tests (extend `artifact_ref/mod.rs`'s `mod tests`, using `tempfile::tempdir` like the existing resolution tests):

- Bead: `exact` for a root page and for a descendant page; `unknown_project` for a foreign prefix; `missing` with a
  populated `candidates` list when the store exists but the page does not; `missing` when `bead_stores` is empty;
  `ambiguous` across two stores sharing a prefix; correct locator text.
- Agent: `exact` from a global payload; `exact` from a local payload plus `agent_owner`, asserting
  `rendered == "agent:<global>"`; `exact` for a legacy machine-qualified directory name; `missing` when no root is
  configured and when the directory is absent; a family member (`…9w--code`) resolving to its own agent directory.
- Canonicalize: bead root page, bead descendant page, agent `README.md`, and the negative cases (a bead page whose
  filename contradicts its lineage directory; an agent `chat.md`).

## Phase: core_editor — Editor surfaces for the new kinds

Work in `crates/sase_core/src/editor/`.

`diagnostics.rs`: `analyze_artifact_refs` currently skips resolution for `Commit | Bug` because they have no filesystem
identity. Bead and agent references _do_ resolve to paths, so they must stay on the resolve path and produce
`unresolved_artifact_ref` diagnostics when their page is absent. Do not add them to that skip list. Confirm
`known_artifact_ref_kinds` now reads from the shared `at_reference::BUILTIN_ARTIFACT_REF_KINDS` constant introduced in
`core_grammar`.

`completion.rs`:

- `build_artifact_ref_kind_completion_candidates` already derives its rows from `known_artifact_ref_kinds(context)` and
  labels builtins `"builtin artifact kind"`, so the two new kinds appear with no change. Verify this rather than
  duplicating logic.
- `build_artifact_ref_payload_completion_candidates` currently early-returns for `matches!(kind, "commit" | "bug")`.
  Leave that guard alone and add two enumerating branches, both honoring `ARTIFACT_REF_MAX_RESULTS` and the
  local-only-no-network contract:
  - `"bead"`: for each `context.bead_stores` entry, enumerate Markdown files under `root/pages/` with the existing
    `bounded_relative_files` helper and map each back to its bead id (`<lineage>/README.md` → `<lineage>`,
    `<lineage>/<id>.md` → `<id>`). Detail: `"bead · <project>"`.
  - `"agent"`: for each `context.agent_roots` entry, enumerate the immediate child directories of `root/agents/` that
    contain a `README.md`. Insertion is the global directory name. Detail: `"agent · <project>"`.

Tests: extend `editor/completion.rs` and `editor/diagnostics.rs` test modules — the bare-`@` kind menu now lists six
builtins in the documented order; a `@bead:` payload query enumerates ids from a temp store; a `@agent:` payload query
enumerates global names; an unresolved `@bead:` reference produces exactly one `unresolved_artifact_ref` diagnostic; a
`@bead:` reference inside a prompt literal zone produces none.

## Phase: py_facade — Python models and resolution context

Reminder: `sase repo open sase-core …` then `just install` before starting, so the rebuilt extension carries the merged
wire.

`src/sase/artifact_ref_models.py`:

- `ARTIFACT_REF_WIRE_SCHEMA_VERSION = 2`.
- `BUILTIN_ARTIFACT_REF_KINDS = ("commit", "chat", "bug", "file", "bead", "agent")` — appended, order matters.
- `ArtifactRefKindType` / the two literal-membership sets in `ArtifactRefPayload.from_wire` and `ArtifactRef.from_wire`
  gain `"bead"` and `"agent"`.
- `ArtifactRefPayload` gains `id: str | None = None` and `name: str | None = None`, added to the `to_wire` field tuple.
  These mirror the Rust wire keys one-for-one, which is the existing convention for every payload field.

`src/sase/artifact_refs.py` — `artifact_ref_context` gains bead/agent context. Keep the new discovery in a **new
module**, `src/sase/artifact_ref_entity_context.py`, called from `artifact_ref_context`, because `artifact_refs.py` is
already at 747 lines. The new module exports something like `collect_bead_stores(store, project_name)`,
`collect_agent_roots(project_key, project_name)`, and `local_agent_owner()`.

Discovery rules:

- **Bead store**: `store.kind_root(BEADS_SIDECAR_ROLE)` from the already-resolved `SddStore`. The `issue_prefix` comes
  from `<root>/config.json`; read it with `sase.bead.config.load_config` or an equivalent small read. This is an
  ~80-byte file — acceptable on the context-build path. **Never read `issues.jsonl` here.**
- **Agent root**: `hidden_sidecar_clone_dir(project_key, AGENTS_SIDECAR_ROLE)` from `src/sase/_linked_repo_paths.py:64`,
  i.e. `~/.sase/projects/<key>/repos/agents`. The project key comes from matching `project_filter` (which
  `artifact_ref_context` already computes) against `context.projects`' name/key/aliases.
- **Owner**: `get_agent_owner_identity()` via `sase.config`. It is nullable and absence must be tolerated.
- **Project display name**: use the existing `_project_display_name` resolution so locators show `sase`, never
  `gh_sase-org__sase`, per the **Show Project Names, Never ProjectSpec Keys** convention.

Every lookup is best-effort and individually wrapped, exactly like the existing `collect_repo_inventory` and
`list_project_records` calls in that function: a missing or unreadable sidecar must degrade to an empty list, never
break the rest of the context.

`ArtifactRefContext` gains `bead_stores`, `agent_roots`, and `agent_owner` fields plus their `to_wire()` projection.
`known_kinds` already unions `BUILTIN_ARTIFACT_REF_KINDS` with document roles, so it picks up the new kinds with no
change — verify, do not duplicate.

Because `ArtifactRefContext.to_wire()` is what `build_artifact_ref_lsp_catalog_payload` writes into the LSP catalog, the
LSP inherits the new context for free.

Tests: extend `tests/test_artifact_refs.py`.

- Parse/render round trips through the Python facade for both kinds, asserting `kind_type` and the new payload fields.
- Resolution through a fake context: bead `exact`, bead `unknown_project`, bead `missing` with candidates, agent `exact`
  from a local name asserting the canonical `rendered`, agent `missing`.
- Fragment rejection surfaces as a `RuntimeError`/`ValueError` from `parse_artifact_ref`.
- `_context()` helper gains bead-store/agent-root fixtures; a context built with no beads sidecar and no agents sidecar
  still resolves document and chat references normally.
- A test asserting the wire schema check rejects a stale core (assert the constant is 2 and that `check_record_schema`
  raises on `schema_version: 1`).

## Phase: py_cli — Prompt expansion and `sase artifact` support

`src/sase/artifact_refs.py`:

- `_artifact_ref_replacement`: add `"bead"` and `"agent"` to the path-returning branch (currently
  `{"document", "chat", "file"}`), producing `@<resolved path>`. No fragment annotation is possible — fragments are
  rejected at parse.
- `_resolve_for_launch` needs no change (its retry path is commit-specific).

`src/sase/artifact_cli/references.py`:

- `ResolvedArtifactReference.is_filesystem_backed` gains `"bead"` and `"agent"`, which is what lets `sase artifact path`
  print a path instead of exiting 2.
- `resolution_error_lines` gains a kind-aware hint line when the status is not `exact`:
  - `bead` → `hint: no published page for <id>; run \`sase bead page refresh\``
  - `agent` → `hint: no published page for <name>; run \`sase agent sync\``
  - `bead` + `unknown_project`, when the derived prefix matches a known project name → name that project explicitly:
    `hint: project <name> has no bead store in this reference context`.

  This is the difference between "missing" and _actionable_, and it is the whole reason the resolver returns candidate
  paths.

`sase artifact show` needs no code change: non-`file` kinds already flow through `_print_resolution`, which prints kind,
reference, status, locator, path, and candidates. Verify with a test rather than editing.

`sase artifact open` needs no code change: `_viewer_command` falls through to `_is_text`, and
`mimetypes.guess_type("README.md")` yields `text/markdown`, so pages open in the text viewer. Verify with a test.

Tests: extend `tests/test_artifact_ref_preprocessing.py` (a prompt containing `@bead:` and `@agent:` expands to
`@<path>` tokens; an unresolved one exits 1 with the hint text) and `tests/main/test_artifact_handler.py`
(`show`/`path`/`open` for both kinds, including exit codes and stderr hints, plus `-j/--json` shape).

## Phase: py_ace — ACE `@` menu payload rows for beads and agents

`src/sase/ace/tui/widgets/artifact_ref_completion.py`:

- `ArtifactRefPayloadSource` gains `"bead"` and `"agent"`.
- `_artifact_ref_payload_panel_title` gains `bead` → `"bead: beads"` and `agent` → `"agent: agents"`.
- `ArtifactRefCompletionCatalog` gains `beads: tuple[...] = ()` and `agents: tuple[...] = ()`.
- `_payload_inventory` gains the two branches.

Put the two loaders and their candidate dataclasses in a **new module**,
`src/sase/ace/tui/widgets/_artifact_ref_entity_catalogs.py`, so `artifact_ref_completion.py` (670 lines) stays under the
700-line info threshold.

- **Beads**: `sase.core.bead_read_facade.list_issues(<beads root>)`. Cache by `issues.jsonl` `(st_mtime_ns, st_size)`
  exactly as `_read_cached_artifact_index` caches the artifact index. Sort by `updated_at` descending, cap at 500. Row:
  payload = bead id, label = title, detail = `"<status>"` (plus `epic`/`phase`), age = `updated_at`.
- **Agents**: `os.scandir(<agents root>/"agents")`, keeping directories that contain a `README.md`. Measured on the live
  5,433-entry sidecar: 5.5 ms for names, 23 ms with stats — safe off-thread, and this is already an off-thread worker
  (`_ARTIFACT_REF_WORKER_GROUP` in `_artifact_ref_highlight.py`). Sort by directory mtime descending, cap at 500. Row:
  payload = the **global** directory name (decision 8), label = `present_agent_name(global)`, detail = `"<project>"` or
  the foreign owner prefix, age = directory mtime.

`load_artifact_ref_completion_catalog` calls both loaders. Both must be individually exception-tolerant and return `()`
on any failure, matching `_load_chat_candidates`.

`src/sase/ace/tui/widgets/_prompt_input_bar_completion_rows.py` — `_ARTIFACT_SOURCE_BADGES` is a direct dict lookup and
will `KeyError` on an unknown source, so both entries are mandatory:

```python
"bead":  ("[◆] ", "bold #FFD700"),   # matches the bead-id color in plans_rendering.epic_text/phase_text
"agent": ("[A] ", "bold #FF5FD7"),   # distinct from all five existing badge hues
```

Each payload menu is single-kind, so badges never mix within one menu; the hues are still chosen to be mutually distinct
for cross-menu recognition. Adjust only if the visual snapshot shows a contrast problem.

Highlighting in `_artifact_ref_highlight.py` needs no change — it is driven by `known_kinds`.

Test constants that enumerate the builtin set must be updated:

- `tests/ace/tui/widgets/test_artifact_ref_completion.py:45` `_KINDS`
- `tests/ace/tui/widgets/test_prompt_artifact_ref_highlight.py:22` `_KNOWN_KINDS`
- `tests/ace/tui/visual/_ace_prompt_png_snapshot_helpers.py:180` known-kinds frozenset

New tests: bead and agent payload rows are produced from fake catalogs with the right payload/label/detail split; the
agent row inserts the global name while displaying the local one; the two new badges render at the documented width in
`tests/ace/tui/widgets/test_at_reference_completion_rendering.py`; both new kinds highlight as known.

`tests/ace/tui/visual/test_ace_png_snapshots_at_reference_completion.py` builds its rows explicitly, so the PNG golden
should **not** change. If it does, that is a signal something unintended shifted — investigate before running
`--sase-update-visual-snapshots`.

## Phase: py_copy — ACE copy mode yields bead and agent references

Implements decision 9.

`src/sase/artifact_refs.py` — `reference_for_entry_target`:

- `_reference_for_plan_row` for `row_kind in {"epic", "phase"}` now returns
  `parse_artifact_ref(f"bead:{issue.id}").rendered` instead of reading the `design` field. `proposal` and `archive` rows
  are unchanged.
- Add a sibling entry point for Agents-tab rows returning `agent:<global name>`, resolved from the row's agent name plus
  the local owner (`globalize_owned_agent_name` / `current_owner_agent_name_lookup_candidates` in
  `src/sase/core/agent_identity_facade.py`). Keep it a separate function rather than overloading the `subtab` string,
  since the Agents tab is not an Artifacts sub-tab.

`src/sase/default_config.yml` (`ace.keymaps.copy_mode.keys`) — per the **Default Keymap Config** convention:

- `artifacts_plans`: add `design: "d"`.
- `agents`: add `reference: "at"`.

`src/sase/ace/tui/actions/clipboard/`:

- `_artifact_reference_resolution.py` / `_artifact_references.py`: plans rows resolve `bead:` refs; add the agent path.
- `_agents.py`: add `_copy_agent_reference` for `%@`, notifying clearly when the selected row is a clan, family
  container, or workflow step rather than an agent — the Agents tab nests clan → members → family members, so non-agent
  rows are common.
- `_core.py::_handle_agents_copy_key`: dispatch the new `reference` key.
- Plans copy: add the `design` action returning the bead's `design` plan reference (the previous `%@` behavior),
  notifying when the bead has no design plan.

Documentation and UI sync required by `src/sase/ace/CLAUDE.md`:

- Update the `?` help modal content for both tabs, honoring the 57-character box width and 32-character keybinding
  description cap.
- Re-check `keybinding_footer.py`: add a footer entry only if the new binding's availability is genuinely conditional on
  the selected row; otherwise it belongs in the help modal only.
- `docs/ace.md` copy tables (lines ~114-117 for the Artifacts sub-tabs, plus the Agents tab table).

Tests: extend `tests/ace/tui/test_artifacts_copy_mode.py` and the Agents-tab copy tests — `%@` on an epic row copies
`bead:<id>`; `%@` on a phase row copies the phase's own id; `%d` copies the design plan reference; `%@` on a proposal
row still copies `plans:`; `%@` on an Agents-tab agent row copies the global `agent:` reference; `%@` on a clan row
warns. Also update `tests/test_keymaps_app_bindings.py` if it enumerates copy-mode keys.

## Phase: docs — Documentation sweep

- `docs/configuration.md` (~line 3231, the `sase artifact` section): add `bead:` and `agent:` to the accepted-reference
  sentence, state that neither accepts `#L`/`#page=`/`#t=` fragments, and note that `path`/`open` work for both because
  they resolve to generated pages.
- `docs/ace.md` (~line 3107, "`@` reference completion"): document the two new kinds, that bead rows come from a
  bounded, mtime-cached bead-store snapshot and agent rows from a bounded scan of the project's agents sidecar, and that
  the agent row inserts the durable global name while displaying the local one.
- `docs/editor.md` (~line 76): the sentence "`commit` and `bug` references receive shape validation but no completion
  enumeration or resolution request" must now scope that to `commit`/`bug` only and state that `bead` and `agent`
  enumerate and resolve locally from sidecar pages, still with no network access. Update the completion, diagnostics,
  and semantic-highlighting rows of the capability table.
- `docs/llms.md` (~line 1420, the preprocessing table and the paragraph below it): bead and agent references expand to
  `@path` tokens like document, chat, and file references.
- `docs/beads.md`: `@bead:<id>` addresses a bead's generated page; the page must be published
  (`sase bead page refresh`); the address is lexical and needs no bead-store read.
- `docs/agents_sidecar.md`: `@agent:<name>` addresses `agents/<global-name>/README.md`; local names are accepted and
  canonicalized to global; the page must be published (`sase agent sync`).
- `docs/cli.md` (~lines 275-279): adjust the `sase artifact` row wording if it enumerates kinds.
- Note the single-project resolution scope (decision 5) once, in `docs/configuration.md`, as a documented limitation.

Run `just fmt-md-check` — Markdown formatting is part of `just check`.

## Phase: pin — Raise the published `sase-core-rs` floor

This phase is **blocked on an external event**: release-plz must publish the sase-core release that first contains the
wire-schema-2 artifact-reference support. If that release has not landed, report the block rather than hand-editing
sase-core versions.

Following the pattern of `e9b17a884` ("build(deps): require sase-core-rs>=0.12.15 for the `@` reference menu"):

1. Raise the `sase-core-rs` floor in `pyproject.toml:46` to that published version.
2. Refresh `uv.lock`.
3. Update the declared-minimum assertion in the telemetry smoke-tool test that tracks the pin.
4. Run `tools/validate_sase_core_rs_version` and `tools/check_sase_core_rs_bindings` against the declared minimum.

This is the guard that stops a fresh non-dev install from resolving to a core that classifies `bead:` as a document
role. Decision 7's schema bump makes such an install fail loudly rather than silently; this phase makes it not happen at
all.

## Verification

Beyond `just check` in every Python phase and `cargo test` / `cargo clippy` in every Rust phase, the epic is done when
these hold against the live `sase` project:

```bash
sase artifact show bead:sase-9z          # exact, locator sase/sase-9z, path .../pages/sase-9z/README.md
sase artifact path bead:sase-9z.1        # .../pages/sase-9z/sase-9z.1.md
sase artifact show agent:9w              # rendered agent:bbugyi200.athena.9w
sase artifact path agent:bbugyi200.athena.9w
sase artifact show bead:sase-9z#L4       # malformed: fragments unsupported
sase artifact show bead:zzzz-1           # unknown_project, with the project hint
```

In ACE: a bare `@` lists six builtin kinds ending in `bead` and `agent`; `@bead:` opens a payload menu of real bead ids
with titles; `@agent:` opens a menu displaying local names and inserting global ones; both highlight as known kinds;
`%@` on an epic row copies `bead:<id>` and on an Agents-tab agent row copies `agent:<global>`. In an LSP-enabled editor,
`@bead:sase-9z` is clean and `@bead:sase-does-not-exist` is diagnosed.
