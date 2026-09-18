---
tier: epic
title: sase-core service foundations
goal: 'sase-core owns every piece of shared service-host behavior that later phases
  of the service-host epic (sase-11y) build on: the proc-wire `service` block with
  per-service retention, the `service.procs` config composer, the pure restart-decision
  function, the locked boot-scoped service state store, and the versioned service
  status snapshot. Each piece has PyO3 bindings and a thin typed Python facade, and
  the Procs query dialect gains the `service` and `svc:` fields.

  '
phases:
- id: proc-service-block
  title: Proc wire service block, per-service retention, Procs query fields
  depends_on: []
  size: medium
  description: 'proc-service-block: add the optional additive `service` block `{name,
    mode, source}` to proc rows and reserve requests in sase-core (no schema bump),
    validate it when a row is created, keep the newest 20 terminal rows per named
    service proc instead of counting them against the generic cap, add a shared service-name
    vocabulary module, and carry the block through the Python Proc/ProcReserve/ObservedProc
    models into new `service` and `svc:` Procs query fields.'
- id: service-config
  title: service.procs config composer, schema, defaults, and loader
  depends_on:
  - proc-service-block
  size: medium
  description: 'service-config: add the sase-core `service.procs` composer (field-by-field
    merge, whole-list replacement, explicit enabled:false, per-field provenance, plugin
    entries default-disabled, project-local layers ignored, reserved builtin names,
    oneshot rejected, invalid entries marked unavailable) with a `service_config_compose`
    binding; add the `service:` and `ace.procs` schema, the scheduler/gateway defaults,
    and the `sase.service.config` loader that fails closed on section-level errors.'
- id: restart-state
  title: Restart decisions and the locked service state store
  depends_on:
  - service-config
  size: medium
  description: 'restart-state: add the pure `decide_service_restart` function (restart
    policy, clean-exit rules, orchestrator-identical backoff and crash-loop accounting)
    and the flock-guarded `~/.sase/service/state.json` store (machine-local enablement
    overrides, boot-id-keyed stops, markers, host record) with bindings, plus the
    Python restart, state, paths, and boot-id facades.'
- id: status-wire
  title: Enablement resolution and the service status snapshot wire
  depends_on:
  - restart-state
  size: medium
  description: 'status-wire: add enablement-provenance resolution, the schema-versioned
    service status snapshot (pure state derivation plus a change token that ignores
    heartbeat churn), and atomic snapshot read/write in sase-core with bindings, plus
    the `sase.service.status` Python facade.'
proposed_by: bbugyi200.athena.sase-11y.2
parent_bead: sase-11y.2
create_time: 2026-09-16 15:15:21
status: done
bead_id: sase-11y.2.1
---

- **PROMPT:** [prompts/202609/core_service_foundations.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/core_service_foundations.md)
- **PARENT:** [202609/service_host_1.md](https://github.com/sase-org/sase--plans/blob/main/202609/service_host_1.md)
- **BEAD:** [sase-11y.2.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11y/sase-11y.2.1.md)

# Plan: sase-core service foundations (phase `core-service` of epic sase-11y)

This epic implements bead **sase-11y.2**, the `core-service` phase of the parent epic
**sase-11y**, "Service host and Services tab". Every phase worker MUST first read:

```bash
sase artifact read plan:202609/service_host_1.md "Implementing a sase-11y.2 sub-phase"
sase artifact read research:202609/service_host_and_services_tab/service_host_and_services_tab.md "Implementing a sase-11y.2 sub-phase"
```

The parent plan's "core-service" section and research §4–§6 give the requirements. This
plan makes the concrete decisions. Downstream consumers are **sase-11y.4**
(service-host: host runtime and CLI), **sase-11y.7** (services-tab), and **sase-11y.3**
(supervision-lib, which later delegates restart decisions to this epic's function).

## Decisions made while planning

These refine or correct the parent plan. The land step records them in the sase-11y.2
close note so the sase-11y.4 and sase-11y.7 workers see them.

1. **No `PROC_WIRE_SCHEMA_VERSION` bump.** The parent plan says "schema version bump",
   but that would lose data. The proc store (`crates/sase_core/src/procs/store.rs`)
   drops rows whose `schema_version` is not in `SUPPORTED_PROC_WIRE_SCHEMA_VERSIONS`,
   and deletes them on the next rewrite (see
   `unknown_fields_are_tolerated_and_malformed_rows_are_dropped_on_rewrite`). Also,
   `validate_reserve_request` requires the exact current version, and the Python
   `models.py` hardcodes 3. A bump would make any older `sase_core_rs` still running on
   the machine (an old TUI or orchestrator) delete every newly written row. The block is
   therefore an optional additive field, like the existing `xprompt_proc` block (added
   in sase-core `92a4fc4`/`b9c6ff0` without a bump). Accepted residual risk: an older
   binary that rewrites the store drops the unknown `service` field from rows until it
   is upgraded. The host is gated behind the `service_host` beta flag, and `sase update`
   restarts every process together.
2. **The service block is validated only when a row is created** (`append_proc` /
   `reserve_proc`). Reads and updates stay lenient, so a future mode or source value
   never makes older binaries drop the row or refuse to settle it. The block is not part
   of `ProcUpdateWire`, so it cannot be changed after creation.
3. **Retention**: a named service-proc row has a block with `source != "transient"` and
   a non-empty `name`. Named terminal rows are exempt from the generic
   `procs.history_limit`; each name keeps its newest `SERVICE_PROC_HISTORY_LIMIT = 20`
   terminal rows. Transient oneshots and all other rows keep the generic cap. Active
   rows are never pruned (unchanged).
4. **The Procs default query lives at `ace.procs.default_query`, not
   `tui.procs.default_query`.** There is no `tui` root key; TUI settings live under
   `ace`, and `ace.artifacts.stitches.default_query` is the existing precedent. This
   epic only adds the schema entry and the default (`"-service"`). sase-11y.7 wires the
   seed.
5. **An invalid entry is unavailable, not fatal.** A bad `service.procs.<name>` entry is
   still returned, with `available: false` and its reasons, so the Services tab can
   render `✕ name  unavailable: …` and the host never launches it. Only section-level
   errors set `fatal: true`: `service` is not a mapping, it has a key other than
   `procs`, or `procs` is not a mapping. On a fatal error, `load_service_config()`
   raises. This is the per-entry form of "fail closed like `load_axe_config()`".
6. **Restart semantics.** Policies are `always | on-failure | never` (default
   `on-failure`). Exit code 0 is always clean, and `success_exit_codes` adds more clean
   codes, as systemd's `SuccessExitStatus=` does. Termination by SIGHUP, SIGINT,
   SIGPIPE, or SIGTERM (portable numbers 1, 2, 13, 15) is also clean, per systemd's
   `on-failure` rule. A spawn error is a failure. Whenever a restart is scheduled, the
   backoff and crash-loop accounting reproduces `src/sase/axe/orchestrator.py` exactly:
   - A healthy run (≥ 300 s since start) resets the state.
   - Backoff starts at 1 s, doubles, and is capped at the maximum (default 60 s).
   - Failures older than 60 s leave the window.
   - A crash loop is 3 or more failures in the window, and it notifies once per episode.

   There is no permanent give-up on a crash loop (the orchestrator has none). As a
   result, policy `always` with the default tuning is behavior-identical to today's
   routine supervision.

7. **Every service wire uses epoch-second `f64` timestamps.** This matches the
   `agent_hold` precedent and makes age and countdown math trivial. Python renders the
   timestamps for display.
8. **Enablement overrides live in the state store.** `sase service proc enable|disable`
   writes to `~/.sase/service/state.json`, which is machine-local and not
   chezmoi-synced. The store also holds boot-scoped stops, markers, and the host record.
   A stop is active only while its recorded boot id equals the current one (two `None`
   values count as equal).

## Rules for every phase

- **Repos.** The Rust work happens in the linked `sase-core` repo. Open it with
  `sase repo open sase-core -r "<reason>"` and use only the printed path. Both repos get
  a `commit` decision in the final declaration.
- **Rust core boundary.** Composition, validation, wire shapes, the locked store, the
  status derivation, and restart decisions live in sase-core. Python facades only
  discover layers and resolve paths, boot ids, and clocks, then call the binding and
  wrap the result in frozen dataclasses. Do not reimplement logic in Python.
- **sase-core rules** (read its `AGENTS.md`):
  - Never edit crate/workspace versions or any `CHANGELOG.md`.
  - Verify with sase-core `just check` (`./scripts/check.sh all`), never with
    `cargo test -p sase_core` alone.
  - The workspace enables `serde_json`'s `preserve_order`.
- **Bindings.** Add bindings to `crates/sase_core_py/src/lib.rs`: the `//!` API list at
  the top, the `#[pyfunction]` definitions next to the related helpers, the
  `m.add_function` registration, and the binding-existence assertions in its test
  module. Python must call every binding as `require_rust_binding("<literal name>")`,
  because `tools/check_sase_core_rs_bindings` scans for literal names.
