---
tier: epic
title: Make the remote screenshot send-keys hint apostrophe-safe
goal: The printed remote send-keys hint carries an arbitrary single-line key through
  both the local and remote shells without syntax errors or argument splitting.
phases:
- id: shell-safe-key-template
  title: Preserve arbitrary key text across both SSH shell boundaries
  depends_on: []
  size: small
  description: 'shell-safe-key-template: repair the printed hint and prove apostrophes
    and other harmless shell metacharacters reach tmux as one exact argument.'
proposed_by: bbugyi200.athena.sase-123.7.6.land
parent_bead: sase-123.7.6
create_time: 2026-09-18 02:08:30
status: wip
bead_id: sase-123.7.6.5
---

- **PROMPT:** [prompts/202609/remote_send_keys_hint_quoting.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/remote_send_keys_hint_quoting.md)
- **PARENT:** [202609/screenshot_residual_contracts.md](https://github.com/sase-org/sase--plans/blob/main/202609/screenshot_residual_contracts.md)
- **BEAD:** [sase-123.7.6.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-123/sase-123.7.6.5.md)

# Plan: Make the remote screenshot send-keys hint apostrophe-safe

This is the only remaining implementation gap found while landing `sase-123.7.6`. Phase
`sase-123.7.6.2` correctly preserved the remote unique tmux target and added a
two-shell-boundary test, but the test covered double quotes and not apostrophes. The
current helper builds a fully quoted remote command containing `'<KEY>'`, quotes that
command again as the SSH argument, and then presents the result as a textual template.
Replacing `<KEY>` with `literal key's value | echo; true` breaks the outer local-shell
quote and `/bin/sh -n -c` reports an unterminated quoted string.

## Phase: Preserve arbitrary key text across both SSH shell boundaries {#shell-safe-key-template}

Update `src/sase/screenshot/remote.py` so the printed `remote_send_keys_hint` has an
explicit, usable substitution contract and does not embed the replacement point in
nested shell quotes that an apostrophe can terminate. Keep the actual remote tmux target
shell-quoted and keep SSH's remote-command joining behavior in mind. A suitable design
may carry the key through standard input or another single-boundary channel, with
`<KEY>` replaced by one ordinary local-shell-quoted token; prefer the smallest portable
POSIX-shell form and do not add a new runtime dependency. Preserve the existing
machine-readable output key and the unique `@window_id` target contract.

Extend `tests/main/test_screenshot_remote_command.py` to execute the printed hint
through the existing fake local SSH and remote shell harness after substituting a key
that contains spaces, an apostrophe, double quotes, a dollar sign, a pipe, and a
semicolon. Assert that fake tmux receives exactly one key argument with the original
bytes and that none of the metacharacters execute. Retain coverage for a target with
spaces, both quote styles, pipes, semicolons, and dollar signs. Keep a simple key such
as `Enter` convenient in the printed hint, and adjust the handler-output assertion to
the repaired representation without weakening it to a substring that would miss the
target or transport structure.

Run the focused remote screenshot command tests and the screenshot command lane. Then
run `just fix` and `just check` under the project verification rules. This child epic's
land agent must re-audit this one phase and use its managed parent-bead relationship to
resume the interrupted landing of `sase-123.7.6`; closing the parent epic, running the
parent Symvision/full verification, and marking the parent plan done are deliberately
outside this implementation phase.
