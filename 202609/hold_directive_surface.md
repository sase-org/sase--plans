---
tier: epic
title: The %hold prompt directive
goal: 'A prompt can arm a durable agent hold with `%hold`, behind the `agent_holds`
  beta flag. Python, the Rust typed launch planner, and the shared editor contract
  all parse it the same way. Agent and proc launches arm the hold when they are submitted,
  excluding their own kin and getting a priority boost. Approval previews list what
  each hold captures, broad holds need interactive confirmation, and `%hold` is rejected
  when combined with `%repeat` or `%dispatch`.

  '
phases:
- id: directive-surface
  title: Parse %hold everywhere behind agent_holds
  depends_on: []
  size: large
  description: 'directive-surface: create the agent_holds beta flag. Add a Rust hold-directive
    collector with a canonical formatter and a selector expansion, plus the `hold`
    contract entry and the new Hood value role. Parse %hold into the typed agent and
    proc unit wires, including previews, the dispatch-prompt re-render, and plan-time
    diagnostics for self holds, cycles, %repeat and %dispatch. Route Python directive
    parsing through the same collector.'
- id: arm-runtime
  title: Arm holds at launch submission
  depends_on:
  - directive-surface
  size: large
  description: 'arm-runtime: extend the Rust hold store with a pending-launch armer
    kind, arm-time rejection of the armer''s own and kin names, and an armer rebind
    that keeps the record''s timing. Arm from the agent runner bootstrap and from
    typed-plan submission, rebind the hold when a unit dispatches, release units that
    never dispatch, and apply the implied armer priority.'
- id: preview-confirm
  title: Preview captures and confirm broad holds
  depends_on:
  - directive-surface
  size: medium
  description: 'preview-confirm: list each hold and its live pending capture in launch
    previews. Add a capture-threshold config value, gate broad holds behind a TUI
    modal and a tty prompt in sase run, and document the directive in docs/xprompt.md.'
proposed_by: bbugyi200.athena.sase-11l.5
parent_bead: sase-11l.5
create_time: 2026-09-16 13:44:45
status: wip
bead_id: sase-11l.5.1
---

