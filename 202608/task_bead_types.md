---
tier: epic
status: done
title: Plugin-extensible task bead types
goal: "Every new task bead carries a required, plugin-extensible `task_type` whose
  declared fields, validators, and body template are validated in the Rust core; the
  effective catalog is a deterministic function of a project's committed configuration
  rather than of whichever plugins happen to be installed on the current machine; and
  every surface that shows a task bead shows a distinctly colored type chip.

  "
phases:
  - id: core-bead-wire
    title: Task type on the bead wire and store
    depends_on: []
    size: medium
    description:
      "core-bead-wire: add optional `task_type` and `task_type_fields` to the bead wire,
      reducer, and SQLite mirror in sase-core, with cross-field and slug-shape
      validation but no membership list."
  - id: core-type-spec
    title: Task-type spec validation, digest, and body rendering in Rust
    depends_on:
      - core-bead-wire
    size: medium
    description:
      "core-type-spec: add the task-type spec wire, its validators, its stable digest,
      field-value validation, body-template rendering, and the committed snapshot format
      to sase-core."
  - id: use-prefix
    title: Required plugin prefix for every `use:` field
    depends_on: []
    size: medium
    description:
      "use-prefix: require `<plugin>@<id>` on artifact-ref and file-hook `use:` values,
      migrate every in-tree and chezmoi-managed config, and report legacy bare values as
      hard errors."
  - id: plugins-required
    title: Required-plugin project config and graded enforcement
    depends_on:
      - use-prefix
    size: medium
    description:
      "plugins-required: add the `plugins.required` project config section, resolve it
      against installed distributions, cross-check it against `<plugin>@` prefixes, and
      enforce it differently per surface."
  - id: registry
    title: Task-type discovery, catalog assembly, and diagnostics
    depends_on:
      - core-type-spec
      - plugins-required
    size: medium
    description:
      "registry: add the `sase_task_types` hookspec and entry-point group, the
      project-config source, and the central validator that turns discovered specs into
      one deduplicated catalog with provenance and diagnostics."
  - id: builtins
    title: Builtin task types and the `sase bead task-type` command group
    depends_on:
      - registry
    size: medium
    description:
      "builtins: author the bug, ci, feature, flake, and memory builtin specs and expose
      the catalog through a `sase bead task-type` group."
  - id: create
    title: Typed task creation, field values, and the rendered body block
    depends_on:
      - core-bead-wire
      - builtins
    size: medium
    description:
      "create: extend the `-T` grammar to `task(<slug>)`, add repeatable `-f/--field`
      values, validate them, render the body block below the description, and add
      task-type filters to reading surfaces."
  - id: presentation
    title: Task-type chips on every bead surface
    depends_on:
      - create
    size: medium
    description:
      "presentation: add the shared task-type presentation module with a distinct accent
      per type and route every CLI, ACE, bead-page, gate-preview, and helper surface
      through it."
  - id: triage
    title: Per-type corroboration thresholds
    depends_on:
      - create
    size: small
    description:
      "triage: make the `+1` bar a per-type value with a spec default of zero, keep the
      global knob for untyped legacy beads, and thread it through every triage and
      stale-cleanup consumer."
  - id: snapshot-memory
    title: Committed catalog snapshot and the generated task-type memory note
    depends_on:
      - builtins
    size: medium
    description:
      "snapshot-memory: write `sase/task_types.json` from `sase memory init`, render a
      new generated short memory note from it, move the discovered-work instructions
      into that note, and update the new-task skill."
  - id: install-offer
    title: Missing-plugin gate offering to install
    depends_on:
      - plugins-required
    size: medium
    description:
      "install-offer: raise one gate per project whose required plugins are missing,
      offering to install them interactively while agent contexts still fail closed."
  - id: github-type
    title: The `github` task type and mirror wiring
    depends_on:
      - registry
      - create
    size: small
    description:
      "github-type: register the agent-uncreatable `github` task type from the
      sase-github plugin and have the external issue mirror stamp it."
  - id: enforce
    title: Make `task_type` required end to end
    depends_on:
      - create
      - presentation
      - triage
      - snapshot-memory
      - github-type
    size: small
    description:
      "enforce: flip task creation to require a type in both the Rust core and the CLI
      once every teaching and rendering surface is in place."
  - id: docs-verify
    title: Documentation, glossary, and end-to-end verification
    depends_on:
      - install-offer
      - enforce
    size: medium
    description:
      "docs-verify: document the registry and required-plugin config, add the glossary
      terms, and verify the whole feature end to end."
proposed_by: bbugyi200.athena.05c
bead_id: sase-p3
create_time: 2026-09-09 19:51:47
---

- **PROMPT:**
  [prompts/202608/task_bead_types.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/task_bead_types.md)
- **BEAD:**
  [sase-p3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-p3/README.md)

# Plan: Plugin-extensible task bead types

## Why

Task beads are the one bead type agents create on their own, and today they are untyped
free text. A flaky test, a confirmed red build, a stale memory note, and an out-of-scope
feature idea all arrive as the same shape, so triage cannot tell them apart, no surface
can show what a bead _is_, and the instructions that tell agents when to file one are
three static bullets that no code can check.

This epic gives task beads a required `task_type` drawn from a catalog that builtins,
plugins, and project config all contribute to through one declarative spec — and makes
that catalog a function of a project's committed files rather than of the current
machine's installed plugin set.

## Context and verified current behavior

**Beads already have a type; tasks have no subtype.** `IssueType` is
`{plan, phase, task, flag}` (`src/sase/bead/model.py:20`) mirrored by the compiled
`IssueTypeWire` in sase-core. It carries nearly every structural invariant: the
`parent_id` rules, plan-only tier and Patch metadata, task-only `ready`/`snoozed`, the
`flag` ⇔ `FlagRecord` equivalence, and the `size` restriction — enforced both in
`Issue.validate` and in SQLite `CHECK` constraints
(`crates/sase_core/src/bead/schema.rs`). There is no `metadata`, `labels`, or `kind`
column, and `IssueWire` does not deny unknown fields, so unknown JSON keys are dropped
rather than rejected.

