---
tier: tale
title: Make ci_watch a release-please merge notifier
goal: "ci_watch never creates gates or launches agents, only squash-merges eligible
  release-please pull requests, and reliably sends polished per-release and
  per-failing-repository SASE notifications with actionable CI evidence.

  "
size: medium
model: codex/gpt-5.6-sol
proposed_by: bbugyi200.athena.0a6
create_time: 2026-08-21 20:18:39
status: wip
---

# Plan: Make `ci_watch` a release-please merge notifier

## Outcome and boundaries

Simplify `bugyi_chop_ci_watch` in the external `bbugyi200/bugyi-chops` repository and
its host configuration in the linked `chezmoi` repository. Use `/sase_repo` to open both
repositories before reading or changing them, and honor each repository's own
`AGENTS.md` instructions.

After this change, the chop has exactly two user-facing responsibilities:

1. Observe the configured GitHub repositories and send one excellent SASE notification
   for each active failing-job incident.
2. Squash-merge eligible release-please PRs and send one excellent SASE notification for
   each PR it submits.

The only GitHub mutation the chop may perform is the guarded `gh pr merge` operation. It
must never call `sase agent`, `sase launch`, `sase gate`, `sase run`, or emit an Axe
launch proposal. Do not remove or change SASE's general gate/launch framework: remove
only the launch-approval behavior owned by this plugin. Continue observing
`sase-org/sase-core` for CI failures, but stop recognizing or merging release-plz PRs.

At planning time, `sase-org/sase#284` and `sase-org/sase-telegram#19` are open,
non-draft, mergeable release-please PRs with green PR rollups. The default branch of
`sase-org/sase` also has newer workflows in flight, so treat those numbers as rollout
evidence rather than bypassing the normal fail-closed merge guards. The configured
one-merge-per-tick cap and dependency order mean two eligible PRs may require two
five-minute chop cycles.

## Simplify the plugin around release observation

Refactor `src/bugyi_chops/ci_watch.py` so the useful CI classification and release
safety checks remain, while all repair-launch machinery disappears:

- Delete `AgentProbe`, the agent-list probe, `LaunchGateClient`, launch request/status
  calls, CI-fix prompt/name construction, red debounce and episode tracking, gate caps,
  gate polling, fix ledgers/streak files, and every fix/gate/agent counter and report
  field. Remove their imports, constants, config fields, and dead helpers rather than
  leaving disabled branches behind.
- Rename the narrow SASE adapter to express its sole purpose as a notifier. It may only
  invoke `sase notify create`; make command failures explicit so required notifications
  can be retried and surfaced in the chop result.
- Keep strict repository validation, actstat allowlisting, token redaction, bounded
  GitHub queries, current-default-branch reconciliation, and fail-closed handling for
  unknown, missing, pending, stale, or malformed evidence.
- Replace release-generator polymorphism with release-please-only behavior. Accept
  `release_repositories` as a list of configured repositories, recognize only
  `release-please--...` and `release-please/...` heads (including component branches),
  and check only the release-please/publish generator before merging. Reject the old
  generator mapping so stale configuration fails clearly.
- Preserve the existing merge invariants: exactly one candidate per repository, the
  configured default base, non-draft, mergeable and clean, a non-empty all-completed
  green check rollup, a green current default branch, an idle release generator,
  deterministic dependency order, `max_merges_per_tick`, explicit live mode, a final
  PR/head re-read, squash merge, and `--match-head-commit` race protection.
- Keep `merge_enabled` as an operational kill switch and keep dry-run side-effect-free:
  a dry run may render decisions but may neither merge nor notify.

Model failing evidence explicitly instead of reducing it to job names. For each failed
job retain its workflow/run name, job name, normalized conclusion, job URL when GitHub
supplies one, and bounded failing-step names. Produce the same typed evidence whether it
came from actstat's settled commit or the bounded current-HEAD GitHub fallback. A stale
settled failure may remain red while a newer HEAD is running, but both SHAs and that
unsettled condition must be visible to the user.

## Make notifications reliable, actionable, and beautiful

Use one small versioned state document for recent submitted releases and active failure
incidents. Migrate the useful recent-merge rows from the existing release ledger,
consider legacy merge notifications already delivered to avoid retroactive spam, and
stop reading the old fix/streak ledgers. Do not delete the old files automatically.

Give delivery at-least-once semantics:

- A failing repository gets one notification for a SHA-independent fingerprint of its
  current workflow/job/conclusion/failing-step set. Keep the incident active while the
  same failure persists, replace it when that evidence changes, and clear it after a
  non-red observation so a later recurrence is announced again. Persist unsuccessful
  delivery and retry it on the next tick.
- Record a successful merge durably with `notification_sent: false` before notifying.
  Mark it delivered only after `sase notify create` succeeds, and retry undelivered
  merge records on later ticks. Prefer a rare duplicate after a crash to silently losing
  a notification.
- If both merges and failures occur in one tick, send every per-PR and per-repository
  notification; neither class suppresses the other. Count attempted, sent, and failed
  notifications, and return `check_error` when a required delivery or report publish
  fails while preserving retry state.

Notification copy should be concise in the inbox and complete without opening another
surface:

- Release: ship icon; repository, PR number, version/title, target branch, short head
  SHA, and submitted/merged outcome.
- Failure: alert icon; repository, default branch and failing SHA; one bounded line per
  failed job in `workflow › job — conclusion` form; failing steps; direct job/run URL;
  and an explicit note when the failure belongs to an older settled SHA while current
  HEAD is still unsettled. Collapse overflow with an accurate remaining-count line.

