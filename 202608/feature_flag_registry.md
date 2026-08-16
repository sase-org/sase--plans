---
tier: tale
title: The typed feature-flag registry, resolver, and snapshot
goal:
  SASE has a code-owned boolean feature-flag registry whose values resolve through the
  existing config layer chain into one immutable per-process snapshot, a strict
  SASE_FEATURE_FLAGS transport that pins every child process to the parent's resolved
  values, and a `feature_flags` block generated into the config JSON Schema from the
  registry.
size: medium
proposed_by: bbugyi200.athena.sase-nb.2
bead: sase-nb.2
create_time: 2026-08-16 12:50:59
status: wip
---

- **PARENT:** [202608/feature_flags.md](feature_flags.md)
- **BEAD:**
  [sase-nb.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-nb/sase-nb.2.md)

# Plan: The typed feature-flag registry, resolver, and snapshot

This is the `registry` phase (bead `sase-nb.2`) of epic `sase-nb`,
`plan:202608/feature_flags.md`. It builds the flag mechanism itself and has **no
dependency on the `flag` bead type** — nothing here reads or writes a bead. The registry
ships **empty of temporary flags**; phase `consumer` (`sase-nb.9`) adds the first two
real entries, and phase `lint` (`sase-nb.5`) adds the static and bead-status enforcement
that consumes this phase's public API.

## Grounding

Verified in this workspace at `c9ef67510` with the linked `sase-core` checkout at
`0.27.12`.

| Fact                                                                       | Evidence                                                                                                                                              |
| -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| The layer chain is `default` → `plugin:*` → `user` → `overlay:*` → `local` | `load_config_layers` (`config/layers.py:126-262`) returns `ConfigLayer`s in that order                                                                |
| Every layer already carries its own unmerged `data` and `path`             | `ConfigLayer.data` / `.path` (`config/layers.py:69-88`)                                                                                               |
| Layer loading is ~6.5 ms warm and uncached                                 | measured: `import 178.8ms merged 46.6ms layers 6.5ms layers2 6.4ms`                                                                                   |
| ACE excludes project-local config _inside_ its handler                     | `set_include_local_config(False)` at `main/ace_handler.py:161`, after `main()` dispatch                                                               |
| The schema root is closed                                                  | `sase.schema.json` has `additionalProperties: false` and 44 top-level properties                                                                      |
| Named `properties` + open `additionalProperties` renders per-flag rows     | verified against the live binding: `feature_flags2.demo_flag` → `kind: "scalar"`, `types: ["boolean"]`, `has_default: true`                           |
| An **empty** `properties: {}` still classifies as a closed container       | verified: `feature_flags` → `kind: "object"`, `leaf: false`, no children; `has_props` is `Some` for an empty map (`sase-core config/schema.rs:52-55`) |
| `additionalProperties: {"type": "boolean"}` type-checks unknown keys       | verified via `config_validate`: `{"unknown_flag": 3}` → `type_mismatch`, `{"unknown_flag": true}` → `[]`                                              |
| Registry-owned defaults are precedent                                      | 14 of 44 schema keys have no `default_config.yml` entry (`artifact_refs`, `timezone`, …)                                                              |
| Children inherit env for free at 15 spawn sites                            | 15 `os.environ.copy()` call sites; none has an allowlist                                                                                              |
| The agent runner is a real second process boundary                         | `axe/run_agent_runner.py:257` `main()`, spawned by path from `agent/launch_spawn.py:104`                                                              |
| One env gate is evaluated at **import** time                               | `ace/tui/bindings.py:257` appends a `Binding` at module scope                                                                                         |
| Tests already have per-test env scrubbing and cache resets                 | `_clear_agent_env_vars` (`tests/_conftest_environment.py:241`), `_clear_config_caches` (`tests/_conftest_runtime.py:94`), both `autouse=True`         |
| Symbols a later epic phase will consume are whitelisted, not deleted       | `--epic-symbol "sase-n4(...)"` entries in `Justfile:307-313`                                                                                          |
| Extensionless `tools/` checkers are the established shape                  | `tools/check_test_wait_helpers`, `tools/validate_changelog`; typechecked by `tools/typecheck_extensionless_tools`                                     |
| No `feature_flags` identifier exists anywhere in the tree yet              | `grep -rn "feature_flag" src/ tests/ docs/` returns nothing                                                                                           |