- **sase verification.** Run `just install` first; it rebuilds `sase_core_rs` from the
  linked sase-core checkout. Then run `just fix`, then `just check`.
- **Core pin.** Do not edit `sase-core-revision.txt`: the new sase-core commit does not
  exist until the host commits it. Record
  `PROPOSED FOLLOW-UP: ratchet sase-core-revision.txt past <phase> core commit` on your
  own phase bead. The land step ratchets once.
- **Symvision.** A new public Python symbol with no non-test consumer yet gets an
  `--epic-symbol '<bead>(<symbol>)'` line in the Justfile `_lint-symvision` recipe. Key
  it to its first real consumer: **`sase-11y.4`** for host/CLI consumers,
  **`sase-11y.7`** for TUI-only consumers. Never key an entry to `sase-11y.2` or its
  sub-phase beads; those close with this epic, and a stale entry breaks other agents'
  `just check`. Add entries only for symbols Symvision actually reports.
- **Do not touch** these files, which concurrent phases own:
  - `src/sase/procs/service.py` (sase-11y.1 renames it).
  - `src/sase/axe/orchestrator.py` and the new supervision module (sase-11y.3).
  - The TUI tab, gear, and footer code (sase-11y.7).

  Passing a `service=` block through `submit_proc_request` belongs to sase-11y.4 and
  sase-11y.8.

