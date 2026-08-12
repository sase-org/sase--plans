---
tier: epic
title: Retire the code_quality audit chops and gate CI-fix launches
goal: "The `code_quality` lumberjack and its `audit_bugs`/`audit_improvements` agents
  never run again, the `bugyi_chop_recent_*` scripts are gone from bugyi-chops, and
  `bugyi_chop_ci_watch` stops launching `ci_fix.*` agents through Axe: it files one
  durable LaunchApproval gate per distinct CI failure so the user approves or rejects
  each repair launch at their convenience, and never files a duplicate gate.

  "
phases:
  - id: retire_audits
    title: Remove the code_quality lumberjack and the recent-audit chops
    depends_on: []
    size: small
    description:
      "retire_audits: delete the `code_quality` lumberjack from the chezmoi-managed axe
      config, apply and restart axe so its lumberjack process stops, then delete the
      `recent_audits` module, its tests, its two console-script entry points, and its
      README coverage from bugyi-chops."
  - id: gate_ci_fix
    title: Replace ci_watch fix proposals with LaunchApproval gates
    depends_on:
      - retire_audits
    size: medium
    description:
      "gate_ci_fix: stop emitting `proposed_launches` from `bugyi_chop_ci_watch` and
      instead create one durable LaunchApproval gate per mature red CI failure, with a
      self-sufficient prompt, a version-2 fix ledger recording each gate's request id,
      and layered suppression that makes a duplicate gate impossible; update the report,
      tests, and README to match."
  - id: rollout
    title: Roll the new ci_watch out to the live host and verify
    depends_on:
      - gate_ci_fix
    size: small
    description:
      "rollout: bump the package version, push bugyi-chops, drop the now-inert
      `wait_runners: 0` from the `ci_watch` lane, reinstall the plugin from git, and
      verify with a dry run plus a live tick that the gate path files exactly one gate
      and no duplicates."
proposed_by: bbugyi200.athena.yi
create_time: 2026-08-12 10:38:38
status: wip
---

- **PROMPT:**
  [prompts/202608/retire_audit_chops_and_gate_ci_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/retire_audit_chops_and_gate_ci_fixes.md)

# Plan: Retire the code_quality audit chops and gate CI-fix launches

## Background

Two pieces of Bryan's personal automation are being retired or reshaped. Both live
outside this repository, split across two other repos:

- **`bbugyi200/bugyi-chops`** — the community SASE plugin that supplies the chop
  scripts. Open it with `sase repo open gh:bbugyi200/bugyi-chops -r "<reason>"` and use
  the printed path for every read and write. Never clone it, and never read it over the
  web.
- **`chezmoi`** — the linked repo holding the machine's SASE configuration. Open it with
  `sase repo open chezmoi -r "<reason>"`. The axe configuration for this host lives in
  `home/dot_config/sase/sase_athena.yml`.

Both repos are conventional git checkouts on `master`. Prior bugyi-chops work was landed
by SASE agents directly on `master`; commit through the `/sase_git_commit` skill from
inside the opened checkout, exactly as the existing commit history did. Never commit
with raw `git` commands.

### What `code_quality` is today

`home/dot_config/sase/sase_athena.yml` defines a `code_quality` lumberjack
(`interval: 60`, `wait_runners: 0`) containing exactly two chops:

- `recent_bug_audit` → script `bugyi_chop_recent_bug_audit`, inhibited by the
  `audit_bugs` hood
- `recent_improvement_audit` → script `bugyi_chop_recent_improvement_audit`, inhibited
  by the `audit_improvements` hood

Both are gated by a `git.commits_since` trigger with `threshold: 200` and
`checkpoint: on_action_success`, and both fan out over
`for_each: {source: projects, names: [sase]}`. They are the only chops in that
lumberjack, so the whole lumberjack goes away with them.

Their scripts live in `src/bugyi_chops/recent_audits.py` (172 lines), which is a
self-contained module: nothing else in the package imports it.
`tests/test_recent_audits.py` covers it, and `pyproject.toml` exposes it through two
`[project.scripts]` entries.

### What `ci_watch` does today

`bugyi_chop_ci_watch` (`src/bugyi_chops/ci_watch.py`) sweeps `actstat` across five SASE
repos each tick, classifies each repo's default branch from terminal job evidence,
guards release-PR merges, and publishes a durable release report. Everything except the
CI-repair path stays exactly as it is.

