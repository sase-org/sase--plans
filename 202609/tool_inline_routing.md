---
tier: epic
title: Mechanical inline-vs-monitor routing for sase tool run
goal: "Every provider adapter that has a hard synchronous-command ceiling exports it as
  SASE_PROVIDER_SYNC_CEILING_SECONDS. Every tool catalog entry has a Rust-validated
  duration class (short, long, or unbounded). Before starting anything, sase tool run
  refuses an agent's inline run of a tool whose class floor meets that ceiling, and
  prints the exact monitor command to use instead. `check` keeps running inline, no
  existing definition digest moves, and sase-17e is closed.

  "
phases:
  - id: core-duration-class
    title: Rust duration class, inline fit, and calibration
    depends_on: []
    size: medium
    description:
      "core-duration-class: in sase-core, add the optional duration_class catalog field
      (validated, excluded from the definition digest), the class-floor table, and the
      fit and calibration functions with PyO3 bindings and tests."
  - id: provider-ceiling
    title: Provider adapters export their synchronous ceiling
    depends_on: []
    size: small
    description:
      "provider-ceiling: add an optional provider hook for the hard synchronous-command
      ceiling (Muse 600 s with the synchronous shell, Claude from BASH_MAX_TIMEOUT_MS),
      export it around every provider invocation, and scrub it at agent, monitor, and
      proc boundaries."
  - id: catalog-duration-class
    title: Pin the core, declare classes, and show them in sase tool list
    depends_on:
      - core-duration-class
    size: medium
    description:
      "catalog-duration-class: ratchet the sase-core pin, add the Python facades,
      declare check-full as long in sase/sase.yml using corpus evidence, and add a CLASS
      column plus calibration diagnostics to sase tool list, with docs."
  - id: ceiling-refusal
    title: sase tool run refuses inline runs that cannot fit
    depends_on:
      - provider-ceiling
      - catalog-duration-class
    size: medium
    description:
      "ceiling-refusal: before any reconcile, reservation, or spawn, refuse an agent's
      inline named-tool run whose class floor meets the exported ceiling (exit 2), print
      the monitor and prepared-completion forms, and update tests, docs, and the
      sase_monitor skill source."
proposed_by: bbugyi200.athena.0u2
create_time: 2026-09-29 16:47:55
status: wip
---

- **PROMPT:**
  [prompts/202609/tool_inline_routing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/tool_inline_routing.md)

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                  | Why                                                                                        |
| ------------ | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| derives-from | [research:202609/sase_tool_e6_e8_go_no_go/sase_tool_e6_e8_go_no_go.md][1] | E6-E8 go/no-go; scopes sase-17e as ceiling export plus a no-prediction class gate.         |
| implements   | bead:sase-17e                                                             | Duration classes, the provider ceiling export, and the pre-start refusal in sase tool run. |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202609/sase_tool_e6_e8_go_no_go/sase_tool_e6_e8_go_no_go.md

<!-- sase:links:end -->

# Plan: Mechanical inline-vs-monitor routing for `sase tool run` (sase-17e)

## Why, and what the research changed

`sase-17e` (read it with `sase bead read sase-17e -r '<why>'`) asks for routing that no
longer depends on prompt rules alone. Today the only rules are the Muse single-turn
directive in `src/sase/llm_provider/muse.py`, `/sase_monitor`'s "Decide Before You
Start", and `lint_and_test.md`'s list of known-long commands. The bead asks for:

- a duration class per catalog entry (`short`, `long`, `unbounded`);
- `SASE_PROVIDER_SYNC_CEILING_SECONDS` exported by provider adapters;
- a refusal in `sase tool run`, before anything starts, that prints the command to use
  instead;
- the corpus used for calibration, never as the gate.

It crosses the sase-core boundary.

The controlling context is
`research:202609/sase_tool_e6_e8_go_no_go/sase_tool_e6_e8_go_no_go.md` (read it with
`sase artifact read`). Its findings shape this design:

- **Kills.** Muse killed 160 inline `check` runs at 540 s on athena, and 57 of them were
  rerun. `check` has a p50 of about 2.5–4.5 minutes, but 20% of runs exceed 540 s on
  athena and 50% on apollo.
- **Unneeded hand-offs.** 38% of handed-off `check` runs finished in under 5 minutes, so
  every hand-off that wasn't needed cost a turn boundary.