- **No feature flag.** Nothing in this epic launches a service proc. The only
  user-visible change is the pair of additive Procs query fields.
- **Tests.** Rust unit tests sit next to each module. Python tests go under a new
  `tests/service/` package (with `__init__.py`) or next to the existing proc and query
  tests named below. Keep Python files under the `toobig` limits.

## Phase: proc-service-block

**sase-core**

- New module `crates/sase_core/src/service/mod.rs` (`pub mod service;` in `lib.rs`)
  holds the shared vocabulary:
  - Mode constants `SERVICE_PROC_MODE_DAEMON = "daemon"` and
    `SERVICE_PROC_MODE_ONESHOT = "oneshot"`.
  - Source constants `builtin`, `plugin`, `user`, and `transient`, plus a
    `SERVICE_PROC_SOURCES` array.
  - `RESERVED_BUILTIN_SERVICE_PROCS = ["gateway", "scheduler"]`.
  - `pub fn validate_service_proc_name(name) -> Result<(), String>`: 1–64 characters,
    matching `^[a-z0-9][a-z0-9_-]*$`. Names become paths
    (`~/.sase/service/procs/<name>/`), YAML keys, and `svc:` values.
- `procs/wire.rs`:
  - Add `ProcServiceWire { name: Option<String>, mode: String, source: String }`. Omit
    `name` when it is `None`.
  - Add
    `#[serde(default, skip_serializing_if = "Option::is_none")] service: Option<ProcServiceWire>`
    to `ProcWire` and `ProcReserveWire`, but not to `ProcUpdateWire`.
  - Re-export `ProcServiceWire` from `procs/mod.rs` and `lib.rs`.
- `procs/store.rs`:
  - `proc_from_reserve_request` copies the block.
  - A new `validate_service_block` runs from `append_proc` (before taking the lock) and
    from `validate_reserve_request`. It trims `name` (empty becomes `None`) and returns
    `InvalidProc` unless:
    - `mode` is a known mode and `source` is a known source;
    - `source == "transient"` implies `mode == "oneshot"` and no name;
    - any other source has a name that passes `validate_service_proc_name`.
  - Add `pub const SERVICE_PROC_HISTORY_LIMIT: usize = 20`. `apply_retention` partitions
    terminal rows into named-service groups (per the decision above; newest 20 kept per
    name) and everything else (the generic `history_limit`). Pruned-log reporting is
    unchanged: only `proc-store`-owned logs are reported.
- `tests/python_wire_parity.rs`: add `service: None`. The fixture JSON must stay
  byte-identical, which proves the field is additive.
- Rust tests:
  - A row without a block serializes without a `service` key.
  - Reserve copies the block, and a later `update_proc` on other fields preserves it.
  - Invalid blocks are rejected on append and on reserve.
  - A stored row with an unknown mode or source is still loaded and still updatable.
  - Retention: two named services with 30 terminal rows each, plus 120 generic and
    transient terminal rows at `history_limit = 100`, leave exactly 20 rows per service
    and 100 generic rows. Active rows survive.
  - Extend the proc binding round-trip test in `sase_core_py` with a reserve that
    carries a block.

**sase**

- New `src/sase/procs/service_meta.py`:
  - A `ProcServiceBlock` frozen dataclass (`name: str | None`, `mode: str`,
    `source: str`).
  - `from_dict`, which is lenient and returns `None` for a non-mapping.
  - `to_dict`, which omits `name` when it is `None`.
  - An `is_named` property.
  - Constants mirroring the Rust vocabulary.
- `src/sase/procs/models.py`: `Proc.service` and `ProcReserve.service` are
  `ProcServiceBlock | None`. `from_dict` parses the block, and both `to_dict` methods
  emit `service.to_dict()` or `None`. `ProcReserve.to_dict` is currently a generic
  `getattr` loop, so the block must be converted explicitly. Add `Proc.service_name` /
  `is_service` convenience properties if the query adapter uses them.
- `src/sase/ace/tui/_proc_observer_models.py`: add
  `ObservedProc.service: ProcServiceBlock | None = None`.
  `_proc_observer_store.store_proc_row` copies it.
- `src/sase/ace/query_profile/profiles/_procs.py` adds two fields:
  - A bool field `service` ("a service proc run (daemon or oneshot)"). A bare `service`
    means `service:true`, so `-service` hides every service row.
  - A string field `svc` (`exact_match=True`, negatable, "service proc name").
- `src/sase/ace/tui/_proc_query.py`: `_proc_query_row` always sets `service` and sets
  `svc` only for named blocks. `_RowCacheKey` gains the block's name and mode.