The repair path today works like this:

1. A repo classified `RED` accumulates a streak keyed on its failing-job fingerprint; a
   fix is only considered once the streak reaches `red_debounce_ticks`
   (`mature_red_repos`).
2. `AgentsGate.probe()` runs `sase agent list -j`; any live agent named `ci_fix` or
   `ci_fix.*` suppresses every fix that tick (`fix_in_flight`).
3. Otherwise, up to `max_fix_proposals_per_tick` repos each get a `sase notify create`
   announcement and a stashed prompt, and at the end of `build_ci_watch_result` each
   becomes a
   `result.propose(prompt, f"gh:{repo}", agent_name="ci_fix.<slug>.@", dedupe_key=...)`
   entry in the chop result's `proposed_launches`.
4. **Axe then launches that agent on its own**, with no user in the loop. That is the
   behavior being removed.

The dedupe key is already exactly the right identity for this work:
`ci_fix:{repo}:{failing_job_fingerprint}:e{episode}`, where `episode` increments only
when a repo is observed going red and then green again. A durable `ci_watch_fixes.json`
ledger in the chop's `state_dir` persists per-repo episode state and the set of keys
already announced.

## Design decisions

These were settled during planning; implement them as written rather than re-deriving
them.

### Use the LaunchApproval gate kind, not a hand-authored custom gate

SASE already has a registered gate kind purpose-built for this: `launch`
(`LaunchApproval`, `auto_policy: "forbidden"`, so it can never be auto-resolved).
Creating one is a single CLI call:

```bash
sase launch request -f <payload.json> -o json -s ci_watch
```

The critical property that makes this usable from a chop: `sase launch request` only
_waits_ for the answer when it detects it is running inside an agent, which it decides
purely from the `SASE_AGENT` environment variable
(`running_agent_context_requires_launch_approval()` in
`sase/src/sase/agent/launch_request.py`). Axe composes every chop script's environment
through `_compose_chop_subprocess_env()` in `sase/src/sase/axe/chop_script_runner.py`,
which calls `scrub_agent_identity_env()` and removes `SASE_AGENT` and every
`SASE_AGENT_*` variable. A chop therefore takes the non-waiting branch: the request is
created durably, the descriptor is printed, and the command exits immediately. It will
not block the tick.

Do **not** hand-author a `sase gate create` custom gate whose approve command shells out
to `sase run`. LaunchApproval already owns hash-verified approve/reject commands, a
rendered launch preview of the prompt, reject feedback capture, and dispatch on
approval; a custom gate would reimplement all of it with weaker guarantees.

The request payload is schema version 1:

```json
{
  "schema_version": 1,
  "prompt": "<full prompt, see below>",
  "reason": "CI red in sase-org/sase at f14b98c: sync-release-metadata",
  "approval": "required",
  "max_slots": 1
}
```

