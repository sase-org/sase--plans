---
tier: tale
title:
  Gate commands accept a last-line JSON result, so chatty commands stop failing after
  approval
goal:
  A gate option or action whose command succeeds but printed other output before its
  JSON result no longer fails after the reviewer approved, and apollo's F11 measurement
  resumes.
size: medium
decisions:
  f11_followup:
    ask:
      How should the apollo F11 measurement resume, given that the reinstall already
      succeeded?
    choices:
      launch:
        The implementer requests a fresh F11 agent via /sase_run; you cancel both failed
        gates
      restart:
        After landing you pull athena's checkout and press R, rerunning the full apollo
        install
    default: launch
    why:
      apollo is already installed; restart repeats a 13-minute install to get one JSON
      line
    answer: launch
proposed_by: bbugyi200.athena.1ad
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 10:06:06
status: wip
---

# Gate commands fail after approval when anything else reaches stdout

## Diagnosis

**What the replacement gate actually did.** `1ab--code` staged gate `custom-61fd7f53-…`
("Reinstall apollo for sase-1s5 F11 (retry with -y)"). It was approved at 09:24 EST and
ran until 09:38 EST. **The reinstall succeeded.** The error record's stderr ends with
`✓ sase now runs this checkout (99db9e6) with sase-core 5c0e49c (pin 5c0e49c)` and
`service proc gateway restarted`. Its stdout ends with `{"status":"installed"}`, which
the script prints only after the `ssh … <<'REMOTE'` heredoc has exited 0 under
`set -eu`.

