---
tier: epic
title: Lane-level runner-slot gating for AXE lumberjacks
goal: 'Every agent an AXE lumberjack launches from a script chop proposal carries
  a `%wait(runners=N)` directive taken from a new `axe.lumberjacks.<name>.wait_runners`
  config key, so the `code_quality` lane can be configured to hold its audit agents
  until the machine is quiet instead of competing with ordinary work for the global
  runner pool.

  '
phases:
- id: core_wait_runners
  title: Rust core accepts the lumberjack wait_runners key
  depends_on: []
  size: medium
  description: 'core_wait_runners: add `wait_runners` to `LUMBERJACK_KEYS` in sase-core''s
    AXE validator, validate it as a non-negative integer with a new `validate_nonnegative_integer`
    helper, and cover accept/reject cases in the axe_chop unit tests and the config
    parity suite.

    '
- id: sase_plumbing
  title: Plumb wait_runners through sase and inject the directive
  depends_on:
  - core_wait_runners
  size: medium
  description: 'sase_plumbing: add `wait_runners` to `LumberjackConfig`, parse it,
    add it to the bundled JSON schema and the AXE entry editor basics, thread it from
    the lumberjack down to `prepare_chop_proposals`, emit the `%wait(runners=N)` line
    from `scaffolded_prompt` unless the proposal already declares one, show it in
    `sase axe lumberjack list`, and cover it with tests and docs.

    '
- id: enable_code_quality
  title: Require the published core and turn the lane on
  depends_on:
  - sase_plumbing
  size: small
  description: 'enable_code_quality: bump the `sase-core-rs` window in pyproject.toml
    to the release that carries the Rust change, then set `wait_runners` on the `code_quality`
    lumberjack in the chezmoi-managed sase_athena.yml.

    '
create_time: 2026-07-28 08:54:03
status: wip
bead_id: sase-af
---

- **PROMPT:** [202607/prompts/lumberjack_wait_runners.md](prompts/lumberjack_wait_runners.md)

# Plan: Lane-level runner-slot gating for AXE lumberjacks

## Goal

Give an AXE lumberjack a lane-wide runner-slot policy. Setting `wait_runners: <N>` on a lumberjack makes every agent
that lane's script chops launch carry `%wait(runners=N)`, so those agents park in the pre-launch runner-slot queue until
at most `N` other agents hold a runner slot. The motivating case is the user's `code_quality` lane, whose two
commit-audit chops should not compete with interactive work for the global agent pool.

## Current state

- The `code_quality` lumberjack is configured in the chezmoi-managed `sase_athena.yml`, not in this repo. It runs two
  script chops (`recent_bug_audit`, `recent_improvement_audit`) that emit `proposed_launches` and are guarded by
  `inhibit_if: agent_hood`, `run_every: 60m`, and a `git.commits_since` trigger with `checkpoint: on_action_success`.
- Chop launch prompts are assembled in exactly one place: `scaffolded_prompt` in `src/sase/axe/chop_proposal_models.py`.
  It emits `#<workspace>`, an `%id`/`%clan` block, optional `%model` and `%effort`, an optional `%wait:<name>` line for
  a `wait_on` dependency, and then the chop's own prompt body. All three consumers — `plan_chop_proposals`,
  `proposal_previews` (dry-run/preview), and `launch_chop_proposals` (both the single-agent and the clan-batch path) —
  build their prompt through that one function.
- `LUMBERJACK_KEYS` in `crates/sase_core/src/axe_chop/config.rs` is
  `["description", "interval", "chop_timeout", "chops", "env"]`. The Rust validator is fail-closed, so adding
  `wait_runners:` to a lumberjack today produces an `unknown_key` diagnostic and `load_axe_config()` raises
  `AxeConfigError`. The bundled `src/sase/config/sase.schema.json` has `additionalProperties: false` on the lumberjack
  object and would reject it too.
- `%wait(runners=N)` already exists end to end on the agent side: `resolve_wait_runners_args`
  (`src/sase/xprompt/_directive_values.py`) parses it, `PromptDirectives.wait_runners` carries it, and
  `wait_for_runner_slot` (`src/sase/axe/run_agent_wait_slots.py`) uses it as the admission threshold. Nothing in AXE can
  set it today; only a human-authored prompt or an ACE wait edit can.