- **Forecasts.** A p10–p90 band is 18.6× wide, too wide to route on.
- **Recommendation.** Handle `check` with reactive escalation (`sase-17g`: start inline,
  move to a monitor at the ceiling without restarting). `sase-17e`'s ceiling export is
  the piece `sase-17g` needs. The duration-class refusal is "optional and advisory at
  most".

**Design consequence: a duration class is a floor, not a forecast.**

- The class states the least time the tool essentially always takes.
- `sase tool run` refuses only when that floor meets or exceeds the caller's kill
  ceiling, which is when running inline cannot succeed.
- `check` fails fast and often finishes in 2–4 minutes, so it stays `short`. It is never
  refused and gains no hand-offs. Its over-ceiling tail remains `sase-17g`'s problem.
- On a correctly declared catalog the refusal can only fire where the ceiling would have
  killed the run anyway. That keeps it inside `decisions:guarded-recipes`, whose reopen
  condition is "any false refusal". The refusal refuses; it never redirects.
- The corpus only calibrates declarations: `sase tool list` flags a class the ledger
  contradicts. It never gates a run.

## Contract (applies to every phase)

### Duration classes and floors

| Class                      | Floor           | Meaning                                                                                                                                             |
| -------------------------- | --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `short` (default if unset) | 0 s             | Can finish in minutes, including tools whose arguments or early failure make them short (`check` fails fast; `test -- <path>`). Always fits inline. |
| `long`                     | 600 s           | Essentially never finishes in under 10 minutes (`check-full` is a superset of the full test suite, whose bare `test` TYPICAL is 15m37s on athena).  |
| `unbounded`                | none (no floor) | No upper bound, or waits on something external (servers, watchers, deploy waits). Never fits under any ceiling.                                     |

**Rule:** a run fits inline when no ceiling is present, or when the class is not
`unbounded` and its floor is less than the ceiling. Equivalently, refuse exactly when a
ceiling is present and the floor is at least the ceiling (an `unbounded` floor is
infinite). Rust owns the floors and this rule; Python never restates them.

### Catalog field

- `duration_class: short | long | unbounded` is an optional key on a `tools:` entry in
  the project's `sase/sase.yml`.
- Rust validates it. An unknown value is a catalog error (exit 2) naming the entry and
  field.
- Follow the `receipt:` precedent:
  - Exclude it from the definition digest, so declaring or changing a class never moves
    TYPICAL history, triage identity, or receipt coverage.
  - Serialize it only when declared, so undeclared entries stay byte-identical.
- Ad-hoc runs have no class and are always `short`.
- An older installed core rejects `duration_class` as an unknown field
  (`deny_unknown_fields`). The pin bump and the first declaration therefore land in the
  same phase, and `just install` heals a stale wheel. This is the rollout already
  documented for `receipt:`.

### Ceiling environment

`SASE_PROVIDER_SYNC_CEILING_SECONDS` is a positive integer number of seconds: the most
time one synchronous command can run in the caller's provider harness before the harness
kills it.

- **Who sets it.** The agent runner sets it around each provider invocation, from the
  execution provider's declaration. It is removed when that provider declares no
  ceiling.
- **Values:**
  - Muse: `600` when `muse_synchronous_shell` is on (`_MUSE_SYNC_CEILING_SECONDS`), and
    nothing when the flag is off.
  - Claude: the effective `BASH_MAX_TIMEOUT_MS` divided by 1000. That is SASE's 4 h
    default unless the user overrides `BASH_MAX_TIMEOUT_MS`.
  - Every other built-in provider: nothing. Declare only a ceiling the adapter itself
    sets or documents in code; never guess one.
- **Scrubbing.** The variable is scrubbed at every agent, monitor, and proc boundary, so
  a monitor, a proc, or a child agent never inherits its starter's ceiling.
- **Malformed values.** A non-integer, zero, or negative value is treated as absent. The
  check fails open.
- **Consumers.** This epic's refusal is the first consumer. `sase-17g`'s ceiling-bounded
  `wait` will be the next.

### Refusal

`sase tool run` refuses only when all four conditions hold:

1. `SASE_AGENT` is non-empty.
2. The ceiling is valid.
3. The run is a named catalog tool.
4. The Rust fit call says the tool does not fit.

When it refuses:

- It exits `2` before `reconcile_unsettled_tool_runs`, before any reservation, before
  any ToolRun row is written, and before the child starts.
