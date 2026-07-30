---
tier: epic
title: Structured sase variables (nested lists and maps) across every display surface
goal: 'A sase variable holds any JSON value — string, number, boolean, null, list,
  or map — nested arbitrarily, with one canonical validation model and one canonical
  renderer so ACE, the agents sidecar, Telegram, notifications, and the CLI all display
  structured values identically and beautifully.

  '
phases:
- id: var-value-model
  title: Canonical structured value model, storage, and renderers
  depends_on: []
  size: medium
  description: 'var-value-model: introduce the JSON value model (validation, normalization,
    caps) and the single canonical inline/block renderer in sase.core, widen agent_meta.json
    storage and readers to structured values, and keep every existing consumer compiling
    and behaving unchanged.'
- id: core-wire-json
  title: Full JSON output-variable values in the sase-core scan wire
  depends_on: []
  size: medium
  description: 'core-wire-json: generalize OutputVariableValue in the sase-core agent-scan
    wire from text-or-string-list to a bounded JSON value, release sase-core, bump
    the sase-core-rs pin here, and widen the Python wire marker type.'
- id: var-cli-jinja
  title: Authoring and consuming structured variables (CLI, Jinja, STOP, skill, docs)
  depends_on:
  - var-value-model
  size: medium
  description: 'var-cli-jinja: add the `--json` value modifier and a `sase var list`
    display subcommand, pass containers into the Jinja `agents` namespace with JSON-shaped
    stringification, generalize STOP truthiness, and update the sase_var skill source
    and reference docs.'
- id: ace-var-display
  title: ACE renders structured variables in agent, clan, and tribe panels
  depends_on:
  - var-value-model
  - core-wire-json
  size: medium
  description: 'ace-var-display: widen the ACE agent/clan/tribe variable models and
    loaders to structured values and render them with the canonical line renderer
    and per-kind styling at every fold level.'
- id: sidecar-var-publication
  title: Agents sidecar publishes and renders structured variables
  depends_on:
  - var-value-model
  size: medium
  description: 'sidecar-var-publication: accept structured values in v2 publication
    validation and the portable-metadata sanitizer, and render them in agent and family
    sidecar pages with inline table previews plus fenced blocks.'
- id: notify-var-display
  title: Completion notifications and Telegram render structured variables
  depends_on:
  - var-value-model
  size: medium
  description: 'notify-var-display: widen the completion-notification variable snapshot
    and render structured values in the sase-telegram plugin''s completion message
    and agent detail rows using the canonical renderer.'
create_time: 2026-07-30 17:00:13
status: done
bead_id: sase-bf
---