- `chop_timeout` is the existing precedent for a lumberjack-level default: it is parsed in `parse_lumberjacks`, stored
  on `LumberjackConfig`, and threaded as `chop_timeout_default` from `lumberjack.py`, `src/sase/axe/cli.py`, and
  `src/sase/ace/tui/actions/axe_chop_run.py` into `run_configured_chop_once` → `run_script_chop_once`.
- `sase-core-rs` is pinned in `pyproject.toml` as `>=0.12.1,<0.13.0`, and the sase-core workspace is at `0.12.1`. A new
  accepted config key is a feature, so release-plz will cut `0.13.0`, which is outside the current window.
- Dev installs do **not** use that window: the `Justfile` builds `sase_core_rs` from the linked sase-core checkout and
  passes `--overrides` so the pyproject constraint is ignored. Phase 2 is therefore fully testable before the core
  release exists.

## Design decisions

These are settled; implementing agents should not re-litigate them.

### 1. `%wait(runners=N)` means "at most N _other_ agents are running"

`may_start` in `src/sase/core/runner_slots/_admission.py` refuses admission when `running_count > threshold`, and
`running_root_agent_count` only counts records that already have `run_started_at` — the candidate itself has not claimed
yet, so it is never in that count. The directive-completion hint says the same thing: "start when at most this many
agents are already running". Concretely:

- `wait_runners: 0` — start only when **no** other agent holds a runner slot.
- `wait_runners: 1` — start when at most **one** other agent is running.

The originating request named `%wait(runners=1)` but described the behavior as "not launched unless there are no other
running sase agents". Those are different: the behavior described is `runners=0`. **This plan configures the
`code_quality` lane with `wait_runners: 0` in Phase 3**, because the described behavior is the actual requirement and
the mechanism was incidental. The value is a single config integer, so switching to `1` is a one-character edit in
`sase_athena.yml` — no code change — if the literal directive is preferred after all.

The absence of the key keeps today's behavior exactly: no directive is emitted and the agent falls back to the implicit
`max_running_agents - 1` threshold.

### 2. The key is lumberjack-level only, named `wait_runners`

The name mirrors the directive keyword (`%wait(runners=...)`), the `AgentInfo.wait_runners` field, and the
`waiting.json` marker key, so the config, the prompt, and the runtime marker all use one word for one concept.