- Tests:
  - `tests/test_procs_facade_models.py`: round-trip and lenient parse.
  - `tests/test_procs_facade_retention.py`: per-service retention through the real
    binding.
  - `tests/test_query_profile_procs.py`: the new fields.
  - `tests/ace/tui/test_proc_query.py`: `-service`, `service`, `svc:gateway`, and
    `-svc:gateway`.

## Phase: service-config

**sase-core** — `crates/sase_core/src/service/config.rs`, modeled on `config/axe.rs` but
much simpler (no aliases or legacy lists).

- **Wires** (`SERVICE_CONFIG_WIRE_SCHEMA_VERSION = 1`):
  - `ServiceConfigComposeRequestWire { layers: Vec<ConfigLayerInputWire> }`.
  - `ServiceConfigCompositionWire { schema_version, fatal: bool, procs: Vec<ServiceProcConfigWire>, diagnostics: Vec<ConfigDiagnosticWire>, ignored_layers: Vec<String> }`,
    with `procs` sorted by name.
  - `ServiceProcConfigWire` has these fields:
    - Identity and availability: `name`, `description: Option`, `available: bool`,
      `unavailable_reasons: Vec<String>`.
    - Origin: `source` (the kind of the first layer that mentions the name: `builtin`
      for the default layer, `plugin` for a plugin layer, otherwise `user`) and
      `declared_by` (that layer's label, `name` or `name:path` as `axe.rs`'s
      `layer_label` builds it).
    - Enablement: `enabled`, and
      `enablement: ServiceEnablementSourceWire { explicit: bool, layer: Option<String>, layer_kind: Option<String>, path: Option<String> }`.
    - Launch: `mode`, and `launcher: Option<ServiceLauncherWire>`, a tagged enum that is
      either `{kind:"command", command: Value, argv: Vec<String>}` or
      `{kind:"builtin", builtin: String}`. A string command becomes
      `["/bin/sh", "-c", s]`; a list becomes argv as-is.
    - Runtime settings: `cwd: Option`, `env: BTreeMap<String,String>`, `restart`,
      `success_exit_codes: Vec<i32>`, `stop_signal` (normalized `SIGTERM` form),
      `stop_timeout_seconds: f64`, `after: Vec<String>`, `log_max_bytes: u64`.
    - Provenance: `field_provenance: Vec<{field, layer, path: Option<String>}>`.
- **Layers** are processed from lowest to highest priority.
  - A layer without a `service` key is skipped.
  - A layer with `kind == "local"` is skipped, with a `warning` diagnostic
    `service_config_ignored_in_project_layer`, and its label goes into `ignored_layers`.
    The host resolves machine-level layers only.
  - Unknown layer kinds (for example, synthetic test layers) are treated as `user`.
  - `service` must be a mapping whose only key is `procs`, and `procs` must be a
    mapping. Otherwise the phase emits an `error` diagnostic with path `service` or
    `service.procs` and sets `fatal: true`.
- **Merging and field rules.** Each present field in a layer entry replaces the prior
  value whole, including lists (`command`, `success_exit_codes`, `after`). The one
  exception is `env`, which merges key by key (the later key wins). An explicit
  `enabled: false` is a value like any other. Each field's provenance records the layer
  label and file path that supplied it. Each field is validated per contribution;
  diagnostics carry the layer and the path `service.procs.<name>.<field>`, and every
  `error` marks the entry unavailable with a readable reason. The allowed fields are
  exactly these:

  | Field                | Rule                                                                                                                                                                                         |
  | -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | `description`        | string                                                                                                                                                                                       |
  | `enabled`            | bool                                                                                                                                                                                         |
  | `mode`               | only `daemon`. `oneshot` is rejected with `service_oneshot_not_configurable` ("oneshot service procs are transient-only; use `sase service proc run`").                                      |
  | `command`            | non-empty string, or non-empty list of non-empty strings                                                                                                                                     |
  | `builtin`            | one of the reserved builtins                                                                                                                                                                 |
  | `cwd`                | non-empty string                                                                                                                                                                             |
  | `env`                | mapping from names matching `[A-Za-z_][A-Za-z0-9_]*` to strings. Values may contain well-formed `${NAME}` references, and `$$` is a literal `$`. Expansion happens at launch, in sase-11y.4. |
  | `restart`            | `always`, `on-failure`, or `never`                                                                                                                                                           |
  | `success_exit_codes` | list of ints in `0..=255`                                                                                                                                                                    |
  | `stop_signal`        | `HUP`, `INT`, `QUIT`, `KILL`, `USR1`, `USR2`, or `TERM`, with or without a `SIG` prefix                                                                                                      |
  | `stop_timeout`       | number with `0 < n ≤ 3600`                                                                                                                                                                   |
  | `after`              | list of strings                                                                                                                                                                              |
  | `log_max_bytes`      | int ≥ 4096                                                                                                                                                                                   |

  Any other key is rejected with `unknown_service_proc_field`.