- It prints the refusal, the `--next` monitor form, and the prepared-completion form on
  stderr (see phase `ceiling-refusal`).

These never evaluate the check:

- humans and CI (no `SASE_AGENT`);
- monitors and procs, which scrub agent identity and the ceiling;
- the monitor `_adopt` worker and the TUI's `execute_handoff`, which never call
  `execute_tool_run`;
- the non-agent `-H` path.

There is **no bypass flag**. Running inline past the ceiling is killed anyway, and a
monitor always works. A misdeclared class is fixed in the project-owned catalog.

**Rollback** is the catalog itself: with no `long` or `unbounded` declaration, nothing
is refused.

No feature flag is needed. The phase order never exposes half a feature: the class is
visible in `list` before the refusal exists, and the refusal lands complete in one
phase.

### Boundary

- **Rust `sase_core::tool_run` owns** the class enum, validation, digest exclusion, the
  floor table, the fit decision, and the calibration verdict.
- **Python owns** the env export, the CLI refusal and its message, the `list` rendering,
  docs, and skills, as thin adapters.
- Open the linked sase-core checkout only with `sase repo open sase-core -r '<why>'`,
  work in the printed path, and read that repo's `AGENTS.md` first.

## Phase `core-duration-class` (sase-core)

Work in the linked sase-core checkout. Follow its `AGENTS.md` recipe "Add a core
function and expose it to Python", and never run bare `cargo`.

1. **Wire.** In `crates/sase_core/src/tool_run/wire.rs`:
   - Add `ToolDurationClassWire` (serde `snake_case`: `short`, `long`, `unbounded`).
   - Add `ToolDefinitionWire.duration_class: Option<ToolDurationClassWire>` with
     `#[serde(default, skip_serializing_if = "Option::is_none")]`, beside `receipt`.
   - Keep `deny_unknown_fields`. Keep `TOOL_RUN_WIRE_SCHEMA_VERSION` at 1: the change is
     additive, like `receipt`.
2. **Digest.** Do not add the field to `ToolDefinitionIdentity` in `catalog.rs`.
   Preserve a declared value as declared (`Some(Short)` stays `Some(Short)`). Tests:
   - The digest is identical for undeclared, `short`, `long`, and `unbounded`.
   - Normalized output omits the key when it is undeclared.
   - An unknown class string fails normalization with a message naming the value.
   - Add a golden fixture beside `fixtures/definition_check.json` for a definition with
     `duration_class: long`.
3. **Policy module.** Add a new file, for example `tool_run/duration.rs`, exported
   through the `tool_run` facade.
   - **Floors.** Use named constants: short 0 s, long 600 s, unbounded none.
   - **Fit.** `duration_fit(request) -> response`.
     - The request is a versioned wire carrying `duration_class: Option<...>` (`None`
       means `short`) and `ceiling_seconds: Option<u64>`.
     - The response carries the effective class, `floor_seconds: Option<u64>` (`None`
       only for `unbounded`), the echoed ceiling, and `fits_inline: bool`, per the rule
       above.
     - A ceiling of `0` is an error; Python passes only validated positive values.
   - **Calibration.** `duration_calibration(request) -> response`.
     - The request carries the class, `typical_duration_ms: Option<u64>`, and
       `typical_sample_count`. These are the existing TYPICAL fields `sase tool list`
       already gets from `tool_run_summary`.
     - The response always carries the effective class, plus `calibration: Option<...>`
       holding `kind`, `suggested_class`, the numbers, and a one-line `summary` so every
       frontend renders the same words.
     - Stay silent below `DURATION_CALIBRATION_MIN_SAMPLES = 10`.
     - `long` with TYPICAL under 600 s: the floor is above typical, so the declared
       class may refuse runs that would fit. Suggest `short`.
     - `short` with TYPICAL at or above 600 s: TYPICAL meets the `long` floor. Suggest
       `long`, but only if the tool's arguments and failures never shorten it.
     - `unbounded` with enough normally exited samples: suggest `long` or `short` by
       TYPICAL.
     - Otherwise return `None`.
   - Unit-test every branch, including the boundary (floor equal to the ceiling is
     refused) and the sample minimum.
