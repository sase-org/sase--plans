---
tier: tale
title: Merge the ACE notification Release tab into the CI tab
goal:
  Every ci_watch notification — CI repair proposals and release merge/blocked notices
  alike — lands in a single ACE notification-panel `Ci` tab, and no `Release` tab is
  produced for new notifications.
proposed_by: bbugyi200.athena.va
create_time: 2026-08-07 20:15:42
status: wip
---

# Plan: Merge the ACE notification Release tab into the CI tab

## Problem

The ACE notification panel currently shows two adjacent tabs, `Ci` and `Release`, that
hold the same kind of work. The user wants one `Ci` tab holding everything both tabs
used to hold.

## Why the tabs exist (established by investigation — do not re-derive)

- Both tabs are **tag tabs**, and every row in either of them comes from the same
  sender: the `ci_watch` AXE chop. Empirically, over the whole local notification store,
  `sase notify list --tag ci -j` and `sase notify list --tag release -j` return only
  rows whose `sender` is `ci_watch` (20 `ci` rows, 13 `release` rows at planning time).
- ACE does not decide tab names. The Rust core assigns each notification exactly one tab
  by a fixed precedence (`crates/sase_core/src/notifications/tabs.rs`, `tab_key_for`):
  snoozed → muted → declared gate `panel` → HITL gate action → error → **the first
  stored tag** → `general`. The Python side mirrors that precedence in
  `src/sase/ace/tui/modals/notification_modal_tags.py` (`_notification_modal_tab_key`),
  and the panel, top-bar indicator, and mobile snapshot all consume the same
  classification.
- So the `Release` tab exists purely because the chop tags those notifications
  `["release"]`, making `release` their first tag. A row's later tags render as chips on
  the row but never create a tab (documented in `docs/notifications.md`, "Tags").
- The chop lives in an **external repo**: `gh:bbugyi200/bugyi-chops`, module
  `src/bugyi_chops/ci_watch.py`. It is installed on this machine as a git-direct-URL
  plugin (`sase plugin show bugyi-chops` reports `Installed ✓ git (direct URL)`), so the
  live chop only picks the change up after the plugin is reinstalled from git.

## Decision

Fix it at the sender: give the two release notifications the tag list
`["ci", "release"]` instead of `["release"]`.

- First tag `ci` ⇒ the core routes them into the existing `Ci` tab, so the `Release` tab
  is never created again.
- Keeping `release` as a **second** tag is deliberate, not decorative: it preserves
  `sase notify list --tag release` as a filter, and the row keeps a visible `release`
  chip so release rows stay distinguishable inside the merged tab (their per-row `🚢`
  icon does too, versus `🛠` for CI repairs).

No change is needed in the `sase` repo or in `sase-core`: the classification is generic
and already correct, and nothing in either repo special-cases `release` or `ci`.

### Rejected alternative (do not implement)

Adding a configurable tag→tab alias (e.g. `ace.notification_tabs.ci.aliases: [release]`)
so the core folds one tag into another tab. Rejected: `classify_notification_tabs` is a
pure row-in/tabs-out function with no config input today, so this would mean new core
wire/API surface, config plumbing into Rust, Python parity in
`_notification_modal_tab_key`, docs, and tests across two repos — a large, permanent
feature to work around one sender's tag choice that the same user owns. If a future need
arises for merging tags from _senders you do not control_, revisit it then.

## Implementation

All code changes are in `gh:bbugyi200/bugyi-chops`. Open it with the `/sase_repo` skill
first (`sase repo open gh:bbugyi200/bugyi-chops -r "<reason>"`) and use only the printed
path for reads and writes. The paths below are relative to that checkout.

1. `src/bugyi_chops/ci_watch.py` — two `agents.notify(...)` call sites near the end of
   the release section (around lines 2128–2153 at planning time; anchor on the note text
   rather than the line number):
   - the merged-release notification, whose first note is
     `f"Merged release PR #{pr.number} for {repo}"`: change `tags=["release"]` to
     `tags=["ci", "release"]`.
   - the blocked/pending-release notification, whose note is
     `f"Release PR #{number} for {repo} needs attention: ..."`: same change.
   - Leave everything else on both calls untouched — `icon="🚢"`, `action="ViewReport"`,
     and `action_data` (`report_path` / `report_title: "Releases"`) all still apply.
   - Do **not** touch the CI-repair notification (`tags=["ci"]`, `icon="🛠"`); it is
     already correct.