**`--size` is the working precedent for "required going forward".** `create_issue`
rejects a task without a size in one place
(`crates/sase_core/src/bead/mutation.rs:201`), the CLI prints a matching error
(`src/sase/bead/cli_crud_create.py:116`), and legacy sizeless tasks stay readable. This
epic copies that three-part shape exactly.

**The `-T` grammar already parses parameterized types.** `parse_type_arg`
(`src/sase/bead/cli_crud_create.py:31`) accepts `task`, `plan(<path>[,<parent>])`,
`phase(<parent>)`, and `flag(<key>,<date>,<release>)`. Bare `task` is the only
unparenthesized form, so `task(<slug>)` needs no new option and no `-T` collision.

**The store is event-sourced.** Canonical state is `beads/events/**`; `issues.jsonl` is
a projection and `beads.db` is a rebuildable cache whose absence is only a doctor
warning. A new nullable column on that mirror is cheap and needs no table rebuild.

**`sase.artifact_providers` is the right template, and it is the only declarative one.**
Its hook returns plain mappings; discovery collects builtins first, then entry points
sorted by name, tagging each with
`ArtifactProviderProvenance{group, name, package, version, builtin}` and isolating
per-plugin load failures into `error` diagnostics (`_discovery.py`); validation dedupes
by both id and kind with distinct diagnostic codes (`_validation.py:68-88`); and each
spec gets a Rust-computed sha256 digest over the normalized spec. Rust already owns a
property-type vocabulary (`crates/sase_core/src/artifact_ref/provider_spec.rs:28-37`)
and a reserved-slug list. `sase-core` has no plugin loader and must not grow one: it
validates plugin-produced JSON and owns the closed enums.

**Generated agent instructions are committed and drift-gated.** `sase validate` runs
`init memory --check` first (`src/sase/main/validate_handler.py:32`) and `just check`
runs `sase validate`. `sase/memory/sase.md`, `sase_beads.md`, and `sase_sizes.md` are
rendered from packaged templates in `src/sase/main/init_memory/templates/`, and
`src/sase/main/init_memory/staleness.py` exists solely to warn that a foreign sase build
can answer a drift check differently. A catalog read from the live plugin environment
would therefore red-build whoever has the "wrong" plugin set, for a change they did not
make.

**Triage policy is global and its predicates are pure.** `bead.task_triage` has three
knobs; `src/sase/bead/task_triage_policy.py` is two pure functions whose callers own the
clock and the threshold. Per-type thresholds are a lookup change, not a rewrite.

**The Rust floor is not a two-release dance here.** `.github/workflows/ci.yml` builds
`sase-core` from source for feature and master lanes; only `release-core-floor-smoke`
(release branch only) installs the published floor, and `tools/ratchet_core_window`
reconciles that floor automatically at release time. The practical constraint is only
that a sase-core change must land on sase-core master before the sase phase that calls
its binding lands on sase master. `tools/check_sase_core_rs_bindings` requires every
`require_rust_binding` name to be a statically analyzable literal.

**`use:` today is an unprefixed provider id.** `sase/sase.yml` carries `use: plan` and
`use: research`; the chezmoi-managed global config carries `use: research-highlights`;
`src/sase/main/_repo_init_config.py:181` scaffolds `use: plan`. A missing artifact-ref
provider is a fail-soft diagnostic (`sidecar_ref_config.py:337`) and a missing file-hook
provider only warns and silently skips the hook (`src/sase/config/file_hooks.py:403`).

**There is no `plugins` key in project config.** `src/sase/config/sase.schema.json` is
`additionalProperties: false` at every level, so a `plugins` section is a genuinely new
registration. `sase plugin install` only works from a `uv tool install sase`
environment, refuses to run from a dev checkout's virtualenv, and restarts axe on
success — so an agent can never self-heal a missing plugin.

## The contract

### D1 — a new orthogonal field named `task_type`

`IssueType` keeps meaning "lifecycle family"; `task_type` means "flavor of discovered
work" and is valid only when `issue_type == task`. Widening `IssueType` is rejected: it
is a compiled enum a Python plugin can never extend, and every invariant in the table
above would dissolve.

The name is `task_type`, not `type` or `kind`. `type` collides with `-T/--type`,
`sase bead list --type`, the ACE `type:` search token, `BeadTypeValue`, and the mobile
`bead_type` payload key. `kind` collides twice over: it is the gate-envelope
discriminator (`FLAG_TRIAGE_KIND`, `TASK_TRIAGE_KIND`, `BEAD_SNOOZE_KIND`,
`BEAD_STALE_CLEANUP_KIND`) and the artifact-ref provider discriminator with a
Rust-enforced reserved list.

`task_type` is a `String` on the wire, never an enum: bead stores are git-synced across
machines, a bead authored with `sase-github` installed will be read without it, and
`IssueTypeWire` has no `serde(other)` fallback.

### D2 — one declarative spec, wherever it comes from

```yaml
schema_version: 1
task_type: flake # snake_case slug, immutable identity
label: Flaky test # chip and picker label
summary: A test that fails and then passes on an unchanged tree.
when_to_use: >-
  File one when a test or lint failed, a rerun on the same tree passed, and you did not
  cause the failure. Record the fail rate and whether it reproduces serially.
glyph: "≈" # optional, exactly one terminal cell
accent_color: "#00D7D7" # optional, #RRGGBB
agent_creatable: true # optional, default true
default_size: null # optional; unset for every builtin (see D8)
fields:
  - name: node_id # snake_case
    label: Test node ID # optional
    type: string # string | enum | integer | date
    required: true # optional, default false
    role: [data, template] # optional, default [data, template]
    help: The pytest node ID, e.g. tests/foo.py::test_bar
    pattern: '\S+::\S+' # string-only validator
  - name: evidence
    type: string
    required: true
    role: [template]
body_template: |
  ## Flake report

  - **Test:** `{{ node_id }}`

  {{ evidence }}
triage:
  min_plus_ones: 1 # optional, default 0
```

