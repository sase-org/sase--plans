---
tier: tale
title: Settle approved sudo gates whose command output exceeds the gate input budget
goal:
  An approved sudo gate settles and launches its successor whenever its completed ledger
  is valid, whatever the command output size. Trimmed output is explicitly marked, and
  any detached finalize failure raises a visible, deduped notification saying whether
  the privileged commands already ran.
size: medium
proposed_by: bbugyi200.athena.134
proposed_with: claude/opus
create_time: 2026-10-23 09:48:49
status: wip
---

# Fix: approved sudo gates with large command output never settle

## Problem

Bryan approved the `bob-cli-an.3` sudo gate
(`sudo-193ab068-d2b6-4f0c-b767-3e9e47911d88`, project `gh_bobs-org__bob-cli`) at least
twice. Each time the password was accepted and then nothing happened: the gate stayed
`SUDO`/pending, no successor agent launched, and no notification explained why. It
looked as if the privileged commands never ran.

## Diagnosis (verified)

**The suspicion is wrong.** The privileged commands ran both times. Settlement is what
failed afterward, silently.

Evidence:

- Two detached finalize procs ran for this gate: `j2fszrw6r1dk` (09:26) and
  `8h6q8p147d46` (09:32). Both exited 1 (`sase proc show 8h6q8p147d46`).
- The executor handoff ledger
  (`~/.sase/sudo/exec/sudo-193ab068-d2b6-4f0c-b767-3e9e47911d88/ledger.json`) has
  `outcome: "completed"`. All six commands (`daemon-reload`, `verify-units`,
  `nft-ruleset`, `iptables-v4`, `iptables-v6`, `listeners`) have `status: "ran"` and
  `exit_code: 0`.
- The bundle `journal.jsonl` and `errors/*.json`, plus the tail of both proc logs, show
  the same failure each time: `code: input_too_large`, `target: option approve input`,
  `input is 66387 bytes; at most 65536 are allowed`. Recomputing the canonical JSON of
  `{"command_ids": [...6 ids], "receipt": <ledger>}` gives exactly 66,387 bytes.
- The request used `output_to_agent: "full"`. The output tails were 26.9 KB (nft), 16.9
  KB (iptables), 14.6 KB (ip6tables) and 6.5 KB (ss), so the ledger was 66,275 bytes.

Root cause: a contract mismatch between the Rust sudo ledger bounds and the generic
Python gate input bound.

1. `sase-core` `crates/sase_core/src/sudo.rs` allows up to `SUDO_MAX_OUTPUT_TAIL_BYTES`
   (64 KiB) per entry under `full` (`ledger_tail_bound` in
   `crates/sase_gateway/src/sudo_runner/execution.rs`; `tail` is 8 KiB per entry). The
   whole ledger may be up to `SUDO_LEDGER_MAX_BYTES` (256 KiB).
2. On the host, `sase.sudo.answer_ops.apply_approved_receipt` passes the whole ledger as
   the approve option's input:
   `option_inputs={"approve": {"command_ids": ..., "receipt": ledger}}`.
3. `accept_gate_decision` (`src/sase/notification_gates/decision.py`) and
   `execute_gate_selection` (`src/sase/notification_gates/executor.py`) run
   `check_input_bounds`. That enforces `MAX_INPUT_BYTES = 64 KiB`, a bound designed for
   reviewer-authored input (`src/sase/notification_gates/input_bounds.py`). Any ledger
   whose JSON is larger than about 64 KiB is rejected deterministically, every time.
   Even `tail` can trip this once there are about 8 or more chatty commands.

Contributing defects:

- **Invisible failure.** The pre-acceptance rejection is recorded through
  `recorded_rejection`, which writes `errors/` only and publishes no notification.
  `detach_finalize._record_finalize_failure` adds a journal `operation_ran` event and
  likewise publishes nothing. The reviewer saw "execution started" in the TUI and then
  silence. The only signal was a red finalize proc.
