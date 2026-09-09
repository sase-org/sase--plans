---
tier: tale
title: Finish the queue-directive landing prerequisites
goal:
  The published dependency floor, Neovim completion coverage, and retired flag state all
  match the unconditional queue-directive contract.
size: medium
proposed_by: bbugyi200.athena.sase-yj.land
bead: sase-yj
create_time: 2026-09-09 06:57:54
status: wip
---

- **PARENT:** [202609/queue_directive.md](queue_directive.md)
- **BEAD:**
  [sase-yj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yj/README.md)

# Plan: Finish the queue-directive landing prerequisites

## Context

Epic `sase-yj` moved queue admission from `%wait` to `%queue` / `%q`. Its four phases
are closed, but the landing audit found three concrete gaps that belong to the epic:

- `pyproject.toml` and `uv.lock` stop at `sase-core-rs` 0.32.50. That release introduced
  the queue APIs but still gates `%queue`, advertises `runners` and `priority` on
  `%wait`, and omits queue completion when the flag is absent. The unconditional
  contract first appears in the published `v0.32.52` tag. Running the
  queue/runtime/editor corpus against the locked 0.32.50 wheel currently gives 9
  failures and 33 passes.
- The approved plan requires a real Neovim/LSP queue-completion smoke regression, but
  `sase-nvim` has no queue-specific test or `%queue` / `%q` assertion.
- Temporary flag bead `sase-yl` remains open even though the Python registry entry and
  disabled branch were removed.

The only child proposal, from `sase-yj.1`, asked that V1 remote dispatch reject `%queue`
as wait-equivalent. Current `scan_dispatch_directive` and
`tests/test_xprompt_dispatch_directive.py` already do that, so it needs no new task.

## Outcome

The minimum published Rust dependency provides the final unconditional queue contract,
regressions prevent the floor from moving below that release, Neovim proves queue
completion through its real LSP client, and the retired flag bead is closed only after
those facts are verified. Do not close `sase-yj`, run its final Symvision landing pass,
or mark its linked epic plan done here; those remain with the resumed land agent.

## Implementation

1. In `sase`, raise the lower `sase-core-rs` bound from 0.32.50 to 0.32.52 and refresh
   `uv.lock` through the normal uv workflow. Add a focused landing guard, following the
   existing published-core-floor pattern, that asserts the declared lower bound can
   never fall below 0.32.52. Keep the upper compatibility window unchanged.
2. Prove the dependency fix against the published wheel, not only the linked Rust
   checkout: install/sync the locked dependency without local sources and rerun
   `tests/test_queue_directive.py`, `tests/test_xprompt_directive_contract.py`,
   `tests/test_xprompt_directive_completion_parity.py`, and
   `tests/test_xprompt_dispatch_directive.py`. Then restore the normal linked-core
   development install before running repository checks.
3. Open `sase-nvim` with `sase repo open` and add a small headless smoke test modeled on
   its existing LSP smoke files. Start the real `sase-xprompt-lsp` through the plugin,
   request completion from a live buffer, and assert at minimum that queue name/alias
   completion is visible, `%queue(` and `%q:` expose the expected queue fields or
   numeric values, and `%wait(` does not expose `runners=` or `priority=`. Apply at
   least one returned text edit to the live Neovim buffer and assert the resulting queue
   syntax so the test covers client edit handling rather than only inspecting raw JSON.
4. Reconfirm that `queue_directive` is absent from the SASE feature-flag registry and
   that no disabled queue branch remains. Close `sase-yl` normally with a note naming
   the unconditional runtime/editor contract, the 0.32.52 floor, and the passing
   published-wheel and Neovim evidence.

## Verification

- Run the new Neovim smoke with its documented
  `nvim --headless -u NONE -c "set rtp+=." -l ...` entry point and a real current LSP
  command.
- Run the focused published-wheel queue/runtime/editor tests before restoring the
  linked-core install.
- Run `just install`, the focused queue/directive/dispatch tests again against the
  linked core, and `just check` in `sase`.
- Confirm `sase bead show sase-yl` reports `closed` and both changed repositories are
  cleanly handed to host-owned finalization.