## Epic decisions this phase must not revert

These come from `plan:202608/feature_flags.md` §"Decisions a phase worker must not
silently revert". They are restated because this phase is where three of them are
implemented:

- **2.** Nothing about a flag's resolved value ever depends on wall-clock time. There is
  no clock in this package at all.
- **6.** `plugin:*` layers may register flags but may never flip a first-party default.
- **7.** One immutable snapshot per process, resolved at a command or app boundary,
  never at import time. The parent re-encodes the **resolved** snapshot into
  `SASE_FEATURE_FLAGS` for children.
- **8.** The flag registry stays in Python. No `sase-core` change belongs to this phase.
- **9.** The registry is an inventory: no parent flags, no implied flags, no "enable
  everything" switch.
- **10.** Do not introduce a second deprecation system; `UNSUPPORTED_TOP_LEVEL_KEYS` /
  `DEPRECATED_TOP_LEVEL_KEYS` keep owning the config surface.

## Decisions this plan settles

The epic left these open or under-specified. Implement them as written.

**A. Resolution is a pure function; only `snapshot.py` touches the world.**
`resolve_feature_flags(...)` takes definitions, an ordered sequence of layer inputs, an
override mapping, and the raw env string, and returns a `FeatureFlagSnapshot`. It
imports nothing from `sase.config`, opens no file, and reads no `os.environ`. Every
resolution rule is therefore testable without a filesystem or a monkeypatch.

**B. The snapshot is lazy-once, with explicit boundary installs.** `current_flags()`
returns the process snapshot, building it on first access and memoizing it;
`install_process_feature_flags()` forces that build now, emits the one log line, and
exports the env value. It is idempotent and returns the existing snapshot on a second
call.

The epic names `main/entry.py:main` as a boundary, and this plan deliberately does not
install there. `handle_ace_command` calls `set_include_local_config(False)` _inside_ the
dispatch that `main()` performs, so an eager install in `main()` would resolve ACE's
flags with project-local config still included — the exact leak decision 7 exists to
prevent — and would also add ~7 ms of uncached layer IO to the `sase bead` fast path
(`main/bead_fast_path.py`) and to every short command. Lazy-once keeps "one immutable
snapshot per process, never at import time" literally true while letting each boundary
choose when to pin. Boundary installs go where a long-lived process or a child spawner
must be deterministic: ACE, the agent runner, and the axe daemon.

**C. Only `user`, `overlay:*`, and `local` are authoritative for first-party keys.** The
`default` layer is not: defaults live in the registry, so a `feature_flags` block in
`default_config.yml` is a mistake and produces a warning diagnostic. `plugin:*` layers
are skipped silently — a plugin naming a key it will one day register is not an error,
and honoring one would violate decision 6.

**D. Malformed env is fatal; an unknown env key is not.** `SASE_FEATURE_FLAGS` that is
not JSON, is not an object, or holds a non-boolean value raises `FeatureFlagEnvError` —
an operator typed that into this process. An **unknown key** in the env value only
warns. SASE agents run from ephemeral `sase_<N>` workspace clones pinned to different
commits, so a parent exporting a key its child's build does not know is routine, not
exceptional; making that fatal would brick every child during a rollout. The warning is
a diagnostic on the snapshot, so `sase doctor -C flags.overrides` (phase `cli`) can
still escalate it.

**E. An unknown key in a _config file_ warns and is ignored** (epic exit condition), for
the same reason spelled out in the epic: a config file outlives the flag it names.