- **PROMPT:** [prompts/202609/hold_directive_surface.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/hold_directive_surface.md)
- **PARENT:** [202609/hold_directive.md](https://github.com/sase-org/sase--plans/blob/main/202609/hold_directive.md)
- **BEAD:** [sase-11l.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11l/sase-11l.5.1.md)

# Plan: the `%hold` prompt directive

## 1. Context

This epic delivers phase `hold-directive` (bead **sase-11l.5**) of the parent epic
**sase-11l**, "%hold: a reverse-%wait admission barrier". Earlier phases have already
landed:

- the Rust hold store `agent_hold.rs`;
- the `hold-barrier` runner-slot blocker, with family-generation and proc-settlement
  release;
- `%queue` on `%proc` units;
- the `sase agent hold {create,list,release,run,show}` CLI.

The Python adapter for the store is `src/sase/core/agent_hold_facade.py`. The CLI lives
in `src/sase/agents/cli_hold.py`.

This epic adds the prompt surface. The directive is sugar over that store. Before
implementing, every worker must:

- read the parent epic plan (`sase bead show sase-11l`) and its research report
  (`sase artifact read research:202609/reverse_wait_hold_barrier/reverse_wait_hold_barrier.md "<why>"`);
- read `sase/memory/xprompts.md`, `sase/memory/sase_flags.md`, and
  `sase/memory/lint_and_test.md` via `/sase_memory_read`;
- read `sase/memory/cli_rules.md` if a phase touches CLI options.

Sibling phases may run concurrently. Coordinate with them as follows:

- **sase-11l.6** (completion and LSP): builds on this epic's `DirectiveValueRole::Hood`
  and the hold contract entry. This epic adds the role but does **no** candidate
  synthesis.
- **sase-11l.7** (proc targets): evaluates holds for un-dispatched procs. See the
  coordination note in §4.
- **sase-11l.8** (visibility): renders `held_by`. It also edits
  `src/sase/core/agent_hold_facade.py`, so keep facade edits additive and rebase with
  care.
- **sase-11l.10**: removes the `agent_holds` flag later. Keep the flag's Off branch
  explicit and easy to delete.

### Rules for every phase

- **Rust first.** Cross-repo work changes `sase-core` first. Open it with
  `sase repo open sase-core -r "<why>"` (the `/sase_repo` skill).
- **Pin bump.** Bump `sase-core-revision.txt` per `tools/ratchet_core_revision`.
- **Binding checks.** Add every new binding to the module doc index in
  `crates/sase_core_py/src/lib.rs`, and keep `tools/check_sase_core_rs_bindings` and
  `tools/validate_sase_core_rs` green.
- **Backend boundary.** Shared parsing and selector semantics live in Rust. Python does
  launch plumbing, I/O, presentation, and config.
- **Verification.** Run `just check` before finishing. `just check-full` is the land
  agent's gate.
- **No new beads.** Workers must **not** create beads (the flag bead below is the one
  sanctioned exception). Record discovered work as
  `sase bead note sase-11l.5 'PROPOSED FOLLOW-UP: …'`.
- **No memory edits.** Workers must **not** edit memory files. The memory-doc follow-up
  already belongs to sase-11l.10.

## 2. The shared `%hold` contract (all phases implement exactly this)

```text
%hold:planner                                # name selector
%hold:planner,reviewer                       # colon list
%hold:@nightly                               # tribe (positional @)
%hold(pending)                               # freeze WAITING|QUEUED at arm time
%hold(future)                                # fence launches created after arm time
%hold(hood=sase-s7)                          # hood selector (component boundary)
%hold(tribe=nightly)                         # tribe selector (keyword form)
%hold(pending, future, ttl=90m, scope=host)  # widest form, explicit in the text
```

### Occurrences and fields

- `%hold` may appear more than once, and all occurrences are unioned.
- `hood=` and `tribe=` may repeat.
- `ttl=` and `scope=` may each appear at most once per launch unit, counting every
  occurrence. A second one is error `duplicate-hold-field`.
- Name, tribe, and hood sets are sorted and deduplicated so digests stay deterministic.

### Positional values

- `pending` and `future` are reserved words. They set flags.
- `@<tribe>` is a tribe selector.
- Anything else is a name, validated as agent-name-like (letters, digits, `_`, `.`, `-`,
  and a `--` shell suffix is allowed).
- A name selector matches a candidate's agent name, family, clan, or workflow, and (once
  sase-11l.7 lands) a proc shell name.
- The CLI's `-n` stays exact-agent-name, as its help text says. If you think the CLI
  should match the directive, add a `PROPOSED FOLLOW-UP` note instead of changing it.

### Errors (typed codes shared by Rust and Python)

| Input                                          | Code                     | Message requirement                                                                                            |
| ---------------------------------------------- | ------------------------ | -------------------------------------------------------------------------------------------------------------- |
| Bare `%hold`, `%hold()`, `%hold+`, empty colon | `hold-selector-required` | Names every selector: name, `@tribe`, `pending`, `future`, `hood=`, `tribe=`                                   |
| Positional `all`                               | `hold-all-unsupported`   | Names the same selectors and suggests `pending, future, scope=host`                                            |
| `scope=` other than `project` or `host`        | `invalid-hold-scope`     |                                                                                                                |
| `ttl=` outside the SASE duration grammar       | `invalid-hold-ttl`       | Grammar is `90m`, `1h30m`, `45s`; reuse `parse_proc_duration_seconds` and move it to a shared helper if needed |
| Unknown keyword                                | `unknown-hold-keyword`   | Lists `hood=`, `scope=`, `ttl=`, `tribe=`                                                                      |

The `agent_hold_max_ttl` cap depends on config, so Python checks it at arm and preview
time, not at parse time.

### Flag gating (`agent_holds`, beta)

With the flag **off**:

- every non-bare `%hold` form fails with `hold-flag-disabled`: "The %hold directive
  requires the agent_holds beta flag. Enable it with `sase flag enable agent_holds`.";
- a bare `%hold` stays inert prose, the same way disabled `%proc` does;
- editors hide `%hold` from name completion.