- **Retries rerun privileged commands.** Each re-approval clears the dead attempt
  (`claim_execution_record`) and reruns every selected command. This is documented
  behavior (`docs/reference/automation/sudo.md`: "Every approval runs the full selected
  command set again"). Combined with a deterministic settlement failure, though, it
  becomes an endless re-execution loop. Here it was harmless (all six commands are
  read-only or idempotent), but this plan does not change it; see follow-ups.

## Goals

- A valid, completed sudo ledger always settles, whatever its command output size, on
  local, remote, detached and foreground paths, and with ledgers from older runners.
- The settled receipt, `response.json`, and the successor's context stay bounded. When
  output is trimmed, the trimming is explicit, never silent.
- A detached finalize failure (any cause) produces a visible, deduped notification. It
  says whether the commands already ran and how to recover.

Non-goals: changing rerun-on-reapproval semantics; changing the generic gate
`MAX_INPUT_BYTES`; changing the runner binary's per-entry caps.

## Design

### 1. `sase-core`: deterministic ledger fitting (Rust owns the ledger contract)

Open the linked repo with `sase repo open sase-core -r "<reason>"` and read its
`AGENTS.md` first. Ledger shape and output truncation already live in `sase_core`
(`truncate_sudo_output_tail`, `validate_sudo_ledger`), so the fitting algorithm belongs
there too, per the Rust core boundary rule.

In `crates/sase_core/src/sudo.rs`, next to `truncate_sudo_output_tail`:

- Add
  `pub fn fit_sudo_ledger_output(ledger: &SudoLedgerWire, max_json_bytes: usize) -> Result<SudoLedgerWire, SudoWireError>`,
  plus a JSON-value wrapper
  `sudo_fit_ledger_json_value(ledger: &JsonValue, max_json_bytes: usize)` that parses
  and validates like `sudo_validate_ledger_json_value`. Re-export both from
  `crates/sase_core/src/lib.rs` beside the existing sudo exports.
- Semantics:
  - Validate the input ledger first.
  - If `serde_json::to_vec(ledger).len() <= max_json_bytes`, return it unchanged.
    Fitting must be idempotent.
  - Otherwise compute the bytes available to all `output_tail` fields combined. Allocate
    them **max-min fair (water-filling)** on each tail's JSON-escaped length: small
    outputs (for example a 375-byte `systemctl show`) stay whole, and the largest
    outputs share the trimming. Break ties deterministically by entry order.
  - Trim each over-cap tail from the **front** so the end of the output survives,
    respecting UTF-8 boundaries and JSON escaping. Prefix it with a fixed marker line
    such as `[sase sudo: <N> earlier bytes omitted to fit the settlement budget]\n`,
    where N counts omitted original UTF-8 bytes. Marker bytes count against the budget.
    The marker must not contain any credential word followed by `=` (see
    `_CREDENTIAL_WORDS` in sase's `src/sase/sudo/models.py`).
  - Never change `id`, `status`, `exit_code`, `duration_seconds`, `outcome`,
    `request_id`, `manifest_sha256` or `diagnostic`.
  - Return `SudoWireError::validation` if the ledger cannot fit even with every tail
    emptied. This cannot happen within the current id and command-count caps; it is a
    guard.
  - Re-validate the result with `validate_sudo_ledger`. Assert in tests that its
    serialized size is `<= max_json_bytes`.
- In `crates/sase_core_py/src/sudo/mod.rs`, add a binding
  `#[pyo3(name = "sudo_fit_ledger", signature = (ledger, max_json_bytes))]` that mirrors
  `py_sudo_validate_ledger` (`py_to_json_value`, `allow_threads`,
  `sudo_error_to_pyerr`), and register it in the module init.
- Tests:
  - Rust unit tests in `sudo.rs`: no-op when it fits; idempotence; water-filling keeps
    small entries intact; marker and omitted-count correctness; multibyte UTF-8 and
    control-character-heavy output (escape expansion); determinism; the
    impossible-budget error; the result passes validation and fits.
  - A binding test in `crates/sase_core_py/src/sudo/tests.rs`.
  - Run `sase tool run check` in sase-core before finishing (never bare `cargo` or
    `just check`).

The runner binary does not change. Fitting happens on the host at settlement time, so it
also covers remote targets and older installed runners.

### 2. `sase`: fit the receipt at every settlement entry point

- `src/sase/sudo/core.py`: add `fit_ledger(ledger, max_json_bytes) -> dict` to the
  `SudoCoreBinding` protocol and `_RustSudoCoreBinding`. Map binding errors to
  `GateError("invalid_sudo_ledger", "ledger", ...)` like `validate_ledger` does.
- `src/sase/sudo/receipt.py`: add
  `fit_sudo_receipt_for_settlement(receipt, *, selected_command_ids, core=DEFAULT_SUDO_CORE) -> dict`.
  - Budget = `MAX_INPUT_BYTES` (import from `sase.notification_gates.input_bounds`)
    minus `len(canonical_json_bytes({"command_ids": [...], "receipt": {}}))`, minus a
    small safety margin of about 512 bytes for float-formatting differences between
    serde and Python's canonical JSON.
  - Call `core.fit_ledger`, then verify the real approve input with
    `check_input_bounds({"command_ids": ..., "receipt": fitted}, "option approve input")`.
    If it still fails, raise a clear `GateError("sudo_ledger_unfittable", ...)`. That
    should be unreachable.
  - Pass non-mapping receipts through untouched so existing validation reports them.
- `src/sase/sudo/detach_finalize.py` `_run_finalize`: right after
  `wait_for_executor_ledger` / `wait_for_remote_sudo_ledger`, and **before**
  `_authorize_finalize_settlement`, replace `receipt` with the fitted receipt. The
  settlement `authorization_id` is a digest over the ledger, so it then covers exactly
  what becomes durable. Keep this inside the existing `try`, so a fit failure is
  recorded like any other finalize failure. Fitting is deterministic, so replays and
  resumes produce the same authorization id.
- `src/sase/sudo/answer_ops.py` `apply_approved_receipt`: fit before
  `validate_sudo_receipt` and `execute_gate_selection`, using `selected_command_ids`.
  This covers the foreground `--no-detach` path in `src/sase/sudo/cli.py` (`_approve`)
  and the remote paths, and is a no-op for the already-fitted detached path.

### 3. `sase`: make detached finalize failures visible

Extend `_record_finalize_failure` in `src/sase/sudo/detach_finalize.py` (pass it the
receipt when one was read, plus the `state`) to also publish one notification through
`sase.notifications.store.upsert_notification`. Mirror `publish_gate_execution_failed`'s
dedup and `plus_one_note` pattern, but do **not** reuse the generic
`GateExecutionFailed` action: its recovery commands are `sase gate answer ...`, which
refuses sudo approvals. The notification:

- uses `action="ViewErrorReport"`. Set `action_data["error_report_path"]` to the
  finalize proc's log (`sase.procs.logs.proc_log_path(state.finalize_proc_id)`), which
  holds the streamed command output plus the error JSON. Fall back to the bundle's
  `errors/` record. Also include `request_id`, `request_kind: "sudo"` and `bundle_path`.
- has notes that state the gate and lane, the failure `code: message`, and whether the
  commands already ran. Derive the last from the receipt: "commands ran: 6/6 exit 0
  (ledger outcome completed)", or "no ledger; commands may have partially run". Also
  give recovery: `sase sudo answer <id> --run` reruns the selected commands,
  `--command <id>` selects a subset, and `--deny` denies.
- has a deterministic id or `dedup_key` per gate, so repeated failures bump one row
  instead of stacking.
- is dismissed when the gate later settles by any sudo path: successful
  `apply_approved_receipt` and the sudo deny path. Use a small helper next to the
  publisher.

Publishing must be best-effort: wrap it in `try` and log a warning. It must never mask
the original `GateError` or change exit codes.

### 4. Docs and agent skill

- `docs/reference/automation/sudo.md`: read the `docs_writing` reference memory first.
  - Document that the settled ledger is fit to the gate input budget (about 64 KiB
    shared by all selected commands) with water-filling and an explicit omission marker,
    and that full streamed output remains in the finalize proc log.
  - Document that a detached finalize failure now raises a notification saying whether
    the commands ran.
  - Keep the existing rerun-on-reapproval text accurate.
- `src/sase/macros/skills/sase_sudo.md` is a generated skill source; read the
  `generated_skills` reference memory first. Amend the `output_to_agent` bullet: `full`
  is still bounded by a shared ~60 KiB settlement budget. Prefer targeted commands (for
  example `nft list chain ...` or filtered `ss`) over whole-system dumps.

### 5. Follow-up task beads (file through `/sase_new_task`)

- **Settle from the preserved ledger.** Re-approving a gate whose executor already wrote
  a completed ledger, but whose settlement failed, reruns every privileged command.
  Consider settling from the preserved terminal ledger. This is a security-sensitive
  behavior change (default size `large`).
- **Credential-shape false positive.** `contains_credential_shape` scans command
  **output**, so any output containing `token=`, `secret=`, `password=` and so on
  rejects a completed receipt with `credential_field_rejected`. That strands the gate
  the same way. Consider redacting the output instead of rejecting the ledger (size
  `large`).

## Tests (sase)

- `tests/test_sudo_detach.py` (or a new focused module if that file is near the `toobig`
  cap):
  - Reproduce the incident: a fake executor ledger with six entries whose canonical
    approve input is over 65,536 bytes (shape it like the real one: 26.9K/16.9K/14.6K/
    6.5K/375/0). Assert that finalize settles the gate; that `response.json` carries the
    fitted receipt with the omission marker on the largest entries; that small entries
    are intact; that the approve input is `<= MAX_INPUT_BYTES`; and that no failure
    notification exists.
  - A finalize failure (for example a forced `GateError` from the fake core) publishes
    exactly one `ViewErrorReport` notification. A second failure updates the same row. A
    later successful settlement dismisses it.
- `tests/test_sudo_gate_answer.py`: the foreground (`--no-detach`) path with an
  oversized ledger settles.
- `tests/test_sudo_core.py`: the `fit_ledger` adapter calls the binding and maps binding
  errors to `GateError`.
- Existing fakes implementing `SudoCoreBinding` in tests (see
  `tests/_sudo_gate_helpers.py`, `tests/_sudo_acceptance_helpers.py`) need `fit_ledger`.
  Prefer delegating to the real binding where the fake already does so for validation.

## Verification

- In `sase-core`: `sase tool run check`.
- In `sase`: rebuild the local `sase_core_rs` from the linked `sase-core` checkout so
  the new `sudo_fit_ledger` binding is importable (see `docs/rust_backend.md`, e.g.
  `just rust-install`), then `just check`. Read the `lint_and_test` reference memory
  first. A turn that commits both repos gets the `sase-core-revision.txt` pin bumped by
  the host automatically; do not hand-edit it.
- Manual sanity check (read-only, no sudo): run the new
  `fit_sudo_receipt_for_settlement` against a copy of the real ledger at
  `~/.sase/sudo/exec/sudo-193ab068-d2b6-4f0c-b767-3e9e47911d88/ledger.json` (if it still
  exists) with its six command ids. Confirm that the resulting approve input is
  `<= 65,536` bytes and that `verify-units` and `daemon-reload` are untouched. Do not
  modify or delete the live gate bundle or handoff directory.

## Rollout note for Bryan (not an implementation step)

Global installs are human-only. After this lands, reinstall sase. Then re-approve the
stuck gate with `sase sudo answer sudo-193ab068-d2b6-4f0c-b767-3e9e47911d88 --run`; this
reruns the six read-only commands a third time. It should now settle and launch the
`bob-cli-an.3` successor.