- **Field types are the scalar subset of the vocabulary Rust already validates**
  (`string`, `enum`, `integer`, `date`). Do not invent a third list. `boolean`,
  `number`, `datetime`, and `string_list` are deferred until a caller needs them.
- **Validators ship from day one**: `pattern` and `max_length` for `string`, a required
  non-empty `values` list for `enum`, `minimum`/`maximum` for `integer`, and implied ISO
  `YYYY-MM-DD` parsing for `date`. They are the prerequisite for ever expressing
  `FlagRecord` faithfully, and they live in Rust so a second frontend validates
  identically.
- **`role` separates query keys from prose.** `data` fields are filterable and may
  appear in compact rows; `template` fields exist to fill the body block. Most are both.
- **`summary` is capped at 120 characters and one line; `when_to_use` at 400
  characters.** These strings are inlined into every agent's Tier 1 context (D6), so the
  cap is a validation rule, not a style note.
- `body_template` renders through the house Jinja2 engine (`src/sase/mdtemplates.py`)
  with `StrictUndefined`, so a missing variable is an error rather than a blank.

`flag`'s current hard-coded `FlagRecord` is expressible as a three-field spec with
`pattern` validators and no `body_template` — the sanity check that this shape is
general enough — but migrating it is explicitly out of scope (D10).

### D3 — values are stored, not baked into the description

`task_type_fields` is a `BTreeMap<String, String>` on the wire. The declared `type`
drives _validation and rendering_, not storage, which keeps the digest and the event
diff unambiguous and matches how artifact-ref properties already declare a type while
their values arrive as text.

Rendering the template into `description` at create time is rejected: it is unqueryable,
the template can never be corrected retroactively, and `sase bead update -d` would
silently destroy it. **`description` stays the free-text field and the rendered block is
appended below it at display time, never merged in.**

When the owning plugin is absent, `sase bead show` prints the raw key/value pairs under
a `Task type: github (not installed on this machine)` header. A missing plugin is never
a read failure.

### D4 — three sources, one validator, first wins

```
builtin specs (BuiltinTaskTypes)      ─┐
plugin hooks (sase_task_types EPs)    ─┼─→ validate + dedupe + provenance ─→ TaskTypeRegistry
project config (bead.task_types)      ─┘            (+ diagnostics)
```

Ordering is deterministic: builtins, then entry points sorted by name, then project
config in file order. Conflict policy copies `_validation.py` exactly — first candidate
wins, later duplicates are dropped with a `duplicate_task_type` diagnostic naming the
winner's provenance label.

- **Builtin slugs are reserved against plugins.** A plugin claiming `bug` is an `error`
  diagnostic, not a silent override.
- **Project config may override any slug through `use:`** (D5), because a project is the
  most specific authority over its own vocabulary. A project-config entry _without_
  `use:` defines a new slug and may not shadow a builtin or reserved slug.
- **Reserved slugs**: `plan`, `phase`, `task`, `flag`, `untyped`, `unknown`, `all`,
  `none`.

Builtins live in Python, not compiled into Rust: `BuiltinArtifactProviders` already
establishes that builtins register through the same hookspec from a host path inside
`sase`, and Rust never needs the list.

### D5 — `use:` grows a required `<plugin>@` prefix everywhere

`use:` becomes `<plugin>@<id>`, where `<plugin>` is a distribution name or the literal
`builtin`:

| Config                                  | Before                | After                                         |
| --------------------------------------- | --------------------- | --------------------------------------------- |
| `repos.sidecar.builtin.plans.ref.use`   | `plan`                | `builtin@plan`                                |
| `repos.sidecar.custom.research.ref.use` | `research`            | `sase-research-artifacts@research`            |
| `file_hooks[].use`                      | `research-highlights` | `sase-research-artifacts@research-highlights` |
| `bead.task_types[].use`                 | _(new)_               | `builtin@flake`                               |

This is worth the migration because it makes the config self-documenting about which
plugin it needs, makes the "provider is not installed" error name the exact package to
install, and gives `plugins.required` (D7) something to cross-check against.

`use:` semantics are unchanged otherwise: the referenced spec is the base and the
sibling keys deep-merge over it.

**No compatibility window and no feature flag.** A bare `use:` value is a hard error
naming its replacement, and this epic migrates every known config in one phase: the
project config, the chezmoi-managed global config, and the repo-init scaffolder. A
window would be a deprecation whose old branch must stay reachable, which the flags
policy would require a `sunset` flag for; a same-epic migration with an exact error
message is simpler and leaves nothing half-live. The one reliability gap that must close
with it: a file hook with an unresolvable `use:` currently only warns and disappears, so
`use:` failures must also surface as a `sase doctor` error and fail `sase validate`.

### D6 — determinism: a committed snapshot plus required plugins

The failure being designed out: agent A (with `sase-github`) runs `sase memory init`,
the type list gains `github`, A commits; agent B (without it) runs `just check` →
`sase validate` → `init memory --check` → drift → a red build for a change B did not
make. B "fixes" it by regenerating and A goes red.

Two artifacts close it, and the _order they are checked in_ is load-bearing:

1. **`plugins.required`** (D7) is verified **first**. On a machine missing a required
   plugin, `sase memory init` and `sase validate` fail with "required plugin
   `sase-github` is not installed", never with a spurious drift report.
2. **`sase/task_types.json`**, a committed snapshot written by `sase memory init`, is
   the render source for the generated memory note. Two machines that pass check 1
   produce byte-identical output because they render from committed bytes.

The snapshot stores, per type sorted by slug: `task_type`, `label`, `summary`,
`when_to_use`, `glyph`, `accent_color`, `agent_creatable`, `default_size`, `fields`,
`body_template`, `triage`, `source` (`builtin`/`plugin`/`project`), `package`, and
`digest`. It deliberately stores **no version**: a version would churn the snapshot on
every release of the providing package and turn `--check` into a false alarm. The digest
covers spec content, so `--check` can say _"`github` spec digest changed (sase-github
0.4.1 installed); run `sase memory init`"_ instead of showing an opaque diff.

