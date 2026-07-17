---
tier: tale
title: Stabilize residual-freeze soak verification and re-land sase-6j
goal: 'The residual-freeze integration soak reliably detects regressions in the fixed
  startup, modal, and loader-cleanup paths without failing on unrelated CPU contention
  in the parallel test suite, and epic sase-6j is re-landed only after the final integrated
  tree passes its full gates.

  '
create_time: 2026-07-17 09:33:38
status: wip
prompt: 202607/prompts/stabilize_residual_freeze_soak_landing.md
---

# Plan: Stabilize residual-freeze soak verification and re-land sase-6j

## Context

Epic `sase-6j` implemented stale-while-revalidate config tokens, compact hitch telemetry, post-apply loader cleanup with
bounded contention, pump-free startup and modal loading, and a lowered-threshold integration soak. Its implementation
and targeted tests are sound, including the Rust bounded artifact-index delete. The first full landing gate passed
before a late, independent prompt-panel lane ordering commit was integrated.

After fast-forwarding that late commit, the final `just check` run failed only
`tests/ace/tui/test_residual_freeze_soak.py::test_lowered_threshold_soak_keeps_fixed_paths_responsive`. The failure log
recorded a 0.607-second loop/pump hitch while Textual and Rich were rendering an ordinary `OptionList` under the default
16-worker pytest CPU load. It did not name any of the deliberately blocked startup, history, revive, or loader-cleanup
workers. The same soak passes in isolation in 4.73 seconds. The current assertion rejects every hitch anywhere during
the whole app session, so unrelated scheduler/render contention can fail the epic-path test.

The landing audit reopened child `sase-6j.5` and epic `sase-6j`, and restored
`sase/repos/plans/202607/tui_residual_freeze_elimination.md` to `status: wip`.

## 1. Make the soak precise and load-robust

Refine the residual-freeze soak so it continues to prove the product contract: each deliberately slow startup,
prompt-history, revive-agent, and loader-cleanup body must remain off the event loop and Textual message pump, user
input must be accepted while that body is blocked, and the fixed path must not produce a watchdog event. Scope watchdog
evidence to the controlled slow-work windows or otherwise distinguish a fixed-path regression from unrelated test-host
CPU contention. Preserve meaningful lowered thresholds and the invariant that each stub stays blocked longer than the
configured hitch threshold; do not solve the flake by disabling hitch detection or by raising thresholds beyond the
exercised delay.

Add or adjust focused tests for any new event-window/filtering helper so a real fixed-path hitch still fails. Keep the
existing direct responsiveness assertions (tab changes and modal typing while workers are blocked), since those are the
strongest end-user signal. Avoid production behavior changes unless the audit finds an actual product defect rather than
a test-harness defect.

## 2. Revalidate on the integrated tree

Re-run the soak repeatedly both alone and in a contention representative of the default parallel suite. Run the
phase-specific watchdog, config-cache, loader, startup/modal, and Rust bounded-delete tests as needed, then run
`just check` on the current integrated revision. Re-run the key-to-paint benchmarks if the fix touches production TUI
paths; a test-only precision fix need not reinterpret unrelated uppercase panel-navigation timings. Review commits that
land after `b7b64b719` and integrate any overlap before the final gate.

## 3. Re-land the epic

Only after the final integrated `just check` is green, record the remediation commit in `sase-6j.5`, close `sase-6j.5`,
and close the epic with `sase bead close sase-6j`. After closing, run `just symvision` so the expired epic-symbol
whitelist is evaluated; remove any stale whitelist entries or unused code it reports and re-run the proportionate
checks. Finally set `status: done` in `sase/repos/plans/202607/tui_residual_freeze_elimination.md` and verify the epic,
all children, plan frontmatter, and worktree state.

## Risks

- A threshold-only change can hide a real regression; the controlled worker duration and a negative regression case must
  keep the test sensitive.
- Filtering all generic render stacks would be too broad because a real UI-loop regression may also surface while
  rendering. Prefer explicit controlled windows and direct interaction deadlines over stack-name allowlists.
- The epic must remain open and its plan WIP until the final integrated gate is green; close the child before the parent
  and run Symvision after parent close.
