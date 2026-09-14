---
tier: tale
title: ACE sudo review and terminal authentication handoff
goal:
  Typed sudo requests can be reviewed command by command in ACE and executed through a
  credential-free terminal handoff with accurate status and failure recovery.
size: medium
proposed_by: bbugyi200.athena.sase-110.4
bead: sase-110.4
status: done
---

- **PARENT:**
  [202609/agent_sudo_requests.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_sudo_requests.md)
- **BEAD:**
  [sase-110.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-110/sase-110.4.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-110.4](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-110.4.md)
- **COMMITS:**
  - [bfd22d8](https://github.com/sase-org/sase/commit/bfd22d8df3f168ac232ec428ca83944d5d650b5a)
    — feat(sudo): add ACE review terminal handoff

# ACE sudo review and terminal authentication handoff

## Objective

Complete phase `sase-110.4` by giving typed `SudoRequest` notifications a bespoke ACE
review experience and a synchronous, terminal-attached Authenticate-and-run path. The
implementation must preserve the approved security boundary: ACE never collects or
transports credential material, every selected command is visibly reviewed, and
authentication failures or cancellation leave the gate pending and answerable.

## Implementation

1. Extend the typed sudo answer contract for ACE without weakening the existing gate
   validation. Add an explicit run mode and reviewed command-selection input to the
   `sase sudo answer` parser/handler, materialize a manifest containing only the chosen
   commands in their original order, and keep receipt/hash validation tied to exactly
   that subset. Preserve headless deny support, require a controlling TTY before any
   approve/run work, and return structured outcomes that distinguish completion, command
   failure, authentication failure, cancellation, timeout, lock contention, and runner
   errors while leaving non-terminal failures unsettled.

2. Add a single-active sudo-authentication lease under the SASE home directory. Acquire
   it non-blockingly around the terminal handoff, record enough non-secret request/host
   identity to produce a useful contention message, release it reliably on success,
   exception, and interruption, and make stale ownership recoverable without ever
   persisting credential input, authentication transcripts, hashes, lengths, or attempt
   counts.

3. Implement a dedicated `SudoRequestModal` and immutable modal-data/result models. Load
   and hash-verify the gate bundle off-pump with `asyncio.to_thread`, project the
   manifest into an all-selected command checklist, and render the untrusted reason,
   verified host/run-as/cwd/environment policy, requester/project/expiry, risk badges,
   concise argv rows, and a full-detail view for argv/environment/snapshot hashes. Give
   Space command-toggle semantics, `v` detail toggling, `d` deny-with-feedback, and an
   always-explicit primary **Authenticate & run** action; refuse an empty selection.
   Export/style the modal through the existing ACE modal conventions, including narrow
   layouts and deterministic visual-fixture support.

4. Dispatch `SudoRequest` from the notification modal flow before the generic custom
   gate path. On denial, use the existing headless-safe gate execution path. On
   approval, dismiss the modal and enter `app.suspend()` (handling
   `SuspendNotSupported`), print the trusted SASE banner, and run
   `sase sudo answer <id> --run` with the selected command IDs attached to the
   controlling terminal rather than through a durable proc. Restore ACE in every
   outcome, refresh the notification/agent fast paths, and show an honest completion or
   retryable-error toast; never auto-retry and never settle after auth failure, Ctrl-C,
   timeout, or missing TTY.

5. Wire typed gate status presentation so local running listings, agent-list rows,
   Focus/detail phase dividers, and gate sections display `SUDO` while pending and
   `SUDOED` after successful settlement instead of collapsing typed gate shells to the
   generic `GATE`/`WAITING INPUT` labels. Keep this data-driven from the gate shell's
   recorded status pair so unrelated gate kinds retain current behavior and no new
   zero-pending refresh or keystroke work is introduced.

6. Add the standing custom-gate warning when agent-authored title, notes, query, option
   labels, or preview text asks for a sudo/root password. Render **Never enter your
   system password here** prominently without treating ordinary typed `SudoRequest` text
   as a credential field or adding any secret-input plumbing.

## Verification

- Add focused sudo CLI tests for subset ordering/hash binding, all-selected defaults,
  TTY refusal, authentication/cancellation pending behavior, successful settlement, and
  single-active lease acquisition/release/contention.
- Add ACE loader/dispatch/modal tests for off-pump bundle verification, command
  checklist toggling, full-detail rendering, empty-selection refusal, deny feedback,
  suspend success, `SuspendNotSupported`, Ctrl-C/auth failure, accurate toasts, refresh,
  and proof that the durable gate proc path is never used for approval.
- Add status/listing/Focus tests for `SUDO` and `SUDOED`, regression coverage for
  generic gate labels, and custom-gate password-warning positive and false-positive
  cases.
- Add PNG snapshot cases for the sudo review modal and pending/settled lock-panel state,
  following the existing visual suite conventions.
- Run the targeted sudo and ACE test modules, formatting/lint checks, `just check`, and
  the relevant visual snapshot assertions. Confirm the TUI stall watchdog remains clean
  during a manual fake-runner suspend/resume check and that the zero-pending path gained
  no polling, startup, refresh, or per-keystroke work.
- Before closing `sase-110.4`, run `sase bead epic-symbols sase-110.4`, resolve or
  re-key every remaining phase-owned symbol, and close only this phase with a note
  summarizing the verified tests and handoff behavior.
