---
tier: tale
title: Configurable per-branch gate-shell follow-up
goal:
  Gate shells launch the correct secure successor for each configured terminal branch.
size: medium
proposed_by: bbugyi200.athena.sase-ud.7
bead: sase-ud.7
create_time: 2026-08-26 19:47:53
status: wip
---

- **PARENT:** [202608/gate_shells.md](gate_shells.md)
- **BEAD:**
  [sase-ud.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ud/sase-ud.7.md)

# Configurable per-branch gate-shell follow-up

## Objective

Complete phase `sase-ud.7` by making a settled gate shell choose its successor policy
from the compiled terminal branch, compose a directive-safe decision prompt, and launch
that successor through the shared family-shell substrate. Preserve the gate-shell
contract that an unmapped branch launches nothing, `results` is the default output, and
the shell's terminal artifact is visible before a successor can resolve `#fork`.

## Implementation

1. Tighten the additive v3 `shell` model in `src/sase/notification_gates/model_shell.py`
   so each `branches.<key>` directly accepts `prompt`, `output`, `fork`, `model`,
   `status`, and `accent`. Keep top-level `next` as the inherited default for mapped
   branches, validate compiled and reserved (`timeout`, `stopped`, `failed`) keys at
   creation, normalize scalar/list output policies, and preserve explicit `prompt: null`
   as follow-up suppression.
2. Add a gate-specific prompt composer under `src/sase/gate_shell/` that renders fixed
   labelled decision metadata for answered, timeout, stopped, and failed outcomes;
   includes per-option result-schema JSON, the bounded untrusted log tail, and/or a
   retained-log pointer according to the composable `none|results|tail|file` policy;
   widens fences for hostile content; and places only the routing prefix outside one
   disabled xprompt region. Map `fork: family|shell|none` and an optional next model
   through the shared shell routing helpers without templating user data.
3. Add a gate follow-up launcher that reuses the shared family-successor launch,
   starter-settle, workspace-claim transfer/fallback, and dropped-prompt persistence
   machinery. Wire settlement to clear stale/default follow-up metadata for unmapped
   branches, apply mapped status/accent/policy, publish the decision record, chat, done
   marker, and artifact index before launch, then persist launch/degraded/failure
   disposition and release claims correctly for both `inherit` and `release` policies.
4. Add focused model, prompt-golden, launcher, and settlement tests. Cover answered
   results, results plus tail, no follow-up, timeout, stopped, failed, result/secret
   boundaries, hostile fence widening, family/shell/no-fork routing, explicit models,
   workspace inheritance/release, launch degradation, and artifact-before-launch
   ordering. Update existing gate-shell tests for the direct branch schema.

## Verification

- Run the focused gate-shell and notification-gate model/CLI tests while iterating.
- Run `just check` after all repository changes.
- Run `sase bead epic-symbols sase-ud.7`, resolve or re-key every remaining symbol, and
  close only `sase-ud.7` with a note naming the verified checks.