2. Grep the module for any remaining `tags=["release"]` to confirm both sites were
   changed and no third one exists.

## Tests (same repo, `tests/test_ci_watch.py`)

1. The merged-release notification assertion compares a whole notification dict and
   currently ends with `"tags": ["release"]` (in `test_live_merge_...`/the merge-cap
   test around line 1202). Update it to `"tags": ["ci", "release"]`.
2. The mixed fix+release test currently asserts
   `{notification["tags"][0] for notification in agents.notifications} == {"ci", "release"}`
   (around line 1535). This assertion is exactly the behavior being changed: make it
   assert that **every** notification's first tag is now `ci` — e.g.
   `{notification["tags"][0] for notification in agents.notifications} == {"ci"}` — and
   add an assertion that the release row still carries `release` as a later tag, so the
   test pins both halves of the decision rather than just the tab.
3. `test_blocked_release_notification_is_debounced_and_not_repeated` (around line 1455)
   covers the second call site but asserts no tags today. Add
   `assert agents.notifications[0]["tags"] == ["ci", "release"]` there so both call
   sites are pinned by tests.
4. Leave `test_agents_gate_parses_global_list_and_sends_json_notifications` (around
   line 1938) alone: it exercises the generic `AgentsGate.notify` pass-through and its
   `tags=["release"]` argument is arbitrary test data, not the chop's tagging policy.

Tag order matters to the outcome, so assert the full list (`["ci", "release"]`) rather
than set membership: `sase notify create` preserves sender order after normalization,
and a `["release", "ci"]` regression would recreate the `Release` tab.

## Verification

1. In the `bugyi-chops` checkout, run the repo's own gates against a SASE virtualenv, as
   its README documents:
   ```bash
   BUGYI_CHOPS_VENV_BIN=<path-to-a-sase-source-checkout>/.venv/bin just install
   BUGYI_CHOPS_VENV_BIN=<path-to-a-sase-source-checkout>/.venv/bin just check
   ```
   (`just check` = ruff format/lint + mypy + pytest + wheel/sdist build + twine check.)
   Use the SASE workspace you are running in for `<path-to-a-sase-source-checkout>`; run
   `just install` there first if its `.venv` is stale.
2. No `sase`-repo gates apply: this plan changes no files in the `sase` repo, so there
   is nothing for `just check` to cover there.
3. Sanity-check the classification without waiting for a real release event:
   ```bash
   printf '%s\n' '{"icon":"🚢","notes":["probe: release row lands in Ci"]}' \
     | sase notify create -s ci_watch -t ci -t release
   ```
   then open the ACE notification panel (`i`) and confirm the new row appears under `Ci`
   with a `release` chip and **no** `Release` tab. Dismiss the probe row afterward.

## Landing

Commit in the `bugyi-chops` checkout with the `/sase_git_commit` skill only (never raw
`git commit`). That repo uses Conventional Commits scoped to the chop — e.g.
`fix(ci-watch): route release notifications to the ci tab`. Do not bump the package
version or push a `v<version>` tag: releases there are tag-driven and PyPI publication
is not required for this machine, which installs the plugin from git.

## Deployment (requires user approval)

The running chop uses the installed plugin, so the change is inert until the plugin is
reinstalled from git:

```bash
sase plugin install bugyi-chops -g
```

This mutates the live environment and restarts axe, so propose it through the
`/sase_gate` skill and let the user approve rather than running it unprompted.

## Expected end state and the one leftover row

The notification modal lists **unread** notifications only, so after deployment the
`Release` tab disappears as soon as the existing unread `release`-tagged rows are read
or dismissed. At planning time there is exactly one such row ("Release PR #95 for
sase-org/sase-core needs attention: base branch not green"). No store migration or
retagging of historical notifications is needed or wanted — read/dismissed rows never
render a tab, and rewriting the JSONL store by hand is out of scope. Mention this to the
user in the completion report so a lingering `Release` tab before that row is cleared is
not mistaken for a failed change.

## Out of scope

- Any tab-alias/merge feature in `sase` or `sase-core` (see rejected alternative).
- Styling the merged tab. The `ci` tab has no entry in `ace.notification_tabs`, so it
  renders with the hashed auto-palette color and the generic `#` tag-kind icon. Giving
  it a dedicated icon/color would mean editing the user's chezmoi-managed
  `~/.config/sase/sase.yml` (via `/sase_repo`), which the user has not asked for.
- Changing the `ci_watch` chop's report, ledger, debounce, or merge behavior in any way.