Read authority, stated once and applied everywhere:

| Consumer                                  | Source                                                                            |
| ----------------------------------------- | --------------------------------------------------------------------------------- |
| generated memory note and AGENTS.md       | snapshot only                                                                     |
| `sase memory init --check`, `sase doctor` | live registry compared against the snapshot                                       |
| create-time membership, field validation  | live registry only                                                                |
| display, chips, body rendering            | live registry, falling back to the snapshot, falling back to a degraded `unknown` |

### D7 — `plugins.required`

```yaml
plugins:
  required:
    - sase-github
    - sase-research-artifacts>=0.2
```

A list of PEP 508 requirement strings validated with
`packaging.requirements.Requirement` (`packaging` is already a dependency) and checked
against installed distribution versions, which `src/sase/plugins/inventory.py` already
extracts. This is not `repos.linked`: a linked checkout of `sase-github` does not
install it.

Every non-`builtin` `<plugin>@` prefix appearing anywhere in the project config must be
listed in `plugins.required`, or config validation reports an error. That single rule is
what keeps a project's declared dependencies honest.

Enforcement is graded by blast radius:

| Surface                                                          | Behavior                                                                                          |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `sase memory init`, `sase validate`                              | hard error — writing a knowingly incomplete instruction file is the whole failure being prevented |
| `sase bead create -T 'task(<slug>)'` for a missing plugin's slug | hard error naming the plugin and the install command                                              |
| `sase doctor`                                                    | new `plugins.required` check, `ERROR` severity, beside `plugins.resources`                        |
| interactive human CLI and ACE                                    | a gate offering to install (D9)                                                                   |
| agent / non-interactive contexts                                 | fail closed with the human-directed command; never auto-install                                   |
| `sase bead show` / `list` of an unknown type                     | degraded render, never a failure                                                                  |

### D8 — required, immutable, no backfill

Following `--size` exactly:

1. **Wire and read**: optional forever. Existing beads without a type stay valid.
2. **Create**: required. Bare `-T task` errors _and prints the agent-creatable types_.
   Rust rejects an empty `task_type` on task creation in one place; Python owns
   membership and field validation.
3. **Display**: a typeless legacy task renders as a dim `untyped` chip — a presentation
   label, never a catalog member, never offered at creation.

**`task_type` is immutable.** It is not on `BeadIssueUpdateFieldsWire`, and
`sase bead update --task-type` does not exist; the identity of a report does not change
because someone misfiled it. `sase bead update` rejects an attempt with a message
pointing at close-and-recreate.

**No heuristic backfill.** A wrong type on hundreds of historical beads is worse than an
honest `untyped`, and the store is event-sourced, so a bulk rewrite is a real event
mutation. `default_size` stays unset for every builtin so `-z/--size` remains the
intentional choice the sizes memory requires.

### D9 — distinct color per type, from one module

A new `src/sase/task_type_presentation.py` mirrors `src/sase/bead_type_presentation.py`
and is the only place any surface gets a task-type glyph, accent, chip, or CLI cell.

- A spec's declared `glyph`/`accent_color` wins.
- A type that declares neither gets a color by stable hash of its slug into a curated
  palette, so a plugin author needs no coordination and a slug's color never moves.
- Catalog assembly emits a `duplicate_task_type_color` **warning** naming both types and
  telling the author to declare `accent_color`.

Builtins declare theirs so the core set is hand-tuned and distinct from the four
existing issue-type accents (`#FFD700` plan, `#87D7FF` phase, `#D787FF` task, `#FF875F`
flag) and from `_FLAG_SOON_ACCENT` (`#FFAF00`):

| slug      | glyph | accent    |
| --------- | ----- | --------- |
| `bug`     | `⨯`   | `#FF5F5F` |
| `ci`      | `⚙`   | `#D7D700` |
| `feature` | `✦`   | `#5FD75F` |
| `flake`   | `≈`   | `#00D7D7` |
| `memory`  | `▤`   | `#8787FF` |
| `github`  | `⑂`   | `#B2B2B2` |

Every glyph must measure exactly one terminal cell under the pinned visual fixtures;
swap any that does not and record the swap.

### D10 — non-goals

- **Extracting or migrating `flag`.** It is an `IssueType` with a due-date gate that
  launches a removal agent, a Rust-validated payload, deliberate exclusion from GitHub
  mirroring and from the unique `external_ref` index, and a registry that is code-owned
  in this tree. Migrating it rewrites `issue_type` in an event-sourced store and makes
  flag beads eligible for `ready`, `snoozed`, and `+1`, none of which is meaningful. A
  type that needs a different _lifecycle_ is an issue type, not a task type. Its
  eventual home is project-local task-type config, after this registry has proven
  itself.
- Plugin-defined issue types or custom gate kinds.
- Per-type `stale_after_days`; that knob stays global.
- Historical backfill or reclassification.
- `boolean`, `number`, `datetime`, and `string_list` field types.
- Rust-side catalog membership enforcement (the snapshot makes it possible later; Python
  owns it in v1).
- Audited reads for `sase bead task-type show`. The glossary's `-r/--reason` log is
  deliberately not copied here.

### D11 — why no feature flag

Nothing user-reaching lands before it is ready. `task_type` is optional at every layer
until the final `enforce` phase, which flips both the Rust and the Python requirement
after the memory note, the skill, the chips, and the `github` type are all in place. The
`use:` prefix is a same-epic migration with no surviving old branch (D5), not a
deprecation window. If the owner would rather have a compatibility window for bare
`use:` values, that window is a `sunset` flag created with `sase flag new` — but this
plan does not take it.

---

## Phase 1: Task type on the bead wire and store

Work in the linked `sase-core` repository, opened through `/sase_repo`.