**F. A `local` override of a `scope: "global"` flag warns and is ignored.** Project
scope is a property of the definition, not of the writer.

**G. The env export carries every registered key, not just the non-default ones.** This
is the literal reading of decision 7 and is what actually closes the hazard: a flag the
parent resolved to its registry default would otherwise be flipped by the child's own
project-local config mid-operation. The cost is that `source` reads `env` for nearly
everything in a child process, which is a presentation problem phase `cli` already owns
("`env` provenance is surfaced prominently").

**H. `override_flags(...)` sits exactly where the epic put it** — above `local`, below
env. Because that means an ambient `SASE_FEATURE_FLAGS` in a developer's shell would
beat a test's own override, test hermeticity is bought by scrubbing the variable per
test (deliverable 7), not by reordering the chain.

## Deliverables

### 1. `src/sase/feature_flags/models.py`

Frozen dataclasses and the two `Literal` aliases. No behavior beyond lookups.

```python
FlagKind = Literal["beta", "wip", "sunset", "ops"]
FlagScope = Literal["global", "project"]
FlagSource = Literal["default", "user", "overlay", "local", "override", "env"]

@dataclass(frozen=True)
class FeatureFlagDefinition:
    key: FeatureFlag
    kind: FlagKind
    description: str
    default: bool
    scope: FlagScope
    bead: str | None = None      # flag bead id; required unless kind == "ops"
    rationale: str = ""          # required when kind == "ops"

@dataclass(frozen=True)
class FeatureFlagDecision:
    key: str
    enabled: bool
    default: bool
    source: FlagSource
    source_detail: str           # "overlay:sase_athena.yml", a path, or ""
    overridden: bool             # a layer above the registry default supplied a value

@dataclass(frozen=True)
class FeatureFlagDiagnostic:
    severity: Literal["warning", "error"]
    code: str                    # "unknown_key", "not_boolean", "scope_violation", …
    message: str
    source: str                  # layer name, "env", or "registry"

@dataclass(frozen=True)
class FeatureFlagSnapshot:
    decisions: Mapping[str, FeatureFlagDecision]
    diagnostics: tuple[FeatureFlagDiagnostic, ...]
```

`FeatureFlagSnapshot.__post_init__` re-wraps `decisions` in `types.MappingProxyType` via
`object.__setattr__`, so a caller holding the snapshot cannot mutate it. It exposes
`enabled(key: FeatureFlag) -> bool`, `decision(key: FeatureFlag) -> FeatureFlagDecision`
(both raising `FeatureFlagError` on an unregistered key), and
`non_default() -> tuple[FeatureFlagDecision, ...]` returning the `overridden` decisions
in key order. `FeatureFlagDefinition.validate()` enforces the `bead`/`rationale`
invariant and a `snake_case` key so a bad definition fails at registry import; phase
`lint` re-checks it statically.

`FeatureFlagError` and `FeatureFlagEnvError(FeatureFlagError)` live here.

### 2. `src/sase/feature_flags/registry.py`

```python
class FeatureFlag(StrEnum):
    """Every SASE feature flag key. Add members through `sase flag new`."""
    # Intentionally empty: the `consumer` phase adds the first two flags.
```

An `enum.StrEnum` with no members is legal in 3.12 and iterates empty.
`FEATURE_FLAG_DEFINITIONS: Mapping[FeatureFlag, FeatureFlagDefinition]` is the table
(empty in this phase, wrapped in `MappingProxyType`), plus
`feature_flag_definitions() -> Mapping[str, FeatureFlagDefinition]` keyed by the string
value — the form the resolver, the schema generator, and phase `lint` all consume. A
module-level assertion pins every table entry's `key` to its mapping key and calls
`validate()`, so a hand-edited table cannot import.

Add a module docstring stating the two rules a definition author must follow:
`remove_by` never appears here (it lives on the bead — epic decision 1), and no
definition may be added by hand rather than through `sase flag new` (phase `cli`).

