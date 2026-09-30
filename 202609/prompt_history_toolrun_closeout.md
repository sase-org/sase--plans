---
tier: tale
size: small
title: Keep nested ToolRun launches out of prompt history and land sase-1d8
goal: A ToolRun command that invokes sase run creates no prompt-history row, and epic
  sase-1d8 is closed with its approved plan marked done.
proposed_by: bbugyi200.athena.sase-1d8.land
bead: sase-1d8
status: done
---

- **PARENT:**
  [202609/prompt_history_human_only.md](https://github.com/sase-org/sase--plans/blob/main/202609/prompt_history_human_only.md)
- **BEAD:**
  [sase-1d8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1d8/README.md)

# Remaining integration work

Epic `sase-1d8` implemented its four phases and all are closed. Its approved plan is
`plan:202609/prompt_history_human_only.md`. The landing review found one gap introduced
by later commit `018061f6f2` for active epic `sase-1cx`: an agent-only detached ToolRun
runs inside a proc, whose supervisor clears `SASE_AGENT`;
`sase.tool.executor_process.child_env` then sets `SASE_TOOL_RUN_ID` for the command. If
that command calls `sase run`,
`src/sase/main/query_handler/_launch.py::_sase_run_ingress_origin` currently classifies
it as `typed` because it checks only `SASE_MONITOR_ID` and `SASE_GATE_COMMAND`. The new
row violates this epic's human-only invariant. Classify a nonempty `SASE_TOOL_RUN_ID` as
generated at this ingress. Keep this check at `sase run`, since the central history
writer also serves human mobile and Telegram traffic through proc boundaries.

## Steps

1. In `_launch.py`, use the existing `SASE_TOOL_RUN_ID` marker from
   `sase.tool.executor_process` (or a local constant if importing that module would
   create an import cycle) in `_sase_run_ingress_origin`. Update its docstring. Confirm
   a plain terminal `sase run` still resolves to `typed`, and a blank marker does not
   suppress history.
2. Extend `tests/main/test_sase_run_history_ingress.py`: clear `SASE_TOOL_RUN_ID` in its
   environment helper; cover nonempty and empty marker values; run the mocked launcher
   path to verify a nested ToolRun records no row even after `SASE_AGENT` is absent.
   Verify a payload claiming `history_origin=typed` cannot upgrade it. Run the targeted
   tests and `just check` (do not run `just check-full`). If `just check` stops in the
   already-recorded `sase-1cj.12` prediction validator skew, record that known external
   failure and the targeted result on `sase-1d8`; do not change prediction code as part
   of this epic.
3. Finish the landing in this same coding turn. Review the existing `sase-1d8` LAND
   TRIAGE AND INTEGRATION note and confirm the four phase implementations and concurrent
   commits still match the approved plan. Run `sase bead epic-symbols sase-1d8`; resolve
   or re-key every listed entry to a still-open later bead. Close with
   `sase bead close sase-1d8 --note "Verified generated write gate, canonical-text launch paths, TUI provenance, safe prune and current ToolRun ingress integration; all phase notes and follow-ups triaged; no unresolved epic symbols"`.
   Do not force the close merely to make it pass. Run `just symvision` after closing and
   distinguish this epic's whitelist from the seven unrelated active `sase-1cx`
   unused-public symbols already recorded there. Finally use `sase repo open plans` and
   set `status: done` in the frontmatter of `plan:202609/prompt_history_human_only.md`.
   `sase-1d8` has no `parent_bead`, so no ancestor close is needed.
