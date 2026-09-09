---
tier: epic
title: Powerful SASE variable discovery and retrieval
goal:
  Make SASE output variables easy to inspect, search, aggregate, and retrieve across
  agent history without weakening their existing reliability guarantees.
phases:
  - id: core-variable-index
    title: Add an indexed output-variable query contract to sase-core
    depends_on: []
    size: medium
    description:
      "core-variable-index: project stored output variables into the agent artifact
      index and expose typed, deterministic history queries through Rust and PyO3."
  - id: show-and-list-cli
    title: Replace current-agent list with show and build historical list
    depends_on:
      - core-variable-index
    size: medium
    description:
      "show-and-list-cli: add the show command, redesign list around the core history
      query, and provide rich and machine-readable output with filtering and limits."
  - id: selector-get-cli
    title: Add the variable selector language and get command
    depends_on:
      - show-and-list-cli
    size: medium
    description:
      "selector-get-cli: implement exact, global, hood, key-wildcard, and JSON-path
      selectors with predictable output for humans, shells, and agents."
  - id: skill-and-integration
    title: Synchronize the sase_var skill and verify the complete workflow
    depends_on:
      - selector-get-cli
    size: small
    description:
      "skill-and-integration: teach agents only the new show replacement, exercise
      end-to-end behavior, and run repository-wide verification without exposing
      historical discovery in the skill yet."
proposed_by: bbugyi200.athena.02u
bead_id: sase-mg
create_time: 2026-09-09 19:51:08
status: wip
---

- **PROMPT:**
  [prompts/202608/powerful_variables.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/powerful_variables.md)
