---
create_time: 2026-07-10 11:21:09
status: done
prompt: 202607/prompts/remove_wait_time_shape_validation.md
tier: tale
---
# Plan: Treat Every Positional `%wait` Value as an Agent Dependency

## Goal

Fix agent launches such as:

```text
#gh:sase %w:4h #fork:4f ...
```

Today this fails before the agent is named or started because directive extraction decides that `4h` looks like a
duration and rejects `%w:4h` with a migration hint. That contradicts the current directive contract:

- positional `%wait:<value>` / `%w:<value>` values are agent or workflow dependencies;
- `%wait(time=<value>)` and `#t:<value>` are explicit time waits.

After this change, positional wait values will never be interpreted or validated as times. `%w:4h` will wait for the
agent named `4h`, while `%wait(time=4h)` and `#t:4h` will retain their existing four-hour meaning.

## Root Cause and Design Decision

`resolve_wait_agent_args()` in `src/sase/xprompt/_directive_values.py` currently sends every non-empty positional wait
argument through two time parsers before accepting it as an agent name:

- `is_canonical_duration()` rejects values such as `4h`, `5m`, and `30s`;
- `parse_absolute_time()` rejects or time-validates values such as `1430` and date/time-shaped strings.

This validation was added as a hard-cutover migration aid when time waits moved out of positional `%wait` syntax. A
later fix allowed leading-zero names such as `05s`, but explicitly left canonical duration-shaped names unresolved. That
residual limitation is now observable because SASE's base-36 auto-name sequence naturally generates names such as `4h`.

The parser cannot correctly infer intent from the value shape:

- duration- and clock-shaped strings are valid agent names;
- a dependency is allowed to name an agent that has not launched yet, so checking the current registry would reject
  valid future dependencies;
- adding quoting or an escape hatch would preserve needless ambiguity when the directive already has separate positional
  and `time=` channels.

Therefore the durable rule should be syntax-based, not value-based: positional values are dependency names, and only the
explicit `time=` keyword or `#t` shorthand invokes time parsing.

## Implementation

### 1. Remove positional time-shape validation

Update `src/sase/xprompt/_directive_values.py` so `resolve_wait_agent_args()` continues to resolve bare `%wait` to the
most recent agent, but appends every non-empty positional argument directly to `directives.wait` without calling a
duration or absolute-time parser.

Keep `resolve_wait_time_args()` unchanged. It remains the sole validation path for `%wait(time=...)`, including duration
normalization, absolute-time validation, conflicting time waits, and helpful errors for non-time keyword values.

Remove imports that become unnecessary in the positional path. If `is_canonical_duration()` has no production consumer
after this change, remove that workaround-specific helper from `src/sase/xprompt/_directive_time.py`, its facade export
from `src/sase/xprompt/directives.py`, and its isolated unit tests. Keep `parse_duration()` and `parse_absolute_time()`
as the time-only parsing API.

### 2. Replace migration-guard tests with semantic regression tests

Update the duplicated positional-wait coverage in:

- `tests/test_directives_wait.py`
- `tests/test_directives_name_wait.py`

Replace the tests that expect migration errors for `%wait:5m`, `%wait:1430`, and `%wait:300415/0900` with assertions
that time-shaped positional values are preserved in `directives.wait` and do not populate `wait_duration` or
`wait_until`. Include `%w:4h` as the primary regression for the reported failure, and cover both duration-shaped and
clock-shaped values so neither removed branch can return later.

Retain explicit-time coverage showing that `%wait(time=4h)` / `#t:4h` still produce a duration and that
`%wait(time=1430)` still produces an absolute-time wait. If the canonical-duration helper is removed, delete only its
helper-level cases from `tests/test_directives_time_parsing.py`; all tests for the actual explicit time parsers remain.

Add or extend a runner/extraction integration test (prefer `tests/test_agent_names_extract.py` or the nearest existing
launch metadata suite) to prove a prompt containing `%w:4h` reaches launch metadata with `wait_for == ["4h"]`. This
guards the user-visible launch path, rather than relying exclusively on parser unit tests.

### 3. Align user documentation with the unambiguous syntax contract

Update `docs/xprompt.md` to remove the paragraph describing positional time-shaped values as migration errors and the
canonical/non-canonical distinction. State directly that every positional `%wait` value is an agent/workflow name and
that timed waits must use `%wait(time=...)` or `#t`.

Keep the old `%time:<value>` removal note because that is a separate, still-valid migration rule. No generated memory or
provider instruction files should be changed.

## Scope and Compatibility

- No persisted metadata or scheduler change is required: accepted positional values already flow through
  `PromptDirectives.wait`, `AgentInfo.wait_names`, and `agent_meta.json`'s `wait_for` field.
- No change is needed in `sase-core`; the failing value-shape check is in the Python xprompt parser, while core-backed
  name generation merely makes the collision likely.
- Existing prompts using the supported timed forms are unchanged.
- A stale pre-migration `%wait:5m` prompt will now mean “wait for agent `5m`,” not “wait five minutes.” This follows the
  documented positional contract and is the necessary tradeoff for supporting valid and future agent names without
  state-dependent parsing.
- Do not add registry-aware validation, special quoting, or auto-name exclusions; those approaches either remain
  semantically incorrect or spread the ambiguity into other layers.

## Verification

1. Run focused directive and launch tests covering positional waits, explicit time waits, time parsing, and launch
   metadata (including the exact `%w:4h` regression).
2. Run `just install` because this is an ephemeral workspace.
3. Run the repository-required `just check` and resolve any lint, typing, or test failures.
4. Confirm the documentation sweep contains no remaining claim that positional duration/clock-shaped `%wait` values are
   time waits or migration errors.