Also build and atomically publish one durable `CI WATCH` report for notification
actions. Use semantic tones and restrained glyphs: a compact health headline, a
repository table, detailed failing-job rows, a release table with PR decisions, recent
submissions, and a small factual footer for live/dry-run mode and update time. Point
notifications at it with `ViewReport` only after the new report validates and publishes;
inline notification details remain sufficient if report publication fails. Preserve the
old release-report file so historical notifications do not break, but send all new
notifications to the new combined report.

## Tests and documentation

Rewrite `tests/test_ci_watch.py` around the smaller contract rather than retaining tests
for removed branches. Cover at least:

- one notification per red repository, multiple red repositories in the same tick,
  multiple jobs and failing steps, URL/SHAs/unsettled-head presentation, bounding and
  token redaction;
- stable incident deduplication, changed evidence, green reset, corrupt/legacy state,
  failed-notification retry, and merge-notification retry;
- simultaneous merge and failure notifications;
- release-please component branch matching, rejection of release-plz and the old config
  mapping, every merge guard, generator-busy handling, reread/head races, dependency
  ordering, cap behavior, and merge-command failure;
- dry-run and missing live context producing no merges and no notifications;
- structured report validation and atomic last-good-report preservation;
- subprocess argument assertions proving that the implementation can call only actstat,
  GitHub reads/merge, and `sase notify create`, with no agent, launch, gate, or run
  command path.

Update `README.md`, module/package descriptions, examples, and configuration snippets to
describe a release-please merge notifier, its notification incident semantics, and its
safety gates. Remove all documentation for CI-fix agents, LaunchApproval, repair
prompts, release-plz, and proposal-oriented `ci_watch` behavior. Leave `toobig_split`'s
proposal behavior and documentation unchanged. Because this removes a configured
behavior and config shape, bump `bugyi-chops` to the next pre-1.0 minor version
(`0.6.0`) and refresh its lock metadata, but do not create a release tag or publish to
PyPI as part of this task; the live rollout intentionally installs the reviewed Git
revision.

In `home/dot_config/sase/sase_athena.yml` in chezmoi:

- remove `ci_watch.chops.ci_watch.inhibit_if` entirely;
- replace the release-generator mapping with the release-please repository list;
- remove `max_fix_proposals_per_tick`, `red_debounce_ticks`, and `fix_enabled`;
- narrow `merge_order` to release-please repositories while preserving dependency order
  and the one-merge-per-tick cap;
- rewrite the lumberjack and chop descriptions so they plainly state the merge-only
  mutation boundary and rich per-incident notifications.

Keep the full `repos` allowlist, including repositories that are watched only for
failure notifications, and keep the dedicated actstat config.

## Verification, durable rollout, and monitored handoff

Run `just install` and `just check` in `bugyi-chops`. Run `just check` in chezmoi and
review a targeted chezmoi diff for the SASE host config. Do not proceed to the live
rollout with dirty unrelated files or failing checks.

The live plugin installer rejects numbered-workspace paths, so make the rollout durable
before observing it:

1. Use `/sase_git_commit` in each changed repository to create and push reviewed commits
   (the removal in `bugyi-chops` is a breaking `feat(ci-watch)!`; the chezmoi change is
   configuration). Do not use raw `git commit`. Verify both repositories are clean and
   not ahead of upstream.
2. As required by chezmoi's `AGENTS.md`, run `chezmoi update -a --force` after its
   commit so the simplified host config is applied.
3. Reinstall the pushed plugin with `sase plugin install bugyi-chops --git --refresh`.
   Let its successful install restart Axe so code and configuration change together.
4. Verify `sase axe status -j` is healthy and `sase axe chop list -j` shows `ci_watch`
   resolved, with an empty `inhibit_if`, no removed fix variables, and the new
   description. Run one verbose manual dry run for diagnostics and assert it performs no
   mutation or notification. Capture the open release-please PR baseline and the current
   notification/report state before waiting; do not force a live chop run.

The implementation agent must then use `/sase_monitor` as its last action to wait for
two complete five-minute scheduler opportunities plus a small buffer:

```bash
sase monitor start \
  --command 'sleep 660' \
  --reason 'Allow two scheduled ci_watch cycles to submit all currently eligible release-please PRs' \
  --timeout 12m \
  --start-status 'WAITING FOR CHOP' \
  --stop-status 'CHOP WINDOW ELAPSED' \
  --next 'Inspect the monitor outcome, then verify the live ci_watch configuration and latest runs. Compare the captured release-please baseline with GitHub: every PR that remained eligible through its scheduled turn, including sase-org/sase#284 when applicable, must be merged, and each merge must have a detailed ci_watch notification. Use /sase_notify to verify one notification per currently failing repository when failures exist, including its job and step evidence, and inspect the combined ViewReport. Confirm ci_watch created no launch proposal or LaunchApproval gate. If any expected PR was not submitted, diagnose the exact guard, workflow/job, deployment, or chop failure and use /sase_plan to author, validate twice, and propose the required corrective plan instead of applying an ad hoc fix. If all expectations pass, report the PR URLs, merge times, notification evidence, and chop run IDs.'
```

`sase monitor start` has no independent model option and directives inside `--next` are
literal. This tale therefore explicitly routes its implementation agent to
`codex/gpt-5.6-sol`; the monitor successor inherits that routing and is the requested
fresh verification agent. Do not poll after starting the monitor.