- **Post-merge checks** (entry-scoped):
  - The name must pass `validate_service_proc_name`.
  - The entry needs exactly one of `command` or `builtin`.
    - Neither: the entry is unavailable with "no launcher: set `command` or `builtin`".
      If only overlay/user layers mention the entry, append "(is the plugin that
      declares it installed?)".
    - Both: error.
  - `builtin` must equal the entry name.
  - The reserved names `scheduler` and `gateway` must use their own builtin.
  - `after`:
    - A self-reference is an error.
    - An unknown target is a `warning` (`unknown_service_proc_after`), and the target is
      dropped from the effective list.
    - Every available entry on an `after` cycle becomes unavailable
      (`service_proc_after_cycle`).
- **Defaults**:

  | Field                  | Default                                                 |
  | ---------------------- | ------------------------------------------------------- |
  | `enabled`              | `true`, or `false` when the declaring layer is a plugin |
  | `mode`                 | `daemon`                                                |
  | `restart`              | `on-failure`                                            |
  | `success_exit_codes`   | `[]`                                                    |
  | `stop_signal`          | `SIGTERM`                                               |
  | `stop_timeout_seconds` | `10.0`                                                  |
  | `log_max_bytes`        | `2_097_152`                                             |
  | `env`                  | `{}`                                                    |
  | `after`                | `[]`                                                    |

  When `enabled` is not explicit, `enablement.explicit` is `false`.

- **Binding**: `service_config_compose(request: dict) -> dict`. A malformed request
  raises `ValueError`.
- **Rust tests** cover:
  - A one-line string command becomes `sh -c` argv.
  - An overlay replaces a plugin list `command` and `success_exit_codes` whole; they are
    not concatenated.
  - Overlay `enabled: false` beats the default, overlay `enabled: true` re-enables a
    plugin default-disabled entry, and plugin-declared entries default to disabled.
  - The default-layer gateway entry is disabled explicitly.
  - A local layer is ignored with a warning.
  - Rejections: `mode: oneshot`, reserved-name misuse, a builtin/name mismatch, and an
    unknown field.
  - `env` merges key by key, and bad `${` syntax is rejected.
  - `stop_signal` is normalized.
  - An unknown `after` target warns; an `after` cycle marks its entries unavailable.
  - Field provenance shows the overriding layer and its file path.
  - An overlay-only partial entry is unavailable and its reason carries the plugin hint.
  - A non-mapping `service` value is fatal.

**sase**

- `src/sase/config/sase.schema.json`:
  - Add a top-level `service` object (`additionalProperties: false`) whose `procs`
    object uses `additionalProperties: {"$ref": "#/$defs/serviceProc"}`.
  - `$defs.serviceProc` mirrors the field rules above. Use `mode` enum `["daemon"]`,
    type unions for `command`, and closed `additionalProperties`. Each description
    states the default and the semantics (strings run through `sh -c`; lists replace
    whole across layers; plugin entries default disabled; project-local `sase.yml`
    entries are ignored).
  - Under `ace.properties`, add
    `procs: {type: object, additionalProperties: false, properties: {default_query: {type: string, default: "-service", …}}}`.
    The description says the value seeds the Procs query bar only when no committed
    query is persisted.
- `src/sase/default_config.yml`:
  - Add a commented `service.procs` block with
    `scheduler: {builtin: scheduler, description: …}` and
    `gateway: {builtin: gateway, description: …, enabled: false}`. The comment says
    nothing runs until the `sase service` host does.
  - Add `ace.procs.default_query: "-service"`.
  - `test_default_config_matches_public_schema` must still pass.
- New package `src/sase/service/`:
  - `__init__.py` has a docstring only and no eager imports.
  - `config.py` holds frozen dataclasses (`ServiceLauncher`, `ServiceEnablementSource`,
    `ServiceFieldProvenance`, `ServiceProcConfig`, `ServiceConfigComposition` with
    `get(name)`) and `ServiceConfigError(Exception)`, which carries the diagnostics.
  - `compose_service_config(layers=None)` mirrors
    `sase.axe.config_backend.compose_axe_config`: `load_config_layers()`, then
    `serialize_config_layer`, then `require_rust_binding("service_config_compose")`. It
    passes every layer through, because Rust owns the local-layer rule.
  - `load_service_config()` caches on `current_config_token()` plus the serialized
    layers, as `sase.axe.config._effective_axe_composition` does. It raises
    `ServiceConfigError` when `fatal` is set.
  - Reuse `sase.config` diagnostic dataclasses where they fit.
- Tests:
  - `tests/service/test_service_config.py`:
    - The real default config composes to an enabled `scheduler` and a disabled
      `gateway` with no diagnostics.
    - Synthetic `ConfigLayer` stacks exercise overlay replacement and a local-layer
      warning.
    - A fatal section raises.
  - `tests/test_config_schema.py` accepts `tunnel: {command: "autossh -M 0 -N box"}` and
    `ace.procs.default_query`, and rejects `mode: oneshot` and unknown entry fields.
- Symvision: key epic entries for the loader surface to `sase-11y.4`.

## Phase: restart-state

**sase-core** — the restart function

`crates/sase_core/src/service/restart.rs` holds the pure decision function.