- **BEAD:**
  [sase-mg](https://github.com/sase-org/sase--beads/blob/main/pages/sase-mg/README.md)

# Powerful SASE variables

## Outcome

Keep `agent_meta.json::output_variables` as the source of truth, but make those values
discoverable through four clear commands:

- `sase var set` continues to attach bounded JSON-shaped values to the current run.
- `sase var show [AGENT_NAME]` owns the old current-agent inspection behavior.
- bare `sase var` continues to delegate to `sase var list`, whose new job is historical
  discovery and aggregation.
- `sase var get SELECTOR [SELECTOR ...]` retrieves precise values using a compact
  selector language.

The new history describes the final stored variable snapshot in every indexed agent
artifact. A later `set` of the same key in one run still replaces that run's earlier
value; this feature does not introduce an append-only mutation/event log. Across runs,
distinct values are identified by canonical JSON, so `1`, `1.0`, and `"1"` retain their
types and do not collapse accidentally.

## Command contracts

### `sase var show [AGENT_NAME]`

With no positional argument, read the current `SASE_ARTIFACTS_DIR` directly so a running
agent sees its own newest writes even if the persistent index has not refreshed yet.
Resolve the display identity from artifact metadata and `SASE_AGENT_NAME` using the
existing identity helpers. `SASE_AGENT` is currently the launcher sentinel `1`, not an
agent name; accept a non-sentinel `SASE_AGENT` only as a compatibility fallback when
neither canonical source is available. This preserves the user's intended no-argument
workflow without mistaking `1` for an agent.

With `AGENT_NAME`, resolve the newest visible exact-name artifact deterministically;
`--project` can narrow otherwise repeated historical names. An unknown agent is an
error, while a known agent with no variables succeeds with a clear empty-state message.
The default pretty form retains the existing canonical YAML-shaped variable block;
`-f/--format json` emits the compact, sorted variable map. Add `-c/--color` and
`-p/--project`, with every public long option carrying a short alias and help/options
remaining alphabetized.

### `sase var list`

List unique variable keys, most-recently-seen first. Under each key, list its distinct
typed values, also most-recently-seen first, with every distinct agent name that
contributed that value in the selected history. Repeated runs of one named agent count
as occurrences but the associated agent-name list is deduplicated. Show occurrence,
distinct-value, and agent counts plus explicit truncation markers so a limit never
silently looks complete.

Support these options:

- `-a/--agent GLOB`, repeatable, including SASE hood-aware `hood.*` matching (the hood
  root and every `hood.`-prefixed member).
- `-c/--color {auto,always,never}`.
- `-f/--format {pretty,json,jsonl}`, defaulting to a colored, grouped `pretty` view.
- `-H/--hidden` to include hidden indexed agents; visible history is the default.
- `-k/--key GLOB`, repeatable and case-sensitive.
- `-n/--limit KEYS[:VALUES]`, default `20:5`; zero means unlimited for that dimension,
  and a single number changes only the key limit while retaining the default value
  limit.
- `-p/--project PROJECT`, repeatable and resolved through project display names/aliases
  rather than exposing ProjectSpec keys.
- `-r/--reverse` to invert the normal recent-first order deterministically.
- `-s/--since DATE` and `-u/--until DATE`, using the established local-time date grammar
  and inclusive day boundaries against agent launch time.
- `-v/--value TEXT`, repeatable literal case-insensitive substring matching over scalar
  text and compact canonical JSON.
- `-V/--value-json JSON`, repeatable exact typed matching after normalizing with the
  output-variable value rules; make it mutually exclusive with `--value`.

Repeated filters within one dimension are ORed; different dimensions are ANDed. JSON
uses a versioned envelope containing normalized filters, requested/effective limits,
total and returned counts, truncation state, grouped values, agents, projects, and
timestamps. JSONL emits one self-contained value group per line. Neither machine format
contains ANSI escapes.

### `sase var get SELECTOR [SELECTOR ...]`

Use this grammar, parsed from the right so dotted agent names stay unambiguous:

```text
[SCOPE.]KEY[PATH ...]

SCOPE := AGENT_NAME | * | HOOD.*
KEY   := [A-Za-z_][A-Za-z0-9_]* | *
PATH  := [NONNEGATIVE_INTEGER] | ["JSON map key"]
```

Examples and semantics:

- `status` returns the newest visible `status` occurrence globally.
- `results[0]` returns element zero from the newest global `results` value.
- `build.status` returns `status` from the newest exact `build` artifact.
- `research.foo.report["summary"]` treats `research.foo` as the dotted agent name and
  traverses the selected `report` map.
- `research.*.status` returns the newest `status` per named member of the `research`
  hood, including the root `research` agent when present.
- `*.status` returns the newest `status` per agent name across visible history.
- `build.*` returns every key from the newest exact `build` artifact.

An unscoped key chooses the newest matching occurrence. Exact-agent selectors choose the
newest artifact for that name. Global and hood selectors collapse older repeated runs to
the newest value per agent name and key. Multiple selectors preserve argument order,
then deduplicate identical resolved records. JSON-path traversal happens after the
occurrence is selected and reports type, missing-key, and out-of-range failures
precisely. No dot-style map traversal is accepted because it would conflict with dotted
agent names.

Support `-c/--color`, `-f/--format {pretty,raw,json,jsonl}`, `-H/--hidden`, `-n/--limit`
(default 20 for wildcard expansion; zero unlimited), and repeatable `-p/--project`.
`pretty` is the human default with source attribution. `raw` requires exactly one
resolved value and prints strings verbatim while emitting every other type as compact
canonical JSON, making command substitution safe. JSON returns a versioned envelope and
JSONL emits one attributed match per line. Invalid selectors are usage errors; valid
selectors with no match, ambiguous project scope, or failed traversal are clear nonzero
query errors rather than empty success.

## Architecture and reliability

Shared discovery, selector, grouping, ordering, and limit behavior belongs in
`sase-core`; Python owns argparse, current-process identity resolution, date-bound
normalization, and Rich/plain rendering.

Extend the persistent agent artifact index with a normalized `agent_output_variables`
projection keyed by `(artifact_dir, variable_key)`, containing canonical `value_json`
and joined/indexable agent, project, visibility, and artifact timestamp context.
Populate it in the same transaction as every artifact upsert, remove stale key rows when
a value map shrinks or an artifact disappears, and rebuild/backfill it during the index
schema bump. The artifact files remain authoritative; this table is only a regenerable
projection. Add indexes supporting recent key lookups and agent/key/time lookups, while
keeping writes under the existing artifact-index lock and busy-timeout policy.

Define Rust request/response wires for occurrences, grouped history, truncation
metadata, and selector matches. Export them from `sase_core`, expose one PyO3 query
entry point, mirror the additive wire models/schema version in Python, and keep
canonical JSON normalization aligned with existing variable caps. The query must use a
single operation clock/date normalization, stable tie-breakers (`timestamp`, project,
artifact path, key), bounded wildcard expansion, and tolerant handling of legacy rows
whose metadata has no usable agent name. Do not read every historical `agent_meta.json`
from the CLI or create a competing cache.

## Phase details

### 1. Indexed core contract (`core-variable-index`)

In `sase-core`, add the normalized projection table, schema migration/rebuild support,
upsert/delete synchronization, indexes, typed request/response wires, and a PyO3
binding. Implement canonical value identity, visibility/project/agent/key/value/time
filters, deterministic aggregation, independent key/value limits, and truncation counts.
Cover fresh indexes, upgrades with existing `record_json`, rebuilds, key
replacement/removal, artifact deletion, structured/unicode values, type-distinct values,
hidden rows, repeated agent names, date bounds, filters, and tie-breaking with Rust and
binding tests. Run the `sase-core` `just check` gate.

### 2. Show and historical list (`show-and-list-cli`)

Mirror the core wires and schema version in the Python facade, resolving index lifecycle
refresh before queries. Refactor the current handler so `show` owns direct/current and
explicit-agent display, while `list` builds the historical request. Add reusable limit
and filter parsers, excellent examples and alphabetized help, Rich rendering with
color-policy compliance, stable JSON/JSONL serializers, and explicit empty/truncated
states. Keep bare `sase var` delegated centrally to the new list. Test parser aliases,
invalid limits/JSON/date bounds, current identity fallbacks, direct-read freshness,
explicit-agent resolution, project display names, filtering combinations, output
schemas, color modes, Unicode/multiline/container values, and zero/unlimited limits.

### 3. Selector get (`selector-get-cli`)

Add the selector AST/parser and resolution semantics to the Rust domain layer, reusing
the indexed occurrences and SASE hood definition. Expose selector diagnostics and
matches through the binding/facade, then add the Python `get` handler and its four
formats. Test dotted/hyphenated/digit-leading names, hood root inclusion, global and key
wildcards, repeated runs, multiple-selector ordering/deduplication, nested list/map
paths and escaped map keys, missing/type/range failures, raw single-value enforcement,
limits, visibility/project filters, and stable machine envelopes. Include concise help
examples for human exploration and shell/agent consumption.

### 4. Skill synchronization and end-to-end verification (`skill-and-integration`)

Update only the canonical generated-skill source `src/sase/xprompts/skills/sase_var.md`:
replace current inspection examples with `sase var show` and
`sase var show --format json`. Do not teach the skill about the new historical `list` or
selector `get` commands yet. Update source-content tests and preview generation with
`sase skill init --diff`; do not deploy global generated skills from an unlanded tree.

Add end-to-end CLI coverage over a temporary multi-project artifact index, including an
old-schema upgrade, concurrent `set` refresh, structured values, history filters,
selectors, machine-output round trips, errors, and empty states. Reconcile command/help
documentation and any fixtures that assumed `list --json` meant the current agent. Run
`just install`, `just check`, and, because this changes the shared
artifact-index/binding surface, run `just check-full` through `/sase_monitor` with an
explicit result-handling `--next` action before landing the combined tree. Re-run
`sase-core`'s `just check` after the final binding contract is fixed.

## Non-goals

- Do not let one agent mutate another agent's variables.
- Do not add unset/delete commands or an event log of overwritten values.
- Do not expose `list`/`get` usage through `/sase_var` until a later explicit request.
- Do not hand-edit generated provider skill installations or memory files.
- Do not duplicate core query behavior in Python or the TUI.
