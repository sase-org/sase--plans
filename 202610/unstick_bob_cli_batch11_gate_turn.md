---
tier: tale
title:
  Unstick bob-cli batch 11 by fixing gate-turn CLI routing and relaunching the gated
  ref-id phase
goal:
  The batch-11 ref-id phase runs again and ends in a real gate turn, and `sase gate
  create --turn` registers gate-turn rows.
size: medium
decisions:
  phase_size:
    ask:
      Should the relaunched ref-id phase run at size medium (the @medium model) instead
      of its original small?
    default: true
    why:
      The small first run missed the answer-time environment; a stronger rerun is
      cheaper than a stall
    answer: true
  gate_env_doc:
    ask:
      Should the sase_gate skill source document the cwd and environment that gate
      option commands run in?
    default: true
    why:
      The second failure came from assuming commands inherit the creating agent's
      environment
    answer: true
proposed_by: bbugyi200.athena.148
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-25 13:38:48
status: wip
---

# Unstick bob-cli batch 11: fix `sase gate create --turn` routing, then relaunch the gated ref-id phase

Epic `bob-cli-an.6.6.6.5.6.5.5.6.6.6` (Backlog batch 11) is stuck:

- Phases `.2`–`.5` are closed.
- Phase `.1` (`ref-id-live-repair`) is still `open`, and its agent is `DONE`.
- The land agent `bob-cli-an.6.6.6.5.6.5.5.6.6.6.land` is `WAITING` on that phase bead.
  Nothing will ever close it.

This tale fixes the sase defect that caused the stall. It then relaunches the phase with
guidance that works on the sase that is installed today.

## Diagnosis (verified while planning)

