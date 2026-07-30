---
tier: tale
title: Make `ci_watch` CI-repair proposals actually launch and notify once
goal:
  The `ci_watch` chop proposes CI-repair agents under the template name `ci_fix.<slug>.@` so proposals stop being
  rejected by a permanent agent-name collision, notifies exactly once per distinct failure it proposes a fix for instead
  of once every two lumberjack ticks forever, and scopes its dedupe key to the current red episode so a failing-job set
  that recurs after the repo goes green can be fixed again.
create_time: 2026-07-30 06:55:06
status: done
---

- **PROMPT:** [202607/prompts/ci_watch_fix_launch_and_notify.md](prompts/ci_watch_fix_launch_and_notify.md)
- **AGENTS:**
  - bbugyi200.athena.p6--code
- **COMMITS:**
  - [788410a](https://github.com/bbugyi200/bugyi-chops/commit/788410a7461d7565b24cb44a6c1f18ab9e76e4ff) — fix(ci-watch):
    isolate repair launches and notifications

# Plan: Make `ci_watch` CI-repair proposals actually launch and notify once

## Incident summary

The user receives a "Proposed a CI repair for sase-org/sase at <sha>" notification from the `ci_watch` Lumberjack chop
roughly every ten minutes, and no PR ever appears. Two independent diagnostic agents (`p5.cld`, `p5.cdx`) reached the
same root cause; every claim below was re-verified from primary evidence before this plan was written.

Evidence:

- `~/.sase/axe/logs/lumberjack-ci_watch.log` contains **25** occurrences of:

  ```
  Skipped proposal 1 (ci_fix.sase): explicit agent name collision:
  Agent name 'ci_fix.sase' is taken. Try 'ci_fix.sase1'. Proposal skipped.
  ```

  first at 2026-07-29 11:52:51, last at 2026-07-30 06:05:45, each immediately followed by
  `Released 1 once-per key(s) after agent-name collision skip`.

- Exactly **one** repair agent has ever launched: `Launched proposal 1 as ci_fix.sase (PID 3114280)` at 2026-07-29
  02:32:34. Its transcript (`~/.sase/chats/202607/gh_sase_org__sase-ace_run-ci_fix_sase-260729_023232.md`, artifacts at
  `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202607/29/20260729023232`) shows it correctly concluded the
  Python 3.14 failure was a superseded load-dependent flake and deliberately made no changes (`"reason": "no_changes"`).
- The agent-name registry still reserves `ci_fix.sase` (verified via `get_reserved_agent_names()`; owner path
  `.../artifacts/ace-run/202607/29/20260729023232`). SASE agent names are durable reservations — they are never freed
  when an agent finishes — so this collision is permanent, not transient.
- The framework's once-per store `~/.sase/axe/lumberjacks/ci_watch/chops/ci_watch/seen.json` holds exactly **one**
  entry, `ci_fix:sase-org/sase:b2176c14e4f91a62` (the 02:32 launch). Every later key was released by the collision-skip
  path, which is why the loop re-arms.
- Zero `ci_fix_*` PRs exist on any of the watched repos.

## Root cause

Three defects interact. The first explains the reported symptom; the second and third survive fixing the first and each
independently reproduce the same "notification with no PR" experience.

### 1. The proposed agent name is a fixed literal, so it collides forever (primary)

`src/bugyi_chops/ci_watch.py` proposes with a hardcoded explicit name in two places:

```python
ledger_repos[repo]["proposal"] = {
    "agent_name": f"ci_fix.{slug}",       # ~line 1716, ledger record
    "dedupe_key": dedupe_key,
}
...
result.propose(                            # ~line 2057
    prompt,
    f"gh:{repo}",
    proposal_id=f"fix_{slug}",
    agent_name=f"ci_fix.{slug}",           # ~line 2061
    dedupe_key=dedupe_key,
)
```

`slug` is `safe_fragment(repo.rsplit("/", 1)[-1])`, so the name is always `ci_fix.sase` for `sase-org/sase`. In the
`sase` repo, `prepare_chop_proposals` (`src/sase/axe/chop_proposal_planning.py`) marks a configured name as
`explicit_agent_name=True`, and `launch_chop_proposals` (`src/sase/axe/chop_proposal_launch.py:179-186`) catches
`AgentNameLaunchCollisionError` for explicit names and _skips_ the proposal rather than auto-suffixing it. Because names
are durably reserved, this can never succeed again. No agent launches, so no PR is ever created.

`ci_watch` is the only chop still doing this. `recent_audits.py` and `toobig_split.py` already use `@` templates
(`audit_bugs.<project>.@`, `split_file.<slug>.@`); commit `eff60e6` ("fix: use templates for audit agent names") fixed
the audit chops and missed `ci_watch`. No built-in `sase` chop sets a literal explicit `agent_name`.

### 2. The notification fires inside the chop body, before AXE decides anything

```python
counters["fix_proposed"] += 1
_mark(ledger_repos, repo, reason="fix_proposed")
if not agents.notify(                      # ~line 1710
    [f"Proposed a CI repair for {repo} at {failure.sha[:7]}"],
    tags=["ci"],
):
    ledger_repos[repo]["notification"] = "failed"
...
streaks.pop(repo, None)                    # ~line 1722
```

The chop cannot observe the launch outcome — AXE evaluates once-per dedupe and launches only _after_ the chop returns
its result document. So the notification is emitted unconditionally, and it is emitted **every time the chop
re-proposes**, with nothing damping it:

- `streaks.pop(repo, None)` resets the red-debounce counter at propose time, so with `red_debounce_ticks: 2` and
  `interval: 300` the chop re-proposes every ~10 minutes;
- on a collision skip AXE releases the once-per key, so the dedupe key never damps it either.

Fixing defect 1 removes the collision path but **not** this loop. The observed log already contains the surviving
variant at 2026-07-29 02:53:08, ~20 minutes after the one successful launch:
`Once-per proposal 1: duplicate key=ci_fix:sase-org/sase:b2176c14e4f91a62` — a skipped proposal that still notified. Any
repo that stays red with a stable failing-job set and no live `ci_fix` agent reproduces a notification every two ticks
indefinitely.

### 3. A failing-job set can only ever get one fix attempt, forever

`dedupe_key = f"ci_fix:{repo}:{failure.fingerprint_key}"`, and `fingerprint_key` is
`sha256("\0".join(failing_job_names))[:16]` — deliberately SHA-independent, but also date- and episode-independent. The
framework's once-per store is a capacity-bounded FIFO (default capacity 1024; this chop has one entry), so in practice
the key is permanent and nothing ever releases it.

Consequence: the 2026-07-29 fixer examined "Python 3.14 tests" failing, correctly changed nothing, and permanently
consumed `ci_fix:sase-org/sase:b2176c14e4f91a62`. If that exact job set breaks again next month — a genuine regression
rather than a flake — the chop will propose, AXE will skip it as a duplicate, and no fixer will ever run for it again.

This also contradicts the chop's own `main()` docstring, which claims: "A fixer landing either changes the failing job
set or turns the repository green; either outcome resets the debounce streak and releases its SHA-independent key."
Nothing releases the key. Either the behavior or the docstring has to change; this plan changes the behavior, because
the documented intent is the correct one.

## Why this is a `bugyi-chops`-only change

All three defects live in the chop. This was traced end to end and exercised against the live SASE name registry before
writing this plan. The implementer should not need to touch the `sase` repo; if something below turns out to be wrong,
stop and report rather than expanding scope.

1. **Rust proposal validation accepts the template.** For a non-clan proposal, `validate_chop_proposal` checks
   `agent_name` only with `validate_token` (non-blank, no whitespace, no NUL). These proposals set no `clan`.
2. **Chop planning passes it through verbatim.** `plan_chop_proposals` allocates concrete names only for clan groups; a
   non-clan proposal keeps `proposal.agent_name` as-is, and `scaffolded_prompt` emits `%id(ci_fix.sase.@, tribe=chop)`.
3. **Launch-name validation tolerates templates.** `validate_launch_name_requests`
   (`src/sase/agent/launch_validation.py:254`) skips reservation and collision checks entirely for template requests
   (`if request.name_template: continue`), so the `AgentNameLaunchCollisionError` path is never reached.
4. **The parent allocates the concrete name before spawn.** `plan_single_agent_name` on the exact scaffolded prompt
   returns `SASE_AGENT_PLANNED_NAME=ci_fix.sase.0`.
5. **Verified interactively for every configured repo.** For all five watched repos, `ci_fix.<slug>.@` parses (prefix
   `ci_fix.<slug>.`, empty suffix), renders `ci_fix.<slug>.0 / .1 / .z / .00`, allocates around the already reserved
   `ci_fix.sase`, and passes both `preflight_launch_name_requests` and `validate_launch_name_requests`:

   ```
   ci_fix.sase.@           alloc=ci_fix.sase.0           validate=OK
   ci_fix.sase-core.@      alloc=ci_fix.sase-core.0      validate=OK
   ci_fix.sase-github.@    alloc=ci_fix.sase-github.0    validate=OK
   ci_fix.sase-telegram.@  alloc=ci_fix.sase-telegram.0  validate=OK
   ci_fix.sase-nvim.@      alloc=ci_fix.sase-nvim.0      validate=OK
   ```

   The hyphenated slugs are fine — a single `-` is not the reserved `--` family separator.

6. **The `ci_fix` hood guard still works.** The in-flight guard matches `name == "ci_fix" or name.startswith("ci_fix.")`
   (~line 1684), which `ci_fix.<slug>.<token>` satisfies. The existing `test_ci_fix_hood_matching` parametrization
   already covers a three-segment name (`ci_fix.sase.child`).
7. **The chezmoi config needs no change.** `home/dot_config/sase/sase_athena.yml` (lane `ci_watch`, `interval: 300`,
   `wait_runners: 0`, `red_debounce_ticks: 2`, `max_fix_proposals_per_tick: 1`) references the `ci_fix` _hood_ and the
   debounce policy, never the agent-name format. **Do not edit the chezmoi repo.**

## Changes

All changes are in the `bugyi-chops` repo. Open it first with the `/sase_repo` skill
(`sase repo open gh:bbugyi200/bugyi-chops -r "<reason>"`) and use the printed path as the only path for reads and
writes. It is not a registered SASE project, so there is no project workspace for it.

### 1. `src/bugyi_chops/ci_watch.py` — template agent name

Replace both fixed names with the `@` template, so SASE allocates the concrete token at launch:

```python
# `@` lets SASE allocate a short alphanumeric suffix at launch while keeping the
# agent in the `ci_fix` hood that gates concurrent repairs.
agent_name = f"ci_fix.{slug}.@"
```

Use it at the ledger record (~line 1716) and at `result.propose(...)` (~line 2061). Both sites recompute `slug`
identically today; a small shared helper (`_fix_agent_name(repo)`) alongside the existing `_dedupe_key`-style helpers is
preferable to duplicating the f-string, but keep it to one short function. Leave `proposal_id=f"fix_{slug}"` and the
`#pr(ci_fix_{slug}_{sha7})` rollover in `_fix_prompt` unchanged — the PR name must stay stable per failing commit.

Match the surrounding comment density: this module is lightly commented, so one short comment on the `@` marker is
enough.

### 2. `src/bugyi_chops/ci_watch.py` — episode-scoped dedupe key and notify-once ledger

Add one new persisted state document next to the existing streak file and release ledger, following
`_load_release_ledger` / `_write_release_ledger` exactly in shape: `"version": 1`, defensive per-row validation, silent
fallback to an empty document on `OSError` / `json.JSONDecodeError` / version mismatch, atomic write through the
existing `_atomic_write_json`, and bounded/pruned contents.

New module constants next to the existing ones (~lines 44-56):

```python
MAX_ANNOUNCED_FIXES = 100
FIX_LEDGER_FILE_NAME = "ci_watch_fixes.json"
FIX_LEDGER_RETENTION_DAYS = 90
```

Document shape:

```json
{
  "version": 1,
  "repos": { "sase-org/sase": { "episode": 3, "red": true } },
  "announced": { "ci_fix:sase-org/sase:4c02102423db4bbd:e3": "2026-07-30T06:05:45-04:00" }
}
```

**Episode counter.** In the red-streak loop (~lines 1644-1666), maintain a per-repo episode that survives green periods
(unlike `streaks`, which is popped both when a repo is not red and at propose time, so it cannot carry this):

- for a repo whose state is not `RED`: if the state is `RepoState.GREEN` and the stored `red` flag is true, increment
  `episode` and clear `red`. Bump on `GREEN` only — `PENDING`, `NO_CI`, and `ERROR` are unsettled or unknown, not
  resolutions, and must not rotate the key;
- for a repo whose state is `RED`: set `red` to true, leaving `episode` alone;
- record the value in the decision ledger via `_mark(ledger_repos, repo, fix_episode=<episode>)` so a tick's reasoning
  stays auditable.

**Dedupe key.** Extend it with the episode:

```python
dedupe_key = f"ci_fix:{repo}:{failure.fingerprint_key}:e{episode}"
```

This keeps the key SHA-independent (the existing `assert SHA not in proposal["dedupe_key"]` coverage must keep passing)
while making a recurrence after a green period a genuinely new event.

**Notify once per key.** Gate only the notification on the `announced` map — do **not** gate whether the proposal is
emitted. AXE's once-per store stays the single authority on launch dedupe, so a proposal whose launch AXE releases
(genuine launch failure) can still be retried:

```python
if dedupe_key in announced:
    ledger_repos[repo]["notification"] = "already_announced"
elif agents.notify([...], icon="🛠", tags=["ci"]):
    announced[dedupe_key] = now.isoformat()
else:
    ledger_repos[repo]["notification"] = "failed"
```

Record the key **only** when `agents.notify` returns true, mirroring the release side's
`if notification_ok: announced_pending[...]["notified"] = True` (~line 2021), so a failed notification is retried on the
next tick.

Improve the notification body while here: keep the existing first line, and add a second line naming the failing jobs
(bounded through the existing `_bounded`, and truncated to a small count so the payload stays small), so a single
notification is actually actionable. Do not add an `action` / `action_data` — there is no durable CI report to open yet
(see out of scope).

**Persist and prune.** Write the fix ledger immediately after the existing `_write_streaks(invocation, streaks)` call
(~line 1724), wrapped in the same `try` / `except OSError` + `invocation.logger.warning(...)` pattern the release ledger
uses, so a state-dir write failure degrades to extra notifications rather than failing the tick. On load, drop:

- `repos` rows and `announced` keys whose repo is not in `config.repos`;
- `announced` entries older than `FIX_LEDGER_RETENTION_DAYS`;
- oldest-first overflow beyond `MAX_ANNOUNCED_FIXES`.

### 3. `tests/test_ci_watch.py`

The suite shares `tmp_path` as `state_dir` across ticks and `_vars()` defaults to `red_debounce_ticks=1`, so the notify
ledger is observable across `_build` calls. Audit every multi-tick test for a now-suppressed second notification before
adding new coverage.

Update:

- `test_red_idle_emits_one_pinned_sanitized_proposal`: `proposal["agent_name"] == "ci_fix.sase.@"`, plus an assertion
  that the raw slug alone is no longer the whole name. It uses `FakeAgents(notify_ok=False)` and asserts
  `ledger["repositories"][REPO]["notification"] == "failed"` — that must still hold, and the failed notification must
  **not** be recorded in the fix ledger.
- `test_dedupe_key_is_stable_across_head_shas`: must keep passing unchanged. Both ticks are red, so the episode does not
  move and the key stays stable across a changed HEAD SHA. This is the guard that the episode suffix did not
  accidentally reintroduce SHA sensitivity.

Add:

- an intervening-green test: red tick → green tick → red tick produces a **different** dedupe key than the first red
  tick, and the `ci_watch_fixes.json` `episode` for that repo incremented exactly once;
- a `PENDING` / `NO_CI` counter-test: an unsettled or CI-less tick between two red ticks does **not** change the key;
- a notify-once test: two consecutive red ticks with the same failing-job set emit exactly one entry in
  `FakeAgents.notifications`, and the second tick's decision ledger records `notification == "already_announced"` while
  still emitting its proposal;
- a notify-retry test: a first red tick with `notify_ok=False` records nothing in the ledger, and a following red tick
  with a working `notify` does notify;
- a corrupt/absent fix-ledger test mirroring `test_intervening_green_and_corrupt_streak_reset_debounce` and
  `test_release_ledger_accumulates_prunes_and_recovers_from_corruption`: a truncated `ci_watch_fixes.json` falls back to
  an empty document and the tick still succeeds;
- coverage that the pruning rules hold (unconfigured repo dropped, retention cutoff applied, `MAX_ANNOUNCED_FIXES`
  respected).

`test_ci_fix_hood_matching`, `test_live_ci_fix_agent_suppresses_all_mature_red_repos`, and
`test_ci_fix_ledger_names_are_bounded_and_redacted` exercise _probe_ names rather than proposed names and need no
change.

### 4. Documentation

- `src/bugyi_chops/ci_watch.py` `main()` docstring (~lines 2084-2091): the claim that a fixer landing "releases its
  SHA-independent key" is currently false. Restate it to describe the implemented behavior: one repair proposal per
  failing-job set per red episode, where an observed green resolution starts a new episode, and the repair agent is
  named `ci_fix.<slug>.@` so SASE assigns the concrete token at launch.
- `README.md` (~lines 19-24): "Its CI-fix proposals are emitted only after a global `sase agent list -j` probe reports
  zero live agents" has been stale since commit `156a2d9` scoped the gate to the `ci_fix` hood. Correct it to say the
  probe suppresses proposals only while a live `ci_fix` hood agent exists, and note the `ci_fix.<slug>.@` template name.
  Leave the rest of the embedded sample config alone.

## Behavior changes to accept

- **Repair agent names gain a token.** `ci_fix.sase` becomes `ci_fix.sase.0`, `.1`, ... The stale `ci_fix.sase`
  reservation stops mattering and needs no cleanup; leave it in the registry.
- **One fixer per failing-job set per red episode, not per lifetime.** A job set that breaks, gets a fixer, goes green,
  and breaks again later now gets a second fixer. Rate stays bounded by `red_debounce_ticks: 2`,
  `max_fix_proposals_per_tick: 1`, `interval: 300`, and the `ci_fix` hood in-flight guard.
- **Redundant proposals are still emitted and still counted.** Because notification is gated but proposing is not, a
  repo that stays red keeps emitting a proposal every two ticks that AXE skips as a duplicate, and `fix_proposed` still
  counts it. This is deliberate: it keeps AXE's once-per store authoritative over launch dedupe. The visible cost is log
  lines, not notifications.
- **A repo that stays red forever gets exactly one fixer for that job set.** If the fixer cannot fix it, there is no
  further notification until the repo goes green or the failing job set changes. That is the intended damping; see out
  of scope for the alternative.

## Verification

In the `bugyi-chops` checkout obtained through `/sase_repo`:

```bash
just install
just check   # ruff format --check, ruff check, mypy, pytest, build
```

`just check` must pass before reporting completion.

Optionally, from a `sase` workspace, confirm the scaffolded prompt carries the template:

```bash
sase axe chop run ci_watch -L ci_watch --dry-run --chop-verbose
```

and check that the previewed proposal prompt contains `%id(ci_fix.sase.@, tribe=chop)`. For a non-clan proposal the
preview intentionally shows the raw template rather than a speculative concrete name; the recorded launch shows the real
one.

No files change in the `sase` repo, so `just check` in the `sase` repo is not required.

## Remediation runbook (operator steps, after the code fix lands)

1. Reinstall the plugin so AXE runs the fixed script — `sase plugin list -j` reports `bugyi-chops` with
   `install_type: git`, `version 0.4.0`, so this pulls the repo rather than PyPI and no release is required:

   ```bash
   sase plugin install bugyi-chops
   ```

   A `pyproject.toml` version bump is optional and up to the user.

2. Confirm the lumberjack restarted onto the new script and watch one tick:

   ```bash
   tail -f ~/.sase/axe/logs/lumberjack-ci_watch.log
   ```

3. At the next mature-red tick, expect `Launched proposal 1 as ci_fix.<slug>.<token>` instead of
   `Skipped proposal 1 (ci_fix.sase)`. As of 2026-07-30 06:41 the sweep reads `red=0 pending=1`, so this fires on the
   next confirmed terminal failure rather than immediately.
4. Nothing needs to be deleted. `~/.sase/axe/lumberjacks/ci_watch/chops/ci_watch/seen.json` holds one legacy key
   (`ci_fix:sase-org/sase:b2176c14e4f91a62`) in the old un-episoded format; it can never collide with a new `...:eN` key
   and will age out of the bounded store. The `ci_fix.sase` name reservation is likewise harmless.

## Out of scope / follow-ups (raise with the user; do not implement here)

- **`sase`-side hardening of the collision-skip loop.** Agent names are permanent reservations, so retrying an identical
  explicit name can never succeed — yet `src/sase/axe/chop_runner_script_result.py:531-539` releases the once-per key
  after every collision skip, guaranteeing an unbounded retry loop for any chop that proposes a literal name. That
  release was deliberate (commit `e39816a1f`, "Release once-per reservations ... across skipped proposals"), and
  reversing it could regress a chop that intends to retry under a sequential name, so it needs the user's decision. A
  purely additive alternative is to surface repeated name-collision skips as an AXE fault (error digest or notification)
  instead of only a `status: skipped` run and a chop log line — nothing currently escalates them, which is why this ran
  for over a day unnoticed. After this tale, no chop proposes a literal explicit name, so the trap has no live victim.
- **Launch-outcome notifications.** The honest fix for "the notification does not know whether the agent launched" is a
  per-proposal notification emitted by AXE after launch. The chop proposal schema has no such field
  (`validate_chop_proposal` rejects unknown fields; the allowed set is `id`, `prompt`, `workspace`, `workspace_ref`,
  `agent_name`, `clan`, `clan_summary`, `tribe`, `model`, `effort`, `env`, `extra_env`, `dedupe_key`, `wait_on`), so it
  would need a `sase-core` wire change plus `sase` framework work.
- **A durable CI report and a `ViewReport` action** on the repair notification, matching what commit `4f4fdd6` gave the
  release side. Only `ci_watch_releases.report.json` is published today.
- **A retry cooldown for persistently red repos**, so a job set that stays broken for N days gets another fix attempt
  without needing an intervening green. This is a policy call.

## Deliverable

A single focused change in `bbugyi200/bugyi-chops` covering `src/bugyi_chops/ci_watch.py`, `tests/test_ci_watch.py`, and
`README.md`, verified by `just check`. Commit only if the user asks or a post-completion finalizer triggers it; in that
case use the `/sase_git_commit` skill from inside the `bugyi-chops` checkout.