Add to `IssueWire` and `BeadCreateRequestWire` (`crates/sase_core/src/bead/wire.rs`):

```rust
#[serde(default, deserialize_with = "deserialize_option_non_empty_string")]
pub task_type: Option<String>,
#[serde(default, skip_serializing_if = "BTreeMap::is_empty")]
pub task_type_fields: BTreeMap<String, String>,
```

Deliberately **not** added to `BeadIssueUpdateFieldsWire` (D8).

Validation, in the existing issue-validation path:

- `task_type` and `task_type_fields` must both be empty unless `issue_type == task`.
- `task_type`, when present, is a non-empty snake_case slug bounded at 32 characters.
- Every `task_type_fields` key is a non-empty snake_case name bounded at 64 characters.
- `task_type_fields` may not be non-empty while `task_type` is absent.
- **No membership list and no closed SQL `CHECK`** — plugin types are an open set.

Store and reducer:

- Thread both fields through the create event, the reducer, `issues.jsonl` projection,
  and the JSONL codec.
- Add `task_type TEXT` and `task_type_fields TEXT NOT NULL DEFAULT '{}'` to the `issues`
  table in `crates/sase_core/src/bead/schema.rs`, with
  `CHECK(task_type IS NULL OR issue_type = 'task')` and an index on `task_type`. This is
  a new column on a rebuildable mirror, not a table rebuild; do not extend
  `issue_type_migration_sql`.

`create_issue` keeps accepting a task without a `task_type` for now; the requirement
flips in the `enforce` phase so nothing user-reaching breaks mid-epic.

Expose both fields through the `sase_core_py` bindings that already carry issue
payloads, and cover round-tripping, the cross-field rejections, and the SQLite CHECK
with tests. Run `just check` from the sase-core repo root before committing there.

## Phase 2: Task-type spec validation, digest, and body rendering in Rust

Work in the linked `sase-core` repository. Model the new module on
`crates/sase_core/src/artifact_ref/provider_spec.rs`, which is the closest existing
shape.

Add `crates/sase_core/src/task_type/` with:

- `TaskTypeSpecWire` and `TaskTypeFieldSpecWire` matching D2 exactly, at
  `schema_version: 1`. Reuse the scalar subset of the existing `PROPERTY_TYPES`
  vocabulary rather than declaring a new list.
- `validate_task_type_spec` — slug shape; reserved slugs (D4); required `label`,
  `summary`, `when_to_use`; the 120/400-character caps with a single-line `summary`;
  `#RRGGBB` accent; single-cell `glyph` via `unicode_width`, as `validate_tab_icon`
  already does; unique snake_case field names; per-type validator keys rejected on the
  wrong field type (for example `pattern` on an `integer`); `values` required and
  non-empty for `enum`; a compiling `regex` for `pattern`; a non-empty `role` subset of
  `{data, template}`; and every `{{ name }}` placeholder in `body_template` naming a
  declared field with `template` in its role.
- `task_type_spec_digest` — a stable sha256 over the normalized spec, exactly as
  `artifact_ref_provider_spec_digest` does.
- `validate_task_type_field_values(spec, values)` — returns one typed error per problem:
  missing required field, unknown field name, and per-type validator failures. This is
  the function a web frontend would call, so it must not depend on any Python-side
  catalog.
- `render_task_type_body(spec, values) -> String` — the pure Markdown block appended
  below a bead's description. Empty output when the spec declares no `body_template`.
- `TaskTypeSnapshotWire` plus `parse_task_type_snapshot` /
  `serialize_task_type_snapshot` for the D6 snapshot file, so the committed catalog is
  readable without Python plugin discovery. Serialization is deterministic: types sorted
  by slug, keys ordered, no version field.

Register every function in `crates/sase_core_py/src/lib.rs` under names that are
statically analyzable string literals, since `tools/check_sase_core_rs_bindings` scans
for exactly that. Cover each validator and the renderer with tests, including the
binding tests (`cargo test -p sase_core` alone is not sufficient in that repo).

## Phase 3: Required plugin prefix for every `use:` field

Introduce one shared parser — `parse_plugin_qualified_id(value) -> (plugin, id)` — used
by every `use:` consumer, accepting exactly `<plugin>@<id>` where `<plugin>` is
`builtin` or a distribution name and `<id>` is the provider or spec id. Anything else is
an error whose message names the correct replacement when the bare value resolves
unambiguously against the live registry, for example:

```
repos.sidecar.custom.research.ref.use: 'research' is missing its plugin prefix;
use 'sase-research-artifacts@research'
```

Wire it into:

- `src/sase/sidecar_ref_config.py` (`REF_USE_CONFIG_KEY` handling), where the provider
  lookup additionally verifies that the resolved provider's provenance package matches
  the declared prefix, so a moved provider is caught rather than silently accepted.
- `src/sase/config/file_hooks.py` (`_resolve_file_hook_provider`). **Close the
  silent-skip gap**: an unresolvable or unprefixed `use:` must still fail-soft for the
  running process, but must now also produce a diagnostic that `sase doctor` reports as
  `ERROR` and that fails `sase validate`, so a disappeared hook can never go unnoticed.
- `src/sase/config/sase.schema.json` — a `^[A-Za-z0-9._-]+@[a-z0-9][a-z0-9_-]*$` pattern
  on both `use` properties.

Migrate every known config in this phase:

- `sase/sase.yml`: `use: plan` → `use: builtin@plan`, `use: research` →
  `use: sase-research-artifacts@research`.
- The chezmoi-managed global config's `use: research-highlights` →
  `use: sase-research-artifacts@research-highlights`. Open the chezmoi repo through
  `/sase_repo`, edit only that value, and follow the commit-then-deploy rule from the
  generated-skills memory.
- `src/sase/main/_repo_init_config.py` (two `("use", "plan")` sites) → `builtin@plan`,
  and any fixture or doc that shows a bare `use:`.

Update the `Artifact Reference` glossary entry in `sase/sase.yml`, which currently says
`use: <provider>`, and regenerate with `sase memory init`.