`-s ci_watch` sets the source-surface label the gate shows the user ("Source:
ci_watch"). Write the payload to a temporary file and pass it with `-f`; do not try to
pass the prompt through `-p` (quoting a multi-line prompt through argv is needless
risk).

Read the descriptor from stdout as JSON and keep `request_id`; that is the durable
handle for dedupe.

### The gate prompt must now be self-sufficient

Axe used to inject the workspace, agent name, and lane-level launch scaffolding around a
proposal's prompt. A LaunchApproval prompt gets none of that, so the prompt itself must
carry everything:

```
#gh:<owner>/<repo> %i:ci_fix.<slug>.@ %w(runners=0)

#pr(ci_fix_<slug>_<sha7>, status=ready)

#actstat(repo=<owner>/<repo>)

Repair the current default-branch CI failure in <owner>/<repo>.
... (existing evidence body: pinned run URL, pinned commit, failed job list, unsettled note) ...
```

Four things to get right:

- **`%i`, not `%n`.** The `%name`/`%n` directive has been **removed** from SASE; it now
  raises a targeted migration error (`_DEPRECATED_DIRECTIVE_MESSAGES` in
  `sase/src/sase/xprompt/_directive_types.py`). The live spelling is `%id`/`%i`. The
  `.@` template marker still works and is still resolved at launch to the next auto-name
  token, so `%i:ci_fix.sase.@` keeps repair agents in the `ci_fix` hood with a unique
  suffix — which is what the existing `sase agent list -j` hood suppression relies on.
- **`%w(runners=0)` replaces the lane's `wait_runners: 0`.** Per `docs/axe.md`,
  `wait_runners` applies _only_ to agents emitted through a script chop's
  `proposed_launches`. Once ci_watch stops emitting proposals, the lane setting is
  inert, so the idle-machine behavior has to move into the prompt.
- **Keep the existing `#pr(...)` and `#actstat(...)` lines and the whole evidence body**
  exactly as `_fix_prompt()` builds them today, including the `head_unsettled` note.
- **The workspace ref is `#gh:<owner>/<repo>`** — the same value the proposal passed as
  its workspace, minus the `gh:` → `#gh:` spelling change.

### Run `sase launch request` with an explicit, stable cwd

`create_launch_approval_request()` records `dispatch.cwd = Path.cwd()` into the request,
and on approval `dispatch_approved_launch_request()` does `os.chdir(cwd)` before
launching, failing outright if that directory no longer exists. Axe runs chop scripts
with `cwd` set to the lumberjack state dir, which is an Axe implementation detail.
Invoke the subprocess with an explicit `cwd=Path.home()` so the recorded dispatch
directory is stable and meaningful. The prompt names its workspace explicitly, so the
cwd does not affect which repo the approved agent works in.

### Duplicate suppression: two layers plus the existing guards

"Never send a duplicate gate" is the hard requirement. Implement it as a strict ledger
rule backed by a liveness check, in this evaluation order per tick (first match wins,
and each records its reason on the repo's ledger row for the report):

1. `fix_enabled` is false → `fix_disabled`.
2. The `sase agent list -j` probe fails → `agents_check_failed`.
3. A live agent named `ci_fix` or `ci_fix.*` exists → `fix_in_flight` (unchanged from
   today; an approved gate becomes exactly such an agent, so this also covers "the last
   gate was approved and is running").
4. **Any gate recorded in the ledger is still `pending`** → `gate_pending`. This is a
   global, one-at-a-time rule across all repos: while the user has an unanswered CI-fix
   gate, no new gate is filed for anything.
5. **The candidate's dedupe key is already present in the ledger's `gates` map** →
   `already_gated`. A key is recorded the moment its gate is created and is never
   re-gated, whatever the gate's outcome. A rejected, cancelled, or timed-out gate
   therefore does not come back; only a genuinely new failure does, because the key
   changes when the failing-job fingerprint changes or when the repo goes green and
   starts a new episode.
6. The per-tick cap `max_fix_proposals_per_tick` is reached → `fix_cap_reached`
   (unchanged).
7. Otherwise, create the gate.

Poll pending status with `sase gate show -k launch -i <request-id> -j`, which prints a
`status` field that is exactly one of `pending`, `answered`, `cancelled`, or `timeout`.
Treat any failure to read or parse that output as **not pending** — but note the key
stays in the ledger regardless, so an unreadable bundle can never produce a duplicate;
the worst case is that a new failure becomes eligible sooner.

Bound the work: only poll request ids actually recorded in the ledger, stop at the first
`pending`, and cap the number of `sase gate show` calls per tick (reuse the existing
`MAX_LEDGER_NAMES`-style constant approach, e.g. `MAX_GATE_POLLS_PER_TICK = 10`).

### Ledger v2

Bump `ci_watch_fixes.json` to `"version": 2`. `_load_fix_ledger()` already returns an
empty ledger for any other version, so the old v1 file is discarded cleanly on first
run; that is acceptable, because no gates exist under the old scheme and per-repo
episode counters re-derive within a tick or two.

```json
{
  "version": 2,
  "repos": { "sase-org/sase": { "episode": 3, "red": false } },
  "gates": {
    "ci_fix:sase-org/sase:0123456789abcdef:e3": {
      "request_id": "launch-<uuid>",
      "created_at": "2026-08-12T10:03:43.606657-04:00"
    }
  }
}
```

Replace the `announced` map with `gates`. Keep the pruning discipline that
`_prune_announced_fixes()` already implements — drop rows whose key does not parse,
whose repo is no longer configured, or whose timestamp is older than
`FIX_LEDGER_RETENTION_DAYS`, then cap the retained rows — and rename it to match its new
job. `_fix_key_repo()`'s key-shape validation carries over unchanged.

Write the ledger before creating gates would be wrong (a crash would suppress a gate
that was never filed), and writing only after would be wrong too (a crash would lose a
gate that _was_ filed and let it duplicate). Record each new gate into the in-memory
ledger and flush the ledger to disk immediately after each successful gate creation, so
an interrupted tick can never re-file an already-created gate.

### Drop the separate "Proposed a CI repair" notification

The gate creates its own notification, so the existing `agents.notify(...)` announcement
for fixes becomes a second notification for one event, and its wording ("Proposed a CI
repair…") would be wrong once nothing is proposed to Axe. Remove it and let the gate be
the single user-facing signal; the failing-job evidence the notification carried is
already in the `reason` field and in the prompt the gate previews. Release notifications
(merged / blocked) are untouched.

## Phase 1 — Remove the code_quality lumberjack and the recent-audit chops

Do the configuration removal **before** the script removal. The reverse order would
leave Axe with a chop whose script no longer exists.

### Configuration (chezmoi)

In `home/dot_config/sase/sase_athena.yml`, delete the entire `code_quality:` lumberjack
block — its `description`, `interval`, `wait_runners`, and both `recent_bug_audit` and
`recent_improvement_audit` chops. It sits between the `run_every` and `refresh_docs`
lumberjacks; leave both of those, and the `ci_watch` lumberjack, exactly as they are.
Nothing else in either `sase_athena.yml` or `sase.yml` references `code_quality`,
`audit_bugs`, `audit_improvements`, or the audit scripts, so there is no other config
edit.

Then, in order:

1. Commit the chezmoi change through `/sase_git_commit`.
2. Run `chezmoi update -a --force` (required by the chezmoi repo's own instructions) so
   `~/.config/sase/sase_athena.yml` actually changes.
3. Restart axe so the removal takes effect: each lumberjack is a separate process
   spawned from config at orchestrator start, so a running `code_quality` lumberjack
   keeps going until axe restarts. Use `sase axe stop` followed by `sase axe start`,
   then confirm with `sase axe status` and `sase axe lumberjack list` that
   `code_quality` is gone and every other lumberjack is healthy.
4. Confirm no audit chops remain configured: `sase axe chop list` must not mention
   `recent_bug_audit` or `recent_improvement_audit`, and `sase axe chop doctor` must be
   clean.
5. Remove the orphaned state directory `~/.sase/axe/lumberjacks/code_quality/`. It holds
   only that lumberjack's status, metrics, logs, and per-chop checkpoints, and nothing
   recreates or reads it once the lumberjack is unconfigured. Do this only after step 3
   confirms the lumberjack is stopped.

Do not touch `~/.sase/diffs/`, `~/.sase/reverted/`, or any existing beads, Patches, or
agent records produced by past audit runs. This phase stops future runs; it does not
rewrite history.

### Package (bugyi-chops)

1. Delete `src/bugyi_chops/recent_audits.py` and `tests/test_recent_audits.py`.
2. Delete the `bugyi_chop_recent_bug_audit` and `bugyi_chop_recent_improvement_audit`
   entries from `[project.scripts]` in `pyproject.toml`. Leave `bugyi_chop_ci_watch` and
   `bugyi_chop_toobig_split`.
3. Update `README.md`: the intro says the package "supplies four console scripts" — it
   now supplies two; drop both audit rows from the script table, delete the whole
   "Recent-commit audits" section including its YAML sample, and replace the
   `recent_bug_audit` example in "The chop result contract" with an equivalent example
   drawn from a surviving script.
4. Verify nothing dangles: no reference to `recent_audits`, `recent_bug`,
   `recent_improvement`, `audit_bugs`, or `audit_improvements` may remain anywhere in
   the repo.
5. Run `just install` then `just check` in the bugyi-chops checkout (lint, mypy, pytest
   with its 90% coverage gate, wheel/sdist build, twine check). If the default
   `just install` cannot resolve a compatible `sase`, use the documented escape hatch:
   `BUGYI_CHOPS_VENV_BIN=<path-to-a-sase-venv>/bin` in front of both commands.
6. Commit through `/sase_git_commit`. Do not bump the version and do not push a release
   tag — phase 3 owns the version bump and the rollout.

## Phase 2 — Replace ci_watch fix proposals with LaunchApproval gates

All work is in the bugyi-chops checkout: `src/bugyi_chops/ci_watch.py`,
`tests/test_ci_watch.py`, and `README.md`. Read the "Design decisions" section above
before starting; it settles the gate kind, the prompt shape, the suppression order, and
the ledger schema.

### Implementation

- **Add a launch-gate client.** Follow the existing adapter pattern: a small class
  taking the `sase` executable and the injectable `CommandRunner` protocol, so tests can
  substitute a fake exactly as they do for `ActstatClient`, `GitHubReader`, and
  `AgentsGate`. It needs two operations: create a launch request (write the JSON payload
  to a temp file, run `sase launch request -f <file> -o json -s ci_watch` with
  `cwd=Path.home()`, parse `request_id` out of stdout) and read a gate's status
  (`sase gate show -k launch -i <id> -j`, parse `status`). Reuse the module's existing
  discipline: bound and redact text through `_bounded()`, raise `CiWatchError` for
  adapter failures, and validate the parsed JSON shape rather than trusting it. Note
  that `CommandRunner` currently supports an `input_text` keyword; a temp file is still
  the right choice here because `sase launch request` reads its payload from a path, not
  stdin.
- **Rewrite the fix branch** of `build_ci_watch_result` to the seven-step order in
  "Duplicate suppression" above, replacing the `notify` call and the stashed
  `_prompt`/`_dedupe_key` handoff. The `mature_red_repos` computation, the streak
  bookkeeping, the episode bookkeeping, and `streaks.pop(repo, None)` after a successful
  gate all stay as they are.
- **Delete the proposal emission.** The loop after `result_with_summary(...)` that calls
  `result.propose(...)` goes away entirely; ci_watch stops emitting `proposed_launches`.
  Everything else the result carries — counters, report, evidence ledger — stays.
- **Extend the prompt builder** so `_fix_prompt()` (or a thin wrapper) produces the
  self-sufficient prompt described above, and keep `_fix_agent_name()` as the source of
  the `%i:` template value.
- **Update counters and the report.** Replace `fix_proposed` with `fix_gated`, and add
  `gate_pending_suppressed` and `gate_errors` alongside the existing `fix_suppressed`
  umbrella. Add the new reasons (`gate_pending`, `already_gated`, `gate_failed`) to the
  reason allowlist in `_repo_evidence()` so they render in the report's EVIDENCE column
  instead of falling through. Update the report footer facts and the `main()` docstring,
  which still describes proposals and Axe launches.
- **Fail closed on gate errors.** A non-zero exit or unparsable descriptor from
  `sase launch request` increments `gate_errors`, marks the repo `gate_failed`, and
  leaves the tick otherwise intact — never let it raise out of the tick and turn the
  whole sweep into a `check_error`. If the command exits 0 but the descriptor cannot be
  parsed, still record the dedupe key with an unknown request id: a gate may well have
  been created, and not duplicating it matters more than being able to poll it.
- **`actionable`** becomes `fix_gated or merged`, so a tick that only files a gate
  reports `ok` rather than `no_op`.

### Tests

Extend `tests/test_ci_watch.py` in its existing style — fake `CommandRunner`s capturing
argv, the `FIXED_NOW` clock, and `run_chop_main` for end-to-end result assertions. Cover
at least:

- a mature red repo files exactly one gate, with the payload containing
  `#gh:<owner>/<repo>`, `%i:ci_fix.<slug>.@`, `%w(runners=0)`, the `#pr(...)` rollover,
  and the failing-job evidence;
- the result contains no `proposed_launches`;
- the very next tick with the same failing fingerprint files **no** second gate
  (`already_gated`);
- a recorded gate still `pending` suppresses a gate for a _different_ red repo
  (`gate_pending`);
- an `answered`, `cancelled`, or `timeout` gate does **not** resurrect its own dedupe
  key;
- a green→red episode change, and a changed failing-job fingerprint, each produce a new
  gate;
- a live `ci_fix.*` agent still suppresses everything (`fix_in_flight`), and a failing
  agent probe still yields `agents_check_failed`;
- `sase launch request` failing counts `gate_errors`, marks `gate_failed`, keeps the
  rest of the tick (including release handling) working, and leaves the key un-recorded
  so a later tick may retry;
- a `sase launch request` that exits 0 with an unparsable descriptor still records the
  key;
- a v1 ledger on disk loads as empty and is rewritten as v2, and v2 pruning drops
  unconfigured-repo, expired, and overflow rows (adapt the existing
  `test_fix_ledger_prunes_*` test);
- `sase launch request` is invoked with `cwd` set to the home directory.

### Documentation

Update `README.md`'s `ci_watch` coverage: the script table row, the paragraph describing
the `ci_fix.<slug>.@` proposal template and the accepted probe race, and the "chop
result contract" text asserting Axe filters duplicate proposals and tracks linked
agents. Add a short section documenting the new flow: what the gate looks like, that
approval is required and never automatic, the exact suppression order, and the ledger
key that guarantees one gate per failing-job fingerprint per red episode.

Then run `just install && just check` and commit through `/sase_git_commit`.

## Phase 3 — Roll out and verify on the live host

1. **Version.** Bump `version` in `pyproject.toml` from `0.4.0` to `0.5.0` — removing
   two console scripts is a breaking change for anyone whose config references them.
   Commit through `/sase_git_commit`.
2. **Push.** Ensure `master` in the bugyi-chops checkout is pushed to `origin`, because
   the plugin on this host is a git install that resolves from the repository, not from
   PyPI. Do **not** create or push a `v0.5.0` tag: tagging triggers the PyPI publish
   workflow, and that release decision belongs to the user. Say so explicitly in the
   completion report rather than deciding it.
3. **Config.** In the chezmoi repo's `home/dot_config/sase/sase_athena.yml`, remove
   `wait_runners: 0` from the `ci_watch` lumberjack — it only ever applied to
   `proposed_launches`, which ci_watch no longer emits, and `%w(runners=0)` in the gate
   prompt now carries that behavior. Refresh the `ci_watch` lumberjack and chop
   `description` blocks, which currently describe proposing "at most one debounced
   CI-fix agent per tick" and lane-level `wait_runners`; they must describe the gate
   flow instead. Leave every `vars` entry alone — `max_fix_proposals_per_tick` keeps its
   name and now caps gates per tick. Commit, then `chezmoi update -a --force`.
4. **Install.** Run `sase plugin install bugyi-chops -g` to reinstall the plugin from
   its git repository. That command restarts axe on a successful change, which is also
   what picks up the config edit; confirm with `sase axe status` afterwards. Preview
   with `-n` first if anything looks uncertain.
5. **Dry run.** `sase axe chop run ci_watch -L ci_watch --dry-run --chop-verbose`.
   Confirm the summary line, the report, and the counters; a dry run must not create a
   gate.
6. **Live verification.** Let one real tick run (or run the chop once without
   `--dry-run`) against current CI state and confirm:
   - if a repo is mature-red, exactly one LaunchApproval notification appears
     (`sase notify list -j --tag launch`), and
     `sase gate show -k launch -i <request-id> -j` reports `pending`;
   - `ci_watch_fixes.json` in `~/.sase/axe/lumberjacks/ci_watch/` is version 2 and holds
     that one gate row;
   - the next tick files **no** second gate and reports `gate_pending` or
     `already_gated`;
   - the chop result contains no `proposed_launches`, and no `ci_fix.*` agent was
     started without approval (`sase agent list -j`). If CI happens to be all-green, say
     so plainly in the completion report and verify what can be verified (clean tick, no
     gate, ledger intact) rather than manufacturing a failure.
7. Report the outcome honestly, including anything left undone — in particular the
   unpushed release tag.

## Out of scope

- Any change to `toobig_split`, `refresh_docs`, the release-merge path, the release
  report, or the actstat sweep and classification logic.
- Any edit to SASE memory files (`sase/memory/*.md`, `AGENTS.md`, or the generated
  `CLAUDE.md` / `GEMINI.md` / `OPENCODE.md` / `QWEN.md` shims) in any repo. This plan
  does not carry permission for those, and a plan file is not a substitute for the
  user's explicit permission.
- Publishing `bugyi-chops` to PyPI, and editing the GitHub repo description (which still
  advertises "recent-commit audits"). Both are the user's call; mention them, do not do
  them.
- Deleting historical artifacts from past audit runs: saved diffs, reverted diffs,
  beads, Patches, and agent records all stay.