With the flag **on**, the full behavior applies.

### Other rules

- **No alias.** `%h` is `%hide`.
- **Scope** defaults to `project`.
- **TTL** defaults to `agent_hold_default_ttl` and is capped by `agent_hold_max_ttl`.
- **Rejected combinations:**
  - with `%dispatch`, in both the Rust typed planner and the Python dispatch scan;
  - with `%repeat`, in both the Python repeat launcher and the Rust typed planner.
- **Proc units:** `%hold` is allowed on `%proc` units. Never add it to
  `proc_forbidden_directives`.
- **Limitation:** the frozen `pending` snapshot cannot capture un-dispatched procs. Only
  lexical selectors and `future` fence them. Document this in phase 3.

## 3. Phase directive-surface: flag, parser, contract, typed planner, Python parse

### 3.1 Flag scaffolding (the one sanctioned bead creation)

Run `sase flag new agent_holds -k beta` with:

- `-d "Beta gate for the %hold prompt directive."`
- `--when-enabled`: "%hold parses, arms durable holds at launch submission, and appears
  in editor completion."
- `--when-disabled`: "any non-bare %hold fails to parse with an error naming the flag;
  bare %hold is inert prose; editors hide it."
- `--remove-when`: "every sase-11l phase has landed and the flag-removal phase makes
  %hold unconditional."

The parent epic sanctions this `sase flag new` call. Its removal is sase-11l.10's job.

Then:

- Paste the printed entry into `src/sase/feature_flags/registry.py`.
- Run `tools/sync_feature_flags_schema`, and keep `tools/check_feature_flags` green (it
  requires a non-test reference).
- Add an `agent_holds_enabled()` helper next to the directive adapter.
- Add the flag to `launch_feature_flag_keys()` in `src/sase/xprompt/queue_directive.py`.
- Add it to the fixed CLI-completion flag list in
  `src/sase/completion/candidates/catalog_build.py`.
- Add an LSP env pin, `SASE_AGENT_HOLDS`:
  - Python side: `src/sase/integrations/xprompt_lsp.py`, mirroring
    `_apply_typed_launch_units_flag`;
  - Rust side: `crates/sase_xprompt_lsp/src/server.rs`, beside `TYPED_LAUNCH_UNITS_ENV`,
    feeding the enabled-flag list.

### 3.2 Rust collector: new `crates/sase_core/src/hold_directive.rs`

Model it on `queue_directive.rs`. It must stay pure: it never reads config.

- **Occurrence wire.** Reuse the `QueueOccurrenceWire` shape (`source`, `source_span`,
  `args`, `has_plus_suffix`), either by sharing the type or by adding an identical
  `HoldOccurrenceWire`.
- **Fields wire.** `HoldFieldsWire` has:
  - `names`, `tribes`, `hoods` (`Vec<String>`);
  - `pending`, `future` (`bool`);
  - `ttl` (the canonical authored text, `Option<String>`);
  - `ttl_seconds` (`Option<u64>`);
  - `scope` (`Option<…>` holding `project` or `host`).

  Mark every field `#[serde(default, skip_serializing_if = …)]` so hold-free payloads
  serialize exactly as they do today.

- **`collect_hold_fields_with_flags(occurrences, flags)`** returns `{ fields, errors }`
  with the §2 codes and source spans.
- **`format_hold_directive(fields) -> String`** renders the canonical
  `%hold(<names>, @<tribes>, pending, future, hood=…, tribe=…, ttl=…, scope=…)`. A test
  must assert that parsing the formatted output returns the original fields.
- **`hold_fields_to_selectors(fields, pending_artifact_dirs) -> AgentHoldSelectorsWire`**
  is a pure function:
  - each name goes into `names`, `clans`, and `workflows`;
  - it also goes into `families` only when `parse_agent_family_name` accepts it;
  - tribes are normalized without the `@`;
  - hoods are validated with the store's hood rules (make `normalize_hood_vec`
    `pub(crate)`).
