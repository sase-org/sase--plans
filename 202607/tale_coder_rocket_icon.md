---
tier: tale
title: Distinguish tale coder launch with a rocket icon
goal: 'Tale approval gates show a rocket for the Launch coder agent option while retaining
  the green checkbox for the primary Tale submission and for epic approval.

  '
create_time: 2026-07-18 14:04:50
status: done
prompt: 202607/prompts/tale_coder_rocket_icon.md
---

# Plan: Distinguish tale coder launch with a rocket icon

The tale gate currently assigns `✅` to both the grouped `Tale` submit action and its `Launch coder agent` member
option. That makes two actions with different scope look equivalent. The option is authored in the neutral plan-gate
request, while direct and legacy ACE modal callers construct equivalent fallback data, so both paths must use one
tier-aware presentation contract.

## Canonical gate presentation

- Extend the plan-gate presentation helpers so an option's icon can depend on the authored tier, alongside the existing
  tier-aware label helper.
- Assign `🚀` only to the tale `approve` option labeled `Launch coder agent`. Preserve `✅` for epic approval, `💾` for
  plan commit, `❌` for reject, and `💬` for feedback.
- Keep `TALE_PLAN_SUBMIT_GROUP` unchanged as `✅ Tale`; this preserves the primary grouped action and makes its icon
  intentionally distinct from the coder-launch member.
- Use the canonical helper when serializing neutral gate requests and when ACE constructs display-only fallback gate
  data, preventing loaded and direct modals from drifting.

## Regression coverage

- Update plan-gate contract and end-to-end assertions to require `🚀` for the tale coder-launch option while explicitly
  retaining `✅` for the Tale group and epic option.
- Strengthen ACE modal coverage so both a neutral gate loaded from its bundle and a direct/fallback tale modal render
  the rocket on `Launch coder agent` without changing the `Tale` submit control.
- Refresh the affected wide and narrow tale-gate PNG goldens and inspect their generated artifacts to confirm that the
  intended option icon is the only visual behavior change. Leave the epic snapshot unchanged unless verification reveals
  an unintended dependency.

## Validation

- Install the workspace's current development dependencies with `just install`, then run focused plan-gate, end-to-end,
  ACE modal, and tale-gate visual tests while iterating.
- Run `just test-visual` to validate the refreshed PNG snapshots, then run the repository-required `just check` before
  handoff.

The change is presentation-only: option identifiers, query branches, default selection, command execution, result
translation, and coder launch behavior remain unchanged.