- **PROMPT:** [202607/prompts/structured_sase_variables.md](prompts/structured_sase_variables.md)
- **BEAD:** [sase-bf](https://github.com/sase-org/sase--beads/blob/main/pages/sase-bf/README.md)

# Plan: Structured sase variables (nested lists and maps)

## Motivation

`sase var set` stores a flat `dict[str, str]`. Agents that need to hand a _collection_ to a later agent today either
encode it themselves (comma-joined paths, hand-rolled JSON that every consumer must re-parse) or set N numbered
variables. Both are lossy, both push parsing into prompts, and neither renders well anywhere.

The fix is to make the value model match what agents actually produce: **a sase variable holds any JSON value**. Strings
stay the common case and behave exactly as they do today; lists and maps become first-class, nest arbitrarily, flow into
Jinja as real Python containers, and render through one canonical formatter so ACE, the agents sidecar, Telegram, the
completion notification, and the CLI all show the same shape.

## Current state (verified anchors; line numbers as of master @ 84d47aa78)

**Storage and validation (Python, this repo)**

- `src/sase/core/agent_output_variables.py` is the single writer. `MAX_OUTPUT_VARIABLES = 256` and
  `MAX_OUTPUT_VARIABLE_VALUE_BYTES = 8_192` (:19-20); key rule `[A-Za-z_][A-Za-z0-9_]*` (`_KEY_RE` :22,
  `_validate_output_variable_key` :76-83); value normalization is CRLF→LF, NUL rejection, 8 KiB UTF-8 cap
  (`_normalize_output_variable_value` :86-99); `set_agent_output_variables` (:49-73) merges under an `flock` and writes
  atomically with `json.dump(..., indent=2, sort_keys=True)` (`_write_json_object_atomic` :124-129);
  `_string_output_variables` (:102-109) is the tolerant reader and **silently drops every non-string value**.
- `parse_output_variable_assignments` (:26-39) splits `KEY=VALUE` on the first `=`.

**CLI**

- `src/sase/main/parser_var.py` — `sase var` has exactly one child, `set` (:32-71), with a mutually exclusive value
  source group: `-f/--value-file` and `-v/--value` (:59-71).
- `src/sase/main/var_handler.py` — `handle_var_command` (:18-25) prints `Usage: sase var {set}` for anything else;
  `_handle_var_set` (:28-61) requires `SASE_AGENT=1` + `SASE_ARTIFACTS_DIR`; `_output_variables_from_args` (:64-88)
  enforces "value-source form takes exactly one bare KEY" and strips at most one trailing newline for `--value-file`.

**Consumption**

- Jinja: `src/sase/agent/output_variable_context.py` builds `{"agents": {agent_key: variables}}`
  (`build_agent_output_variable_context` :87-140), typed `dict[str, dict[str, str]]` throughout
  (`_merge_agent_variables` :326-336, `read_waited_agent_output_variables` :291-303). The environment is a plain
  `Environment(loader=BaseLoader(), undefined=StrictUndefined, autoescape=False)` (`src/sase/xprompt/_jinja.py:55-64`),
  so Jinja's built-in `tojson` filter is available.
- Repeat `STOP`: `src/sase/axe/run_agent_repeat_stop.py` — `STOP_OUTPUT_VARIABLE` (:24), falsy set
  `{"", "0", "false", "no", "off"}` (:28), `_is_stop_value` (:31-33), `RepeatStopDecision.stop_value: str` (:45-46),
  `detect_repeat_stop` (:49-72). The value is re-propagated into the stopping slot by
  `src/sase/axe/run_agent_runner_repeat.py:49`.

**Display surfaces (all five)**

1. **ACE TUI agent panel** — `src/sase/ace/tui/widgets/prompt_panel/_agent_output_variables.py`:
   `append_agent_output_variables_section` (:37-83), `_append_flat_variables` (:126-140), `_append_attributed_variables`
   (:143-161). Colors `_COLOR_KEY = "bold #87D7FF"`, `_COLOR_VALUE = "#5FD75F"` (:26-27); multi-line strings already
   render as `key:` followed by 2-space-indented lines. Wired in at
   `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py:185`.
2. **ACE clan / tribe panels** — `ClanVariableEntry` (`src/sase/ace/tui/models/_agent_clan_sections.py:87-93`, frozen
   slots dataclass with `value: str`), `aggregate_agent_output_variables` (:337-354); rendered by
   `_agent_display_clan_sections.py::append_variables_section` (:80-116) and
   `_agent_display_tribe.py::_append_variables` (:334-372) at three fold levels (COLLAPSED count / EXPANDED one-line
   `member.name = <first meaningful line>` / EXHAUSTIVE full body). Clan wiring at `_agent_display_clan.py:229-240`,
   tribe at `_agent_display_tribe.py:650-654`, tribe aggregation at `agent_tribe_summary.py:509`.
3. **ACE loaders / state** — `Agent.output_variables: dict[str, str]` (`src/sase/ace/tui/models/_agent_state.py:322`),
   filled from the Rust wire (`_loaders/_meta_enrichment_wire.py:80`) or straight from `agent_meta.json`
   (`_loaders/_meta_enrichment_filesystem.py:110-111` via `_meta_enrichment_common.string_output_variables` :158-165,
   which drops non-strings).
4. **Agents sidecar (published git repo)** — `src/sase/agents_sync/rendering_variables.py`: `output_variables` (:15-27)
   keeps only `str` values, `render_agent_variables` (:30-63) and `render_family_variables` (:66-101) emit markdown
   tables with `_DISPLAY_VALUE_LIMIT = 200` (:12) and a meta.json link when truncated. Publication validation
   `src/sase/agents_sync/v2_validation.py::validate_output_variables` (:95-124) **rejects** non-string values; the
   portable-metadata sanitizer `src/sase/agents_sync/inventory_io.py::_portable_output_variables` (:49-66) **drops**
   them.
5. **Completion notification + Telegram** — `src/sase/axe/run_agent_runner_finalize.py::_completion_output_variables`
   (:191-205, filters `STOP`) is JSON-encoded into `action_data["output_variables"]` (:314-325). The integrations
   projection carries them too (`src/sase/integrations/_agent_list_entry_models.py:129`, built at
   `_agent_list_entry_builder.py:156`). In the **sase-telegram linked repo** (which depends on `sase>=0.1.0` and imports
   `sase.*` freely): `src/sase_telegram/formatting.py::_format_output_variables_section` (:1441-1485, with
   `OUTPUT_VARIABLE_VALUE_MAX = 300` :67 and `OUTPUT_VARIABLES_MAX_DISPLAYED = 20` :70), called at :1571-1573; and
   `src/sase_telegram/agent_format.py:270-276` renders a one-line `Outputs` detail row.

**Rust core (sase-core linked repo, currently at v0.15.0; this repo pins `sase-core-rs>=0.14.2,<0.15.0`)**

- `crates/sase_core/src/agent_scan/wire.rs:176-182` already defines
  `#[serde(untagged)] enum OutputVariableValue { Text(String), List(Vec<String>) }`, used at :250; scanner coercion
  `coerce_output_variable_map` (`crates/sase_core/src/agent_scan/scanner.rs:866-898`) is called at :992-995 and accepts
  only strings and all-string arrays. Re-exports at `src/lib.rs:183` and `src/agent_scan/mod.rs:46`; parity cases at
  `crates/sase_core/tests/agent_scan_parity.rs:1077-1092`. `serde_json::{Map, Value}` is already used in this wire
  module (:148, :272, :390). **This repo has not consumed that release**, so today lists are dropped end to end.

**In-flight work this epic supersedes**

Epic bead `sase-be` ("Vars-driven commit finalization with exclusion-based staging",
`plans:202607/commit_vars_finalizer.md`) contains two variable phases: `sase-be.1` (`list-vars-rust`) is **closed** and
is exactly the v0.15.0 Rust change above; `sase-be.2` (`list-vars-python`) is open with no work landed and no agent
running. See design decision 9.

## Design decisions

1. **A sase variable is a JSON value.**
   `VarValue = str | int | float | bool | None | list[VarValue] | dict[str, VarValue]`. One sentence defines the whole
   model, it matches what `--json` input naturally produces, and it round-trips through `agent_meta.json`, the scan
   wire, the notification payload, and the sidecar with no encoding layer. Coercing numbers/booleans to strings would be
   silently lossy (`42` vs `"42"`, `3.0` vs `"3.0"`); rejecting them would make JSON input hostile. Strings remain the
   default and behave exactly as they do today.
2. **Variable names keep today's rule; nested map keys are ordinary strings.** Top-level names stay
   `[A-Za-z_][A-Za-z0-9_]*` because they are Jinja attributes under `agents["producer"]`. Nested map keys may be any
   non-empty NUL-free UTF-8 string within the per-string cap, so paths, hyphenated names, and human labels work; Jinja
   reaches them with `["..."]` subscripts.
3. **Map keys are sorted; list order is preserved.** Sorting is already what storage does
   (`json.dump(..., sort_keys=True)`) and what the Rust `BTreeMap`/`serde_json::Map` default gives, it keeps the
   published agents sidecar diff-stable across runs, and it makes every surface agree. List order is semantic and is
   never reordered. This is a documented contract, not an implementation detail.
4. **Reliability caps are structural, not just byte-based.** Keep `MAX_OUTPUT_VARIABLES = 256` and
   `MAX_OUTPUT_VARIABLE_VALUE_BYTES = 8_192` (now: per _string leaf_, so no existing value changes validity). Add
   `MAX_OUTPUT_VARIABLE_DEPTH = 8` (nesting), `MAX_OUTPUT_VARIABLE_NODES = 1_024` (total containers + leaves per
   variable, which subsumes any per-list length cap), and `MAX_OUTPUT_VARIABLE_ENCODED_BYTES = 65_536` (compact JSON of
   one whole variable). Reject `NaN`/`Infinity` and integers outside the signed 64-bit range so Python, `serde_json`,
   and `json.dumps(allow_nan=False)` in the sidecar all agree on what is storable.
5. **`--json` is a modifier, not a new input path.** `sase var set` gains one flag, `-j/--json`, meaning "the value(s)
   are JSON-encoded". It composes with all three existing forms — `sase var set 'tags=["a","b"]' -j`,
   `sase var set cfg -j -v '{"retries":3}'`, `sase var set report -j -f report.json`, `... -j -f -` for heredocs and
   pipes — instead of adding `--json-value`/`--json-file` twins. Without `-j`, nothing about today's parsing changes.
   `-v/--value` deliberately stays non-repeatable: "one occurrence is a string, two are a list" is exactly the kind of
   ambiguity this model exists to remove.
6. **One canonical renderer, two forms, every surface.** A new `src/sase/core/output_variable_display.py` is the only
   place that decides how a variable looks:
   - **Block form** — a YAML-shaped tree returned as structured `VarLine` records (indent, optional key, optional list
     bullet, text, and a `kind` tag) so ACE and the CLI can color per kind while plain-text surfaces just join them.
     Strings render bare and are quoted only when ambiguous (empty, leading/trailing whitespace, or parseable as a
     number/bool/null, or opening with a YAML indicator); multi-line strings use a `key: |` block scalar with indented
     lines, which is what ACE already does today; empty containers render `[]` / `{}`.
   - **Inline form** — a single-line preview: `{duration_s: 42.5, passed: true, suites: [unit, integration]}`, truncated
     with `…` to a caller-supplied budget. This is what one-line contexts use (ACE EXPANDED triage rows, sidecar table
     cells, the Telegram `Outputs` row).

   Rendering must stay a pure function of already-loaded in-memory values with no I/O and no re-parsing, per the TUI
   performance rules (render paths never stat, glob, or parse JSON).

7. **The shared renderer lives in Python `sase.core`, and that is deliberate.** The Rust-core boundary litmus flags
   cross-frontend display logic, but output-variable storage, validation, and rendering already live in
   `src/sase/core/*.py` — Rust only _projects_ the scan wire — and the one other frontend, the sase-telegram plugin,
   depends on `sase` and imports `sase.*` directly, so a single Python module yields exactly one implementation with no
   duplication. Moving the model into Rust mid-epic would fork the source of truth. Migrating storage + rendering into
   `sase_core` behind the binding is recorded as follow-up work, not part of this epic.
8. **Containers reach Jinja as real containers that stringify as JSON.** `{% for h in agents["build"].cfg.hosts %}` and
   `{{ agents["build"].cfg.retries }}` work with no filter. To keep `{{ agents["build"].cfg }}` from emitting a Python
   repr (`{'a': 1}`) in a prompt, the `agents` context wraps containers in thin `dict`/`list` subclasses whose `__str__`
   is compact JSON. `| tojson` keeps working for explicit control.
9. **This epic supersedes `sase-be.2` and unblocks `sase-be.4`.** `sase-be.1` (already released as sase-core v0.15.0) is
   generalized here by `core-wire-json`; `sase-be.2` (`list-vars-python`, no work landed) is entirely contained in
   `var-value-model` + `var-cli-jinja` at a strictly wider scope and should be closed as superseded once those two land.
   `sase-be.4` (`sase commit --vars`) then records `commit_exclude_files` as a real list through
   `set_agent_output_variables`. Closing or re-scoping `sase-be.2` is the user's call; this plan does not do it.
10. **Unlit surfaces degrade to invisible, never to broken.** Every reader keeps its existing "drop what I do not
    understand" behavior until its phase lands, so a structured value set between phases simply does not appear on a
    not-yet-updated surface. The same property covers version skew between `sase-core-rs` and this repo.

### Worked example of the canonical block form

```
report_path: dist/report.md
status: ok
metrics:
  duration_s: 42.5
  passed: true
  suites:
    - unit
    - integration
findings:
  - file: src/a.py
    severity: high
  - file: src/b.py
    severity: low
notes: |
  first line
  second line
empty_list: []
version: "3"
```

`version` is quoted because a bare `3` would read as a number; `notes` uses a block scalar; map keys are sorted; list
order is untouched.

## Phase details

### var-value-model — canonical model, storage, and renderers

1. New `src/sase/core/output_variable_values.py`:
   - `VarValue` type alias, `MAX_OUTPUT_VARIABLE_DEPTH = 8`, `MAX_OUTPUT_VARIABLE_NODES = 1_024`,
     `MAX_OUTPUT_VARIABLE_ENCODED_BYTES = 65_536` (keep the two existing caps where they are and re-export for
     compatibility).
   - `normalize_var_value(key, value) -> VarValue` — strict writer-side validation raising `ValueError` with an
     actionable message naming the offending JSON path (e.g. `findings[1].severity`): CRLF→LF and NUL rejection and the
     8 KiB cap per _string leaf_ (including nested map keys), depth/node/encoded-byte caps, `NaN`/`Infinity` and
     out-of-int64 rejection, map keys must be non-empty strings. **Gotcha: `bool` is a subclass of `int` — check `bool`
     before `int` everywhere.**
   - `coerce_var_value(value) -> VarValue | None` — tolerant reader for data already on disk: returns `None` for
     anything unsupported so readers keep dropping rather than raising.
   - `coerce_var_map(raw) -> dict[str, VarValue]` — the structured replacement for `_string_output_variables`.
   - `encode_var_value(value) -> str` / `decode_var_value(text) -> VarValue` — compact, sorted-key JSON used for
     canonical comparison, hashing, and the notification payload.
2. New `src/sase/core/output_variable_display.py` implementing design decision 6: `VarLine` (frozen slots dataclass:
   `indent: int`, `bullet: bool`, `key: str | None`, `text: str | None`,
   `kind: Literal["string","number","boolean", "null","list","map","block"]`),
   `format_var_value_lines(value, *, max_lines=None) -> tuple[list[VarLine], bool]` (the bool reports truncation),
   `format_var_value_block(value, *, indent="  ", max_lines=None) -> tuple[str, bool]`,
   `format_var_value_inline(value, *, max_chars) -> str`, `var_value_is_container(value) -> bool`, and
   `var_value_preview(value, *, max_chars)` (first meaningful line for scalars, inline form for containers) so the
   existing `first_meaningful_line` triage callers have a drop-in.
3. `src/sase/core/agent_output_variables.py`: `set_agent_output_variables` and `read_agent_output_variables` become
   `Mapping[str, VarValue]` / `dict[str, VarValue]`, delegating validation to `normalize_var_value` and reading through
   `coerce_var_map`; `parse_output_variable_assignments` keeps producing strings (the `-j` path in `var-cli-jinja`
   parses first, then normalizes). Storage format is unchanged — still `agent_meta.json["output_variables"]`, still
   `flock` + atomic sorted-key write.
4. Type-widening only (no behavior change) for direct callers so `just lint` stays green and nothing regresses:
   `src/sase/agent/output_variable_context.py` (`dict[str, VarValue]` through `_merge_agent_variables`,
   `read_waited_agent_output_variables`, `_variables_with_plan_file` — note `plan_file` stays a string),
   `src/sase/axe/run_agent_runner_finalize.py::_completion_output_variables`,
   `src/sase/axe/run_agent_runner_repeat.py:49`, `src/sase/integrations/_agent_list_entry_models.py:129` and
   `_agent_list_entry_builder.py:156`.
5. Tests: new `tests/core/test_output_variable_values.py` (round-trip of every JSON shape; each cap boundary with its
   error message; `bool`-before-`int`; NUL/CRLF in nested leaves; oversized nested key; int64 boundary; NaN rejection)
   and `tests/core/test_output_variable_display.py` (block form for the worked example above, quoting rules, block
   scalars, empty containers, list-of-maps indentation, inline truncation, `max_lines` truncation flag). Extend
   `tests/main/test_var_handler.py` with structured round-trips through the storage API.
6. Verification: `just install`, targeted pytest, `just check`.

### core-wire-json — full JSON values in the scan wire

Work happens in the **sase-core linked repo**; open it with `/sase_repo` (`sase repo open sase-core -r "<reason>"`) and
use only the printed path.

1. `crates/sase_core/src/agent_scan/wire.rs:176-182`: replace the `Text | List` enum with `serde_json::Value` (the
   module already imports `Map`/`Value`), so `output_variables: BTreeMap<String, Value>` at :250. Keep the public name
   `OutputVariableValue` as a type alias if it keeps the re-exports at `src/lib.rs:183` and `src/agent_scan/mod.rs:46`
   stable; otherwise update both.
2. `scanner.rs::coerce_output_variable_map` (:866-898): accept any JSON value that satisfies the structural caps from
   `var-value-model` (depth ≤ 8, ≤ 1024 nodes, encoded size ≤ 64 KiB); drop entries that exceed them. The scanner stays
   soft-failing — it never errors on malformed marker JSON. Leave `coerce_str_str_map` (:846-864) alone; it serves
   `output_types`.
3. Confirm `serde_json` is not built with `preserve_order`, so nested object keys stay sorted and match the Python
   writer's `sort_keys=True` (design decision 3). Confirm `crates/sase_core_py` needs no change (snapshots cross the
   boundary as JSON-shaped dicts); refresh any doc comment that spells out the `output_variables` shape.