- **Exports and bindings.** Re-export from `crates/sase_core/src/lib.rs`. Add pyo3
  bindings `collect_hold_fields`, `format_hold_directive`, and
  `hold_fields_to_selectors` beside the queue bindings, and add doc-index rows.
- **Tests:** every selector shape, every error code, flag on and off, union and
  duplicate rules, canonical round-trip, and name expansion (a `--` shell name must not
  enter `families`).

### 3.3 Rust editor contract

**`editor/wire.rs`:**

- Add `DirectiveValueRole::Hood` (serde `hood`). It gets no candidate synthesis here, so
  it falls through to static suggestions; synthesis belongs to sase-11l.6.
- Add `directive_feature_flag`: `"hold" => Some("agent_holds")`.
- Add `"hold"` to `directive_metadata_supports_colon`.
- Add a `hold` arm to `directive_examples` and a snippet recipe.

**`editor/directive.rs`:**

- Add a `HOLD_KEYWORDS` spec. The parity tuple, in alphabetical order, is
  `("hood", "scope", "ttl", "tribe")`:

  | Keyword | Role       | Repeatable | Suggested values       |
  | ------- | ---------- | ---------- | ---------------------- |
  | `hood`  | `Hood`     | yes        | —                      |
  | `scope` | `FreeText` | no         | `project`, `host`      |
  | `ttl`   | `Duration` | no         | `DURATION_SUGGESTIONS` |
  | `tribe` | `Tribe`    | yes        | —                      |

- Add a `hold` entry placed right after `queue` in `DIRECTIVES`:
  - `alias: None`, `allows_multiple: true`, `syntax_forms: COLON_PAREN`;
  - `positional_role: Some(Agent)`;
  - positional suggestions `pending` and `future`, each with documentation;
  - a description such as "Hold selected pre-run agents until this launch settles".
- Add `"hold"` to `mixes_positional_and_keyword_clauses`.
- Update the ordered contract-matrix lists in `editor/directive.rs` (test
  `contract_covers_the_audited_directive_matrix`) and in
  `crates/sase_core_py/src/lib.rs`.

**Completion.** Name completion hides `%hold` when the flag is off, which follows
automatically from `directive_feature_flag`. Verify this with a test.

### 3.4 Rust typed launch planner (`crates/sase_core/src/agent_launch/mod.rs`)

**Parsing in `classify_typed_launch_unit`:**

- Add a `"hold"` match arm that collects occurrences (like `"queue"`) and removes their
  regions. With the flag off, leave a bare `%hold` in place, matching Python.
- After the loop, run `collect_hold_fields_with_flags` and turn its errors into unit
  diagnostics.
- Add `hold: Option<HoldFieldsWire>` (`serde default` + `skip_serializing_if`) to both
  `AgentUnitWire` and `ProcUnitWire`, and fill it for both kinds. Update the
  `ProcUnitWire` literals in `mod.rs` and in the `admission.rs` test.

**Dispatch prompt, preview, and digest:**

- `agent_unit_dispatch_prompt_with_flags` (in `admission.rs`) re-emits
  `format_hold_directive`, next to the existing `format_queue_directive` re-emit.
- `render_launch_approval_preview` appends ` hold=<canonical directive>` to agent and
  proc lines when a hold is present.
- Digests change only for units that carry a hold.

**Plan-time diagnostics:**

- `validate_dispatch_combinations` gains `has_hold`, which adds `%hold` to its message.
- `hold-with-repeat` fires when the unit came from a repeat fan-out slot or still
  carries `%repeat`.
- `hold-self` fires when a name selector equals the unit's own explicit identity,
  family, or clan.
