---
tier: tale
title: Close mini-xprompt landing gaps
goal:
  Mini-xprompt retargeting and target selection are conflict-safe, verified off-thread,
  and documented accurately.
size: medium
proposed_by: bbugyi200.athena.sase-rl.land
bead: sase-rl
create_time: 2026-08-21 07:00:10
status: wip
---

- **PARENT:** [202608/targeted_mini_xprompt.md](targeted_mini_xprompt.md)
- **BEAD:**
  [sase-rl](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rl/README.md)

# Close mini-xprompt landing gaps

Finish only the correctness and verification gaps found while landing epic `sase-rl`.
The existing target catalog, pane lifecycle, persistence flow, keymap, visuals, and
documentation remain the base implementation.

## Required changes

1. Make the name-panel verdict refuse a selected destination whose existing physical
   definition is incompatible with mini mode (for example, a shadowed Markdown swarm),
   even when a different effective definition with the same callable name is editable.
   The flow must never present that destination as a fork/override that can overwrite
   the incompatible definition. Add focused catalog/modal regression coverage for this
   precedence combination while preserving the established editable, read-only, fork,
   and fresh-create verdicts.
2. Give a retargeted mini pane the correct comparison/clean baseline for its new
   destination while preserving the user's current body and pane-scoped frontmatter. A
   previously clean draft that differs from the new target must become dirty, render
   accordingly, and trigger the generic auxiliary discard guard; an actually identical
   retarget may remain clean. Add model/widget coverage for both the body preservation
   and the resulting dirty-guard behavior.
3. Add the missing mini-save off-event-loop regression required by the original epic:
   exercise a deliberately slow or thread-observed mini definition read/write and prove
   the Textual loop remains responsive or that the filesystem operation runs on a worker
   thread. Keep all disk reads, hashing, serialization, and writes outside the serial
   app message pump.
4. Correct the raw-placeholder documentation to match the implemented contract: a fresh
   `gx` extraction converts placeholders in the copied origin body and seeds its
   inferred inputs before the mini pane opens; do not claim that arbitrary placeholders
   typed later are converted at save review unless the implementation is deliberately
   extended and tested to provide that behavior.

## Verification

- Run the focused mini-xprompt target-catalog/name-modal, pane lifecycle/model, save
  flow, and relevant documentation/help tests.
- Run `just install` before repository verification, then run `just check`.
- Leave no `--epic-symbol` exemption for this child work.