4. Parity tests (`crates/sase_core/tests/agent_scan_parity.rs:1077-1092`): keep the existing string and string-list
   cases, add nested map, list-of-maps, mixed-type list, numbers/bools/null, and drop-on-cap-violation cases.
5. Release through the repo's normal release-plz flow (versions are release-plz-owned; do not hand-edit version fields).
   Verify with `cargo fmt --all -- --check`, `cargo test --workspace`, and
   `cargo clippy --workspace --all-targets -- -D warnings`.
6. Back in this repo: bump `pyproject.toml:46` to the released version (follow commit 5a8dc1cba for the pin-bump
   pattern; note the current pin is `<0.15.0` and sase-core is already at 0.15.0, so this moves two minors) and widen
   `src/sase/core/agent_scan_wire_markers.py:138` to `dict[str, Any]`-shaped structured values. Extend
   `tests/test_core_agent_scan_wire.py` with nested wire cases.
7. Verification: sase-core's own cargo flow for the Rust change; `just install` + targeted pytest + `just check` here.

### var-cli-jinja — authoring, Jinja, STOP, skill, docs

1. `src/sase/main/parser_var.py`: add `-j/--json` to the `set` parser (keep options alphabetized: `--json`, `--value`,
   `--value-file`), with help text and epilog examples covering all three composed forms plus a heredoc. Add a `list`
   subcommand (`sase var list`, with `-j/--json` for machine output) — per the CLI-rules default-`list` convention,
   `sase var` then delegates to it with the standard notice, replacing the current `Usage: sase var {set}` error path
   (`var_handler.py:18-25`).
