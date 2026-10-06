---
tier: tale
title: Finish and close the grok-root phase (sase-1gu.3)
goal:
  "The grok-root phase's remaining defects are fixed: no test-order leak, a directive
  that names Grok's real wait primitives, and the missing sase-1gj pointer note. Phase
  bead sase-1gu.3 is closed so the record phase (sase-1gu.5) is unblocked."
size: small
proposed_by: bbugyi200.athena.0x6.w0
create_time: 2026-10-06 06:56:55
status: wip
---

# Plan: Finish and close phase bead sase-1gu.3 (grok-root)

## Context

Phase bead `sase-1gu.3` ("Grok root runs receive the directive and project AGENTS.md
once via --rules") belongs to epic `sase-1gu`
(`plan:202610/e1_instruction_scoreboard_and_stopgaps.md`, section "grok-root"). It is
still `in_progress`, and it blocks `sase-1gu.5` (record), which in turn blocks
`sase-1gu.6` (acceptance).

Most of the phase has already landed:

- Commit `724f9ea9c2`
  (`feat(llm-provider): add Grok provider core with docs and registry`) added the
  following to `src/sase/llm_provider/grok.py`: `_GROK_SINGLE_TURN_DIRECTIVE` (it opens
  with the shared marker `SASE single-turn instructions for Grok:`),
  `_grok_rules_text(cwd)` (the directive plus the project root `AGENTS.md` exactly once,
  in SASE-managed projects only), the 120 KiB `_GROK_RULES_ARGV_BYTE_LIMIT` guard, and
  `--rules` on every `_invoke_loop` cycle ahead of the `SASE_LLM_*`/`SASE_GROK_*` extra
  args. It sends no `--trust` and sets no `GROK_CLAUDE_AGENTS_ENABLED`.
- The sunset flag `grok_rules_delivery` is registered in
  `src/sase/feature_flags/registry.py`, and its flag bead is `sase-1gv`.
  `sase flag list` shows it as default on.
- Commit `10a2173a7c` moved the phase's tests into
  `tests/llm_provider/test_grok_provider_rules.py`. They cover `--rules` once, directive
  plus `AGENTS.md` once with no home H1, no `--trust`, the non-managed case, every
  continuation cycle, the over-limit error, both flag states, and a live parse probe.
- Docs are done: `docs/agent_providers.md` has "Instruction delivery", and
  `docs/llms.md` has Grok "Command Construction" and "Skills and Instruction File".
- The local canary is recorded as `sase-1gu.3` note #1. In it, `<human_rules>` held the
  directive marker and the project H1 exactly once, and `agents_md_files: []` was
  unchanged.

Planning found three things still open:

1. **A test-order leak from this phase's own test (latent flake).**
   `test_grok_rules_reach_every_continuation_cycle` sets
   `GrokProvider._pending_interrupt_message = "keep going"` on the **class**, but its
   `finally` only clears the **instance** attribute. The class attribute stays
   `"keep going"`, so the next `GrokProvider()` in the same worker runs an extra
   interrupt-continuation cycle. Reproduced:
   - `pytest tests/llm_provider/test_grok_provider_rules.py tests/llm_provider/test_grok_provider_invocation.py::test_grok_command_construction`
     fails at `mock_process.stdin.write.assert_called_once_with("test prompt")` (called
     2 times).
   - `pytest tests/llm_provider/test_grok_provider_rules.py tests/llm_provider/test_grok_provider_stream.py`
     fails 3 stream tests.
   - Each file passes alone. `tools/run_pytest` uses xdist `worksteal`, so the failure
     depends on which worker picks up the rules file.
2. **The directive does not name Grok's real wait primitives.** The phase spec says
   "Name Grok's actual background and subagent-wait primitives. Check them in a recent
   `system_prompt.txt` tool list." The shipped text says "a yielded result before the
   command exits means the command is still running", which is Codex vocabulary. The
   tool definitions in Grok 1.0.46 (from the canary session's `tool_definitions.json`)
   show:
   - `run_terminal_command` takes `block_until_ms` (**default 30000**, max 36000000). A
     command still running when it expires "moves to the background … You get a task
     id."
   - `get_command_or_subagent_output` takes `task_ids` and `timeout_ms` (max 3600000).
     It is the wait primitive for background commands, subagents, and monitors.
   - `spawn_subagent` defaults to `background: true`. With `background: false` it waits,
     but for at most 10 minutes.
   - Results from `monitor` and `scheduler_create` arrive "in a new turn after your turn
     ends", which never happens in SASE.

   Because the default block is 30 s, a `sase plan propose` that takes up to a minute is
   backgrounded by default. The directive has to tell Grok how to actually wait.

3. **The `sase-1gj` pointer note is missing.** The phase spec says "add a note on
   `sase-1gj` pointing to it [the canary]. Do not close `sase-1gj` here." `sase-1gj` has
   no notes.

No other gaps remain against the phase spec or the bead description.

## Changes

### 1. Fix the test-order leak

In `tests/llm_provider/test_grok_provider_rules.py`,
`test_grok_rules_reach_every_continuation_cycle`:

- Create `provider = GrokProvider()` **before** defining `_fake_run`. Inside
  `_fake_run`, set `provider._pending_interrupt_message = "keep going"` on the instance.
  This mirrors `test_grok_interrupt_preserves_partial_output_and_continues` in
  `tests/llm_provider/test_grok_provider_invocation.py`.
- Remove the `try`/`finally` that only reset the instance attribute.
- Defense in depth: add an autouse fixture in that module that runs
  `monkeypatch.setattr(GrokProvider, "_pending_interrupt_message", None)`. That way any
  future class-level write is restored after each test. It can be folded into the
  existing autouse `_clear_grok_env` fixture.
- Do not touch the production `_pending_interrupt_message` class attribute pattern. It
  is shared by every provider, and production code only writes it on the instance (via
  `setattr(self, …)`).

### 2. Tighten `_GROK_SINGLE_TURN_DIRECTIVE`

In `src/sase/llm_provider/grok.py`, rewrite the constant. Keep the exact opening
`SASE single-turn instructions for Grok:`, which the scoreboard fingerprint and the
`record` phase's marker tests depend on. The intended text is below. Small wording edits
are fine, but every named primitive must stay.

> SASE single-turn instructions for Grok: this session is exactly one turn, and nothing
> can wake you after you end it. Notifications that arrive "in a new turn after your
> turn ends" — from background commands, background subagents, `monitor`, or
> `scheduler_create` — never reach you, and anything still running when you give your
> final response is lost. Run commands in the foreground: set `run_terminal_command`'s
> `block_until_ms` longer than the command takes, because the 30-second default moves
> slower commands to the background. If a command returns a task id instead of an exit
> code, it is still running, not done: call `get_command_or_subagent_output` with that
> task id and a `timeout_ms` until it reports an exit code. Spawn subagents with
> `background: false`, or wait on each `subagent_id` the same way. Handoff commands such
> as `sase monitor start`, `sase plan propose`, `sase pipe`, and `sase questions` can
> take up to a minute before they hand off; wait for the handoff command itself to exit.
> Never end your turn to wait.

- Add `test_grok_directive_names_grok_wait_primitives` to
  `tests/llm_provider/test_grok_provider_rules.py`. It asserts that each of
  `block_until_ms`, `get_command_or_subagent_output`, `spawn_subagent`,
  `scheduler_create`, and `sase plan propose` appears in `_GROK_SINGLE_TURN_DIRECTIVE`,
  and that the old Codex phrase `a yielded result` does not.
- The existing rules tests build expectations from the constant, so they need no
  changes. The scoreboard fixtures use synthetic directive text and need no changes
  either. Confirm both by running them.
- In `docs/llms.md`, under Grok "Command Construction", extend the `--rules` bullet by
  one sentence. It should say that the directive names Grok's own wait primitives
  (`block_until_ms`, `get_command_or_subagent_output`, `spawn_subagent` with
  `background: false`), because `run_terminal_command` backgrounds anything slower than
  its 30-second default. Run `just fmt` afterward so Markdown wrapping matches house
  style.

### 3. Verify

- Run the order-dependence repro, rules file first. It must pass:
  `.venv/bin/python -m pytest -q tests/llm_provider/test_grok_provider_rules.py tests/llm_provider/test_grok_provider_stream.py tests/llm_provider/test_grok_provider_invocation.py tests/llm_provider/test_grok_provider_core.py tests/llm_provider/test_grok_provider_metadata.py`
- Then run `.venv/bin/python -m pytest -q tests/instructions` (the scoreboard and its
  Grok marker cross-check).
- Then run `sase tool run check`, which must pass. Do **not** run `just check-full`.
  Nothing here explicitly requests it.

### 4. Bead bookkeeping (after verification passes)

- Note `sase-1gj` (do **not** close it; `sase-1gu.6` acceptance closes it as superseded
  after its live Grok probe):
  `sase bead note sase-1gj "grok-root (sase-1gu.3) landed the --rules stopgap in 724f9ea9c2: the Grok single-turn directive plus the project root AGENTS.md exactly once, no home layer, no --trust, sunset flag grok_rules_delivery. Live canary evidence is sase-1gu.3 note #1 (directive marker and project H1 once inside <human_rules>; prompt_context.json agents_md_files still []). Leave this bead open: sase-1gu.6 acceptance closes it as superseded after its live Grok probe."`
- Close the phase bead with a note that summarizes the evidence:
  `sase bead close sase-1gu.3 --note "<landed commits 724f9ea9c2 + 10a2173a7c; this turn fixed the continuation-cycle test's class-level interrupt leak and made the directive name Grok's wait primitives (block_until_ms, get_command_or_subagent_output, spawn_subagent background:false, scheduler_create); canary in note #1; sase-1gj pointer note added; sase tool run check passed>"`
  Use the default `done` resolution.
- If `sase bead close` refuses, record why on the bead and report it. Do not use
  `--force`.

## Out of scope

- Closing `sase-1gj` (owned by `sase-1gu.6`), the epic bead `sase-1gu` (owned by its
  land agent), or `sase-1gu.5`/`sase-1gu.6`.
- Any memory file, `AGENTS.md` or its shims, the home layer, or the chezmoi source.
- The `record` phase's actor-qualified sentence, decision record, and marker-consistency
  tests.
- Changes to Codex, Muse, agy, or Claude adapters, or to the production
  `_pending_interrupt_message` pattern.
- A new live canary. The directive change does not touch the delivery mechanism, and
  `sase-1gu.6` runs the live Grok probe on the landed code.