No chop-level `wait_runners` override is added. The request is lane-scoped ("any agent launched by the `code_quality`
lumberjack"), and a per-chop knob would need its own precedence story for something no configured chop needs today. A
chop that genuinely needs a different threshold already has an escape hatch — see decision 4.

### 3. Injection happens in `scaffolded_prompt`, driven by a field on `PreparedChopProposal`

The lane value is stamped onto `PreparedChopProposal.wait_runners` in `prepare_chop_proposals` and read by
`scaffolded_prompt`. This is deliberate: it is the only design where planning, dry-run previews, single-agent launches,
and clan-batch launches cannot drift, because all four already call `scaffolded_prompt` and all four already carry
`PreparedChopProposal` values through `dataclasses.replace`.

The alternative — passing the threshold as a separate argument to each of the three call sites — was rejected because
`chop_runner_script_result.py` reconstructs previews in three places (`proposals` on the parse path, after once-per
filtering, and inside `_updated_proposal_previews`), and each one would be an independent chance to forget the argument.

### 4. A proposal's own `%wait(runners=...)` wins

Two `%wait(runners=...)` values in one prompt is a hard `DirectiveError` ("Multiple %wait(runners=...) values are not
allowed"), which would surface as a chop action failure rather than as a useful message. So `scaffolded_prompt` skips
the lane default when the chop's prompt body already declares one, mirroring how per-chop `timeout` overrides the lane's
`chop_timeout`.

Detection uses a new cheap scanner in `src/sase/xprompt/_directive_scan.py` built on the existing
`_has_protected_pattern_match` helper, so fenced blocks and disabled regions are honored the same way
`has_deferred_start_directive` and `has_model_directive` already honor them. Do **not** call `extract_prompt_directives`
here — it expands xprompt references and is far too heavy for a per-proposal check.

### 5. Separate `%wait` lines, not one merged directive

When a proposal also has a `wait_on` dependency, emit two lines:

```
%wait:some-earlier-agent
%wait(runners=0)
```

Multiple `%wait` lines merge (agent lists accumulate, keywords are single-valued), which is existing, tested behavior.
Merging them into `%wait(dep, runners=0)` would mean re-deriving the combined form for the clan case and the templated
`wait_name` case for no benefit.

### 6. Injection only affects proposal-based script-chop launches

`launch_chop_proposals` is the launch path for script chops that emit `proposed_launches`. It is not how the builtin
`hooks` lane starts mentors, CRS workflows, or fix hooks — those go through their own launchers and stay governed by
`axe.max_agent_runners`. Say so in the docs so nobody expects `wait_runners` on `hooks` to throttle mentors.

### 7. Setting `wait_runners` makes every lane launch a deferred-workspace launch

`has_deferred_start_directive` matches any `%wait`, and `launch_cwd_agents` / `multi_prompt_launch_execution` pass
`deferred_workspace=has_wait`. So an injected `%wait(runners=N)` moves the workspace claim to _after_ the runner-slot
gate (`_prepare_workspace_and_repos` in `run_agent_runner_launch.py`). This is a desirable side effect — a parked audit
agent holds no workspace — but it is a real change in launch shape for lanes that adopt the key, and Phase 2's tests
should pin it rather than discover it later.

---

## Phase 1 — Rust core support

Repo: `sase-core` (open with `/sase_repo`; do not edit it any other way). Its `AGENTS.md` forbids hand-editing versions
— release-plz owns them, so do not touch `Cargo.toml` versions.

### Changes

1. `crates/sase_core/src/axe_chop/config.rs`
   - Add `"wait_runners"` to `LUMBERJACK_KEYS`.
   - Add a `validate_nonnegative_integer` helper next to the existing `validate_positive_integer`. It emits code
     `negative_integer` with message `value must be a non-negative integer` when `value.as_u64()` is `None`. Keep
     `validate_positive_integer` untouched — `interval` still requires a positive value.
   - In `validate_lumberjacks`, after the existing `chop_timeout` block, validate `wait_runners` with the new helper at
     path `axe.lumberjacks.<name>.wait_runners`.

2. `crates/sase_core/src/axe_chop/tests.rs`: a lumberjack with `wait_runners: 0` and one with `wait_runners: 3` validate
   clean; `wait_runners: -1`, `wait_runners: "1"`, and `wait_runners: 1.5` each produce exactly one `negative_integer`
   diagnostic at the right dotted path; a lumberjack with no `wait_runners` produces no diagnostic.

3. `crates/sase_core/tests/config_parity.rs`: one compose case proving `wait_runners` survives layer composition and
   provenance — a base layer setting `wait_runners: 2` on a lumberjack and an overlay setting `wait_runners: 0` must
   compose to `0` with the overlay's provenance and no diagnostics.

No `sase_core_py` change is needed: the validation request is dict-based and the new key is only an addition to an
allow-list, not a wire field.

### Acceptance

- `cargo test` passes in `sase-core`.
- A config carrying `axe.lumberjacks.<name>.wait_runners: 0` validates clean; a negative or non-integer value produces a
  precise `negative_integer` diagnostic naming the source layer.
- Merging the change lets release-plz publish the release Phase 3 will pin.

---

## Phase 2 — sase plumbing and prompt injection

Repo: `sase`. Run `just install` before `just check` — ephemeral workspaces drift, and `just install` is also what
rebuilds `sase_core_rs` from the linked sase-core checkout so Phase 1's change is live locally without waiting for the
published release.

### Config surface

1. `src/sase/axe/_config_types.py`: add `wait_runners: int | None = None` to `LumberjackConfig`, after `chop_timeout`.

2. `src/sase/axe/_config_targets.py`, `parse_lumberjacks`: read `wait_runners` from the raw lumberjack mapping next to
   the existing `chop_timeout = parse_duration(...)` line and pass it through. Absent → `None`. Coerce with `int(...)`;
   the Rust layer has already rejected anything that is not a non-negative integer.

3. `src/sase/config/sase.schema.json`: add a `wait_runners` property to the lumberjack object under
   `axe.lumberjacks.additionalProperties.properties`, `{"type": "integer", "minimum": 0}`, with a description naming the
   semantics from decision 1 ("start a lane agent only once at most this many other agents hold a runner slot; omit to
   use the global `max_running_agents` cap").

4. `src/sase/ace/tui/modals/axe_entry_editor_types.py`: extend `_BASICS_BY_KIND["lumberjack"]` to
   `("description", "interval", "chop_timeout", "wait_runners")`. The AXE entry sheet is schema-driven, so the widget
   and its validation come from step 3 for free; this only fixes the ordering.

### Threading

Follow the `chop_timeout_default` path exactly, adding a keyword-only `wait_runners_default: int | None` alongside it:

5. `src/sase/axe/chop_runner.py`: add the parameter to `run_configured_chop_once` (defaulting to `None`) and
   `_run_script_chop_once`, and forward it.

6. `src/sase/axe/chop_runner_script.py`: add the parameter to `run_script_chop_once` and forward it to
   `process_script_chop_result`.

7. `src/sase/axe/chop_runner_script_result.py`: add the parameter to `process_script_chop_result` and pass it to
   `prepare_chop_proposals`.

8. Call sites that resolve it from the matched lumberjack:
   - `src/sase/axe/lumberjack.py`, `_run_single_chop`: `wait_runners_default=self.config.wait_runners`.
   - `src/sase/axe/cli.py`, `handle_axe_chop_run`: `match.lumberjack.wait_runners`, and `None` in the `_oneshot`
     fallback branch that already sets `chop_timeout_default = None`.
   - `src/sase/ace/tui/actions/axe_chop_run.py`: `match.lumberjack.wait_runners`.

### Injection

9. `src/sase/xprompt/_directive_scan.py`: add `has_wait_runners_directive(prompt: str) -> bool` next to
   `has_deferred_start_directive`, implemented with `_has_protected_pattern_match` and a pattern matching a
   parenthesized `%wait`/`%w` directive carrying a `runners=` keyword, e.g. `(?:^|\s)%(?:wait|w)\([^)]*\brunners\s*=`.
   Export it wherever the module's siblings are exported.

10. `src/sase/axe/chop_proposal_models.py`:
    - Add `wait_runners: int | None = None` to `PreparedChopProposal` (last field, so the existing positional/keyword
      construction in `prepare_chop_proposals` keeps working).
    - In `scaffolded_prompt`, after the existing `%wait:{wait_name}` line, append `%wait(runners={n})` when
      `proposal.wait_runners is not None` **and** `not has_wait_runners_directive(proposal.prompt)`.

11. `src/sase/axe/chop_proposal_planning.py`, `prepare_chop_proposals`: add a keyword-only
    `lumberjack_wait_runners: int | None = None` parameter and set it on every `PreparedChopProposal` it builds. The
    existing `replace(...)` calls in this module and in `chop_runner_script_result.py` carry the field automatically.

### Display

12. `src/sase/axe/cli.py`, `handle_axe_lumberjack_list`: after the `interval` line, print
    `  [dim]wait_runners:[/dim] <n>` when it is set. Omit the line entirely when it is `None` so unconfigured lanes
    print exactly as they do today.

### Tests

13. `tests/test_axe_lumberjack_config_parsing.py`: `parse_lumberjacks` round-trips `wait_runners`, and a lumberjack
    without it parses to `None` (mirror the two existing `chop_timeout` tests).

14. `tests/test_config_schema_automation.py`: extend the existing script-chop case with `wait_runners: 0` so the bundled
    schema accepts it, and add a case asserting a negative value is rejected.

15. New coverage for the injection, in the nearest existing home (`tests/test_axe_chop_result_protocol.py` or a new
    `tests/test_axe_chop_wait_runners.py`):
    - `prepare_chop_proposals(..., lumberjack_wait_runners=0)` produces a scaffolded prompt containing
      `%wait(runners=0)`.
    - `lumberjack_wait_runners=None` produces a prompt with no `%wait(runners` at all.
    - A proposal whose own prompt already contains `%wait(runners=2)` is left alone — the emitted prompt has exactly one
      `runners=` occurrence, and it is the proposal's.
    - A proposal with both a `wait_on` dependency and a lane threshold emits both `%wait:<name>` and `%wait(runners=N)`,
      and `extract_prompt_directives` on the result yields the dependency name and the threshold together (this is the
      test that proves the two-line form actually merges).
    - The clan-batch path in `launch_chop_proposals` puts the directive in every `---`-separated segment.
    - `proposal_previews` shows the injected directive, so `sase axe chop run --dry-run` does not lie.
    - Per decision 7: `has_deferred_start_directive` is true for an injected prompt that has no other wait.

16. `tests/test_axe_cli.py`: `sase axe lumberjack list` prints the `wait_runners` line when configured and omits it when
    not.

17. Sweep for `LumberjackConfig(...)` and `run_configured_chop_once(...)` fixtures that need the new keyword — all new
    parameters default, so nothing should break, but confirm rather than assume.

### Docs

18. `docs/configuration.md`: add a `wait_runners` row to the **Lumberjack fields** table (`int`, not required, no
    default) describing the "at most N _other_ agents" semantics and that omitting it uses the global
    `max_running_agents` cap.

19. `docs/axe.md`: add `wait_runners` to the Lumberjack Configuration YAML example and its field table, and state
    decision 6 explicitly — the key gates agents launched from chop `proposed_launches`, not mentor/hook/CRS workflows.

### Acceptance

- `just check` passes.
- Adding `wait_runners: 0` to any lumberjack in any config layer no longer produces `unknown_key` or
  `additional_property` diagnostics.
- `sase axe chop run <chop> --dry-run` on a lane with `wait_runners` set shows `%wait(runners=N)` in the previewed
  prompt; the same lane without it shows an unchanged prompt.
- A real launch from such a lane produces an agent that reports `Waiting for a runner slot` while other agents are
  running, and starts once they finish.

---

## Phase 3 — Require the published core and enable the lane

Two repos, in this order. Do not start until release-plz has published the sase-core release containing Phase 1.

### 1. sase — pin the published core

`pyproject.toml`: bump `sase-core-rs>=0.12.1,<0.13.0` to the window that requires Phase 1's release (expected
`>=0.13.0,<0.14.0`; use the version actually published). Refresh `uv.lock`, run `just install` and `just check`.

This ordering matters: until the pin is bumped, a user running the published wheel would get an `unknown_key` diagnostic
the moment `wait_runners` appears in their config — which is exactly what step 2 does.

### 2. chezmoi — configure the `code_quality` lane

Open with `sase repo open chezmoi -r "..."` and edit only through the printed path. The file is
`home/dot_config/sase/sase_athena.yml`.

Add `wait_runners: 0` to the `code_quality` lumberjack, next to its `interval: 60`, and extend that lumberjack's
`description` body with one sentence recording the intent — something like:
`Agents launched from this lane wait for an idle machine (%wait(runners=0)) so commit audits never compete with interactive work.`
Keep the description within the grammar the AXE validator enforces (summary line ≤ 100 characters, blank second line,
then the body, ≤ 2000 total).

Leave `run_every`, `refresh_docs`, and `telegram` alone — this change is scoped to `code_quality`.

Then apply the change on the machine the way chezmoi changes are normally applied, and confirm.

### Acceptance

- `sase axe lumberjack list` shows `code_quality` with `wait_runners: 0` and no config diagnostics.
- `sase axe chop run recent_bug_audit --dry-run --force` previews a prompt containing `%wait(runners=0)`.
- The next real `code_quality` launch parks in the runner-slot queue while other agents are running and shows as
  `WAITING` in the ACE Agents tab.

---

## Risks

- **Wrong threshold shipped.** `runners=1` and `runners=0` differ by one concurrent agent, and the originating request
  named one and described the other. Decision 1 picks `0`; if that is wrong it is a one-character config edit, not a
  code change.
- **Starvation on a busy machine.** With `wait_runners: 0`, an audit agent on a machine that always has at least one
  agent running never starts. It parks as a live process polling every two seconds, and nothing reaps it. Two things
  bound the damage: it holds no workspace (decision 7), and the lane's existing `inhibit_if: agent_hood` guard sees
  `WAITING` agents as active (`active_status_for_record` returns `WAITING`, and `_agent_snapshots` marks it
  `active: True`), so the next hourly tick will not stack a second audit behind it. If starvation shows up in practice,
  `wait_runners: 1` is the release valve.
- **Serialized clans.** If a lane with `wait_runners` set ever proposes a clan, every member carries the threshold and
  the members serialize instead of running in parallel. Neither `code_quality` chop proposes clans today, but the docs
  in Phase 2 should mention it so a future clan-proposing chop is not a surprise.
- **Release window between Phases 2 and 3.** After Phase 2 lands, sase reads a key the published core still rejects.
  This is harmless as long as nobody sets it, which is why Phase 3 bumps the pin before touching chezmoi. Do not reorder
  those two steps.