2. `src/sase/main/var_handler.py`: `_output_variables_from_args` parses each value with `json.loads` when `--json` is
   given (clear error naming the form that failed to parse), then normalizes through `normalize_var_value`. The "exactly
   one bare KEY for the value-source form" rule is unchanged; the trailing-newline strip applies only to the non-JSON
   `--value-file` path. `sase var list` reads the current agent's variables and prints the canonical block form with
   per-kind color (keys `bold #87D7FF`, strings `#5FD75F`, numbers `#FFAF5F`, booleans/null `italic #AFAFAF`, list
   bullets dim), or compact JSON under `--json`.
3. `src/sase/agent/output_variable_context.py`: wrap containers in thin `dict`/`list` subclasses whose `__str__` is
   compact JSON (design decision 8), applied where variables enter the `agents` mapping; scalars pass through untouched.
   Keep the wrapper local to this module and confirm `StrictUndefined` attribute/subscript access, iteration, and
   `| tojson` all behave.
4. `src/sase/axe/run_agent_repeat_stop.py`: generalize `_is_stop_value` to `VarValue` — `None`, `False`, `0`, `0.0`,
   `""`, `[]`, `{}` and the existing case-insensitive string set `{"", "0", "false", "no", "off"}` are not-stop;
   anything else stops. `RepeatStopDecision.stop_value` widens to `VarValue` and still propagates verbatim through
   `run_agent_runner_repeat.py:49`.