1. **The phase could not create a gate turn.** The phase needed a `/sase_gate` turn
   whose follow-up records the outcome and closes the phase. `sase gate create --turn`
   failed with `missing_gate_turn_row`, so the agent fell back to a non-turn gate. That
   gate is `custom-cf7c69d1-c91f-4e26-8666-2b05572e2c66`, notification
   `deb2617c-822e-48a7-bb0a-73ea81a675bb`. A non-turn gate has no follow-up agent (phase
   notes #2–#3).
2. **The approved repair failed before writing anything.** Bryan approved option
   `repair` from the TUI at 12:20 EDT. The command exited 1 with
   `no freshly built bob available; refusing to use installed bob`; see the bundle's
   `errors/` record and `journal.jsonl`.
   - Gate option commands run with the bundle directory as cwd and the answering
     process's environment. In `src/sase/notification_gates/command_runner.py`,
     `run_owned_command` passes `env={**os.environ, SASE_GATE_COMMAND: "1"}` and
     `cwd=bundle_path`. That environment is the TUI's, not the creating agent's, so
     `$CARGO_TARGET_DIR` was unusable.
   - The script's fallback, `git rev-parse --show-toplevel` run from `~/bob`, resolved
     the vault rather than a cargo checkout.
   - The live vault is unchanged: `~/bob/job_update_cv.md` still links
     `ref/paper/BryanBugyi_CV_extended_v2`. The notification is now dismissed, and no
     agent will ever see the outcome.
3. **The sase root cause.** `src/sase/main/gate_handler.py` `_handle_gate_create` sends
   a spec to `sase.gate_turn.create_gate_turn` only when
   `isinstance(data.get("shell"), dict)`. That line was written while the block was
   still called `shell`. Two paths now miss it:
   - Any turn CLI flag (`--turn`, `--turn-status`, `--turn-stop-status`, `--next*`) runs
     `_merge_shell_cli_overrides`, which moves `shell` to `turn` through
     `normalize_gate_spec_block` and writes `data["turn"]`.
   - A canonical JSON `turn` block never has a `shell` key.

   Both fall through to `notification_gates.service.create_gate`. There,
   `validate_gate_spec` (`src/sase/notification_gates/validation.py`) correctly refuses
   a custom gate that has a `turn` block but no `turn_row_managed`. So the documented
   form in the `sase_gate` skill always fails. The only form that works is a legacy JSON
   `shell` block passed with no turn CLI flags.
   - That is the workaround `sase-1nv.5` used on 2026-10-24 (its note #1).
   - Existing tests (`tests/gate_turn/test_model_turn.py`) call `create_gate_turn` and
     `create_gate` directly. None of them goes through `sase gate create`.
   - Other agents hit the same failure: `sase-1e3.11`, `sase-1j1.6`, `sase-1my`, and
     `sase-1nv.5`.
   - The gate half of the READY task `sase-1el` is this defect. Its `sase monitor start`
     `NameCollisionError` half is unrelated and out of scope here.

4. **The fix will not reach the relaunched agent.** The global `sase` is an editable uv
   install of the primary checkout `~/projects/github/sase-org/sase`. That checkout
   fast-forwards to master only on a later update, and commits are host-owned and land
   after this turn. So the relaunched phase agent will still run the buggy CLI. Its
   guidance must use the legacy `shell` key workaround, which works on both the old and
   the fixed code.

## Steps

### 1. Route every turn-bearing spec through the gate-turn transaction

In `src/sase/main/gate_handler.py` `_handle_gate_create`:

- Normalize the legacy block once, whether or not any CLI flags were given, using
  `sase.agent.legacy_sase_shell_syntax.normalize_gate_spec_block`. A `ValueError` (both
  `shell` and `turn` present) prints `Error: <message>` and exits 1, matching
  `_merge_shell_cli_overrides`. Normalizing again after the merge is harmless, because
  the helper is idempotent.
- Dispatch on the canonical key: any spec carrying a non-`None` `turn` value goes
  through `create_gate_turn`. Keep the existing sequence: print the descriptor, flush,
  then call `maybe_handoff_gate_from_agent` when `should_handoff` and
  `will_handoff_gate_to_agent_runner()`.
- A non-object `turn` value must surface its `GateSpec` validation error, not
  `missing_gate_turn_row`.
- Specs without a `turn` block still go to `create_gate`, unchanged.
- Do not weaken the `missing_gate_turn_row` guard in `validate_gate_spec`, and do not
  change `create_gate_turn`. `sase-161` (extending the guard to built-in kinds) stays
  out of scope.

Add CLI-level regression tests, for example
`tests/gate_turn/test_gate_create_cli_turn.py`.

- Drive `register_gate_parser` and `handle_gate_command` with the request JSON on stdin.
- Reuse the fixture pattern from
  `test_turn_block_custom_gate_registers_and_lists_row_end_to_end`:
  `gate_home`/`sandbox_home`, `write_project_file`, `make_starter_agent`, and
  `SASE_AGENT_NAME`/`SASE_ARTIFACTS_DIR`.
- Delete `SASE_FINALIZER_OWNED_TURN`.
- Make the handoff inert, for example by monkeypatching
  `will_handoff_gate_to_agent_runner` to return `False` in `sase.main.gate_handler`'s
  import target, or by clearing the runner environment. A test must never signal the
  real runner.

Cover these cases:

| Input                                            | Expected                                                                                                                                                             |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| canonical JSON `turn` block, no flags            | exit 0; the descriptor on stdout; `find_gate_turn_by_gate_id` finds the row; `continuation_mode == "gate_turn"`; the gate is listed by `sase gate list --all --json` |
| turn-less JSON plus `--turn --next "…"`          | same as above; the `next.prompt` is persisted in `request.json`'s `turn`                                                                                             |
| legacy JSON `shell` block, no flags              | still exit 0 with a registered row                                                                                                                                   |
| legacy JSON `shell` block plus `--turn-status P` | exit 0, row registered, pending status `P`                                                                                                                           |
| JSON with both `shell` and `turn`                | exit 1 with the combine error                                                                                                                                        |
| turn-less JSON, no flags                         | plain gate, no gate-turn row (regression guard for the non-turn path)                                                                                                |

Run `sase tool run check`, never `check-full`. Read `sase/memory/lint_and_test.md`
first.

### 2. Document the command environment in the skill source

> [!decision] gate_env_doc

In `src/sase/macros/skills/sase_gate.md`, add one short paragraph to the
command-resources section. It sits just before `## Declare The \`turn\` Block`, near
"Keep command resources narrowly scoped". The paragraph says:

- Option and operation commands run with the gate bundle as their working directory.
- They run in the environment of whichever process executes the answer: the TUI, mobile,
  a proc, or a follow-up runner. They do not run in the creating agent's environment.
- So bake absolute paths and pinned artifacts into the command at creation time. Never
  depend on agent-only variables such as `CARGO_TARGET_DIR`, `SASE_ARTIFACTS_DIR`, or
  the agent's workspace path.

Read `sase/memory/generated_skills.md` before editing. Preview with
`sase skill init --diff`. Do **not** deploy (`sase skill init --force` or
`chezmoi apply`) from this unlanded tree. If a test snapshots skill text, update it.

> [!decision] gate_env_doc = no

Skip this step.

### 3. Record the root cause on `sase-1el`

Append a note with `sase bead note sase-1el "…"`. Keep it to one or two sentences
covering:

- the `gate_handler` dispatch root cause;
- the fix and its regression tests from this tale, noting that the fix lands with this
  agent's commit;
- the legacy-`shell` workaround for older installs;
- that the monitor `NameCollisionError` half is still open and undiagnosed.

Leave `sase-1el` open and do not change its status. Do not create new beads. The
batch-11 phase's `PROPOSED FOLLOW-UP` (note #3) already maps to `sase-1el`, and the
bob-cli lander will dedupe it.

### 4. Relaunch the stuck phase with resume guidance

Run everything here from this workspace. `sase bead` resolves the `bob-cli-*` IDs. Do
not edit any bob-cli file, the live vault, or the gate bundle.

1. **Re-verify the state.**
   - `sase bead read bob-cli-an.6.6.6.5.6.5.5.6.6.6 -r "…"` still shows phase `.1` not
     closed and `.2`–`.5` closed.
   - `sase agent list -a -j` shows the land agent `WAITING`, the phase-1 agent `DONE`,
     and no live member of the phase-1 session.
   - If anything differs (someone already relaunched it, the phase closed, or a new gate
     exists), stop, report what you found, and skip the rest of this step.
2. **Append the resume note** to `bob-cli-an.6.6.6.5.6.5.5.6.6.6.1` with
   `sase bead note`. Do not edit its description or earlier notes. Use this text,
   adjusting only facts you re-verified differently:

   > RESUME GUIDANCE (2026-10-25): The earlier gate never ran its repair.
   >
   > - Bryan approved option `repair` on the non-turn gate
   >   `custom-cf7c69d1-c91f-4e26-8666-2b05572e2c66` at 12:20 EDT. The command exited 1
   >   before any write:
   >   `no freshly built bob available; refusing to use installed bob`.
   > - Gate option commands run with the bundle directory as cwd and the answering
   >   process's environment (here the TUI), not this agent's. So `$CARGO_TARGET_DIR`
   >   was unusable, and the `git rev-parse` fallback found the vault.
   > - The live vault is unchanged: `job_update_cv.md` still links `ref/paper/…`. That
   >   gate is dismissed. Never reuse or re-answer it.
   >
   > Redo phase steps 1–3 as a fresh decision for Bryan:
   >
   > - Re-run the read-only diagnostics with a freshly built bob.
   > - Build bob from this workspace through `$CARGO_TARGET_DIR`. Copy the binary to a
   >   stable path outside every workspace and the vault, for example
   >   `~/.local/state/bob-cli/ref-id-repair/bob-<commit12>`. Record its sha256 and
   >   source commit.
   > - Write every gate command to use that absolute path. Each command must verify the
   >   sha256 first and fail closed. It must export `BOB_DIR=$HOME/bob` before any bob
   >   call, `cd` to the vault explicitly, and depend on no agent-only environment
   >   variable.
   >
   > Create a real gate turn. The installed sase still has the `sase gate create --turn`
   > routing bug (`sase-1el`; the fix is pending in sase).
   >
   > - Put the whole turn policy in the request JSON under the legacy `"shell"` key.
   > - Pass none of `--turn`, `--turn-status`, `--turn-stop-status`, `--next`, or
   >   `--next-*`.
   > - On the installed sase that path registers the gate-turn row and hands off. The
   >   descriptor's `continuation_mode` must read `gate_turn`.
   >
   > Turn policy:
   >
   > - On approve, the follow-up records the command results and clean diagnostics here
   >   and closes this phase.
   > - On reject, it records Bryan's feedback verbatim and closes this phase.
   >   `bob-cli-b8` stays open for the lander.
   > - On `failed` or `timeout`, it records the failure here and checks read-only which
   >   commits already landed. It then creates a corrected replacement gate turn the
   >   same way rather than leaving this phase open.
   >
   > Gate settings:
   >
   > - Keep `auto` false.
   > - The gate notes must say that the earlier approval never executed.
   > - Keep the PDF-rewrite cost and Highlights-closing instructions from the plan.
   >
   > Never fall back to a non-turn gate. If gate-turn creation still fails, record the
   > exact error here and ask Bryan through `/sase_questions` how to proceed.

3. **Apply the phase-size decision.**

   > [!decision] phase_size

   Run `sase bead update bob-cli-an.6.6.6.5.6.5.5.6.6.6.1 -z medium`.

   > [!decision] phase_size = no

   Leave the phase size alone.

4. **Dry-run the relaunch** with `sase bead work bob-cli-an.6.6.6.5.6.5.5.6.6.6 -n`.
   Proceed only if it plans exactly the following:
   - `KILL (WAITING)` of the land agent;
   - `REMOVE (COMPLETED)` of the old phase-1 agent;
   - one wave containing only phase `.1`, with `%model:@medium` when `phase_size` is on
     and `@small` when it is off;
   - one land agent that waits on `.1`.

   Otherwise stop and report the dry-run output.

5. **Relaunch** with `sase bead work bob-cli-an.6.6.6.5.6.5.5.6.6.6 -Y`. Confirm with
   `sase agent list -j` that the new phase-1 agent is running or queued and the new land
   agent is `WAITING`. Do not wait for the phase, its gate, or Bryan's answer. The gate
   turn's follow-up and the lander own the rest.

Do not run `sase update`, `just install`, or any global reinstall. The routing fix goes
live on its own once the primary checkout next fast-forwards.

## Acceptance

- `sase gate create` registers a gate-turn row and hands off for a canonical JSON `turn`
  block and for the `--turn`/`--next*` flags. A legacy `shell` block still works, the
  both-blocks error still fires, and turn-less gates are unchanged. The new CLI tests
  prove each case, and `sase tool run check` passes.
- When `gate_env_doc` is on, the `sase_gate` skill source documents the command cwd and
  environment, and the skill was previewed but not deployed.
- `sase-1el` carries the root-cause and fix note and stays open for its monitor half.
- Phase `bob-cli-an.6.6.6.5.6.5.5.6.6.6.1` carries the resume note, and when
  `phase_size` is on it is size medium.
- The epic was relaunched through `sase bead work`, and the final report shows a live
  phase-1 agent and a waiting land agent.
- No live-vault, bob-cli repository, or gate-bundle file was modified, and no global
  install ran.