**Why the gate still failed.** The error is `invalid_command_output` ("command stdout
must contain one JSON value"), not the earlier `-y` refusal. Stdout also carried the
output of the remote `git checkout master` ("Your branch is up to date with
'origin/master'.") and `git pull --ff-only` ("Updating 8f4e6503d3..99db9e685d",
"Fast-forward", the diffstat). Everything a remote heredoc script prints lands on the
gate command's own stdout. The author redirected `just install-dev` and the gateway
restart with `>&2`, but not the two git lines.

**Same script, second bug.** This bug was already in the original gate
`custom-92df387c`: its error record's `stdout` field holds the same git checkout/pull
output. It was also in the 11-08 gate `custom-196bbcc8`. In both, the installer's `-y`
refusal (exit 2, `command_failed`) fired first and hid it. The 1ab plan fixed only the
failure that surfaced and told the implementer to keep "the same guarded flow", so the
second bug was copied over word for word.

**Root cause.** The gate executor treats all of stdout as the result.
`decode_json_result` in `src/sase/notification_gates/command_runner.py` is just
`json.loads(stdout)`. Options use it in `attempts.py::execute_one_option`, and actions
use it in `operations.py::_run_action_command`. So one stray line from any subprocess
fails the gate. That happens after the reviewer approved and after the command's side
effects ran. Gate commands are write-once shell scripts that nobody can test before
approval. With a remote heredoc it is easy to miss one `>&2`. The `/sase_gate` skill
already says "diagnostics belong on stderr", and that guidance did not prevent this
failure three times. The fix has to be in the mechanism.

**Failure surfacing makes it worse.** The `invalid_command_output` record drops the exit
status (`returncode: null`). The failure note says only "option approve failed:
invalid_command_output". A reviewer can't tell that the command finished its work, so
pressing `R` looks like a reasonable retry, when in fact it reruns the whole install and
fails again.

**Current state.** The gate failed, so its `turn.next` F11 measurement never launched.
Two gate turns are still `settling` and still hold workspace claims: `1a8--gate-0`
(`custom-92df387c`) and `1ab--gate-0` (`custom-61fd7f53`). Athena's global `sase` is an
editable install of `~/projects/github/sase-org/sase`, currently at `8f4e6503d3`. A
Python-only executor fix goes live on athena once that checkout is pulled, with no
reinstall needed.

**Rust boundary.** Gate command execution and result decoding exist only in this repo's
Python. sase-core has no gate command decoder, so nothing here crosses into `sase_core`.

## Changes

### 1. Accept a last-line JSON result for options and actions

In `src/sase/notification_gates/command_runner.py`, change
`decode_json_result(value: bytes)` as follows:

1. Parse `json.loads(value)` exactly as today, so a whole-stdout value (including
   pretty-printed JSON) keeps working.
2. If that raises `json.JSONDecodeError` or `UnicodeDecodeError`, split the **bytes** on
   `b"\n"`, drop lines that are empty after stripping, and `json.loads` the last
   remaining line. Splitting bytes keeps non-UTF-8 junk in earlier lines from blocking a
   clean result line.
3. If there is no such line, or it does not parse either, re-raise the **original**
   exception so both callers' `except (UnicodeDecodeError, json.JSONDecodeError)`
   clauses still match.

Update the docstring to state the contract: the result is all of stdout as one JSON
value or, failing that, the compact JSON value on stdout's last non-empty line. Nothing
may follow it. Both callers keep validating the decoded value against `result_schema`,
so a stray JSON-looking line still has to satisfy the declared schema.

Reword both failure messages, keeping the existing prefix so current `match=` assertions
keep working:

- `attempts.py` (both the recorded `GateError` and the raised one):
  `"command stdout must contain one JSON value, as all of stdout or its last line; send other output to stderr"`.
- `operations.py`: the same text with `action` instead of `command`.

### 2. Say when the command exited 0

- `attempts.py`, invalid-output branch: pass `returncode=completed.returncode` to
  `record_failure_outcome`.
- `operations.py`, invalid-output branch: pass `returncode=completed.returncode` to
  `record_execution_error`.
- `failure_outcome.py::_redacted_message`: for `invalid_command_output` with
  `returncode == 0`, return
  `f"option {option_id} exited 0 without a JSON result on stdout; its work may already have run"`.
  Keep the fixed-summary rule: no stdout or stderr content in the message. Other codes
  and the non-zero path stay unchanged.

### 3. Tests

Add `tests/test_notification_gate_command_output.py` so
`tests/test_notification_gate_execution.py` doesn't grow. Use the existing `create_gate`
/ `gate_spec` / `execute_gate_selection` helpers and the shared `gate_home` fixture
(`tests/_conftest_shared.py`). Cover:

- **Regression (the real failure).** A `#!/bin/sh` command prints git-shaped lines taken
  from the real record ("Your branch is up to date with 'origin/master'.", "Already up
  to date.", "Updating 8f4e6503d3..99db9e685d", "Fast-forward", " INSTALL.md | 3 +++"),
  then `printf '{"status":"ok"}\n'`. Execution succeeds, and `option_results` carries
  `{"status": "ok"}`.
- A pretty-printed whole-stdout JSON result is still accepted.
- A non-UTF-8 byte line before a valid JSON line is accepted.
- Output **after** the JSON line fails with `invalid_command_output`. Its
  `errors/*.json` record has `returncode == 0`, and the current failure message (journal
  `attempt_failed` / `current_gate_execution_failure`) contains `exited 0`.
- A last-line JSON result that violates `result_schema` (for example
  `{"status":"nope"}`) is still rejected.
- **Action path.** Add one case to `tests/test_gate_operations.py`, following its
  existing operation fixtures: an action whose command prints a chatty preamble and then
  `{"summary":"x"}` succeeds through `execute_gate_operation`.

Update `test_hash_mismatch_and_malformed_output_leave_gate_answerable` only if its
`match=` stops matching. With the prefix kept, it shouldn't.

### 4. `/sase_gate` skill: `src/sase/macros/skills/sase_gate.md`

Replace the sentence "A command receives the gate input as JSON on stdin and must print
exactly one JSON value on stdout; diagnostics belong on stderr." (in the paragraph that
begins "Command `argv` arrays are executed without a shell") with:

> A command receives the gate input as JSON on stdin and prints its result as one JSON
> value on stdout: either all of stdout, or one compact line that is stdout's last line,
> with nothing after it. Send everything else to stderr. A remote `ssh host sh <<'EOF'`
> script's output lands on your stdout, so redirect its chatty steps (`git pull`,
> builds) with `>&2`, or start the script with `exec 3>&1 1>&2` and print the result
> with `>&3`.

In the actions paragraph, change "its stdout must be one JSON value" to "its stdout
follows the same result rule". Leave the rest of the skill alone. Do not run
`sase skill init --force` or `chezmoi apply` from this workspace: generated skills
deploy only from a landed tree.

### 5. Reference docs: `docs/reference/automation/notification-gates.md`

- In the **Gate actions** `run_command` bullet, change "Its stdout must be one JSON
  value validated against…" to say that stdout carries one JSON value: all of stdout, or
  a compact value on its last line after other output. It is still validated against the
  action's `result_schema`.