### 3. `src/sase/feature_flags/resolver.py`

Pure, per decision A.

```python
@dataclass(frozen=True)
class FeatureFlagLayerInput:
    name: str                    # "default", "plugin:x", "user", "overlay:a.yml", "local"
    detail: str                  # path, or "" for package-backed layers
    values: Mapping[str, Any]    # the layer's raw `feature_flags` mapping

def resolve_feature_flags(
    *,
    definitions: Mapping[str, FeatureFlagDefinition],
    layers: Sequence[FeatureFlagLayerInput],
    overrides: Mapping[str, bool] | None = None,
    env_value: str | None = None,
) -> FeatureFlagSnapshot: ...
```

Seed every decision from its definition default (`source="default"`,
`overridden=False`), then apply in order:

1. each layer in `layers` order, skipping `default` (warn if it carries values, per
   decision C) and skipping `plugin:*` silently;
2. `overrides` (`source="override"`, `source_detail=""`);
3. `env_value`, parsed by `env.py` (`source="env"`, `source_detail=SASE_FEATURE_FLAGS`).

Per-value rules, each producing a diagnostic and leaving the prior decision intact when
it rejects: unknown key → `unknown_key` warning (decisions D/E); non-`bool` value →
`not_boolean` warning for a file layer, and a fatal `FeatureFlagEnvError` for env
(decision D — `env.py` raises before the resolver sees it); a `local` layer value for a
`scope: "global"` definition → `scope_violation` warning (decision F). Accepting a value
sets `overridden=True` even when it equals the default, because "a layer spoke" is the
fact diagnostics need.

Use `type(value) is bool` rather than `isinstance`, matching the existing config
accessors in `config/core.py`, so `1` and `0` are not silently booleans.

### 4. `src/sase/feature_flags/env.py`

`SASE_FEATURE_FLAGS_ENV = "SASE_FEATURE_FLAGS"` plus:

- `parse_feature_flags_env(raw: str) -> dict[str, bool]` — strict. Raises
  `FeatureFlagEnvError` with the offending text when the value is not JSON, is not an
  object, or holds a non-boolean. Unknown keys pass through; the resolver classifies
  them (decision D).
- `encode_feature_flags_env(snapshot) -> str` — `json.dumps` of every decision's
  `enabled`, `sort_keys=True`, `separators=(",", ":")`, so the exported value is stable
  and diffable (decision G). Returns `"{}"` for an empty registry.
- `apply_feature_flags_env(snapshot, env=os.environ) -> None` — writes the encoded value
  into `env`, which is what makes the 15 `os.environ.copy()` spawn sites inherit it with
  no edit.

### 5. `src/sase/feature_flags/snapshot.py`

The only module in the package that touches process state. Holds the module-global
snapshot, a `threading.Lock` guarding first build, and the override stack.

- `current_flags() -> FeatureFlagSnapshot` — build-once-and-memoize.
- `install_process_feature_flags() -> FeatureFlagSnapshot` — forces the build, logs one
  `log.info` naming every `non_default()` decision as `key=value (source)` plus any
  diagnostics (`log.warning` each), calls `apply_feature_flags_env`, and returns. A
  second call returns the installed snapshot without re-logging.
- `override_flags(**values: bool) -> Iterator[FeatureFlagSnapshot]` — a
  `@contextmanager` that pushes onto the override mapping, drops the memoized snapshot,
  yields the rebuilt one, and restores on exit (including on exception). Nested use
  composes.
- `reset_process_feature_flags() -> None` — test-support reset consumed by conftest.

The build projects `ConfigLayer`s into `FeatureFlagLayerInput`s: import
`load_config_layers` from `sase.config.core` **inside** the function so importing
`sase.feature_flags` stays cheap and cannot pull the config stack at import time. A
layer's `feature_flags` entry that is present but not a mapping becomes a
`not_a_mapping` warning rather than a crash. `detail` is `layer.path or ""`.