## Phase 4: Required-plugin project config and graded enforcement

Add a top-level `plugins` section to `src/sase/config/sase.schema.json` with a single
`required` array of PEP 508 requirement strings, and a resolver module that:

- parses each entry with `packaging.requirements.Requirement`, reporting a config error
  for an unparseable entry or a duplicated distribution name;
- resolves installed versions through `src/sase/plugins/inventory.py`;
- returns a structured report of satisfied, missing, and version-mismatched
  requirements;
- cross-checks that every non-`builtin` `<plugin>@` prefix used anywhere in the project
  config appears in `plugins.required`, reporting the offending config path and the
  missing entry.

Apply the D7 enforcement table:

- Hard error in `sase memory init` and `sase validate`, raised **before** any drift
  comparison so a missing plugin never reports as spurious drift.
- A new `plugins.required` check in `src/sase/doctor/checks_plugins.py` beside
  `plugins.resources`, `ERROR` severity, listing each missing requirement with the exact
  `sase plugin install <name>` command.
- A reusable "fail closed" helper for agent and non-interactive contexts that prints the
  human-directed command and never attempts an install, because `sase plugin install`
  refuses to run from a dev checkout's virtualenv and restarts axe on success.

Add `plugins.required` to `sase/sase.yml` for this project:

```yaml
plugins:
  required:
    - sase-github
    - sase-research-artifacts
```

## Phase 5: Task-type discovery, catalog assembly, and diagnostics

Build `src/sase/task_types/` as a near-transcription of `src/sase/artifact_providers/`:

- `_hookspec.py` — `task_type_specs()` returning
  `Iterable[Mapping[str, Any]] | Mapping[str, Any] | None`, with a `hookimpl` marker
  plugins import.
- `_discovery.py` — add `sase_task_types` to `ENTRY_POINT_GROUPS` in
  `src/sase/plugins/inventory.py`, collect builtins first and then entry points sorted
  by name, tag each candidate with provenance, isolate load failures into
  `entry_point_load_failed` errors, and honor `SASE_DISABLE_PLUGINS` /
  `SASE_DISABLE_PLUGIN_TASK_TYPES`.
- `_models.py` — `TaskTypeProvenance`, `TaskTypeDiagnostic`, `TaskTypeRecord`
  (`task_type`, `spec`, `digest`, `provenance`, resolved `glyph`/`accent_color`), and
  `TaskTypeRegistry` with `by_slug`, `agent_creatable`, and `diagnostics`.
- `_validation.py` — validate each spec through the Rust binding, compute its digest,
  apply D4 conflict policy with `duplicate_task_type` and `builtin_task_type_shadowed`
  diagnostics, apply reserved slugs, and resolve presentation with the
  `duplicate_task_type_color` warning.
- `_project_config.py` — the `bead.task_types` source. An entry with
  `use: <plugin>@<slug>` deep-merges its sibling keys onto the referenced spec and
  replaces that slug in the catalog; an entry without `use:` defines a new slug and may
  not shadow a builtin or reserved slug. Register `bead.task_types` in
  `sase.schema.json`.
- `registry.py` — the memoized public accessor, cached on the same config token
  `file_hooks` uses so a config change invalidates it.

Prefer pluggy over the `sase_config` module pattern here for the reason the artifact
registry did: task types feed a generated, committed file, so a silently dropped type is
a silently wrong instruction file, and only the pluggy path produces `error`-severity
diagnostics that `sase doctor` already surfaces.

Surface the diagnostics through a new `beads.task_types` doctor check.

## Phase 6: Builtin task types and the `sase bead task-type` command group

Author `src/sase/task_types/_builtin.py` with `BuiltinTaskTypes` registering five specs.
Keep field counts small; a type grows fields additively and removing a required field is
a spec major version.

| slug      | fields                                                                                                                                         | notes                                                                                           |
| --------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `bug`     | `location` (string, required, data+template), `repro` (string, required, template), `impact` (string, optional, template)                      | A defect an agent found while doing unrelated work, not an external tracker bug.                |
| `ci`      | `node_id` (string, required, data+template), `sha` (string, optional, data), `why_not_flake` (string, required, template)                      | The complement of `flake`: a confirmed true failure. `triage.min_plus_ones: 0`.                 |
| `feature` | `proposal` (string, required, template), `why_out_of_scope` (string, required, template)                                                       | `why_out_of_scope` is what keeps this from becoming a wish list.                                |
| `flake`   | `node_id` (string, required, data+template, `pattern`), `repro_cmd` (string, optional, data+template), `evidence` (string, required, template) | `triage.min_plus_ones: 1` — the one type that must be corroborated.                             |
| `memory`  | `path` (string, required, data+template), `proposed_change` (string, required, template)                                                       | Close ritual stays "explicit user permission plus `sase memory init`"; say so in `when_to_use`. |

Each `when_to_use` is written for an agent deciding mid-task, fits the 400-character
cap, and is the text that will appear verbatim in every agent's Tier 1 context.

Add `sase bead task-type` with an exact `list` child, so a bare invocation defaults to
`list` through the central `_default_list_subcommands()` wiring — do not re-implement
that per command:

- `list` — a colored table of slug, label, summary, source, and whether agents may file
  it; `-j/--json` for machine-readable output; `-a/--all` to include agent-uncreatable
  types (hidden by default).
- `show <slug>` — label, summary, `when_to_use`, every field with type, requirement,
  role, help, and validator, the body template, the triage threshold, and provenance. No
  `-r/--reason`; reads are not audited here.

Give every public long option a short alias and keep subcommands and options sorted, per
the CLI rules memory.

## Phase 7: Typed task creation, field values, and the rendered body block

**Model and codec.** Add `task_type: str = ""` and `task_type_fields: dict[str, str]` to
`Issue` (`src/sase/bead/model.py`) with `validate()` rejecting either on a non-task
bead, thread them through `_project_mutations.create`, the Rust facade call, the JSONL
codec, the DB rows, and `cli_detail_json.py`.