5. Skill source `src/sase/xprompts/skills/sase_var.md`: document the JSON value model, `-j` on all three forms, the
   sorted-map-key / preserved-list-order contract, the caps, iterating containers in Jinja, and the generalized `STOP`
   truthiness. Iterate with `sase skill init --diff`/`--dry-run` only; do not deploy from this tree.
6. Docs: `docs/configuration.md` `sase var` section (:3117-3155 — forms table gains `--json`, add `sase var list`,
   restate limits and ordering, update the `STOP` paragraph) and `docs/xprompt.md` "Cross-Agent Output Variables"
   (:2010-2050 — a structured example with iteration).
7. Tests: extend `tests/main/test_var_handler.py` (each `-j` form, invalid JSON, cap violations surfacing as exit-1
   errors, `sase var list` text and `--json` output, bare-`sase var` delegation notice), CLI parser tests for
   alphabetized help, `tests/test_agent_output_variable_context.py` (containers reach Jinja, stringify as JSON,
   iterate), `tests/test_run_agent_repeat_stop.py` (structured truthiness matrix), and
   `tests/main/test_init_skills_sources.py` if it pins `sase_var` phrases.
8. Verification: `just install`, targeted pytest, `just check`.

### ace-var-display — ACE agent, clan, and tribe panels