### 6. `src/sase/feature_flags/schema.py` and `tools/sync_feature_flags_schema`

- `feature_flags_schema_block(definitions=...) -> dict[str, Any]` — builds
  `{"type": "object", "description": ..., "additionalProperties": {"type": "boolean"}, "properties": {key: {"type": "boolean", "description": ..., "default": ...}}}`
  in key order. A `kind: "sunset"` definition also emits `"deprecated": true`, which the
  field model already reads (`schema.rs:80-83`).
- `feature_flags_schema_drift(document) -> str | None` — returns a human-readable
  difference, or `None` when the document's `feature_flags` block equals the generated
  block. Phase `lint` rule 2 must call **this** function rather than re-derive the
  block.
- `tools/sync_feature_flags_schema` — extensionless tool, `--check` (default, exit 1 on
  drift, printing the drift and the `--write` hint) and `--write` (rewrites
  `src/sase/config/sase.schema.json` in place, preserving the file's existing 2-space
  indent and trailing newline).

Then run `tools/sync_feature_flags_schema --write` to add the block to
`src/sase/config/sase.schema.json`, taking the top-level property count from 44 to 45.
With an empty registry the block's `properties` is `{}`, which the binding still
classifies as a closed container (grounding), so Config Center gains one empty
`feature_flags` group now and one row per flag as `consumer` lands.

Do **not** add a `feature_flags` key to `src/sase/default_config.yml`: defaults live in
the registry, and 14 existing schema keys already work that way.

### 7. Boundary wiring and test isolation

- `main/ace_handler.py` — call `install_process_feature_flags()` in `handle_ace_command`
  immediately **after** `set_include_local_config(False)` and before `AceApp` is
  constructed. Ordering is the whole point; a comment must say so.
- `axe/run_agent_runner.py` — call it at the top of `main()`, before `_build_run_state`,
  so an agent run pins its flags once and every tool it spawns inherits them.
- `main/axe_handler.py` — call it at the top of `handle_axe_command`, so the daemon and
  every chop it spawns share one resolved set.
- `tests/_conftest_environment.py` — add `SASE_FEATURE_FLAGS` to
  `_clear_agent_env_vars`' `keys_to_clear` set, so an ambient value in the developer's
  shell cannot beat a test's `override_flags` (decision H).
- `tests/_conftest_runtime.py` — call `reset_process_feature_flags()` from
  `_clear_config_caches`, so a snapshot built by one test cannot leak into the next.
- `Justfile` — add `--epic-symbol "sase-nb(<symbol>)"` entries, in the existing
  alphabetical block, for every public symbol this phase ships whose only consumer is a
  later phase (at minimum `FeatureFlag`, `FeatureFlagDecision`, `FeatureFlagDefinition`,
  `FeatureFlagDiagnostic`, `feature_flag_definitions`, `feature_flags_schema_block`,
  `feature_flags_schema_drift`, `override_flags`, `resolve_feature_flags`). Add only the
  ones symvision actually reports; each is self-cleaning once its consumer lands.

`src/sase/feature_flags/__init__.py` re-exports the public surface (`FeatureFlag`, the
three models, the two errors, `current_flags`, `install_process_feature_flags`,
`override_flags`, `SASE_FEATURE_FLAGS_ENV`) so call sites in later phases import one
module.

## Tests

Under `tests/feature_flags/`, mirroring the module split. Resolver tests build
`FeatureFlagLayerInput`s and a synthetic definitions table directly — no tmp_path, no
config files, no clock. Because `FeatureFlag` ships empty, add
`tests/feature_flags/_helpers.py` with `demo_flag(...)` returning a
`FeatureFlagDefinition` whose `key` is `cast(FeatureFlag, "demo_flag")`; a `StrEnum`
member is a `str` at runtime, so every mapping lookup behaves identically and production
call sites keep their enum-only typing.

Cover, at minimum:

- resolution from each authoritative layer in turn, asserting `enabled`, `source`,
  `source_detail`, and `overridden`;
- last-writer-wins across `user` → `overlay:a` → `overlay:b` → `local`;
- the `default` layer warns and does not win; a `plugin:*` layer is ignored **silently**
  (decision C, epic decision 6);
- a `scope: "global"` key in `local` warns and is ignored; a `scope: "project"` key in
  `local` wins (decision F);
- an unknown key in a file layer warns and is ignored; a non-boolean file value warns
  and leaves the prior decision intact;
- `SASE_FEATURE_FLAGS` beats an `override_flags` value (decision H, pinning the chain);
- malformed env — not JSON, a JSON array, a non-boolean value — raises
  `FeatureFlagEnvError`; an unknown env key warns only (decision D);
- `encode_feature_flags_env` → `parse_feature_flags_env` round-trips every decision and
  is byte-stable across two calls; the encoded value carries **every** registered key
  (decision G);
- `apply_feature_flags_env` writes into a supplied mapping, and a snapshot built from
  that mapping reproduces the parent's values — the child-pinning property;
- `current_flags()` returns the identical object across calls; the returned `decisions`
  mapping rejects mutation;
- `override_flags` nests, restores on exception, and restores the previously memoized
  snapshot;
- schema: `feature_flags_schema_block` is stable and ordered; a `sunset` definition
  emits `deprecated: true`; `feature_flags_schema_drift` returns `None` for the shipped
  `sase.schema.json` and a message for a mutated copy — this is the test that keeps the
  committed schema honest;
- schema integration: feed a document containing the generated block to
  `sase_core_rs.config_field_model` and assert `feature_flags` is `kind: "object"` and
  each flag is a `scalar` boolean row, and to `config_validate` and assert a string
  value under a flag is a `type_mismatch` while an unknown boolean key is accepted (the
  downgrade-tolerance property);
- boundaries: `handle_ace_command` installs the snapshot after
  `set_include_local_config(False)` (assert the call order, not just that both ran), and
  `install_process_feature_flags` is idempotent and logs the non-default set exactly
  once;
- no import-time resolution: `importlib.import_module("sase.feature_flags")` followed by
  a `reset_process_feature_flags()`-clean state leaves no snapshot built, and importing
  `sase` performs no resolution.

Add the new directory to any test-collection config that enumerates test roots only if
one exists; `tests/` is globbed today, so no change is expected.

## Out of scope

The `flag` bead type and anything that reads a bead (`bead`, `look`); `remove_by`,
due-ness, and the clock (epic decisions 1-3); `tools/check_feature_flags` and its nine
rules (`lint`); the `FlagTriage` gate (`gate`); `sase flag` and the `flags.*` doctor
checks (`cli`); any real flag entry and the `SASE_CODER_INHERIT_PLANNER_CHAT` /
`SASE_DISABLE_PRETTIER` conversions (`consumer`); `sase/memory/sase_flags.md`, the
glossary, and `docs/` (`memory`); and migrating `ace/tui/bindings.py:257` — it is a
named target for `consumer`, not this phase. No `sase-core` change belongs here (epic
decision 8).

## Verification

Run `just install` first — this is an ephemeral `sase_<N>` workspace — then
`just check`. `just check-full` belongs to the epic's land agent; if `just check`'s
scoped run escalates or reports an unusual selection, hand `just check-full` to
`/sase_monitor` with a `--next` action rather than running it inline.

The phase is done when the epic's exit condition holds: a registered flag resolves
correctly from every layer with correct provenance, a project-scoped override is honored
by the CLI and ignored by ACE, a malformed `SASE_FEATURE_FLAGS` fails loudly, and an
unknown key in a config file warns and is ignored. Because the shipped registry is
empty, "a registered flag" is exercised through the synthetic definitions table, and the
ACE-vs- CLI difference is asserted through the `local`-layer projection rather than by
launching the TUI.