- **Defaults.** Public constants mirror the orchestrator:
  - initial backoff 1.0 s;
  - maximum backoff 60.0 s;
  - healthy run 300.0 s;
  - crash-loop window 60.0 s;
  - crash-loop threshold 3.
- **Wires:**
  - `ServiceRestartPolicyWire`: `always`, `on-failure`, or `never` (serde names as
    written).
  - `ServiceExitWire { exit_code: Option<i32>, signal: Option<i32>, spawn_error: Option<String>, stop_requested: bool }`.
  - `ServiceRestartTuningWire`: all fields `#[serde(default)]` to the constants.
  - `ServiceRestartHistoryWire { started_at: Option<f64>, backoff_seconds, consecutive_failures: u32, recent_failures: Vec<f64>, alert_sent: bool }`.
  - `ServiceRestartRequestWire { policy, success_exit_codes, exit, history, now, tuning }`.
  - `ServiceRestartDecisionWire { schema_version, action: "restart"|"give_up", clean_exit, delay_seconds, restart_at: Option<f64>, reason, crash_loop, notify, history }`.
- **Algorithm:**
  1. A stop request gives up with reason "stopped on request".
  2. Classify the exit as clean or not, per decision 6.
  3. Policy `never`, or policy `on-failure` with a clean exit, gives up. The reason
     names the exit and the policy.
  4. Otherwise the function restarts, applying the orchestrator's accounting step for
     step (see `_schedule_lumberjack_restart`): healthy reset, then
     `consecutive_failures += 1`, then doubling capped backoff, then
     `restart_at = now + backoff`, then window pruning (`< now - window`), then append
     `now`. `crash_loop` is true when there are at least `threshold` recent failures.
     `notify` is `crash_loop && !alert_sent`, and a notification sets `alert_sent`.
  5. On `give_up`, `history.started_at` becomes `None`; all other fields are unchanged.
- **Validation:** `now` and every tuning value must be finite, backoffs must be
  positive, and the threshold must be at least 1.
- **Reason strings** are pinned by tests. The exit detail is one of
  `exited with code N`, `killed by SIGKILL` (portable names for 1, 2, 3, 4, 6, 8, 9, 11,
  13, 14, 15; otherwise `signal N`), or `failed to start: <err>`. The formats are:

  | Outcome            | Reason                                                                           |
  | ------------------ | -------------------------------------------------------------------------------- |
  | Restart            | `"<detail>; retrying in <delay>"`                                                |
  | Crash-loop restart | `"crash-looping (<n> failures within <window>s): <detail>; retrying in <delay>"` |
  | Give up (`never`)  | `"<detail>; restart policy is never"`                                            |
  | Give up (clean)    | `"<detail>; clean exit, not restarting (restart: on-failure)"`                   |

  `<delay>` is formatted like Python `f"{x:g}s"`.

**sase-core** — the state store

`crates/sase_core/src/service/state.rs` holds the locked store.

- **Paths:**
  - `service_state_path(sase_home)` is `<home>/service/state.json`.
  - The lock is `state.json.lock`, taken through `crate::store_lock` with its holder
    file.
  - The lock timeout comes from env `SASE_SERVICE_STATE_LOCK_TIMEOUT` (default 5 s).
- **Wires** (`SERVICE_STATE_WIRE_SCHEMA_VERSION = 1`):
  - `ServiceStateWire { schema_version, enablement: BTreeMap<name, {enabled, updated_at, updated_by}>, stops: BTreeMap<name, {boot_id: Option<String>, stopped_at, stopped_by, reason: Option<String>}>, markers: BTreeMap<key, {recorded_at, recorded_by, detail: Option<String>}>, host: Option<ServiceHostRecordWire> }`.
  - `ServiceHostRecordWire { pid, boot_id: Option, started_at, heartbeat_at, mode: "platform_unit"|"detached"|"foreground", unit: Option, sase_version: Option, error: Option }`.
  - `ServiceStateSnapshotWire { schema_version, state (active stops only), expired_stops: Vec<String>, read_only: bool, diagnostics: Vec<String> }`.
  - `ServiceStateMutationWire` is tagged by `op` (snake_case):
    `set_enablement {name, enabled, actor}`, `clear_enablement {name}`,
    `stop {name, actor, reason}` (records the current boot id), `clear_stop {name}`,
    `set_marker {key, actor, detail}`, `clear_marker {key}`, `record_host {host}`,
    `clear_host {pid}`. `clear_host` is a no-op unless the stored pid matches.
  - The mutation outcome is `{snapshot, changed}`.
- **`read_service_state(home, boot_id)`** takes a shared lock.
  - A missing file yields an empty state.
  - A corrupt file yields an empty state plus a diagnostic, and nothing is written.
  - A newer `schema_version` yields `read_only: true` plus a diagnostic, with
    best-effort parsed state.
- **`mutate_service_state(home, mutation, boot_id, now)`** takes an exclusive lock.
  - A corrupt file is renamed to `state.json.corrupt-<epoch_ms>` with a diagnostic, and
    the mutation starts from an empty state.
  - A newer schema returns a `NewerSchema` error.
  - Stops whose boot id does not match are pruned.
  - The mutation is applied, and the file is written atomically (`NamedTempFile` in the
    same directory, fsync, persist) only when something changed.