- In **Gate inputs**, after the paragraph that starts "An option's command reads its
  input as **stdin JSON**", add two or three sentences. They state the same result rule
  for options. They also say that a command which exits 0 without a valid result fails
  with `invalid_command_output` after its side effects have already run, and that the
  failure note says so.

### 6. Record the outcome on `sase-1s5.3`

Read `sase_beads.md` through `/sase_memory_read` first. Then run:

`sase bead note sase-1s5.3 "F11 apollo reinstall: gate custom-61fd7f53 installed apollo at 2026-11-09 09:38 EST (sase 99db9e6, sase-core 5c0e49c, gateway restarted), but the gate failed invalid_command_output because remote git checkout/pull output reached stdout, so its F11 follow-up never ran. Fix: the gate executor now accepts a last-line JSON result."`

### 7. (`f11_followup` = launch) Request the F11 measurement agent

> [!decision] f11_followup = launch Do this last, after `sase tool run check` passes.
> Use the `/sase_run` skill and follow it exactly. If the skill reports Staged, finish
> through `/sase_final`.

Read the prompt from the failed bundle without editing it: `turn.next.prompt` in
`~/.sase/interaction_requests/custom/custom-61fd7f53-f7db-473c-a0d4-7f869d84e055/request.json`.
Launch one agent in project `gh_sase-org__sase` with `%model:@medium`. Its prompt is
this preamble followed by that `turn.next.prompt` text, word for word:

> Apollo was reinstalled for sase-1s5 F11 at 2026-11-09 09:38 EST (14:38 UTC): sase
> 99db9e6, sase-core 5c0e49c, gateway restarted. Gate custom-61fd7f53 did this. Its
> approve command succeeded; the gate itself failed only on output parsing, so its own
> follow-up never ran. Measure from that install time. Step 1's sleep is already
> satisfied once 15 minutes have passed since then.

Do not stage another reinstall gate, ssh to apollo, reinstall anything, or cancel any
gate yourself. Cancelling the failed gates is the reviewer's call.

> [!decision] f11_followup = restart Skip the launch. The final report must tell the
> user to do these steps after this change lands. (1) Pull
> `~/projects/github/sase-org/sase` on athena to the landed commit; the change is
> Python-only, so no reinstall is needed. Restart the TUI if it runs gate executions
> in-process. (2) Press `R` on gate `custom-61fd7f53`'s "Gate execution failed" dialog.
> That reruns the whole apollo reinstall (about 13 minutes, including scheduler and
> gateway restarts) and then fires its F11 follow-up.

### Final report (both branches)

- Say plainly that apollo's reinstall already succeeded at 09:38 EST, and that only the
  gate's output parsing failed.
- Cancel `custom-92df387c` (`1a8--gate-0`) with `c` in its "Gate execution failed"
  dialog. Don't use `R`, because it still lacks `-y`. With `launch`, cancel
  `custom-61fd7f53` (`1ab--gate-0`) the same way. Both hold workspace claims until then.
- After landing: run `sase skill init --force` from a clean, landed tree to deploy the
  skill text. The executor fix is live on athena only after its sase checkout is pulled.

## Out of scope (deliberately)

- **A separate result channel** (a `SASE_GATE_RESULT` file or fd 3 contract). It would
  be cleaner, but every author would have to adopt it. The last-line rule fixes the
  commands agents already write, including the skill's own examples.
- **Linting command scripts at `sase gate create`.** Creation can't know which
  subprocesses print to stdout.
- **Pretty-printed JSON after other output, or any output after the result line.** These
  still fail, and the new message says how to fix them.
- **Making `R` detect work that is already done.** Rerun semantics stay the same; the
  new failure summary warns the reviewer instead.
- **Memory changes.** No memory note or decision record states the strict stdout rule
  (checked `sase/memory/`). The skill is a generated source, not memory.

## Verification

- Fast loop (run `just install-venv` first if the venv is stale):
  `.venv/bin/pytest tests/test_notification_gate_command_output.py tests/test_notification_gate_execution.py tests/test_gate_operations.py tests/gate_conformance tests/main/test_init_skills_source_content.py -q`.
- **Real-evidence check (read-only).** Feed the `stdout` field of
  `~/.sase/interaction_requests/custom/custom-61fd7f53-f7db-473c-a0d4-7f869d84e055/errors/1794235110285255565-d86324a7f7124b4b84ce3b708e1a02e8.json`,
  encoded to bytes, through the new `decode_json_result`. It must return
  `{"status": "installed"}`.
- Run `just fmt`, then `sase tool run check` (the agent default; do not run
  `check-full`).
