---
tier: tale
title: Finish unified notification-gate integration and land sase-6e
goal: 'The adapter-owned %auto contract is represented consistently in runtime-facing
  completion and documentation, all notification-gate repositories pass their integrated
  checks, and epic sase-6e is closed with post-close symbol cleanup and its original
  plan marked done.

  '
create_time: 2026-07-16 19:53:57
status: done
prompt: 202607/prompts/plan.md
---

# Plan: Finish unified notification-gate integration and land sase-6e

## Context

Epic `sase-6e` landed its seven implementation phases across SASE, `sase-core`, and `sase-telegram`. The current source
has the intended shared gate constructor, neutral request bundles, typed `EpicApproval`, hash-verified command
execution, mechanical producer waits, automatic resolution, and neutral-first legacy readers. The landed parser also
deliberately retains an arbitrary optional `%auto` argument so the gate adapter that eventually receives the request can
validate it.

The landing audit found one unfinished integration seam. Runtime and tests accept an opaque argument such as
`%auto:foo`, but ACE completion, the Rust editor directive registry, and `docs/xprompt.md` still describe `%auto` as a
globally closed, plan-only `plan|tale|epic` enum; the documentation even promises parser-time rejection. The three plan
spellings should remain useful compatibility suggestions, but they must not be presented as the global validation
boundary. Bare `%auto` must also remain capable of first-option question resolution, while launch approval continues to
reject automatic resolution and plan adapters continue to reject unknown or tier-changing arguments.

## Phase 1: Align `%auto` editor and documentation contracts

- In SASE, replace closed-enum naming such as `AUTO_MODES_ORDERED`/`AUTO_MODES` with a clearly named compatibility
  suggestion vocabulary. Remove the now-unused closed set, keep `plan`, `tale`, and `epic` as completion candidates, and
  update ACE directive metadata so it explains adapter-owned automatic gate resolution instead of claiming global
  plan-only validation.
- In `sase-core`, update the editor directive registry and its focused tests to expose the same contract. Keep the three
  plan aliases as suggestions, but make hover/documentation clear that an optional raw argument is interpreted by the
  gate kind rather than by a universal parser enum. Do not add a second gate-policy implementation to the editor core.
- Correct the authoritative `%auto` section and directive table in `docs/xprompt.md`: document raw-argument retention,
  adapter-time validation, bare question first-option behavior, plan/tale/epic compatibility aliases, authored-tier
  conflict rejection, and launch auto rejection. Update other exact user-facing summaries only where they make the same
  false closed-enum or plan-only claim; do not churn examples that validly demonstrate the plan aliases.
- Update focused Python and Rust tests so they distinguish "suggested compatibility arguments" from "accepted parser
  arguments" and preserve regression coverage for `%auto:foo` reaching adapter validation.

## Phase 2: Revalidate the integrated epic

- Re-run focused tests for directive parsing/completion, plan gate auto arguments, question automatic resolution, and
  the Rust editor registry before the full suites.
- In SASE, run `just install` and `just check`, including visual snapshots and the still-open epic Symvision allowances.
  In `sase-core`, run the workspace tests and parity fixtures. In `sase-telegram`, run its complete check against the
  local SASE/core build rather than the older released SASE package, so the neutral plan/question/launch callback paths
  are tested against the APIs they landed with.
- Recheck the post-epic-start integration points from `sase-6g`: parallel-family scan fields must remain present in both
  Rust and Python wire projections, the family directive must compose with the opaque `%auto` parser changes, and the
  later launch-time family-resolution path must preserve `auto_approve_argument` metadata while adding its family
  membership metadata. Keep all linked working trees clean apart from the intentional changes in this plan.

## Phase 3: Close and finalize epic `sase-6e`

- Confirm all seven child beads are still closed and the checks above are green, then run `sase bead close sase-6e`.
- After the close, follow the repository's Symvision memory guidance and run `just symvision`. Remove every expired
  `sase-6e(...)` epic-symbol entry from the `Justfile`, including the current allowances for the launch, plan, and
  question gate command entry points and `notify_plan_approval`; delete genuinely unused code it exposes, and use only
  the repository-approved permanent dynamic-entrypoint mechanism for symbols proven live through generated hashed
  command scripts or other string-based dispatch. Re-run Symvision until it passes without an epic allowance for
  `sase-6e`.
- Open the plans sidecar through `sase repo open` and change only the original epic plan
  `sase/repos/plans/202607/unified_notification_gates.md` frontmatter from `status: wip` to `status: done`.
- Because the landing cleanup changes SASE files, run `just install` and `just check` once more. Re-run any affected
  `sase-core` or `sase-telegram` checks if the post-close cleanup crosses those repositories. Finish by verifying the
  epic is closed, its linked plan is done, all child beads remain closed, and each participating repository contains
  only the intended landing changes.

## Acceptance criteria

- `%auto` parsing remains open to opaque arguments, while each gate adapter owns its valid arguments and defaults.
- ACE and Rust editor completion may suggest `plan`, `tale`, and `epic`, but no user-facing contract calls them the only
  parser-valid values; exact documentation covers question and launch behavior as well as plan behavior.
- The durable gate, typed transport, automatic/manual execution, and legacy-fallback regressions pass across SASE,
  `sase-core`, and `sase-telegram` against mutually compatible local sources.
- `sase-6e` is closed, post-close Symvision passes with no `sase-6e` whitelist entries, and the original epic plan has
  `status: done`.