- **Validation:** names go through `validate_service_proc_name`, marker keys must match
  `^[a-z0-9][a-z0-9_.-]{0,63}$`, actors must be non-empty, and `now` must be finite.
- **Errors:** `Validation`, `LockTimeout`, `NewerSchema`, `Io`, and `Json`. In Python
  they become `ValueError`, `TimeoutError`, `RuntimeError`, `RuntimeError`, and
  `RuntimeError`.
- **Bindings:**
  - `service_restart_decide(request: dict) -> dict`
  - `service_state_read(sase_home: str, boot_id: str | None = None) -> dict`
  - `service_state_mutate(sase_home: str, mutation: dict, boot_id: str | None = None, now: float | None = None) -> dict`
    (a `None` `now` means the system clock).
- **Rust tests:**
  - Restart:
    - An orchestrator-parity sequence: backoffs 1, 2, 4 … capped at 60, and a reset
      after a healthy run.
    - A crash-loop notification fires exactly once per episode and re-arms after a
      healthy run.
    - Window pruning.
    - Under `on-failure`: clean exits (0, a listed success code, SIGTERM) give up;
      SIGKILL restarts; a spawn error restarts.
    - `never`, stop-requested, and invalid input.
  - State:
    - Setting and clearing enablement.
    - A stop expires under a different boot id and is pruned on the next write.
    - `None` boot-id semantics.
    - Markers.
    - The host pid guard.
    - Corrupt-file quarantine on mutate, and report-without-write on read.
    - A newer schema is `read_only` and refuses mutation.
    - Threaded concurrent mutators lose no updates.
    - The lock-timeout env var is honored.

**sase**

- `src/sase/service/paths.py`: `service_dir()`
  (`sase.core.paths.sase_home() / "service"`) and `service_state_path()`.
- `src/sase/service/boot.py`: `current_boot_id() -> str | None`, cached per process. On
  Linux it reads `/proc/sys/kernel/random/boot_id`. On macOS it runs
  `sysctl -n kern.bootsessionuuid` with a short timeout. Otherwise it returns `None`.
  Leave the private helper in `src/sase/monitor/reconcile.py` alone.
- `src/sase/service/restart.py`: typed dataclasses plus
  `decide_service_restart(policy, exit, history, *, now, success_exit_codes=(), tuning=None)`.
- `src/sase/service/state.py`: typed snapshot dataclasses plus these functions:
  `read_service_state`, `set_service_enablement`, `clear_service_enablement`,
  `record_service_stop`, `clear_service_stop`, `set_service_marker`,
  `clear_service_marker`, `record_service_host`, and `clear_service_host`. Each defaults
  `sase_home` to `sase_home()`, `boot_id` to `current_boot_id()`, and `now` to
  `time.time()`, all overridable for tests.
- Tests (`tests/service/test_service_restart.py`, `tests/service/test_service_state.py`)
  run against a tmp `SASE_HOME` through the real bindings.
- Symvision: key the new entries to `sase-11y.4`.

## Phase: status-wire

**sase-core** — `crates/sase_core/src/service/status.rs`

- **Enablement resolution.**
  `resolve_service_enablement(config_enabled, &ServiceEnablementSourceWire, override: Option<&EnablementOverride>) -> ServiceEnablementWire`
  returns
  `{ enabled, provenance: "override"|"config"|"default", layer, path, updated_at, summary }`.
  The `summary` strings are pinned by tests:

  | Provenance                    | Summary                                                                                   |
  | ----------------------------- | ----------------------------------------------------------------------------------------- |
  | Override                      | `enabled here` / `disabled here`                                                          |
  | Config with a file path       | `enabled by <file name>` / `disabled by <file name>` (e.g. `disabled by sase_apollo.yml`) |
  | Config from the default layer | `disabled by default config`                                                              |
  | Config from a plugin layer    | `disabled by plugin <module>`                                                             |
  | Implicit default              | `enabled by default` / `disabled by default (plugin-declared)`                            |

- **Request.** `ServiceStatusRequestWire` has these fields:
  - `generated_at`, `boot_id`.
  - `host: {record: Option<ServiceHostRecordWire>, lock_held, pid_alive: Option<bool>, platform_unit: Option<String>, stale_after_seconds (default 15)}`.
  - `config: ServiceConfigCompositionWire`.
  - `state: ServiceStateWire`.
  - `procs: Vec<ServiceProcObservationWire { name, pid, alive, proc_id, started_at, last_exit: Option<{exit_code, signal, spawn_error, finished_at}>, restart: Option<ServiceRestartDecisionWire>, restarts: u32, reported: Option<{summary, state, updated_at}>, log_path }>`.
    `reported` is the optional per-proc `status.json` contract.