1. Loaders/state: `src/sase/ace/tui/models/_agent_state.py:322` widens to `dict[str, VarValue]`;
   `_loaders/_meta_enrichment_common.py::string_output_variables` (:158-165) is replaced by the structured
   `coerce_var_map` reader (keep a thin alias if other callers exist); `_meta_enrichment_wire.py:80` and
   `_meta_enrichment_filesystem.py:110-111` pass structured values through.
2. Agent panel (`_agent_output_variables.py`): `_append_flat_variables` (:126-140) and `_append_attributed_variables`
   (:143-161) render through `format_var_value_lines`, keeping today's exact output for single-line strings (`key: `
   then value) and today's indent for multi-line strings, and adding per-kind styles for the new kinds. The attributed
   form keeps the role column and its continuation prefix at every nesting depth.
3. Clan/tribe: `ClanVariableEntry.value` (`_agent_clan_sections.py:87-93`) widens to `VarValue`. **Audit hashing first**
   — the dataclass is `frozen=True, slots=True`, so it has a generated `__hash__` that a `dict`/`list` value would
   break; if any call path hashes entries or their snapshots (`ClanInMemorySnapshot`, `TribeSectionSnapshot`, any cache
   key), define `__hash__` over `(member_identity, name, encode_var_value(value))` rather than dropping frozen-ness.
   `append_variables_section` (`_agent_display_clan_sections.py:80-116`) and `_append_variables`
   (`_agent_display_tribe.py:334-372`) use `var_value_preview` for EXPANDED triage rows and `format_var_value_lines` for
   EXHAUSTIVE bodies; COLLAPSED counts are unchanged.