**Grammar.** Extend `parse_type_arg` to accept `task(<slug>)` alongside bare `task`,
returning the slug. Its error message enumerates the accepted forms including the new
one.

**Field values.** Add a repeatable `-f/--field k=v` to `sase bead create`, where a value
of the form `@<path>` reads the value from that file so long prose (`evidence`, `repro`)
does not have to survive shell quoting. Duplicate keys are an error, not a silent
last-wins.

**Validation at create.** Resolve the slug through the registry; on a miss, error with
the available types and, when the snapshot knows the slug, name the plugin that provides
it and the install command. Validate values through the Rust
`validate_task_type_field_values` binding and print every problem at once rather than
one per run.

**Rendering.** The stored description is untouched. `sase bead show`, the ACE bead
detail pane, and bead pages append `render_task_type_body(spec, values)` below the
description under a clear separator. When the spec is unknown to this machine, print the
raw key/value pairs under a `(not installed on this machine)` header instead.

**Immutability.** `sase bead update` has no `--task-type`; an attempt errors with a
message pointing at close-and-recreate.

**Reading surfaces.** Add a repeatable `--task-type` filter to `sase bead list` and
`sase bead search`, a `task_type:` token to `src/sase/bead/filter_query.py` and the ACE
query profiles, and `--task-type untyped` to select legacy beads.

Bare `-T task` still succeeds and creates an untyped bead in this phase; the `enforce`
phase flips that.

## Phase 8: Task-type chips on every bead surface

Add `src/sase/task_type_presentation.py` implementing D9: `task_type_presentation`,
`task_type_chip`, and `task_type_cli_cell`, with the resolution order from D6 (live
registry → snapshot → degraded `unknown`) and a dim `untyped` presentation for legacy
beads. Include the curated fallback palette and a test asserting pairwise-distinct
accents across every task type _and_ the four `BEAD_TYPE_PRESENTATIONS` accents.

Route every surface through it — no surface may build its own label:

- `src/sase/bead/cli_query_render.py` — a fixed-width chip cell after the existing type
  cell, blank-padded for non-task beads so columns stay aligned.
- `src/sase/bead/cli_detail.py` — a `Task type` row.
- `src/sase/ace/tui/widgets/artifacts/beads_rendering.py` and `beads_detail.py` — the
  same chip in rows and the detail pane, beside the existing
  `("Type", bead_type_chip(...))` row.
- `src/sase/ace/tui/widgets/artifacts/bead_filter_bar.py` — a task-type filter control.
- `src/sase/ace/tui/modals/bead_editor_modal.py` — show the type on task beads; the
  field is read-only because the type is immutable.
- `src/sase/ace/tui/modals/wait_modal_beads.py`, `notification_tab_style.py`,
  `prompt_panel/_agent_bead_section.py`.
- `src/sase/bead_pages/roster.py` and `rendering_identity.py`.
- `src/sase/bead/_task_gate_preview.py` — the TaskTriage preview shows the type and its
  field values.
- `src/sase/integrations/_mobile_helper_beads.py` and the JSON payloads — add
  `task_type` and `task_type_fields` while `bead_type` keeps meaning `issue_type`.

Gate previews and bead pages are byte-compared or hashed, so the chip they embed must be
a pure function of the bead and the catalog, with no clock or terminal-width input.

Update the PNG visual snapshots with `--sase-update-visual-snapshots` and inspect the
diff artifacts before accepting.

## Phase 9: Per-type corroboration thresholds

Add `effective_min_plus_ones(issue, *, registry, global_default)` next to the existing
pure predicates in `src/sase/bead/task_triage_policy.py`: a bead with a known
`task_type` uses that spec's `triage.min_plus_ones` (spec default `0`); a bead with no
type, or a type unknown to this machine, uses the global
`bead.task_triage.min_plus_ones`. Keep both functions pure — callers keep owning the
clock.

Thread it through `src/sase/scripts/sase_chop_bead_task_triage.py`,
`sase_chop_bead_stale_cleanup.py`, `src/sase/bead/stale_cleanup_gate.py`, and every
preview that quotes the threshold. Update `src/sase/bead/config.py` and the
`bead.task_triage.min_plus_ones` description in `sase.schema.json` and
`src/sase/default_config.yml` to say it now applies to untyped legacy beads and to types
that declare no threshold. `stale_after_days` stays global (D10).

## Phase 10: Committed catalog snapshot and the generated task-type memory note

**Snapshot.** `sase memory init` writes `sase/task_types.json` at
`resolve_project_layout(root).namespace_root.path / "task_types.json"`, serialized
through the Rust snapshot binding so the bytes are deterministic. It joins the existing
expected-file machinery in `src/sase/main/init_memory/` so `--check` reports it like any
other managed file. `--check` compares live-registry digests against the snapshot and
names the drifting type and its package. The `plugins.required` guard from phase 4 runs
before this comparison.

**Generated short memory note.** Add
`src/sase/main/init_memory/templates/memory-sase-task-types.template.md` and render it
to `sase/memory/task_types.md` as a `short` note whose parent is `AGENTS.md`, alongside
the existing generated `sase.md`. It contains:

- the discovered-work workflow prose **moved out of** `memory-sase.template.md`'s
  `## File Discovered Work As Task Beads` section, which that template loses in this
  phase;
- one block per agent-creatable type: label, slug, `when_to_use`, required and optional
  field names, and the `sase bead task-type show <slug>` pointer.

Types with `agent_creatable: false` are omitted, which is exactly why that field exists.

For the project root the note renders from the committed snapshot. For the home root,
which has no project and no snapshot, it renders from the **builtin catalog only** — not
from the live registry — so the home instruction file stays a pure function of the
installed sase version and never varies with plugins.

**Bead memory.** Update `memory-sase-beads.template.md` so the "Types, Tiers, And
Launching" section documents `-T "task(<slug>)"`, `-f/--field`, immutability, and the
new child note.