4. **Bindings.** Add `tool_run_duration_fit` and `tool_run_duration_calibration` (dict
   in, dict out). Put them in the binding domain that already binds
   `tool_run_normalize_definition` (`crates/sase_core_py/src/telemetry/`). Model them on
   that neighbour, and add binding tests there.
5. **Verify.** Run `sase tool run check` inside the sase-core checkout. It takes about 5
   minutes, is `short`, and runs inline. Use a Conventional Commit subject such as
   `feat(tool-run): add duration classes and inline-fit policy`. This is not `feat!`:
   released sase never sends the field and ignores extra response keys.

## Phase `provider-ceiling` (sase)

This phase is independent of the core and can run in parallel with it.

1. **Constant.** Add `SASE_PROVIDER_SYNC_CEILING_SECONDS_ENV` to
   `src/sase/env_contracts.py`.
2. **Hook.** Add an optional `llm_sync_ceiling_seconds() -> int | None` hookspec in
   `src/sase/llm_provider/_hookspec.py`.
   - Its docstring says: hard per-command synchronous kill ceiling; `None` or omitting
     the hook means none, so third-party providers stay compatible.
   - Expose it through the provider object `get_provider` returns (`_plugin_manager.py`,
     with a `base.py` default of `None` if needed).
   - Validate like `_usage_probe_floor_value` in `_registry_metadata.py`: reject `bool`,
     non-integers, and values ≤ 0. Treat an exception as `None`.
3. **Implementations:**
   - **Muse** (`muse.py`): `_MUSE_SYNC_CEILING_SECONDS` when
     `_muse_synchronous_shell_enabled()`, else `None`.
   - **Claude** (`claude.py`): the effective `BASH_MAX_TIMEOUT_MS`, using the same
     `setdefault` resolution `_run_subprocess` applies (the user's valid positive value
     wins, else `_BASH_MAX_TIMEOUT_MS`), `// 1000`.
   - **Other providers:** nothing.
4. **Export.** In `invoke_agent` (`src/sase/llm_provider/_invoke.py`), after
   `get_provider` resolves the execution provider, set the variable or pop it when the
   provider declares none.
   - Popping matters: it drops any inherited value.
   - Restore the previous value in the existing `finally`, exactly like
     `SASE_FINAL_TURN_NONCE`.
   - This covers every provider's subprocess, because each one copies or inherits
     `os.environ`. It also covers the finalizer follow-up turns inside `run_finalizers`.
5. **Scrub.** Add an exact-key pop to `scrub_agent_identity_env` in
   `src/sase/agent/env_hygiene.py`.
   - That one function already runs at the child-agent, proc and monitor supervisor,
     monitor child-env, and chop-script boundaries.
   - Do not scrub by the `SASE_PROVIDER_` prefix: it would strip the user's
     `SASE_PROVIDER_TEARDOWN_GRACE_SECONDS`.
6. **Tests:**
   - Muse flag on returns 600 and flag off returns `None`.
   - Claude: the default, a valid override, and garbage input.
   - A provider without the hook leaves the variable unset.
   - `invoke_agent`:
     - sets the value during `provider.invoke`;
     - restores or pops it afterwards, including on error;
     - pops an inherited value when the provider declares none.
   - Scrub coverage in `tests/test_agent_env_hygiene.py`.
   - The monitor child env lacks the variable
     (`tests/monitor/test_monitor_start_supervisor.py`).
7. **Docs.**
   - `docs/llms.md`: the Muse single-turn section, the Claude env section, and the env
     table.
   - A new row in `docs/tool.md`'s environment-contract table.

Nothing consumes the value yet, so this phase changes no behavior.

## Phase `catalog-duration-class` (sase)