4. Performance: renderers touch only in-memory values, do no I/O, and are bounded by the node caps, so the detail-panel
   debounce and existing caches are sufficient — do not add new refresh paths. Spot-check with
   `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` (which already exercises `output_variables`) and confirm no p95
   regression.
5. Tests: extend `tests/ace/tui/widgets/test_agent_display_output_variables.py` (nested map, list of maps, mixed
   scalars, empty containers, attributed/family form, unchanged rendering for plain strings),
   `tests/ace/tui/widgets/test_agent_display_tribe.py`, and clan-section/aggregation tests including the hashing audit
   result.
6. Verification: `just install`, targeted pytest, `just check`.

### sidecar-var-publication — agents sidecar

1. `src/sase/agents_sync/v2_validation.py::validate_output_variables` (:95-124): accept structured values, validating
   with the same caps as the writer (per-string 8 KiB, depth, nodes, encoded bytes, 256 keys) and raising
   `AgentsSyncFormatError` with the offending JSON path.
2. `src/sase/agents_sync/inventory_io.py::_portable_output_variables` (:49-66): sanitize recursively via
   `coerce_var_value`, keeping the existing "drop what does not fit" contract and the `json.dumps(allow_nan=False)`
   guarantee in `portable_metadata` (:29-46).
3. `src/sase/agents_sync/rendering_variables.py`: `output_variables` (:15-27) stops filtering to `str`;
   `render_agent_variables` (:30-63) and `render_family_variables` (:66-101) keep one table row per variable using
   `format_var_value_inline` capped at `_DISPLAY_VALUE_LIMIT` (:12), and gain a `#### <key>` + fenced block section
   below the table for every container-valued variable, rendered with `format_var_value_block` under a line/byte budget.
   The existing "values are truncated; see meta.json" link covers both. Output must stay byte-deterministic — this repo
   is published and diffed.
4. Docs: `docs/agents_sidecar.md` (:109, :128-139) — structured values, ordering contract, and the reader-version
   compatibility note.
5. Tests: `tests/agents_sync/` validation and sanitizer suites (structured accept/reject matrix), plus rendering tests
   for the table preview, the fenced blocks, family attribution, and determinism across repeated renders.
6. Verification: `just install`, targeted pytest, `just check`.

### notify-var-display — completion notifications and Telegram

