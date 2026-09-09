---
tier: epic
status: done
title: Feature flags whose removal is a bead, a deadline, and a gate
goal: "SASE has one boolean feature-flag registry resolved through the existing config
  layer chain, every temporary flag is owned by a dedicated `flag` bead carrying a
  date-and-release removal threshold, that threshold raises a FlagTriage gate the owner
  answers with Remove / Extend / Keep / Close, flag beads read unmistakably on every
  surface that renders a bead, and `sase/memory/sase_flags.md` teaches agents when a
  flag is warranted and how to retire one.

  "
phases:
  - id: core
    title: The flag bead type in sase-core
    depends_on: []
    size: medium
    description:
      "core: add IssueTypeWire::Flag and the BeadFlagWire record to the bead wire,
      mirror the snooze-record validation shape, extend every exhaustive match over
      issue type, and give the flag type its CLI glyph and accent."
  - id: registry
    title: The typed registry, resolver, and snapshot
    depends_on: []
    size: large
    description:
      "registry: build the code-owned flag registry, the layered resolver and its
      immutable per-process snapshot, the strict SASE_FEATURE_FLAGS transport, and the
      generated feature_flags block in the config JSON Schema."
  - id: bead
    title: Flag beads in the Python bead layer
    depends_on:
      - core
    size: medium
    description:
      "bead: mirror IssueType.FLAG and FlagRecord through the Python model, the SQLite
      compatibility layer, the wire conversion, and the sase bead create/show/update
      surfaces."
  - id: look
    title: The shared flag visual language
    depends_on:
      - bead
    size: small
    description:
      "look: register the flag bead type's glyph and accent, and add the shared
      bead_flag_presentation module that renders the flag key chip and the
      urgency-graded removal countdown for every surface."
  - id: lint
    title: Registry and bead integrity enforcement
    depends_on:
      - registry
      - bead
    size: medium
    description:
      "lint: ship tools/check_feature_flags with its static registry rules and its
      bead-status rules, wire it into just lint and just validate, and ban import-time
      flag resolution."
  - id: gate
    title: The FlagTriage gate and its reconciler
    depends_on:
      - registry
      - look
    size: large
    description:
      "gate: add the trusted FlagTriage gate contract with its Remove, Extend, Keep, and
      Close options and host effects, and generalize the bead gate reconciler so a due
      flag bead raises exactly one pending gate."
  - id: cli
    title: sase flag and the flag doctor checks
    depends_on:
      - registry
      - look
    size: medium
    description:
      "cli: add the sase flag group with list, new, and show, and register the flags.*
      doctor checks covering registry integrity, override hygiene, and overdue flags."
  - id: ui
    title: Flag beads on every bead-rendering surface
    depends_on:
      - look
    size: large
    description:
      "ui: render flag beads distinctly in the bead CLI rows and detail, bead pages, the
      ACE Beads pane and its modals and filters, the notification and gate panels, and
      the Telegram bead formatter, with a visual snapshot golden."
  - id: consumer
    title: The first two real flags
    depends_on:
      - lint
      - gate
      - cli
    size: medium
    description:
      "consumer: convert one opt-in beta env gate and one disable_* env gate into
      registered flags with real flag beads, and record why plugin discovery cannot be
      the first consumer."
  - id: memory
    title: sase_flags.md, the sase.md pointer, and the docs
    depends_on:
      - ui
      - consumer
    size: medium
    description:
      "memory: add the generated sase/memory/sase_flags.md long memory and its
      sase/memory/sase.md pointer, register the glossary terms, and document the flag
      lifecycle in the user docs."
proposed_by: bbugyi200.athena.03v
bead_id: sase-nb
create_time: 2026-09-09 19:50:15
---

- **PROMPT:**
  [prompts/202608/feature_flags.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/feature_flags.md)