1. **Pin.**
   - Confirm the `core-duration-class` commit is on sase-core's remote master.
   - Run `just ratchet-core-revision`, then `just install`.
   - Confirm the "Check pinned core bindings" lint (`tools/check_sase_core_rs_bindings`)
     passes.
   - If the core commit is not pushed yet, stop and record why on this phase's bead.
     Never bypass binding validation. See `docs/rust_backend.md` ("The CI source
     revision pin").
2. **Facades.** Add `tool_run_duration_fit` and `tool_run_duration_calibration` to
   `src/sase/core/tool_run.py` through `require_rust_binding`, and export them in
   `__all__`. Do not re-implement the default class or the floors in Python. Take the
   effective class from the core responses.
3. **`sase tool list`** (`src/sase/main/tool_handler.py`):
   - Per tool, call calibration with the TYPICAL fields `_list_envelope` already has.
   - Add a `CLASS` column after `TYPICAL`.
   - Add `duration_class` and `duration_calibration` (object or `null`) to each JSON
     tool; the envelope's `schema_version` stays 1 because the change is additive.
   - Print each calibration `summary` as one stderr diagnostic line in human mode.
   - Update `tests/main/test_tool_handler.py`.
4. **Declare classes in `sase/sase.yml`, backed by corpus evidence:**
   - **`check-full: long`.** It is a superset of the full test suite (bare `test`
     TYPICAL is 15m37s on athena).
   - **`test-visual`.** Declare `long` only if athena's corpus supports a floor of at
     least 600 s (`sase tool runs -a -t test-visual`, settled runs). Otherwise leave it
     undeclared and add a `PROPOSED FOLLOW-UP:` note.
   - **`check`, `test`, `install`.** Leave them undeclared (`short`). Add a YAML comment
     on `check` explaining that it is short on purpose: it fails fast, and its
     over-ceiling tail is `sase-17g`'s escalation problem, not a refusal.
   - **Digests.** Record each tool's `sase tool list -j` digest before and after in a
     bead note. They must be identical.
5. **Docs:**
   - `docs/configuration.md` (`### tools`): the field reference.
   - `docs/tool.md`: a "Duration classes" subsection under "Catalog provenance". Cover
     the floors table, the digest exclusion, calibration, and the mixed-core rollout
     note.
6. **Tests.**
   - The loader surfaces a declared class.
   - An unknown class raises `ToolCatalogError`, which names the entry and field.
   - Digest equality with and without the field goes through the real binding.

## Phase `ceiling-refusal` (sase)

1. **Module.** Add a small module, for example `src/sase/tool/routing.py`:
   - `read_sync_ceiling(env) -> int | None`: positive integers only.
   - An `inline_refusal(resolved, env) -> str | None` helper that applies the four
     conditions in the contract. A missing or raising binding fails open (no refusal)
     with one stderr warning.
   - A shared
     `monitor_start_form(words, *, reason, next_text=None, completion_ref=None)`
     builder. Refactor the inline `monitor_form` in `src/sase/tool/handoff_launch.py` to
     use it with **byte-identical** output; `tests/tool/test_handoff.py` pins it.
2. **Call site.** In `execute_tool_run` (`src/sase/tool/executor.py`), put the check
   right after `resolve_run_argv` and `resolve_ownership` succeed, before
   `reconcile_unsettled_tool_runs`. On refusal, print the message and return 2.
3. **Message.** Tune the wording but keep every element: the tool, class, floor,
   ceiling, the variable name, "Nothing was started", and both runnable forms. Quote the
   words with `shlex` like `handoff_launch.py`:

   ```text
   sase tool run: refused before starting check-full: its duration class is long (runs at least 10m), and this agent's provider kills any command at 10m (SASE_PROVIDER_SYNC_CEILING_SECONDS=600). Nothing was started.
   Hand it to a monitor:
     sase monitor start -p verify -r 'run check-full (duration class long)' -n '<what the follow-up should do with the result>' -- sase tool run check-full
   For a final verification gate, prepare host completion first (/sase_final "Prepared Monitor Completion") with verification.command ["sase", "tool", "run", "check-full"], then:
     sase monitor start -p verify -f <ref> -r 'Verify before host completion' -- sase tool run check-full
   ```

   For `unbounded`, say "has no upper bound" instead of the floor.

   Before relying on the prepared form, confirm that `sase final prepare`
   (`src/sase/finalizers/prepare.py`) accepts that verification argv. Also confirm that
   the monitor's exact-command comparison (`src/sase/monitor/host_completion_state.py`,
   `_command_argv`) matches it. If either does not, print the catalog argv instead
   (`just check-full`); a `verify` monitor upgrades it to the named run.

4. **Help and parser.** Add one sentence about the refusal to `sase tool run -h` in
   `src/sase/main/parser_tool.py`. Run `just sync-completion-spec` if
   `tests/completion/test_snapshot.py` drifts. No new option is added.