- **Snapshot.** `build_service_status(request) -> ServiceStatusSnapshotWire` returns
  `{ schema_version: 1, generated_at, change_token, host, procs, orphans, diagnostics }`.
  - `host` has `state`, `pid`, `mode`, `platform_unit`, `started_at`, `heartbeat_at`,
    `heartbeat_age_seconds`, `sase_version`, `error`, and `summary`. Its `state` is:
    - `running` when a record exists, the pid is alive, and the heartbeat is fresh;
    - `stale` when a record exists but the pid is dead or the heartbeat is stale;
    - `starting` when the lock is held but there is no record;
    - `stopped` otherwise.
  - Each entry in `procs` has one row per configured entry, in config order.
    - Copied fields: `name`, `description`, `source`, `declared_by`, `mode`,
      `available`, `pid`, `proc_id`, `started_at`, `last_exit`, `restart`, `restarts`,
      `reported`, `log_path`.
    - `launcher_summary`: `builtin: <name>` or the shell-joined argv.
    - `unavailable_reason`.
    - `enablement`: the resolved `ServiceEnablementWire`.
    - `stop`: the stop override, only while it is active.
    - `desired`: `running` if the entry is available, enabled, and not stopped;
      otherwise `stopped`.
    - `state` and `summary`: see the table below.
  - `orphans` has the same row shape, for observed names that are not in the config.
- **State derivation** (the first matching row wins):

  | Condition                            | `state`       |
  | ------------------------------------ | ------------- |
  | alive                                | `running`     |
  | not available                        | `unavailable` |
  | not enabled                          | `disabled`    |
  | active stop                          | `stopped`     |
  | restart pending with `crash_loop`    | `crash_loop`  |
  | restart pending without `crash_loop` | `backoff`     |
  | `last_exit` present                  | `exited`      |
  | otherwise                            | `stopped`     |

  `summary` is stable text:
  - `running · pid N` for a running proc;
  - the decision `reason`, verbatim, while a restart is pending (never a live countdown;
    clients render the countdown from `restart_at`);
  - the enablement summary for a disabled proc;
  - `unavailable: <first reason>` for an unavailable proc;
  - `stopped until next boot` for a stopped proc.

- **`change_token`**: a sha256 hex digest of the canonical JSON of the snapshot, with
  `generated_at`, `change_token`, `host.heartbeat_at`, and `host.heartbeat_age_seconds`
  blanked. Heartbeats and rebuilds alone never change it.
- **I/O:**
  - `write_service_status_snapshot(path, &snapshot)` validates the schema version,
    creates the parent directory, and writes atomically (temp file in the same
    directory, fsync, rename).
  - `read_service_status_snapshot(path) -> Result<Option<_>>` returns `None` for a
    missing file. A newer schema returns a `NewerSchema` error, and a corrupt file
    returns a `Corrupt` error.
- **Bindings:**
  - `service_enablement_resolve(entry: dict, override: dict | None) -> dict`
  - `service_status_build(request: dict) -> dict`
  - `service_status_write(path: str, snapshot: dict) -> None`
  - `service_status_read(path: str) -> dict | None`
- **Rust tests:**
  - The derivation table and every enablement summary.
  - Token stability across `generated_at` and heartbeat changes, and a token change when
    a proc's state changes.
  - A write/read round-trip.
  - Newer-schema and corrupt reads.
  - Orphans.

**sase**

- `src/sase/service/paths.py` gains `service_status_path()`
  (`service_dir() / "status.json"`).
- `src/sase/service/status.py`: typed dataclasses plus `resolve_service_enablement`,
  `build_service_status`, `write_service_status(snapshot, path=None)`, and
  `read_service_status(path=None) -> ServiceStatusSnapshot | None`. The builder takes a
  `ServiceConfigComposition`, a state snapshot, a host observation, and proc
  observations, and forwards their wire dicts. It keeps each facade's original payload
  so no re-serialization drift is possible.
- Tests (`tests/service/test_service_status.py`) run end to end:
  1. Compose the real default config.
  2. Set a disabled override and a boot-scoped stop through the state facade.
  3. Build, write, and read the snapshot.
  4. Assert the states, summaries, and token equality across a heartbeat-only rebuild.
- Symvision: key the new entries to `sase-11y.4`, or to `sase-11y.7` for symbols only
  the TUI will read.

## Landing (the step that closes sase-11y.2)

1. `sase bead epic-symbols sase-11y.2` prints nothing. Every Justfile entry this epic
   added is keyed to `sase-11y.4` or `sase-11y.7`.
2. Once the last sase-core commit of this epic is on the remote, ratchet
   `sase-core-revision.txt` (`just ratchet-core-revision`, or set the SHA directly) so
   master's "Check pinned core bindings" step sees the new bindings. Then run
   `just install` and `just check`.
3. Run sase `just check-full` through a verify monitor, and sase-core `just check`.
4. Close sase-11y.2. The note summarizes the decisions above, especially:
   - no proc schema bump;
   - `ace.procs.default_query` rather than `tui.procs.default_query`;
   - entry-scoped unavailability;
   - restart and clean-exit semantics;
   - the new binding and facade names sase-11y.4 and sase-11y.7 consume.