1. This repo: `src/sase/axe/run_agent_runner_finalize.py::_completion_output_variables` (:191-205) returns structured
   values; the payload encoding at :314-325 keeps `json.dumps(..., ensure_ascii=False, sort_keys=True)` so `action_data`
   stays `dict[str, str]` and no notification wire change is needed. Confirm the payload stays within whatever size
   limits the notification store enforces, given the 64 KiB per-variable cap. `docs/notifications.md:150` and
   `docs/integrations.md:101` get a sentence each.
2. sase-telegram linked repo (open with `/sase_repo`):
   `src/sase_telegram/formatting.py::_format_output_variables_ section` (:1441-1485) stops coercing with `str(value)`
   and instead renders scalars exactly as today and containers with `format_var_value_block` inside a
   ```fence (the function already uses fences for multi-line strings), keeping`OUTPUT_VARIABLES_MAX_DISPLAYED =
   20`and converting`OUTPUT_VARIABLE_VALUE_MAX =
   300`into a line-aware budget with the existing`…`marker;`escape_markdown_v2`/`_escape_code_entity`handling is unchanged.`src/sase_telegram/agent_format.py:270-276`renders its one-line`Outputs`row with`format_var_value_inline`. Because the plugin already depends on `sase`and imports`sase.\*`,
   it imports the canonical renderer directly — do not reimplement it there.
3. Tests: in this repo, notification-payload tests for structured snapshots and `STOP` filtering; in the plugin repo,
   `tests/test_formatting.py` and `tests/test_show_format.py` for scalar-unchanged, nested-container, empty-container,
   and truncation cases. Run the plugin repo's own lint/test flow there.
4. Verification: `just install`, targeted pytest, `just check` in this repo; the plugin repo's own checks for its
   changes.

## Cross-cutting rules for every phase

- This repo's phases must run `just install` before any other `just` command (ephemeral workspaces) and `just check`
  before finishing. The sase-core phase uses that repo's cargo flow; the sase-telegram phase uses that repo's flow.
- Any repo other than this workspace checkout — sase-core, sase-telegram — is opened only through the `/sase_repo`
  skill, and only the printed path is used for reads and writes.
- New CLI options follow the CLI rules: excellent `-h` output, alphabetized subcommands and options, a short alias for
  every public long option, and colored output where color helps.
- `src/sase/xprompts/skills/*.md` are generated-skill _sources_: iterate with `sase skill init --diff`/`--dry-run` only,
  and never deploy from a dirty or unlanded tree.
- No edits to `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction shims are in scope.
- User-facing text shows project names, never ProjectSpec keys.
- Every phase keeps the "drop what you do not understand" reader behavior so partially-landed phases and `sase-core-rs`
  version skew degrade to invisible, never to broken.

## Risks and notes

- **`bool` is an `int` in Python.** Every isinstance chain in validation, coercion, and rendering must test `bool` first
  or `True` will render as `1` and pass integer caps. Pin this with a test in `var-value-model`.
- **Hashability.** `ClanVariableEntry` is a frozen slots dataclass whose generated `__hash__` breaks the moment a value
  is a `dict`/`list`. The `ace-var-display` phase must audit for hashing (sets, dict keys, cache keys, snapshot
  equality) before widening the field, and use `encode_var_value` for any hash that is needed.
- **Published-artifact determinism.** The agents sidecar is a git repo; sorted map keys and a stable renderer are what
  keep its diffs meaningful. Any nondeterminism here shows up as churn in a shared repo.
- **Two-minor pin jump.** This repo pins `sase-core-rs<0.15.0` while sase-core is already at 0.15.0, so `core-wire-json`
  moves the pin across the already-released `Text|List` breaking change and the new JSON-value one in a single step; the
  parity and wire tests in that phase are the guard.
- **Telegram message limits.** Structured values can be much taller than scalars; the line-aware budget plus the
  existing 20-variable cap must keep completion messages under Telegram's message size limit, with truncation visible
  rather than silent.
- **Secrets.** The existing "output variables are visible metadata, not secret storage" warning matters more once maps
  make it natural to dump whole config objects; keep that warning prominent in the skill and docs.
- **Deliberate non-goals.** Per-key merge (a set that deep-merges into an existing map), path-addressed writes
  (`sase var set cfg.retries=3`), a `sase var get` consumption command, and moving the model into `sase_core` behind the
  Rust binding are all out of scope and are the natural follow-ups.