5. **Tests.** Put them in a new `tests/tool/test_routing.py`, using the fixture-catalog
   pattern from `tests/tool/test_executor.py` and a marker-file argv to prove nothing
   ran:
   - An agent with ceiling 600 and a `long` tool gets exit 2, both forms on stderr, no
     marker, and an empty `tool_run_list`.
   - `unbounded` is refused under ceiling 14400.
   - `long` under ceiling 1800 runs.
   - `short` or undeclared tools run under 600.
   - Ad-hoc runs are never refused.
   - No `SASE_AGENT` with the ceiling set: the tool runs.
   - Malformed ceilings (`abc`, `0`, `-5`, empty) run, because the check fails open.
   - A missing binding fails open with a warning.
   - The `-H` agent refusal output is unchanged.
6. **Docs.**
   - Add a `docs/tool.md` subsection, "Inline routing". Cover:
     - the four conditions, and the floors versus the ceiling;
     - the message and the lack of a bypass;
     - rollback by editing the catalog;
     - how it relates to `sase-17g`.
   - Adjust the "Hand-off and lifecycle control" agent sentence.
7. **Skill source.**
   - In `src/sase/xprompts/skills/sase_monitor.md` "Decide Before You Start", add one
     provider-neutral sentence. It says that `sase tool run` enforces this for catalog
     tools declared `long` or `unbounded`: when the provider's ceiling cannot fit the
     tool, it refuses before starting and prints the monitor command to use, while short
     tools such as this repo's `check` still run inline.
   - Keep provider numbers out of the skill.
   - Leave the Muse directive unchanged; the refusal output describes itself.
   - Do not deploy skills from this phase.

## Landing

1. Confirm each phase's evidence, including the digest-equality note. Run `just fix`,
   then `sase tool run check`.
2. **Cheap live smoke.** Run it in a temporary git project, never against this repo's
   `check-full`. Use a fixture catalog whose `slow` tool is
   `[sh, -c, "sleep 1; touch marker"]` declared `long`.
   - `SASE_PROVIDER_SYNC_CEILING_SECONDS=600 sase tool run slow` exits 2, leaves no
     marker, and writes no run.
   - With `1800`, it runs.
   - Also record `echo "$SASE_PROVIDER_SYNC_CEILING_SECONDS"` from the lander's own
     shell. That is live proof of the export: 14400 under Claude, 600 under Muse, empty
     elsewhere.
3. Close `sase-17e`:
   `sase bead close sase-17e --note "<phases landed; smoke evidence; check stays short by design>"`.
4. Leave a note on `sase-17g` with `sase bead note`. It should say that the
   `SASE_PROVIDER_SYNC_CEILING_SECONDS` contract (set per provider invocation, scrubbed
   at owner boundaries, absent when there is none) is ready for its ceiling-bounded
   `wait`.
5. From the clean landed tree, deploy the changed skill: `sase skill init --force`, then
   `chezmoi apply` if it was skipped (see `generated_skills.md`).
6. File the phases' `PROPOSED FOLLOW-UP:` notes through `/sase_new_task` where they are
   warranted. Likely candidates:
   - a CLASS column in the Admin Center Tools pane;
   - configurable soft ceilings for providers without a hard one, which belongs with
     `sase-17g`;
   - a deferred `test-visual` class.

## How to tell it worked (post-landing observation, not a landing gate)

- `sase tool runs -a -t check-full` shows no inline agent runs signaled at the ceiling
  after landing.
- No `check` run is ever refused, by construction.
- `sase tool list` shows CLASS, and calibration stays silent or points at a real
  mismatch.
- `sase-17g` can read the ceiling without any new plumbing.

## Non-goals

- Inline-then-escalate (`--detach`, the bounded `wait`, `monitor start --join`): that is
  `sase-17g`.
- Forecasts, `run -E`, `sase tool stats`, and ETAs: these are the parked remainder of
  E6.
- Configurable soft ceilings.
- TUI surfaces.
- Receipts.
- **No memory edits.** `lint_and_test.md` and `glossary:tool-catalog` stay accurate. If
  a phase finds stale memory, it records a `PROPOSED FOLLOW-UP:` note.

## Verification (every phase)

- **sase phases:** run `just fix`, then `sase tool run check`. If the workspace venv is
  stale, run `just install` first.
- **The sase-core phase:** run `sase tool run check` in that checkout.
- Do not run `check-full` (`decisions:check-full-is-explicit`).
- Phase workers never create beads. They append `PROPOSED FOLLOW-UP:` notes to their own
  phase bead.