- `hold-cycle` extends `validate_typed_wait_cycles`. For each unit H with a hold whose
  explicit name, `@tribe`, or hood selector matches another unit T's explicit identity,
  tribe, or hood membership (excluding H's kin), add the edge T→H. Run the existing DFS
  over wait and hold edges together and report a distinct message.

**Tests.** Copy the existing queue, proc, and dispatch tests
(`typed_launch_proc_accepts_queue_spellings_and_authored_weight`,
`typed_launch_rejects_dispatch_combined_with_wait_or_family`,
`typed_launch_plan_rejects_wait_cycles`,
`agent_dispatch_prompt_restores_identity_with_queue_directive`). Add:

- proc units carrying a hold;
- preview text;
- digest changes only with a hold;
- dispatch-prompt round-trip;
- each diagnostic;
- flag off.

### 3.5 Python parsing (`src/sase/xprompt/`)

- **New `hold_directive.py`:**
  - a frozen `HoldDirective` dataclass: names, tribes, hoods, pending, future,
    `ttl_seconds`, `ttl_text`, scope;
  - `collect_hold_directive(occurrences)`, which calls the binding with
    `launch_feature_flag_keys()` and raises `DirectiveError` on the first error;
  - `format_hold_directive`;
  - the disabled-message constant.
- **Parser tables and fields:**
  - add `hold` to `_KNOWN_DIRECTIVES` and `_MULTI_VALUE_DIRECTIVES`, with no alias;
  - add `PromptDirectives.hold: HoldDirective | None = None`, documented in the class
    docstring.
- **Collection and extraction:**
  - `_directive_collect.py` gets a dedicated `hold` branch (like `queue`) that builds
    occurrence payloads through a shared helper and honors the flag-off bare rule;
  - `_directive_extract.py` assembles `hold`.
  - Check `strip_known_directives` and the `has_*` predicates in `_directive_scan.py` so
    a flag-off bare `%hold` stays inert.
- **Rejections:**
  - `scan_dispatch_directive` tracks `saw_hold`, and its message becomes "…with %wait,
    %queue, %hold, or %clan…". Update any tests that assert the old message.
  - `src/sase/agent/repeat_launcher.py::_validate_repeat_prompt_directives` rejects
    `%hold` with a message pointing to `sase agent hold run`.
- **Python wire mirror** for the new unit field:
  - `src/sase/core/agent_launch_wire_records.py`;
  - `agent_launch_wire_from_dict.py`;
  - `agent_launch_wire_conversion.py`.
- **Parity.** In `tests/test_xprompt_directive_contract.py`, update the vocabulary, the
  `allows_multiple` set, the keyword tuples
  (`"hold": ("hood", "scope", "ttl", "tribe")`), the syntax forms,
  `feature_flag == "agent_holds"`, and the positional suggestions.
- **Tests:** both flag states (use `override_flags(agent_holds=…)`), every selector
  shape, the rejection matrix, fenced and backticked mentions staying inert, and wire
  round-trips.

This phase arms nothing. With the flag on, `%hold` parses and is stripped, and phase 2
adds the runtime behavior.

## 4. Phase arm-runtime: store extensions and launch-time arming

### 4.1 Rust store (`crates/sase_core/src/agent_hold.rs`)

**New armer kind `AgentHoldArmerKindWire::Launch` (`"launch"`).** It represents a
submitted typed-launch unit that has not dispatched yet.

- It requires `pid` and `done_marker_path`.
- `agent_name`, `family`, and `clan` are optional; fill them when the unit has an
  explicit identity so kin exclusion works before dispatch.
- Its liveness fact is `Launch { pid_alive, done_marker_present }`, and it needs an
  `armer_is_alive` arm.
- Keep `AGENT_HOLD_WIRE_SCHEMA_VERSION` unless a bump proves necessary. Older binaries
  prune an unknown-kind record, which fails open; note that in the commit.

**Arm-time kin rejection inside `arm_agent_hold_until`,** shared by the CLI and the
directive:

- Reject any `names`, `families`, `clans`, or `workflows` selector that equals the
  armer's agent name, family, or clan, or that is a dotted descendant of the armer's
  family (mirroring `armer_kin_excluded`).
- Use a clear message naming the offending value.
- Hood selectors that contain the armer stay legal, because evaluation already excludes
  the armer.

**`rebind_agent_hold_armer(sase_home, old_key, new_armer, liveness, now) -> Option<record>`.**
Under the lock it:

1. removes `old_key`;
2. validates `new_armer`;
3. inserts the record under `new_armer.key`, keeping `created_at`, `expires_at`,
   `scope`, and `selectors` unchanged, and replacing any record already at that key.

It returns `None`, and creates nothing, when `old_key` is absent (expired, pruned, or
released).

**Binding and tests.**

- Add the pyo3 binding `agent_hold_rebind`, with the same error mapping as
  `agent_hold_arm_relative`, plus a doc-index row.
- Tests:
  - rebind keeps the timing, so a `future` rule still matches a candidate created
    between arm and rebind;
  - rebind with an absent old key returns `None`;
  - kin rejection for self, family, clan, and descendant, while hoods are allowed;
  - `launch`-kind liveness pruning.

### 4.2 Python facade and launch glue

**`src/sase/core/agent_hold_facade.py`:**

- `arm_agent_hold` accepts an explicit `armer` and an explicit `selectors` payload. The
  defaults keep today's CLI behavior.
- Add a public `agent_armer_wire_for_artifacts(artifacts_dir)`, renamed from
  `_agent_armer_wire`.
- Add `rebind_agent_hold(old_key, new_armer)`.
- Add a shared `resolve_hold_ttl_seconds(requested)` for the config default and cap, and
  reuse it from `cli_hold._resolve_ttl_seconds`.
- The `launch` kind needs support in three places:
  - `_validate_hold_record`;
  - liveness: alive when the recorded pid, or the bundle coordinator pid in
    `started.json` next to `done_marker_path`, is alive and the done marker is absent;
  - CLI list and show rendering.

**New `src/sase/agent/launch_hold.py`:**

- The key helper `unit_hold_key(request_id, logical_id)` returns
  `launch:<request_id>/<logical_id>`.
- The arm, pre-arm, rebind, and release helpers below.
- Directive name selectors are converted with the Rust `hold_fields_to_selectors`
  binding. `pending` is frozen through the existing `_capture_pending_targets`, scoped
  like the hold.

**`HOLD_ARMER_WAIT_PRIORITY`.** Define it beside `DEFAULT_WAIT_PRIORITY` in
`src/sase/core/runner_slots/_admission_types.py`, with a value below 10 (5 is suggested)
because lower values start first.

### 4.3 Non-typed agent launches: arm in the runner bootstrap

**Where.** In `src/sase/axe/run_agent_runner_bootstrap.py::bootstrap_agent_run`, right
after `extract_directives_and_write_meta`. At that point `agent_meta.json` has the name,
family, clan, and pid; verify the pid. This is before `_wait_flags` and any dependency
wait, so a `%wait`-parked armer is already holding. Carry the parsed hold on a new
`AgentInfo.hold` field set in `src/sase/axe/run_agent_directives.py`.

**What.**

- If the env var `SASE_LAUNCH_HOLD_KEY` is set, rebind that pre-armed key to the agent
  armer.
  - If the record is gone or the rebind fails, release the pre-arm key on a best-effort
    basis and do **not** arm a fresh hold.
  - Otherwise, arm a fresh hold.
- Skip code-refresh passes (`RUNNER_CODE_REFRESHED_ENV`) and retry handoffs. The
  original record lasts until the family settles.
- If arming fails (a kin name, a TTL over the cap, a lock timeout), fail the launch with
  a message prefixed `%hold:`.

**Implied priority.**

- When a hold is present and `%queue(priority=)` was not authored, `_admit_and_launch`
  passes a new `wait_priority_implied=HOLD_ARMER_WAIT_PRIORITY` through
  `wait_for_runner_slot`, then `_try_claim_runner_slot`, then `marker_priority_state`,
  which returns `(value, False)`.
- `waiting.json` records `wait_priority_explicit: false`.
- Do **not** write the implied value into `agent_meta` or `AgentInfo.wait_priority`.
  Doing so would flip `_wait_flags` and render the value as authored.
- Make `src/sase/ace/tui/models/_loaders/_meta_enrichment_wire.py` honor an explicit
  `false`. Infer explicitness from a non-default value only when the flag is absent,
  mirroring `_legacy_marker_priority_explicit`. Verify that the marker wire keeps
  "absent" and "false" distinct.

### 4.4 Typed plans: pre-arm at submission, rebind at dispatch

**Pre-arm.** In `src/sase/agent/launch_admission.py::dispatch_typed_launch_request`,
under the admission lock and before `engine.run`, pre-arm every unit whose payload has a
`hold`:

- Armer fields:

  | Field              | Value                                                            |
  | ------------------ | ---------------------------------------------------------------- |
  | kind               | `launch`                                                         |
  | key                | `unit_hold_key`                                                  |
  | display            | identity or shell name, else logical id, plus a short request id |
  | project            | the unit's project                                               |
  | pid                | `os.getpid()`                                                    |
  | `done_marker_path` | `<admission dir>/receipt.json`                                   |
  | identity fields    | from an explicit identity                                        |

- **Idempotency.** Write a per-unit marker,
  `launch_admission/units/<logical_id>.hold.json`, and never re-arm or re-freeze on
  re-entry.
- **Failure.** If arming fails, fail the whole submission before any unit dispatches.

**Coordinator re-anchor.** `run_coordinator_in_bundle` rebinds every still-`launch` unit
hold to its own pid **before** writing `started.json`. The start ack then implies the
re-anchor happened, so the exit of the inline `sase run` process never orphans a hold.

**Agent dispatch.** `launch_admission_runtime.py::dispatch_agent_unit` adds
`SASE_LAUNCH_HOLD_KEY` to `extra_env`. The runner rebinds as described in §4.3. The
dispatch prompt already carries the canonical `%hold` from phase 1.

**Proc dispatch.** `launch_proc_runtime.py::dispatch_proc_unit` rebinds once the proc is
reserved and visible in the proc snapshot:

- The new armer is
  `{kind: proc, key: proc:<proc_id>, display: shell name or label, proc_id}`.
- If the rebind fails, release the hold.
- The existing `settle_proc_shell` then releases the hold at terminal status.

**Units that never dispatch.** In `launch_admission_engine.py`, the `skipped`,
`launch_error`, and `cancelled` transitions release that unit's `launch:` key through
`release_agent_hold`, with reason "launch unit ended without dispatch". A present
receipt prunes any stragglers through liveness.

**Proc priority.** `src/sase/agent/proc_capacity_admission.py` uses
`HOLD_ARMER_WAIT_PRIORITY` instead of `DEFAULT_WAIT_PRIORITY` for hold-carrying proc
units that have no authored priority.

**Coordination with sase-11l.7.** If sase-11l.7 has already landed, set
`armer_key=unit_hold_key(…)` on the proc's own hold candidate so a pending proc is never
held by its own `future` rule. Otherwise, add a `PROPOSED FOLLOW-UP` note on sase-11l.5
stating that requirement.

### 4.5 Tests

- **Bootstrap arming:**
  - the bootstrap arms an agent armer before the dependency wait: with
    `%wait:x %hold(pending)`, the hold is in the store while the runner is parked;
  - refresh and retry passes neither re-arm nor re-freeze;
  - a kin name fails the launch with a clear error;
  - the flag off writes nothing to the store.
- **Implied priority:**
  - it applies only when no `%q(p=…)` is authored;
  - it is non-explicit in `waiting.json`;
  - the TUI does not render it as authored.
- **Typed plans:**
  - agent and proc units pre-arm, and re-entry is idempotent;
  - the coordinator re-anchors the pid;
  - rebind at agent dispatch keeps `created_at`, so an agent launched between submission
    and dispatch is still `future`-held;
  - a proc rebinds at dispatch and is released at settlement;
  - skipped, cancelled, and `launch_error` units release their hold;
  - a dead coordinator gets its hold pruned.
- **Quiesce composition** `%proc(...) %q:1 %hold(pending, future)`: the proc drains
  while a later-launched agent parks with `hold-barrier`, and that agent is admitted
  after the proc settles.

## 5. Phase preview-confirm: previews, confirmation gate, docs

### 5.1 Capture enumeration

- **Facade helper.** Add a public, fail-soft
  `preview_pending_capture(scope, project) -> capture | None` to the facade as a thin
  wrapper over `_capture_pending_targets`, appended at the end of the file.
- **Formatter.** Add a formatter that produces "captures 4 waiting + 2 queued; skips 3
  running".
- **Previews** (`src/sase/agent/launch_preview.py`):
  - `build_launch_preview_request` records a per-slot hold summary. Non-typed slots are
    parsed fail-soft with `extract_prompt_directives`; typed units use `payload.hold`.
  - `render_launch_preview_markdown` adds a `## Holds` section per slot or unit with:
    - the canonical directive;
    - the scope;
    - the resolved TTL (config default and cap);
    - the live capture line for `pending`;
    - a warning line when `future` is combined with `scope=host`.
- **Surfaces.** The LaunchApproval gate preview and the PDF renderer pick this up
  automatically.

### 5.2 Confirmation gate for broad holds

- **Config.** Add `agent_hold_confirm_capture_threshold` (integer, default 10) to
  `src/sase/default_config.yml`, `src/sase/config/sase.schema.json`,
  `src/sase/config/_settings.py`, and a `src/sase/config/core.py` getter. Use the
  `agent_hold_default_ttl` pattern.
- **Shared predicate.** Add it in a new module, `src/sase/agent/launch_hold_preview.py`
  (phase 2 owns `launch_hold.py`). It returns a confirmation body when any unit either:
  - has `future` with `scope=host`, or
  - has a `pending` capture above the threshold.
- **TUI.** Add a hold preflight to the prompt-bar chain
  (`src/sase/ace/tui/actions/agent_workflow/_launch_provider_guard.py` →
  `_submit_resolved_launch`):
  - A cheap `%hold` substring check keeps the fast path free.
  - Planning and capture run off the UI thread (read `sase/memory/tui_perf.md`).
  - Push `ConfirmActionModal` (DANGER) with the listed capture.
  - On cancel, restore the prompt bar the way the provider-guard abort does.
- **CLI.** `sase run` (`src/sase/main/query_handler/_launch.py`) asks
  `Arm this hold? [y/N]` only when all of these hold:
  - stdin and stdout are TTYs;
  - `SASE_AGENT` is unset;
  - the process is not a durable proc.

  Declining exits 1. Non-interactive launches go ahead, because the arm notification and
  the LaunchApproval preview already list the capture. Follow the confirm helper pattern
  in `src/sase/agents/cli_drain.py`.

### 5.3 Docs and tests

- **Docs.** Add a `%hold` row and section to `docs/xprompt.md` → Directives, covering:
  - syntax;
  - semantics: active from submission, kin excluded, released on family or proc
    settlement, TTL, fail-open;
  - the beta flag;
  - defaults;
  - limitations: `pending` cannot capture un-dispatched procs, no alias, rejected with
    `%repeat` and `%dispatch`;
  - a cross-reference to `sase agent hold`.

  Do not touch `sase/memory/`.

- **Tests:**
  - preview output for typed and non-typed launches;
  - the threshold config;
  - TUI modal confirm and cancel (Textual pilot);
  - CLI behavior with and without a TTY;
  - with the flag off: no Holds section, and the parse error surfaces unchanged.

## 6. Landing notes (for the land agent)

- **Verification.** Run `just check-full` on the combined tree. Also run `cargo test` in
  `sase-core` for `hold_directive`, `agent_hold`, `agent_launch`, and `editor`.
- **Closing.** Run `sase bead epic-symbols sase-11l.5` and resolve every leftover. Then
  close **only** phase bead sase-11l.5, with a note summarizing what was verified. Never
  close sase-11l.
- **Follow-ups.** Triage the `PROPOSED FOLLOW-UP` notes on sase-11l.5, including a
  possible CLI `-n` alignment and the sase-11l.7 `armer_key` requirement.

## 7. Non-goals

- Completion candidate synthesis for `Hood` or proc rows (sase-11l.6).
- Proc-target fencing (sase-11l.7).
- `held_by` rendering, the hold panel, the doctor check, and the deadlock notification
  (sase-11l.8).
- `%wait(hood=)` (sase-11l.9).
- Flag removal (sase-11l.10).
- A `drain=` keyword or a `proc=<id>` selector.
- Global cycle detection beyond the in-plan check.
- A short alias.