**Skill.** Update `src/sase/xprompts/skills/sase_new_task.md`: steps 4 and 5 gain
`--task-type <slug>` for a same-type search before the all-types sweep (semantic
duplicates legitimately cross types — a `flake` that is really a `ci` failure), and step
7 gains choosing a type and supplying its fields. The skill must **not** embed any
project's type list; it points at `sase bead task-type list`. Follow the
commit-then-deploy rule before `sase skill init --force`.

Every edit in this phase is to a packaged template or skill source under `src/`, not to
a canonical memory note under `sase/memory/`. The notes themselves are regenerated by
`sase memory init`. Do not hand-edit any file under `sase/memory/`; a hand-authored note
such as `gotchas.md` is out of scope for this epic and needs its own explicit user
permission.

Regenerate with `sase memory init` and commit the resulting `AGENTS.md`, provider shims,
memory notes, README, and snapshot.

## Phase 11: Missing-plugin gate offering to install

Give the human the install offer that agent contexts deliberately do not get.

Add a chop (`src/sase/scripts/sase_chop_plugins_required.py`, registered in the
five-minute lane beside `bead_task_triage`) that, for each enabled project, compares
`plugins.required` against the installed distributions and raises at most one gate per
project per distinct missing set. Follow the existing gate-spec shape
(`src/sase/bead/_flag_gate_spec.py` is the closest model): a `plugins_required` kind, a
deterministic generation so a re-run does not duplicate a notification, cancellation
when the set becomes satisfied, and a preview listing each missing requirement with the
project that needs it.

Options:

- **Install** — runs `sase plugin install <name>` for each missing requirement. When
  sase is not a `uv tool` install, the command fails fast with the same actionable
  message `sase plugin install` already prints, and the gate stays pending rather than
  reporting a phantom success. Note in the preview that a successful install restarts
  axe.
- **Dismiss** — records the decision so the gate does not immediately return, until the
  required set changes.

Nothing here runs from an agent turn: the chop only raises the gate, and the install
runs from the answering surface.

## Phase 12: The `github` task type and mirror wiring

Work in the linked `sase-github` repository, opened through `/sase_repo`.

Register a `sase_task_types` entry point exposing one spec: `task_type: github`,
`agent_creatable: false`, no fields beyond what `external_ref` already carries, the
glyph and accent from D9, and a `when_to_use` that states plainly that agents never
create this type and that beads of it are created by the external issue mirror.

Do **not** add an `IssueType.GITHUB`. Mirrored issues are already `task` beads created
at `status=open` with `size=small` and an `external_ref`, and the labor split is already
clean: provider-neutral `IssueWire` in sase, `gh issue` calls in sase-github,
reconciliation in `src/sase/external_mirror/`, and `external_ref` uniqueness in
sase-core.

In this repository, have `src/sase/external_mirror/_issue_apply.py` pass
`task_type="github"` when it creates a mirrored bead. The mirror only runs when
sase-github is installed, so the type is always available where it is needed; if it is
somehow absent, fail the mirror run with the `plugins.required` message rather than
creating an untyped bead. `task_type` is immutable, so reconciliation of existing
untyped mirrored beads is unaffected and no backfill happens.

`create.default_status` is not needed: the mirror already creates at `open` and
`task_gate_suppressed` only fires on `ready`, so today's never-flood-triage behavior is
preserved for free.

## Phase 13: Make `task_type` required end to end

The flip, in this order:

1. In the linked `sase-core` repository, add the requirement to `create_issue`
   (`crates/sase_core/src/bead/mutation.rs`) beside the existing size check:

   ```rust
   if request.issue_type == IssueTypeWire::Task && request.task_type.is_none() {
       return Err(BeadError::validation(
           "new task issue creation requires an explicit task type",
       ));
   }
   ```

   Land and verify it there first.

2. In this repository, make bare `-T task` an error in
   `src/sase/bead/cli_crud_create.py` that prints the agent-creatable slugs with their
   summaries, matching the existing `--size` error's shape. Reject a create whose type
   declares required fields that were not supplied, naming each one.

3. Update `-h/--help` for `sase bead create` and every in-repo caller, fixture, and
   example that still creates a bare `task`.

Legacy sizeless-style behavior is preserved: existing untyped beads stay readable,
launchable, and closable, and render as `untyped`.

## Phase 14: Documentation, glossary, and end-to-end verification

**Documentation.** Update `docs/beads.md` with the task-type model, the
`-T 'task(<slug>)'` and `-f/--field` grammar, immutability, and the degraded render;
document the spec shape, the three sources, and the `sase_task_types` entry point in the
plugin docs; document `plugins.required` and the `<plugin>@` prefix in the configuration
docs; and add `sase/task_types.json` to whatever inventory lists managed generated
files.

**Glossary.** Add `Task Type` and `Required Plugin` entries to `memory.glossary` in
`sase/sase.yml`, and update the `Artifact Reference` entry if phase 3 did not already.
Regenerate with `sase memory init`.

**Verification.** Beyond each phase's own tests, prove end to end:

- a full round trip:
  `sase bead create -T 'task(flake)' -f node_id=... -f evidence=@file` → `show` renders
  the body block → `list --task-type flake` selects it → the chip appears in CLI, ACE,
  and bead-page output;
- a type from a project-config `use: builtin@bug` override changes only what it
  declares;
- a plugin-provided type renders degraded, without error, when its plugin is absent, and
  `sase bead create` for that slug fails with the install command;
- `sase memory init --check` is clean on a machine with every required plugin, reports a
  digest change when a spec is edited, and reports the missing plugin — not drift — on a
  machine without one;
- two machines with different _optional_ plugin sets generate byte-identical
  `AGENTS.md`;
- `sase doctor` reports the `plugins.required` and `beads.task_types` checks correctly
  in both healthy and broken states.

Run `just check-full` through `/sase_monitor` with a `--next` follow-up, and update the
PNG visual snapshots if any chip changed a rendered row.