- **BEAD:**
  [sase-nb](https://github.com/sase-org/sase--beads/blob/main/pages/sase-nb/README.md)

# Plan: Feature flags whose removal is a bead, a deadline, and a gate

## Why this shape

The scarce resource here is not flag evaluation. SASE is a locally installed CLI/TUI: a
boolean read from a merged config map is a solved problem, and the consolidated research
report settles that a hosted service, OpenFeature, targeting, and percentages are all
out of scope. The scarce resource is **deletion**, and every structural choice below
exists to make removal a forced, answerable event rather than optional hygiene.

Three mechanisms do three different jobs, and all three are needed:

| Mechanism                          | Job                                                             |
| ---------------------------------- | --------------------------------------------------------------- |
| The `flag` bead                    | Identity, ownership, history, and the work item that deletes it |
| `remove_by` (date **and** release) | The due signal — what makes an untouched flag start asking      |
| The `FlagTriage` gate              | The channel that makes the question unignorable and answerable  |

The research proved the bead alone is self-defeating: the only event a bead-only check
can trip on is _someone closing the bead_, and nobody closes a bead whose entire content
is "remove this flag" while the flag still exists. It equally proved that dates alone
are badly calibrated here — SASE went v0.2.0 to v0.10.0 in three weeks and v0.10.x to
v0.16.0 in the six weeks after — so a threshold is due only when **both** the date and
the release have passed.

The cut into phases follows the three seams that must each be independently true before
a flag can be both usable and un-ignorable:

1. a flag must be **declarable and resolvable** (`registry`),
2. its removal must be **an owned, dated, first-class record** (`core` → `bead` →
   `look`),
3. that record must **ask a question the owner can answer** (`gate`, `cli`, `lint`).

`core` and `registry` share nothing and start together. The bead chain is strictly
sequential because each layer mirrors the one below it. `ui`, `gate`, and `cli` all
consume the visual language and fan out from `look`.

```text
core ── bead ── look ──┬── ui ──────────────────┐
                       ├── gate ──┐             │
registry ──────────────┼── cli ───┼── consumer ─┴── memory
                       └── lint ──┘
```

## Grounding

Verified in this workspace at `9fe82045d`, with the linked `sase-core` checkout at
`34ef5f2`. Line numbers are from those trees.

| Fact                                                         | Evidence                                                                                                                                                                         |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bead types are a closed three-variant enum, twice            | `IssueType` (`bead/model.py:19-22`); `IssueTypeWire` (`sase-core bead/wire.rs:25-30`)                                                                                            |
| Production bead mutations are Rust-backed                    | `core/bead_mutation_facade.py` calls `bead_create`/`bead_update`; `bead/db.py:1-4` is the compatibility path                                                                     |
| The wire already models one optional type-scoped record      | `BeadSnoozeWire` / `SnoozeRecord`, present iff the status is snoozed (`bead/model.py:115-151`, `wire.rs`)                                                                        |
| Type validation is duplicated in both languages, on purpose  | `Issue.validate` (`bead/model.py:230-282`) mirrors `wire.rs:527-593` message for message                                                                                         |
| Rust matches over issue type are exhaustive                  | `issue_type_presentation` (`cli.rs:2228-2243`), `parse_issue_type` (`cli.rs:2420-2427`), `parse_create_type` (`cli.rs:861+`), sort rank (`events.rs:1580-1582`), `search.rs:345` |
| The Python bead type presentation is already centralized     | `BEAD_TYPE_PRESENTATIONS` (`bead_type_presentation.py:38-57`), consumed by 14 modules                                                                                            |
| Filters, query profiles, and the ACE filter bar are derived  | all three read `BEAD_TYPE_VALUES` (`bead/filter_query.py:362`, `ace/query_profile/profiles.py:183`, `bead_filter_bar.py:60`)                                                     |
| A presentation parity test pins the type list and its order  | `BEAD_TYPE_VALUES == ("plan", "phase", "task")` and must equal `IssueType` order (`tests/test_bead_type_presentation.py:23-26`)                                                  |
| Cross-surface presentation modules are an established shape  | `bead_status_presentation`, `bead_time_presentation`, `bead_type_presentation`, `phase_size_presentation`, `snooze_presentation`                                                 |
| The Rust and Python type accents are already exact twins     | `xterm256_foreground_style` maps the three accents to `38;5;220/117/177`, matching `ANSI_TYPE_PLAN/PHASE/TASK`; `#FF875F` maps to `209`                                          |
| One chop already owns every bead-scoped gate, under one lock | `sase_chop_bead_task_triage.py:1-9,59`; `_GATE_KINDS = (TASK_TRIAGE_KIND, BEAD_SNOOZE_KIND)`                                                                                     |
| Its gateable set and its kind choice are two small functions | `gateable_tasks` (`_bead_task_triage_state.py:206-212`), `expected_gate_kind` (`_bead_task_triage_gates.py:30-32`)                                                               |
| A gate kind is a spec, a preview, a response, and effects    | `bead/_task_gate_spec.py`, `_task_gate_preview.py`, `_task_gate_response.py`, `_task_gate_actions.py`, `task_gate.py` facade                                                     |
| Gate options already support typed declared inputs           | `GateInputField` supports `enum` with choices (`notification_gates/model_inputs.py:40-50`); snooze uses one (`snooze_gate_input.py:37-52`)                                       |
| A gate declaring `panel: "beads"` bypasses the Gates tab     | `notification_modal_tags.py:36-49` — BeadSnooze stays out of `HITL_ACTIONS` for exactly this reason                                                                              |
| Config layers are the persistent control plane               | `default` → `plugin:*` → `user` → `overlay:*` → `local` (`config/layers.py`)                                                                                                     |
| The config schema root is closed                             | `sase.schema.json` has `additionalProperties: false` and 44 top-level properties                                                                                                 |
| Registry-owned defaults are precedent, not novelty           | 14 of the 44 schema keys have no `default_config.yml` entry (`artifact_refs`, `github_orgs`, `timezone`, …)                                                                      |
| Project-local config is genuinely invisible to ACE           | `set_include_local_config(False)` (`config/core.py:101`) called from `main/ace_handler.py:162`                                                                                   |
| Child processes inherit env for free at 15 spawn sites       | 15 `os.environ.copy()` call sites; there is no allowlist to extend                                                                                                               |
| Ad-hoc env gates use at least five incompatible conventions  | non-empty, `== "1"`, two independent `_TRUTHY` sets, `{"1","true","yes"}`, `{"1","true","yes","on","soft"}`                                                                      |
| One ad-hoc gate is evaluated at **import** time              | `ace/tui/bindings.py:257` appends a `Binding` at module scope                                                                                                                    |
| Plugin discovery feeds the `plugin:*` config layers          | `discover_plugin_resources("sase_config")`; `is_plugin_disabled` reads env only (`main/plugin_discovery.py:14-25`)                                                               |
| `symvision` is an external package, not ours to extend       | `pyproject.toml:60` pins `symvision>=0.1.0,<0.2.0`; `sase/memory/symvision.md` forbids vendoring                                                                                 |
| The bead-status lint handshake is proven and reusable        | `SASE_SYMVISION_BEAD_STATUS_ONLY=1 BD_COMMAND=tools/sase_bead` (`Justfile:303-304`)                                                                                              |
| `just lint` is a flat list of nine one-line stages           | `Justfile:259-343`; `tools/check_test_wait_helpers` and `tools/validate_changelog` are the shape to copy                                                                         |
| Generated long memory notes are a two-line registration      | `_GeneratedLongMemorySpec` + `_GENERATED_PROJECT_LONG_MEMORY_SPECS` (`init_memory/root_rendering.py:62-145`)                                                                     |
| No top-level `sase flag` command exists                      | neither `sase --help` nor `sase --full-help` mentions a `flag` command                                                                                                           |
| No prior beads exist for this work                           | `sase bead search` finds nothing for "feature flag"                                                                                                                              |

## Decisions a phase worker must not silently revert

**1. `remove_by` lives on the bead, never in the registry.** The registry declares what
a flag _is_ (key, kind, default, scope, description); the bead owns _when it is due_.
This is the decision that makes `Extend` a one-click, attributed, append-only event
instead of a code change, and it puts the deadline's history in the same event stream as
the close, the notes, and the reopen. A phase worker who moves `remove_by` into the
Python registry has destroyed the Extend option's entire reason to exist.

**2. Nothing about a flag's resolved value ever depends on wall-clock time.** The
deadline drives the gate, the doctor, and the lint warning — never the boolean. There
are no time-bombed installs. A flag resolves identically on the day before and the day
after its `remove_by`.

**3. Due-ness is derived, never persisted.** The chop does **not** flip a flag bead to
`ready` when its deadline passes. Flag beads use `open` → `in_progress` → `closed` and
never take `ready` or `snoozed`, so the core's existing "Only task issues can have ready
status" and "Only task issues can have snoozed status" rules stay literally true.
Persisting due-ness would mean an event stream that churns on clock skew and a bead that
has to be un-readied every time a deadline is extended.

**4. The flag bead is a dedicated removal bead — never the epic or implementation
bead.** An epic spanning a release boundary must land with its flag still off, but its
land agent must close the epic bead to land; keying the flag to the epic bead makes that
combination unlandable. The same bug hits the beta case, where "add feature X" closes
months before the flag dies. `sase flag new` creates a `flag` bead in the same change
that adds the flag.

**5. An overdue flag warns in CI and errors in `sase doctor`; the gate is the action
channel.** This is a deliberate divergence from the consolidated research, which
recommends a hard CI failure after a grace window. A pending gate that re-raises every
reconciliation tick, sits in the ACE beads panel, and demands a written reason to defer
is a _stronger_ nag than a red build an owner learns to ignore — and it never breaks an
unrelated change for a deadline it had nothing to do with. The integrity rules (§`lint`
rules 6-8) stay hard errors, because those are real inconsistencies rather than schedule
facts.

**6. `plugin:*` layers may register flags but may never flip a first-party default.**
Installing a plugin must not silently change SASE behavior.

**7. One immutable snapshot per process, resolved at a command or app boundary, never at
import time.** `ace/tui/bindings.py:257` is the cautionary example and a named migration
target. Downstream code receives the snapshot; it does not consult global flag state.
The parent re-encodes the **resolved** snapshot into `SASE_FEATURE_FLAGS` for children,
so a child cannot re-resolve against a different project's config mid-operation.

**8. The flag registry stays in Python.** Not because a core round trip is expensive —
the core floor ratcheted roughly twenty times in the eight days before this plan — but
because no Rust path resolves a flag under any proposal, and
`sase_core/src/config/mod.rs` already carries a
duplicated-`_deep_merge`-with-parity-tests burden that a second boolean-merge surface
would only compound. Promote to `sase_core::feature_flags` on one concrete trigger: a
non-Python frontend must resolve a flag _independently_ rather than receive a resolved
snapshot. The **bead type** is a different question and does cross the boundary, because
the bead store's wire, validation, and event stream are core-owned.

**9. A flag is a router, not a migration protocol.** Persisted and wire formats still
need independent expand-and-contract. The registry is an inventory, not a dependency
graph: no parent flags, no implied flags, no "enable everything experimental" switch.

**10. Deprecation extends the existing key ladder rather than forking it.** Flags own
_behavior_; `UNSUPPORTED_TOP_LEVEL_KEYS` / `DEPRECATED_TOP_LEVEL_KEYS` /
`RETIRED_SDD_SELECTOR_KEYS` own the _config surface_. Do not introduce a second
deprecation system.

## The shape of a flag

A flag has exactly two artifacts, linked by ID in both directions.

**In code** — `src/sase/feature_flags/registry.py`:

```python
@dataclass(frozen=True)
class FeatureFlagDefinition:
    key: FeatureFlag                              # typed enum member, never a bare string
    kind: Literal["beta", "wip", "sunset", "ops"]
    description: str
    default: bool
    scope: Literal["global", "project"]
    bead: str | None                              # flag bead id; required unless kind == "ops"
    rationale: str = ""                           # required when kind == "ops"
```

**In the bead store** — a `flag` bead carrying one `FlagRecord`:

```python
@dataclass(frozen=True)
class FlagRecord:
    key: str                  # the registry key this bead owns
    remove_by_date: str       # ISO date, e.g. "2026-11-14"
    remove_by_release: str    # release threshold, e.g. "0.19.0"
```

The lint and the doctor check both directions: a definition must name a live `flag` bead
whose `flag.key` matches, and a live `flag` bead must have a matching definition. A
closed flag bead with a surviving definition is an error, and so is the reverse.

## Phases

### core — The flag bead type in sase-core

Add the fourth bead type to the core wire, mirroring the snooze record's shape exactly.

_The wire._ Add `IssueTypeWire::Flag` to `bead/wire.rs:25-30` and a `BeadFlagWire`
(`key`, `remove_by_date`, `remove_by_release`) beside `BeadSnoozeWire`, plus an optional
`flag` field on `IssueWire`. Serde is `snake_case`, matching every sibling.

_Validation._ Extend `IssueWire::validate` (`wire.rs:527-593`) with rules that mirror
the existing ones message for message, because the Python layer restates them verbatim:

- `flag` metadata is present if and only if `issue_type == Flag` (the snooze rule's
  shape);
- a flag issue has no `parent_id` and no `tier`;
- `remove_by_date` parses as an ISO date and `remove_by_release` as a release string,
  with the same "must be …" error phrasing the snooze timestamp parser uses;
- `key` is non-empty `snake_case`;
- the existing "Only task issues can carry +1 evidence", "can have ready status", and
  "can have snoozed status" rules are left **unchanged** — they already exclude flag
  beads.

_Exhaustive matches._ The compiler names most of these; the plan names them so none is
resolved by a stub arm. `issue_type_presentation` (`cli.rs:2228-2243`) gains the flag
glyph `⚑` on a new `ANSI_TYPE_FLAG = "\x1b[38;5;209m"` (xterm 209, matching the
`#FF875F` accent `look` registers on the Python side); `compact_type_width` gains the
variant; `parse_issue_type` (`cli.rs:2424`) accepts `"flag"`; `parse_create_type`
(`cli.rs:861+`) accepts `flag(<key>,<YYYY-MM-DD>,<release>)`, following the
`plan(<path>,<parent>)` form exactly and rejecting the arity error the same way; the
display sort rank (`events.rs:1580-1582`) gains `Flag => 3`; `search.rs:345` gains
`"flag"`.

_Storage._ Extend the SQLite projection and the JSONL codec with the flag column,
following the snooze column's migration precedent in `bead/schema.rs`, and add the field
to `bead_create`/`bead_update` field handling so a flag record round-trips through a
mutation.

_Parity._ Add flag cases to `tests/bead_event_parity.rs` and
`tests/python_wire_parity.rs` covering create, update of `remove_by_*`, close, and the
four new validation errors.

The Python side of this epic builds against the local checkout, so no published release
gates the next phase; the `sase-core-rs` floor ratchets at release time as usual.

Exit condition: `cargo test` passes in the core checkout, a `flag` bead round-trips
through create/update/close, and every one of the five listed match sites handles `Flag`
explicitly.

### registry — The typed registry, resolver, and snapshot

Build the flag mechanism itself, with no dependency on the bead type.

_The package._ Add `src/sase/feature_flags/` with `models.py` (`FeatureFlagDefinition`,
`FeatureFlagDecision`, `FeatureFlagSnapshot`), `registry.py` (the `FeatureFlag` key enum
and the definition table), `resolver.py`, `env.py`, and `schema.py`. The registry ships
**empty of temporary flags**; `consumer` adds the first real entries. Call sites take a
typed enum member so a typo fails at type-check time.

_Resolution._ Lowest to highest, mirroring the existing layer chain: registry default →
`user` → `overlay:*` in existing order → `local` (**only** for `scope: "project"` flags)
→ an explicit in-process/test override → `SASE_FEATURE_FLAGS`. `plugin:*` is skipped for
any first-party key. Resolution yields a `FeatureFlagDecision` (`enabled`, `default`,
`source`, `source_detail`, `overridden`) so diagnostics keep the whole story; routing
calls `snapshot.enabled(key)`.

_Validation is the resolver's own job._ Nothing schema-validates ordinary config layers
at startup — `config_validate` runs only on candidate merged config during Config Center
edit planning, which is why domain loaders like `config/file_hooks.py` diagnose their
own unknown fields. So the resolver validates keys, types, and scope itself and emits
diagnostics. Bad **env** input fails loudly, because an operator typed it into this
process; unknown **file** keys warn and are ignored, because a config file outlives the
flag it names.

_The snapshot._ One immutable snapshot per process, built at the command/app boundary
(`main/entry.py:main`, `main/ace_handler.py`, the agent runner and axe entry points),
logged once for the non-default set, injected at routing boundaries. Provide an explicit
`override_flags(...)` context manager for tests so no test reaches for `monkeypatch` on
a module global.

_The transport._ `SASE_FEATURE_FLAGS` is one strict-parsed JSON object of booleans, not
one variable per flag. The parent re-encodes the **resolved** snapshot before spawning,
which the 15 `os.environ.copy()` sites inherit for free. The same fact is a hazard: an
exported flag reaches every descendant, including detached procs and monitors that
outlive the shell that set it, so `env` provenance is surfaced prominently by `cli` and
`doctor`.

_The schema._ Generate the `feature_flags` block into `config/sase.schema.json` from the
registry: named `properties` per key (so Config Center renders one row per flag with its
description, default, and `deprecated` markers through the existing `config_inventory`
contract, with **zero Rust changes**) plus `additionalProperties: {"type": "boolean"}`
so a downgraded install tolerates a key it no longer knows. This exploits `schema.rs`'s
documented rule that a node with named `properties` is a closed `"object"` while an open
object-typed node collapses to one opaque `"map"` leaf. Defaults live in the registry,
not `default_config.yml`; 14 existing schema keys already work that way.

Exit condition: a registered flag resolves correctly from every layer with correct
provenance, a project-scoped override is honored by the CLI and ignored by ACE, a
malformed `SASE_FEATURE_FLAGS` fails loudly, and an unknown key in a config file warns
and is ignored.

### bead — Flag beads in the Python bead layer

Mirror the core type through Python, following the snooze field's path end to end.

_The model._ Add `IssueType.FLAG = "flag"`, the `FlagRecord` dataclass with its
`validate`, and `Issue.flag: FlagRecord | None`. Restate each new core rule in
`Issue.validate` (`bead/model.py:230-282`) with the same message text, because the two
validators are deliberately duplicated and a parity test compares them.

_The plumbing._ Extend `bead/_db_codec.py`, `bead/_db_rows.py`, `bead/_db_schema.py`,
and `bead/_db_migrations.py` with the flag column; extend `core/bead_wire.py` conversion
in both directions; extend `bead/jsonl.py`.

_The CLI._ `parse_type_arg` (`bead/cli_crud.py:59-98`) accepts
`flag(<key>,<YYYY-MM-DD>,<release>)` with the same arity errors the `plan()` form uses.
`sase bead show` gains a `FLAG` section (key, `remove_by`, derived due state);
`sase bead update` accepts `--remove-by <YYYY-MM-DD>/<release>` for the Extend path;
`sase bead list --type flag` works through the existing derived `BEAD_TYPE_VALUES` path.
Follow `sase/memory/cli_rules.md`: alphabetical subcommands and options, a short alias
for every long option, no required options.

_Due-ness._ Add one shared predicate — `flag_removal_due(record, *, today, release)` —
that returns a closed state (`live`, `soon`, `due`) from the **later** of the date and
the release threshold. This is the single definition the chop, the CLI, the doctor, the
lint, and every renderer use; nothing else may recompute it. It is a pure function of
its arguments, so the callers own the clock and tests never need to freeze time
globally.

Exit condition: `sase bead create -T "flag(demo_key,2026-12-01,0.19.0)"` round-trips
through create, show, list, update, and close, and every validation rule rejects with
the same text the core does.

### look — The shared flag visual language

Own the flag's appearance in one place, before three phases render it.

_The type._ Add the `"flag"` entry to `BEAD_TYPE_PRESENTATIONS`
(`bead_type_presentation.py`): glyph `⚑`, accent `#FF875F`, chip
`bold black on #FF875F`, label `Flag`. The accent is a warm coral chosen to sit apart
from plan gold `#FFD700`, phase sky `#87D7FF`, and task violet `#D787FF` while reading
as "temporary, expiring" rather than "error". Update
`tests/test_bead_type_presentation.py`'s order assertion and confirm
`BEAD_TYPE_CHIP_WIDTH` still resolves. The three derived surfaces —
`bead/filter_query.py:362`, `ace/query_profile/profiles.py:183`, `bead_filter_bar.py:60`
— must pick the new value up with no edit; if any needs one, that is a bug in the
derivation to fix here.

_The countdown._ Add `src/sase/bead_flag_presentation.py`, a sibling of
`bead_time_presentation.py` and following its dual-surface shape (a Rich `Text` builder
and an ANSI CLI cell from one presentation record):

- `flag_key_chip(key)` — `⚑ plugins_enabled` on the flag accent, the flag's identity;
- `flag_due_chip(record, *, today, release)` — the urgency-graded removal meter, driven
  by the one `flag_removal_due` predicate `bead` defined:
  - `live` — dim `⧗ 84d · v0.19.0`
  - `soon` — one threshold reached, bold amber `⧗ 12d · v0.19.0`
  - `due` — both reached, bold reverse `DUE` plus `⧗ +6d` overshoot
- `FLAG_DUE_STYLES` — the state-to-accent map every surface shares, so the CLI, the TUI,
  the bead pages, and the gate preview never disagree about what "due" looks like.

Exit condition: a golden test pins the glyph, the accent, and all three countdown states
in both the Rich and ANSI renderings, and the Rust `ANSI_TYPE_FLAG` from `core` is
asserted equal to the Python `cli_style` for the flag type.

### lint — Registry and bead integrity enforcement

Ship enforcement with the registry, because shipping it later is how flag debt happens.

_The tool._ Add `tools/check_feature_flags`, an ordinary extensionless tool matching
`tools/check_test_wait_helpers` and `tools/validate_changelog`. Do not build on
`symvision`: it is a published external dependency (`pyproject.toml:60`) and
`sase/memory/symvision.md` forbids patching or vendoring it. What is reusable is its
_loop_ — an in-tree temporary allowance keyed to a bead, checked against live bead
status, self-cleaning — and its CI handshake.

Static rules, which need no bead store and therefore also run under `just validate`:

1. every non-`ops` definition names a bead; every `ops` definition carries a
   `rationale`;
2. the generated `feature_flags` schema block matches the registry exactly;
3. every registered key has at least one non-test reference — a flag nothing reads is
   already dead;
4. no flag is resolved at module import time, so `ace/tui/bindings.py:257`'s
   anti-pattern cannot return under a new name;
5. no repo-managed config layer overrides an unregistered `feature_flags` key.

Bead-status rules, run with
`SASE_SYMVISION_BEAD_STATUS_ONLY=1 BD_COMMAND=tools/sase_bead`:

6. the named bead exists, is type `flag`, and its `flag.key` equals the definition's
   key;
7. a **closed** flag bead whose definition survives is an error — you cannot mark the
   work done and keep the flag;
8. a **live** flag bead with no definition is an error — the reverse orphan;
9. an **overdue** flag is a warning, never an error (decision 5).

_Wiring._ Add a `_lint-flags` recipe and one line in `lint`, matching the eight sibling
stages. Design it so the unimplemented `_lint-backcompat` design can later add a second
marker source to the _same_ checker rather than shipping a second bead-aware expiry
linter a quarter later.

Exit condition: each of the nine rules has a failing fixture and a passing one, the
static subset runs green under `just validate` with no bead store present, and
`just check` is green.

### gate — The FlagTriage gate and its reconciler

Make a due flag ask a question the owner can answer in one keystroke.

_The contract._ Add the `flag_triage` gate kind as a five-module set mirroring the
TaskTriage layout exactly — `bead/_flag_gate_spec.py`, `_flag_gate_preview.py`,
`_flag_gate_response.py`, `_flag_gate_actions.py`, and the `bead/flag_gate.py` facade
the persisted command wrapper names by path. Every helper stays a pure function of its
arguments, because gate validation rebuilds the spec from the persisted payload and
compares it byte for byte. Register a
`GateAdapter(kind="flag_triage", action="FlagTriage", sender="bead", neutral_only=True, generic_form=True, auto_policy="forbidden")`
and its `kind_validation` module.

_The four options._ Every real answer to "your flag is due" gets a button, so no owner
is ever forced to click the wrong one:

| id       | icon | label  | inputs                                       | host effect                                                                          |
| -------- | ---- | ------ | -------------------------------------------- | ------------------------------------------------------------------------------------ |
| `remove` | 🚀   | Remove | `winner`: enum `enabled` \| `disabled`       | record the winning branch on the bead, then `sase bead work <flag-id>`               |
| `extend` | ⏳   | Extend | `until` (date expression), `release`; reason | rewrite `flag.remove_by_*`; the bead stays `open`                                    |
| `keep`   | ⚑    | Keep   | required rationale                           | launch a promotion worker: `kind` → `ops` with the rationale, or an ordinary setting |
| `close`  | ✕    | Close  | required reason                              | close the bead; `lint` rule 8 catches the orphan if the flag survives                |

`remove` is the primary branch. The `winner` enum is what makes `Remove` actionable
rather than a shrug: the worker is told which branch survives and which is deleted.
`extend` is the one path that defers, and it costs a written reason and a new dated
threshold, so perpetual silent extension is impossible. `keep` exists so that "this was
never temporary" is a supported, recorded answer rather than a broken build; without it,
decision 5's humane lint would have no path to a legitimately permanent flag.

Reuse `snooze_time`'s parser for the `until` expression and
`snooze_duration_result_property`'s echo-back pattern, so the host effect resolves the
instant the reviewer actually chose rather than re-reading free text.

_Presentation._ The gate declares `sender: "bead"`, `panel: "beads"`, `panel_icon: "⚑"`,
tags `["bead", "flag"]`, and a `flag.md` preview rendering the key, both thresholds, the
`look` countdown, the definition's kind and description, the bead's notes, and the
flag's call sites. Because it declares `panel: "beads"`, it routes there by the
higher-precedence panel rule and stays out of `HITL_ACTIONS` — exactly like BeadSnooze —
so **this phase needs no `sase-core` change**.

_The reconciler._ Generalize `sase_chop_bead_task_triage.py` rather than forking it. Its
docstring already frames its invariant as "the one pending gate each live bead may
have", one lane state under one lock; a second chop could only race this one into giving
a bead two gates. Three small edits carry it: `gateable_tasks` becomes a
`gateable_beads` that also returns live `flag` beads whose `flag_removal_due` state is
`due`; `expected_gate_kind` returns `FLAG_TRIAGE_KIND` for a flag bead; `_GATE_KINDS`
gains the third kind. The presentation fingerprint must include the due state and both
thresholds, so an Extend replaces the pending gate instead of leaving a stale one asking
a settled question. The chop keeps its name and state file; renaming it is deliberately
out of scope, and its module docstring is updated to say it owns every bead-scoped gate.

Exit condition: a flag bead whose thresholds have both passed raises exactly one pending
`FlagTriage` gate; Extend rewrites the bead and cancels the gate; Remove launches a
worker carrying the winning branch; the reconciler is idempotent across ticks; and a
bead already holding a task gate never gains a second one.

### cli — sase flag and the flag doctor checks

Give flags a front door for the two questions people actually ask: what is on, and what
is due.

_The group._ Add a top-level `sase flag` group defaulting to `list` through the central
`_default_list_subcommands()` wiring — never a per-command reimplementation. A top-level
noun is right here rather than the research's `sase config flags`: with beads, gates,
and a lifecycle, a flag is its own domain like `sase bead` and `sase patch`, not a
config view.

- `sase flag list` — one colored row per flag: key chip, kind, default, effective value,
  source layer, scope, bead id and status, and the `look` countdown. `env` provenance is
  called out prominently, because an inherited flag in a long-running detached proc is
  otherwise invisible and effectively unfalsifiable.
- `sase flag show <key>` — the full `FeatureFlagDecision` with per-layer provenance, the
  bead's detail, both thresholds, and the flag's call sites.
- `sase flag new <key>` — the scaffold, and the path `sase/memory/sase_flags.md`
  teaches. Creates the `flag` bead with computed defaults (`remove_by_date` = today + 90
  days, `remove_by_release` = the current minor plus two, both overridable through
  `-r/--remove-by`), then prints the registry entry to paste and the both-states test
  checklist. Options `-d/--description`, `-k/--kind`, `-r/--remove-by`, `-s/--scope`,
  `-z/--size`; none required.

_The doctor._ Register `flags.registry` (both-direction registry/bead integrity),
`flags.overrides` (unknown keys, non-boolean values, project-scoped overrides on a
global flag, and env-inherited values), and `flags.due` (overdue flags — an **error**
here, per decision 5). Follow the dotted domain-namespaced convention of `config.repos`,
`llm.registry`, and `axe.chops`.

Exit condition: `sase flag`, `sase flag list`, `sase flag show <key>`, and
`sase flag new <key>` all behave, `-h` output is complete and alphabetized, and each
doctor check has an OK, a WARN, and an ERROR fixture.

### ui — Flag beads on every bead-rendering surface

A flag bead must be unmistakable everywhere a bead is drawn, using only the `look`
vocabulary. This phase adds no new colors, glyphs, or states of its own.

_The bead CLI._ `cli_query_render.py` rows get the flag glyph free from the derived
width computation; add the countdown cell for flag rows. `cli_detail_prose.py`,
`cli_detail_style.py`, and `cli_detail_json.py` gain the `FLAG` section — key, both
thresholds, due state — and the JSON projection gains the same fields, which is also
what `sase-telegram`'s `parse_bead_list_json` reads.

_Bead pages._ `bead_pages/rendering_identity.py:228`, `roster.py:41-46`, and
`rendering_tables.py` render the flag type and the key; the published page for a flag
bead states its thresholds so the markdown mirror is self-describing.

_Counts._ `bead_summary_presentation.py` picks up the flag type automatically through
`BEAD_TYPE_VALUES`; add the due-flag count so `sase bead stats` and the ACE state/count
lane both report it.

_The ACE Beads pane._ Add a `flag_text()` renderer in `beads_rendering.py` beside
`task_text`/`epic_text`/`phase_text`, composing the type glyph, the key chip, the title,
the status, and the countdown. `beads_detail.py` gains the Flag rows. `beads_data.py`,
`beads_navigation.py`, and `beads_options.py` place flags as their own group in the
list. `bead_filter_bar.py` gets `type:flag` free from `BEAD_TYPE_VALUES`; add a `due:`
token for the three countdown states through `bead/filter_query.py` and
`ace/query_profile/profiles.py`. The flag and due counts belong in the shared
state/count lane, never a second identity header —
`docs/artifacts_pane_visual_grammar.md` is the binding contract for this pane and must
be read before touching it, as must `sase/memory/tui_perf.md`.

_ACE modals._ `bead_create_modal.py` (create a flag bead with its thresholds),
`bead_editor_modal.py` (already reads `bead_type_presentation`; verify the flag label),
`bead_close_modal.py`, `wait_modal_beads.py`, and the prompt panel's
`_agent_bead_section.py`.

_Notifications._ `notification_modal_tags.py` gives the `flag` tag the flag accent and
glyph; confirm `gate_debug_modal.py` renders a `FlagTriage` bundle.

_Telegram._ `sase-telegram/bead_format.py` tolerates unknown ALL-CAPS sections by
design; verify `FLAG` round-trips through `bead_show_to_markdown` and add a regression
test with a real flag-bead fixture. Open that repository with `/sase_repo` before
editing it.

_External mirror._ `external_mirror/_issue_apply.py:80,125` mirrors only task beads
today. Make the exclusion of flag beads explicit and commented — flag hygiene is
internal and should not become GitHub issue noise.

_Snapshots._ Add a PNG golden covering a flag bead in all three countdown states
(`tests/ace/tui/visual/`), following the existing beads snapshot fixtures.

Exit condition: every surface in this list renders a flag bead distinctly,
`just test-visual` is green with the new golden accepted deliberately, and no surface
hard-codes a flag color or glyph outside `look`'s modules.

### consumer — The first two real flags

A registry with no consumer proves nothing. Convert two existing ad-hoc env gates, one
per kind, and give each a real flag bead.

**`coder_inherits_planner_chat`** — `kind: "beta"`, default `false`, `scope: "global"`.
Replaces `SASE_CODER_INHERIT_PLANNER_CHAT` (`axe/run_agent_exec_plan_accept.py:444`),
which is already documented in a comment as an opt-in behavior toggle. One site, one
convention (`== "1"`), genuinely user-visible behavior, and both states are testable.
This is the worked example `sase/memory/sase_flags.md` will teach from.

**`prettier_enabled`** — `kind: "sunset"`, default `true`, `scope: "global"`. Replaces
`SASE_DISABLE_PRETTIER` (`file_references.py:529,569`), killing a `disable_*` double
negative and demonstrating the deprecation path for a retired env name: the old variable
keeps working, mapped into the snapshot with a deprecation diagnostic surfaced by
`sase flag list` and `sase doctor -C flags.overrides`. The `shutil.which("prettier")`
availability guard stays — that is capability detection, not a flag.

_Why not `SASE_DISABLE_PLUGINS`._ The consolidated research recommends it as the first
consumer, and this plan deliberately declines.
`discover_plugin_resources("sase_config")` feeds the `plugin:*` config layers, so a flag
resolved from config cannot gate plugin discovery without a bootstrap cycle: config
needs plugins, plugins would need the flag, the flag needs config. Record this finding
on the phase bead. If `plugins_enabled` is ever wanted, it needs an explicit bootstrap
scope resolved from the registry default and `SASE_FEATURE_FLAGS` only — a separate,
later decision, not a v1 requirement.

_Each conversion ships whole._ A `flag` bead created through `sase flag new`, the
registry entry, the call site converted to `snapshot.enabled(...)`, tests covering
**both** states, the generated schema block regenerated, and `tools/check_feature_flags`
green.

Exit condition: two flags resolve end to end from every layer, both appear in
`sase flag list` with live countdowns, both flag beads render across the surfaces `ui`
built, and forcing one bead's `remove_by` into the past raises a real `FlagTriage` gate
that Extend clears.

### memory — sase_flags.md, the sase.md pointer, and the docs

Teach the loop, in as few tokens as it can be taught in.

_The long memory._ Add
`src/sase/main/init_memory/templates/memory-sase-flags.template.md` and register it as a
`_GeneratedLongMemorySpec` in `_GENERATED_PROJECT_LONG_MEMORY_SPECS`
(`init_memory/root_rendering.py:130-145`) with `relative_path`
`sase/memory/sase_flags.md`, `parent=AGENTS_PARENT` so it is indexed in the Tier 2
listing beside `sase_beads.md`, and `detail="generated SASE feature flag memory"`. Add
`_generated_flags_memory_relative_path()` beside its two siblings. The note's
`description` frontmatter is what agents scan to decide relevance, so it must name the
decision it settles:

> Read before adding, deferring, or removing a SASE feature flag or flag bead.

Target under 60 lines. It must cover, and cover only: what a flag is and the four kinds;
the `sase flag new` scaffold as the only creation path; why the bead is a dedicated
removal bead; that both thresholds must pass; that both states are always tested; the
four gate answers and when each is right; that removing the flag means deleting the
losing branch in the change that closes the bead; and the one anti-pattern that matters
— if users are meant to choose forever, it was never a flag, so promote it to an
ordinary config field. It must **not** restate `sase flag -h`, the resolution chain's
implementation, or anything `sase_beads.md` already owns.

_The always-loaded pointer._ Add one short section to
`templates/memory-sase.template.md`. Tier 1 is loaded into every agent's context, so
this is the most expensive text in the epic and gets the tightest budget: **at most 90
words of prose under one heading**, stating only when a flag is warranted and where to
read the rest. Draft:

> ## Feature Flags
>
> You MUST put a feature flag on behavior that reaches users before it is ready: a beta
> feature shipped disabled, a phase that lands user-reachable behavior early, or a
> deprecation whose old branch must stay reachable for now. You SHOULD NOT flag anything
> a user is meant to choose forever — that is an ordinary config field.
>
> Create one only with `sase flag new <key>`, which also files its `flag` removal bead.
> Read `sase/memory/sase_flags.md` with `/sase_memory_read` before adding, deferring, or
> removing any flag.

Both templates must pass `validate_short_memory_structure` and the generated-note
description check; run `sase memory init` to regenerate `AGENTS.md`, the provider shims,
and the memory README. Editing these files is authorized by the prompt that requested
this epic, and no separate approval is needed for the regeneration step.

_The glossary._ Add `Feature Flag` and `Flag Bead` to the `glossary:` block in
`sase/sase.yml`, alphabetically among the existing 24 terms, with `Flag Bead` aliasing
`flag bead`. `sase memory init` regenerates `sase/memory/glossary.md` from it.

_The user docs._ Add a "Feature flags" section to `docs/configuration.md` (the
`feature_flags` config surface, the resolution chain, `SASE_FEATURE_FLAGS`, and the
scope rule); extend `docs/beads.md` with the `flag` type and its lifecycle; add
`FlagTriage` to `docs/notifications.md`'s gate list; and note the `sase flag` group in
`docs/cli.md`.

Exit condition: `sase init -c` reports no memory drift, `sase memory list` shows
`sase_flags.md` as a Tier 2 note with its description, the Tier 1 addition is within its
90-word budget, and `just check-full` is green through a monitor.

## Verification

Every phase runs `just check` before handing off. The land agent runs `just check-full`
through `/sase_monitor` on the combined tree, because this epic touches the broadening
set: the config schema, the bead wire, a chop, and the ACE Beads pane.
`just test-visual` runs with the `ui` phase's golden.

Two whole-epic checks belong to the land agent and to no phase:

1. **The loop closes.** With `consumer`'s two flags live, force one bead's `remove_by`
   into the past, run the reconciler, answer the resulting gate with Extend, and confirm
   the bead's threshold moved, the reason was recorded, and the pending gate was
   replaced rather than duplicated. Then force it again and answer with Remove, and
   confirm the launched worker receives the winning branch.
2. **Nothing resolves at import time.** `tools/check_feature_flags` rule 4 asserts it
   statically; the land agent confirms `python -c "import sase"` performs no flag
   resolution.
