---
tier: epic
title: Fix the athena/apollo audit findings the triage ranked worth fixing
goal: 'Apollo''s and athena''s managed-tmp reaper reclaims dead scratch even with
  SSH sessions open and read-only test trees present, and its jobs report failures
  truthfully. Agent cleanup works while any agent waits on an epic. `sase doctor -j`
  always prints JSON. Artifact-link projection converges past missing or foreign bead
  endpoints. Agent-hood publication drains its quarantine backlog through measured
  fixes and bounded retries. The TUI''s multi-second GC freezes are attributed and
  fixed against sase-1f3''s targets. Task beads sase-1bw, sase-1ne, sase-1nb, sase-1na,
  sase-1c5, sase-1f4, sase-1f3, sase-10x, and sase-1nc are closed with evidence, and
  the stalled epic sase-11o is superseded.

  '
decisions:
  gc_soak:
    ask: Should gc-measure wait out sase-1f3's 24-hour soak before gc-fix starts?
    default: false
    why: Busy-hour and 4-hour windows suffice to attribute; gc-fix's acceptance reruns
      the soak
    answer: false
phases:
- id: reap-core
  title: sase-core reaper removes read-only trees and exempts system-service sessions
  depends_on: []
  size: medium
  description: 'reap-core: in sase-core, make managed-tmp removal repair owner write
    permission inside the candidate tree and retry once, add the system-service ancestry
    exemption to the launch-scratch liveness observer, and surface the blocking process
    in dead-launch and pressure skip reasons.'
- id: reap-sase
  title: Truthful managed_tmp_reap and disk_pressure jobs on the new core
  depends_on:
  - reap-core
  size: medium
  description: 'reap-sase: move the core pin, log removal errors and blocking processes,
    make managed_tmp_reap report check_error on removal failures, make disk_pressure
    reap before notifying with an explicit status table, and update the housekeeping
    and runtime docs.'
- id: bobcli-vault
  title: bob-cli tests stop leaking read-only vault fixtures
  depends_on: []
  size: small
  description: 'bobcli-vault: in the bob-cli repo, restore write permission after
    the read-only completion test locks its fixture vault, and make the test TempDir
    removal repair owner write permission inside its own tree.'
- id: cleanup-encoding
  title: Lossless EpicFollowView encoding for bundles and cleanup payloads
  depends_on: []
  size: medium
  description: 'cleanup-encoding: encode wait_epic_follows explicitly in the shared
    agent bundle encoder, decode it back into EpicFollowView, guard against the next
    non-JSON field, and make the cleanup test harness round-trip payloads through
    JSON. Closes sase-1nb.'
- id: doctor-macros
  title: sase doctor survives unbound macro templates
  depends_on: []
  size: small
  description: 'doctor-macros: expand macros with raise_on_error in the model-macro
    check, skip parameterized macros explicitly while still reporting concrete broken
    presets, and make the diagnostics registry convert SystemExit into an ERROR check.
    Closes sase-1na.'
- id: links-core
  title: sase-core partial bead link projection and per-endpoint receipts
  depends_on: []
  size: medium
  description: 'links-core: add a skip mode to bead link projections that applies
    valid requests and reports missing holders per request, and extend the artifact-link
    publication receipt with per-endpoint outcomes and an acknowledged, parked, or
    pending disposition.'
- id: links-project
  title: Route, isolate, and commit artifact-link bead endpoints per operation
  depends_on:
  - links-core
  size: medium
  description: 'links-project: move the core pin, classify every bead endpoint by
    owning store, project only local endpoints in skip mode, commit chunks that already
    applied when a later chunk fails, and give each operation its own endpoint evidence.'
- id: links-converge
  title: Parked endpoints leave the retry loop and the sweep converges
  depends_on:
  - links-project
  size: medium
  description: 'links-converge: move parked operations into a replayable parked ledger,
    make outbox appends idempotent, sweep per document, report projection failures
    and parked counts in the artifact_link_backfill job status, and expose parked
    counts in doctor.'
- id: pub-measure
  title: Instrument and measure agent-hood publication drains
  depends_on: []
  size: medium
  description: 'pub-measure: supersede the stalled sase-11o epic, add permanent stage
    timing, a sync-lock holder record, and richer outbox doctor output, then measure
    one representative drain on athena and inspect apollo''s wedged clone read-only,
    and publish the findings as a research report.'
- id: pub-fix
  title: Fix the measured publication failure causes
  depends_on:
  - pub-measure
  size: large
  description: 'pub-fix: plan from the measurement report, then stop repo-wide failures
    from charging every request''s attempt budget, harden the drain timeout, bound
    the measured per-drain cost, resolve the stale-commit conflicts, and add a bounded
    retry selector.'
- id: pub-recover
  title: Drain the publication backlog on athena and apollo in bounded batches
  depends_on:
  - pub-fix
  size: medium
  description: 'pub-recover: on both hosts, back up unique unpublished content, retry
    quarantined requests in bounded batches while verifying remote publication and
    a falling backlog, confirm new agents publish, and close sase-10x.'
- id: gc-instrument
  title: GC attribution telemetry and an acceptance-grade freeze report
  depends_on: []
  size: medium
  description: 'gc-instrument: fix tui_freeze_report''s startup-row, rotation, and
    trigger handling and add an acceptance mode, add collection start, input age,
    fault deltas, source revision, and allocator stats to TUI telemetry, and add an
    opt-in GC census. Closes sase-1f4.'
- id: gc-measure
  title: Measure and attribute the TUI GC freezes on restarted TUIs
  depends_on:
  - gc-instrument
  size: medium
  description: 'gc-measure: have the user restart their TUIs onto the instrumented
    code, capture busy-hour and 4-hour windows on both Python versions, attribute
    garbage and survivors by type and producer, correlate long pauses with swap, and
    report against sase-1f3''s targets.'
- id: gc-fix
  title: Fix the measured GC freeze causes
  depends_on:
  - gc-measure
  size: large
  description: 'gc-fix: plan from the measurement report, reduce allocation and retention
    at the measured producers before touching collector policy, resolve the backstop
    quiet-window conflict, and prove sase-1f3''s targets on restarted TUIs. Closes
    sase-1nc.'
proposed_by: bbugyi200.athena.12i
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-20 13:36:11
status: wip
bead_id: sase-1nf
---

- **PROMPT:** [prompts/202610/athena_apollo_audit_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/athena_apollo_audit_fixes.md)
- **BEAD:** [sase-1nf](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nf/README.md)

# Plan: Fix the athena/apollo audit findings the triage ranked worth fixing

## Context

The operational log audit of athena and apollo found six problems. The triage report
re-verified them on 20 October 2026 against sase `9188cc5733` and core `b664b15b`. It
found that epic `sase-1n8` already fixed the disk-burst producer, and it ranked what is
left. This epic implements every item on that ranked list, plus the bob-cli side of
defect B. It has six independent tracks. Phases in different tracks never depend on each
other, and each track lands and is verified on its own, because the acceptance signals
differ (the audit warns against judging them together).

| Rank | Track                                                 | Phases                                          | Beads closed                                                 |
| ---- | ----------------------------------------------------- | ----------------------------------------------- | ------------------------------------------------------------ |
| 1    | The tmp reaper reclaims space (defects A, B, C)       | `reap-core`, `reap-sase`, `bobcli-vault`        | `sase-1bw`, `sase-1ne` (by the land agent after host checks) |
| 2    | Kill and dismiss work while an agent waits on an epic | `cleanup-encoding`                              | `sase-1nb`                                                   |
| 3    | `sase doctor -j` always prints JSON                   | `doctor-macros`                                 | `sase-1na`                                                   |
| 4    | TUI GC freezes                                        | `gc-instrument`, `gc-measure`, `gc-fix`         | `sase-1f4`, `sase-1f3`, `sase-1nc`                           |
| 5    | Publication backlog                                   | `pub-measure`, `pub-fix`, `pub-recover`         | `sase-11o` and `sase-11o.2` (superseded), `sase-10x`         |
| 6    | Artifact-link endpoints converge                      | `links-core`, `links-project`, `links-converge` | `sase-1c5` (by the land agent after host checks)             |

Tracks 4 and 5 are investigation first. Their root causes are not yet proven, so each
starts with a measurement phase, and its fix phase is `large` and plans from that
measurement. Every other track has a known root cause and a fixed design below.

## Shared rules for every phase

- **Repos.** Work in the repo the phase names. Open sase-core with
  `sase repo open sase-core -r "<why>"` and bob-cli with
  `sase repo open bob-cli -r "<why>"`. Use the printed path, and read that repo's
  `AGENTS.md` before editing it. A modified opened repo becomes a `/sase_final`
  repository obligation.
- **Rust boundary.** Domain decisions belong in sase-core
  (`rust_core_backend_boundary`); Python gathers inputs, adapts, and renders.
- **Wire compatibility.** New request fields are optional, with a serde default that
  reproduces today's behavior. New result fields are additive, with a serde default, and
  Python reads them by name. These are bug fixes, not user-selectable behavior, so they
  get no feature flags.
- **Core pin.** If one turn commits both sase-core and sase, the host moves
  `sase-core-revision.txt` automatically. A sase phase that needs an already-landed core
  commit runs `just ratchet-core-revision` (see
  `docs/contribute/rust-backend.md#the-ci-source-revision-pin`). It also rebuilds the
  workspace venv's binding (`just rust-install`) before running sase tests against new
  core APIs.
- **Verification.** Run `sase tool run check` in every repo you changed. Never run
  `just check-full`; it is explicit-only. In sase, read the `lint_and_test` reference
  memory before finishing. Never weaken an existing assertion to get a green run. Every
  behavior change needs a regression test that fails on the old code.
- **Reference memory.** Before editing `docs/`, read `docs_writing`. Before touching
  `tools/`, read `tools`. Before TUI changes, read `tui` and `tui_perf`. Before adding
  CLI options, read `cli_rules`. Before creating artifacts, read `sase_artifacts`.
- **Hosts.** Agents usually run on athena. Reach apollo with
  `ssh apollo 'zsh -lic "<command>"'`, so its normal PATH and providers load (read the
  `tailnet` reference memory). Host work is read-only unless the phase says otherwise.
  Host checks need the landed code installed. If `sase version --json` on a host
  predates the commit you need, ask for `sase update` on that host through `/sase_gate`
  and continue in the gate's follow-up. Never run `just install` or `just install-dev`
  yourself (`decisions:global-install-is-human-only`).
- **The user's TUI.** Only the user restarts their TUI. Ask through `/sase_questions`.
  Wait out long measurement windows with `/sase_monitor`, never inline.
- **Beads.** A phase that names a task bead closes it after verifying the bead's own
  acceptance, with `sase bead close <id> --note "<what you verified>"`. Phase workers
  never create beads. Record `PROPOSED FOLLOW-UP: <summary — detail>` notes on your own
  phase bead instead. Never close this epic's bead.

## Track 1 — the tmp reaper reclaims space

At triage time apollo had 18 GB free. Reclaimable scratch it could not free included 25
GB of cargo targets and 7.6 GB of agent TMPDIRs. Every hourly reap ran
`selected=198 removed=1 failed=197` and `dead_launch_preserved_incomplete=267`, yet
reported `Structured job result: ok`. Three defects cause this:

- **A (`sase-1bw`).** Any SSH login started after a scratch entry was born blocks that
  entry from the dead-launch backstop, the pressure pass, runner-exit cleanup, and
  `sase-1n8.4`'s monitor-settlement cleanup.
- **B (`sase-1ne`).** `fs::remove_dir_all` cannot delete bob-cli's `0555` test vaults.
- **C.** Both jobs report success while failing.

### Phase `reap-core` (sase-core)

**1. Permission repair before removal (`sase-1ne`).** Every pass removes through
`remove_snapshot` in `crates/sase_core/src/managed_tmp.rs`. In its directory branch,
when `fs::remove_dir_all` fails with `PermissionDenied`:

- Walk the candidate's own subtree, never its parent or bucket, using
  `symlink_metadata`, and never follow a symlink.
- For each real directory owned by the current euid that lacks owner write or execute,
  add `u+wx` and keep its other bits. Open the directory with `O_DIRECTORY | O_NOFOLLOW`
  and `fchmod` it through the existing `libc` dependency. Do not use
  `fs::set_permissions` on a path, because it follows symlinks.
- Leave directories owned by anyone else alone; their removal still fails and is
  reported.
- Retry `remove_dir_all` once. On a second failure, report both errors in the existing
  `RemovalStatus::Failed` message.
- Count bytes only on success, as today. File modes do not matter, because unlinking
  needs only the parent directory to be writable.

Tests:

- An aged `agent-tmp/<key>` entry holding `x/vault` at `0555` with `0444` files is
  removed, with its bytes counted.
- A symlink inside a candidate that points at an outside `0555` directory leaves that
  directory's mode and contents untouched, and the candidate is still removed.
- `failed_age_removal_is_reported_without_claiming_reclaimed_bytes` (bucket `0500`)
  still fails as before.
- A dead-launch candidate with a nested `0555` tree is removed.

Follow the AGENTS.md file-size rule: `managed_tmp.rs` is already past it, so put new
tests in a tests submodule if needed.

**2. System-service ancestry exemption (`sase-1bw`).** This goes in
`crates/sase_core/src/launch_scratch_liveness.rs`. Today `observe_unreadable` marks
every older candidate `complete=false` whenever a same-uid process P meets two
conditions:

- P's `environ` or `cwd` is unreadable. That covers non-dumpable processes, and also
  `systemd --user`, which is unreadable because of its capabilities.
- P started after the candidate's birth.

Add this exemption. P does not make a candidate incomplete when all of the following
hold:

- **No same-uid ancestor.** Walk from P's parent upward, reading `<proc>/<pid>/stat`
  (ppid is field 4, which `read_proc_stat` currently discards) and
  `<proc>/<pid>/status`. Every ancestor below pid 1 must have all four `Uid:` values
  (real, effective, saved, filesystem) different from our euid. Any match means no
  exemption. That rejects `sudo -u <me>` (its real uid stays the invoker's) and same-uid
  parents: `systemd --user`, the non-dumpable `sase_sudo_runner`, tmux, and shells.
- **Not an orphan.** P's own ppid is not 1. An orphan's current parent says nothing
  about where its environment came from.
- **Top ancestor is a system service.** The walk ends at the first ancestor T whose ppid
  is 1. The `0::` line of `<proc>/<T>/cgroup` must start with `/system.slice/`. That
  proves T is an init-started system service (`ssh.service`, `sshd@….service`,
  `tailscaled.service`). It is not a root daemon that an agent's `sudo` spawned or
  daemonized, because those keep the agent's `/user.slice/…` cgroup.
- **Pid-reuse guard.** Each ancestor's start time is no later than its child's.
- **Fail closed.** Depth is capped at 32, with a cycle guard. Any missing, unreadable,
  or unparsable `stat`, `status`, or `cgroup` file means no exemption. So does a cgroup
  v1-only host, and so does a host without systemd. Those hosts keep today's behavior.

The verdict is per process, so compute it once per pid per observation. It exempts
OpenSSH session processes (`sshd-session` → `[priv]` → listener) and Tailscale SSH
children. These keep blocking when they started after the candidate, which is
deliberate:

- `gpg-agent`, `ssh-agent` under `systemd --user`, and `(sd-pam)`, because each has a
  same-uid parent.
- Init-adopted orphans.
- Every `sase sudo` path. The user-side runner has a same-uid parent. The synchronous
  `sudo` has real uid = ours. A detached root worker is an orphan in the agent's
  `user.slice`.

Also in this step:

- Count exemptions in a new additive result field `exempted_system_service` (serde
  default 0). Add a summary diagnostic next to the pre-launch and zombie ones.
- Treat `ESRCH` (the process exited mid-scan) as missing in `is_missing`.
- Make per-pid incomplete diagnostics name the process:
  `"<pid> (<comm>, ppid <ppid>): environ unreadable: <error>"`.
- In `managed_tmp.rs`, append the observer's per-pid diagnostics to dead-launch
  `preserved_incomplete` skip reasons and to pressure-pass skip reasons. Today
  `reap_dead_launch` and `apply_pressure_liveness` drop them. Cap them at 8 lines plus
  "and N more".

Tests (extend the module's tempdir proc fixtures):

- Add helpers that write `status` (`Uid:`, `PPid:`), `cgroup`, and a `stat` with a
  configurable ppid and start time.
- **Exempt:** an sshd-like chain makes the candidate complete, with the exemption
  counted. P (same-uid, unreadable, post-birth) → a root `[priv]` process → a root
  listener with ppid 1 in `/system.slice/ssh.service`.
- **Not exempt (stays incomplete):**
  - a parent with real uid = ours and effective uid 0 (sudo);
  - a same-uid parent;
  - P with ppid 1;
  - a root chain whose top is in `/user.slice/…`;
  - a missing ancestor `status`;
  - an ancestor that started after its child;
  - an ancestry cycle.
- Existing tests stay unchanged. Their fixtures use ppid 1 with no `status`, so they
  still fail closed.
- Add a binding round-trip test for the new result field in
  `crates/sase_core_py/src/config/tests.rs`.

The wire schema version does not change, because the field is additive.

**3.** Run `sase tool run check` in sase-core and land the commit. Phase `reap-sase`
moves sase's pin past it.

### Phase `reap-sase` (sase)

1. Run `just ratchet-core-revision` past `reap-core`'s commit, then `just rust-install`.
2. **`managed_tmp_reap`** (`src/sase/scripts/sase_chop_managed_tmp_reap.py`):
   - After each root's `result.describe()`, log up to 5 `removal_errors` and up to 5
     skip reasons. Skip reasons now name the blocking pid and command. If the Python
     result object does not expose them yet, thread them through
     `src/sase/core/disk_footprint_reap_managed_tmp.py`.
   - When combined `failed > 0`, set `result.status = "check_error"` with reason
     `removal_failed`, and keep every counter. Do this after `emit_summary`, the way
     `src/sase/scripts/sase_chop_artifact_run_prune.py` does.
   - Otherwise keep today's `ok`, or `no_op` with `nothing_stale`. Incomplete
     observations alone are not an error.
3. **`disk_pressure`** (`src/sase/scripts/sase_chop_disk_pressure.py`):
   - Call `run_disk_reap(...)` before the notification. Then send one notification that
     includes the reap outcome: changed and failed owners, bytes reclaimed, and space
     available after.
   - A notification failure (any exception from `notify_workflow_complete`, including
     the ENOSPC `ValueError` seen at 04:50) never prevents or undoes a reclaim. Write
     one bounded stderr line and count `notify_failed=1`.
   - After the reap, re-measure free space on the pressured filesystem with a `statvfs`
     (not a full footprint).
   - Set the status after `emit_summary`. Judge owners by their
     `cleanup_outcome["status"]` (`success | failed | blocked | incomplete`), not by
     `DiskReapStep.failed`, which is also true for incomplete observations. Leave
     `ChopResultBuilder.from_summary`'s global mapping alone, because other jobs rely on
     it.

     | Situation                                                                   | Status        | Reason                             |
     | --------------------------------------------------------------------------- | ------------- | ---------------------------------- |
     | No pressure                                                                 | `no_op`       | `space_ok` (unchanged)             |
     | Any owner `failed`, or the notification failed                              | `check_error` | the failed owners, `notify_failed` |
     | Pressure persists after the reap and an owner was `blocked` or `incomplete` | `check_error` | `recovery_blocked`                 |
     | Some owner changed state and none failed                                    | `ok`          | —                                  |
     | Nothing eligible and nothing blocked                                        | `no_op`       | `nothing_eligible`                 |

     Keep `changed`, `failed`, owner names, available bytes, and the recovery target in
     the counters and summary.

4. **Tests.** Extend `tests/test_axe_chop_output_contract_digest_and_reap.py`, updating
   exact-counter assertions only for the new fields.
   - A removal failure gives `check_error` and logs the error lines.
   - Incomplete-only results are not `check_error`.
   - A call-order recorder proves the reap runs before the notification.
   - A notification raising ENOSPC still reaps, gives `check_error`, and writes the
     stderr line.
   - A failed owner with partial effects gives `check_error`, with its counters intact.
   - Pressure that persists after a blocked owner gives `recovery_blocked`.
   - A changed owner gives `ok`, and space ok gives `no_op`.
   - Through the binding, a fake proc root with an sshd-like chain makes
     `observe_launch_scratch_liveness` (and `cleanup_launch_scratch_key`) treat the
     candidate as complete, and an orphan stays incomplete. Use
     `tests/test_managed_tmp_reaper_dead_launch.py` or a sibling file.
5. **Docs.** Update these places so they describe the exemption and its fail-closed
   cases, the permission repair, the richer skip reasons, `check_error` on removal
   failures, and disk_pressure's reap-then-notify order and status table:
   - `docs/reference/automation/scheduler-housekeeping.md` (the `managed_tmp_reap` and
     `disk_pressure` sections);
   - the dead-launch and pressure-liveness paragraphs of
     `docs/reference/config/runtime-and-storage.md`;
   - the liveness wire paragraph in `docs/contribute/rust-backend.md`;
   - the two job descriptions in `src/sase/default_config.yml`, if they state the old
     behavior.
6. Leave `sase-1bw` and `sase-1ne` open, because their acceptance is production reclaim.
   Add a note to each with the test evidence. The land agent closes them after host
   checks.

### Phase `bobcli-vault` (bob-cli repo)

In `tests/cli/completion/vault.rs`, `completion_is_read_only` calls `lock_tree` (`0555`
directories, `0444` files) and never unlocks. `TempDir`'s drop
(`remove_dir_all_if_exists` in `tests/cli/support.rs`) then fails, and every run leaks
`bob-cli-complete-vault-*/vault` under `$TMPDIR`. Apollo had 295 of them.

1. Make `lock_tree` return a guard whose `Drop` restores owner write (directories
   `0755`, files `0644`). Keep it alive for the whole read-only probe, so cleanup works
   even when an assertion panics.
2. In `tests/cli/support.rs`, when `TempDir::drop` hits `PermissionDenied`, add owner
   write to directories inside its own tree (never following symlinks) and retry once.
3. Add a test proving the locked fixture directory is gone after the guard and the
   `TempDir` drop.
4. Run bob-cli's check the way its `AGENTS.md` says, and commit in bob-cli.

## Track 2 — phase `cleanup-encoding` (sase; closes `sase-1nb`)

`agent_state_to_bundle_dict` (`src/sase/ace/tui/models/agent_bundle.py:78-123`) passes
`wait_epic_follows`, a list of `EpicFollowView` dataclasses, through raw.
`from_bundle_dict` has no branch for it either. Several paths serialize through it:
cleanup (`serialize_agent` in `src/sase/ace/tui/actions/cleanup_payload.py`), the kill
and dismiss payloads (which carry the whole `agents_with_children` snapshot, in
`_kill_procs.py` and `_dismissing.py`), `_marking.py`, and dismissed-bundle persistence
(`src/sase/ace/dismissed_agents_bundles.py`). So one waiting agent breaks kill and
dismiss for the whole roster.

1. **Encoder.** Encode `wait_epic_follows` as a list of plain dicts with exactly the ten
   `EpicFollowView` fields (`dataclasses.asdict`; tuples become lists). These are the
   same keys as the Rust `WaitEpicFollowEntryWire` and `waiting.json`. No `default=str`,
   and do not drop the field.
2. **Decoder.** In `from_bundle_dict`, map the field through
   `list(epic_follow_views(value))`. Import it lazily: `sase.core.wait_epic_follow_view`
   must stay out of the TUI startup import closure
   (`tests/ace/tui/test_app_import_budget.py`).
3. **Guard the next field.** Add a test that fails when any bundle-persisted `Agent`
   field holds a value `json.dumps(..., allow_nan=False)` cannot encode. Build it from a
   fixture agent with every nested-type field populated.
4. **Harness.** Make `tests/_agent_cleanup_proc_helpers.py` round-trip the submitted
   payload through `json.dumps` and `json.loads`, the way `src/sase/ops/io.py` does, so
   the kill and dismiss harness tests catch non-JSON payloads.
5. **Tests.**
   - Empty and populated `wait_epic_follows` round-trip equal through the bundle dict
     and JSON, for every persisted state (`launching`, `following`, and `blocked` with a
     reason), covering all ten fields.
   - `serialize_agent` → JSON → `agent_from_json` round-trips equal.
   - A full dismiss through `_dismiss_through_cleanup_subprocess`
     (`tests/ace/tui/actions/test_cleanup_payload.py`) succeeds with a waiting agent in
     `agents_with_children`.
   - A bulk kill payload with one waiting agent among others encodes.
6. Close `sase-1nb`.

## Track 3 — phase `doctor-macros` (sase; closes `sase-1na`)

`_model_preset_tokens` (`src/sase/doctor/checks_config_macros.py:571-620`) expands raw
macro bodies with `process_macro_references(content)`. That skips the launch-time Jinja
render, so `#research/audio(edition={{ audio_edition }})` fails the core enum check.
`_abort_expansion` then calls `sys.exit(1)`, and `SystemExit` escapes every
`except Exception`, which leaves no JSON.

1. Call `process_macro_references(content, raise_on_error=True)`. Catch `MacroError` and
   `MacroValidationError`; the latter is not a `MacroError`.
2. Classify the failure:
   - **Parameterized.** The raw body has template markers (`sase.macro.jinja_inspect`'s
     `has_jinja`). It cannot be evaluated statically, so return a skipped scan, count it
     in `data["skipped_parameterized"]`, and mention the count in the details.
   - **Concrete broken preset.** The raw body has a model directive and still fails.
     Report a WARN problem row: `"<name>: cannot expand: <error>"`.
   - **Anything else.** Skip it as today.
3. **Defense in depth.** `run_check_spec_safely` (`src/sase/diagnostics/registry.py`)
   also catches `SystemExit` (not `KeyboardInterrupt`). It turns it into the same ERROR
   check shape, with "check tried to exit the process (code N)". One misbehaving check
   can then never end `sase doctor` without JSON.
4. Do not weaken the plugin enum, and do not special-case `audio_edition`.
5. **Tests.** Extend `tests/doctor/test_checks_config_model_macros.py`. Use an inline
   ENUM-input macro (`choices=(brief, full)`), so no plugin discovery is needed.
   - A macro referencing it as `#audio(edition={{ audio_edition }})` returns normally
     with `skipped_parameterized == 1`.
   - A concrete invalid model preset still warns.
   - A concrete preset whose nested reference has a bad enum value gives a WARN row.
   - A registry check raising `SystemExit(1)` becomes an ERROR check, and a later
     sentinel check still runs (`tests/test_diagnostics_models.py`).
   - `sase doctor -j -C config.model_macros` with the problematic macro prints one
     parseable JSON document and no prose (`tests/main/test_doctor_command.py`).
6. **Docs.** Say that parameterized macros are skipped, in
   `docs/reference/providers/models.md` (near the check's description) and in
   `docs/reference/diagnostics.md` if it describes the check.
7. **Out of scope.** Raw expansion still runs `$(…)` command substitutions in macro
   arguments, although the docs call this check read-only. Record that as a
   `PROPOSED FOLLOW-UP:`.
8. Close `sase-1na`.

## Track 4 — TUI GC freezes

Athena's TUI (Python 3.14.7) has had main-thread gen-2 collections of 10–32 s, all
triggered as `idle`, `backstop`, or `rss_backstop`. Apollo's (3.12.3) has reached about
6 s. The heartbeat window around the 32 s pause shows about 200k major faults and
0.9–1.9 GiB of swap, and the frozen set decays over time. Several cyclic producers are
plausible but unproven:

- each agents load rebuilds a cyclic `Agent` graph and deep-copies it
  (`copy_agent_graph`);
- fleet projection rebuilds its rows on every apply.

The tools cannot yet attribute the garbage. `tools/tui_freeze_report` also misreads real
startup rows.

### Phase `gc-instrument` (sase; closes `sase-1f4`)

1. **`tools/tui_freeze_report`:**
   - **Startup rows.** Real rows carry an ISO `timestamp`, not `ts`, so today every
     startup window is empty and rows land in `unknown`. Fix that, and fix the fixture
     at `tests/test_tui_freeze_report_tool.py:128` to the real shape.
   - Read `tui_stalls.jsonl.1` along with the active file, behind an opt-out flag.
   - Report the `idle`, `backstop`, `rss_backstop`, `startup_freeze`, and `automatic`
     triggers separately.
   - Count backstop as well as automatic gen-2 collections near input.
   - Use the heartbeat's exact `loop_hitch_seconds` and suppressed totals.
   - Add `--acceptance`, which marks each `sase-1f3` target pass or fail per instance.
2. **Telemetry** (`src/sase/ace/tui/util/gc_telemetry.py`):
   - **`tui_gc_pause` rows** gain:
     - the wall-clock collection start, derived from `start_mono` through the exec-aware
       clock;
     - seconds since the last input at that start;
     - the process major-fault delta across the collection. The callback must stay
       lock-free and I/O-free, as its structural test requires. Take faults from
       `resource.getrusage`, or sample them at flush time from the ring timestamps.
   - **Heartbeat rows** gain `gc.get_stats()`, `sys.getallocatedblocks()`, and the
     Python version.
   - **Startup rows** gain `source_revision`, per `sase-1f4`. Read `.git/HEAD` and its
     ref without a subprocess, captured once at import.
3. **Opt-in census** (`SASE_TUI_GC_CENSUS=1`, default off). Collector calls stay in
   `src/sase/ace/tui/util/gc_policy.py` and run under `gc_trigger`. At most once every
   10 minutes, and only on an idle-triggered full collection, it does two things:
   - **Garbage census.** Run that collection with `gc.DEBUG_SAVEALL` and write a
     `tui_gc_census` row with the 40 most common collected types (`module.qualname`).
     Then clear `gc.garbage` and collect again to free them.
   - **Survivor census.** Record the 40 most common types among surviving tracked
     objects.

   Each row records the census's own cost. Document that it lengthens the pause.

4. **Tests:**
   - the report fixes (real startup rows, the rotated file, the trigger split,
     acceptance evaluation);
   - the new row fields;
   - the census is off by default and rate-limited;
   - the callback's structural test still passes.
5. **Docs.** Update the GC pause and heartbeat rows, frozen-share report, and heap
   sampler sections of `docs/contribute/perf-runbook.md`.
6. Close `sase-1f4`.

### Phase `gc-measure` (operational analysis)

1. **Restart.** Ask the user through `/sase_questions` to restart their ACE TUI on
   athena (and on apollo, if they run one there). The restart should land on code that
   contains `gc-instrument`, with `SASE_TUI_GC_CENSUS=1` set. Confirm it from
   `tui_startup.jsonl` rows carrying the new `source_revision`.
2. **Capture.** Wait at least 20 minutes after the restart. Then, using `/sase_monitor`,
   capture a busy hour and a 4-hour window on one `app_instance_id` per host. Run
   `tools/tui_freeze_report --acceptance` on each host (on apollo over SSH, read-only).

   > [!decision] gc_soak Also wait out the 24-hour soak on the same instance before
   > finishing, and include it in the report.

   > [!decision] gc_soak = no Skip the 24-hour soak here. `gc-fix`'s acceptance runs it
   > on the fixed code.

3. **Attribute.** Answer these from the data:
   - Which types dominate the collected garbage and the survivors?
   - How much of each long pause is swap-in? Correlate pause length with the major-fault
     delta and VmSwap.
   - How much does each candidate producer allocate per refresh?
     - the per-load agent graph rebuild and `copy_agent_graph`
       (`src/sase/ace/tui/actions/agents/_loading_graph.py`);
     - `Agent` back-reference cycles (`models/_agent_state_session.py`);
     - fleet projection on every apply (`actions/agents/_fleet_refresh.py`,
       `models/fleet_agents.py`);
     - artifact-links snapshot copies;
     - anything else the census surfaces.
   - How do Python 3.14.7 and 3.12.3 compare, and prompt-active periods versus idle
     ones?
4. **Report.** Write it in the research sidecar (`sase repo open research`, current
   month directory). Include a before/after table against `sase-1f3`'s targets and the
   `sase-1ez` baselines, and the ranked producers with evidence. Attach it to the phase
   bead with `sase bead ref add`.
5. **Close `sase-1f3`.** Record the measurement. Under `gc_soak = no`, the close note
   says the 24-hour soak moved to `gc-fix`'s acceptance.

### Phase `gc-fix` (sase; plans first; closes `sase-1nc`)

Plan from the `gc-measure` report, then implement.

- **Constraints.**
  - Reduce allocation and retention at the measured producers before changing collector
    policy.
  - Keep the memory backstops.
  - Do not merely raise thresholds.
  - Do not assume moving collection to another thread stops it pausing the process.
- **Known policy conflict to resolve.** `BACKSTOP_QUIET_S=1.0` in `gc_policy.py` lets a
  backstop collection start one second after input. That breaks the "no automatic or
  backstop gen-2 within 2 s of input" target by construction.
- **Acceptance.** Have the user restart the TUI onto the fix on both hosts and Python
  versions, then show every `sase-1f3` target passing with
  `tools/tui_freeze_report --acceptance`:
  - idle-collection p95 ≤ 500 ms;
  - total GC under 3% of wall time;
  - hitches under 0.5% of wall time;
  - no automatic or backstop full collection within 2 s of input;
  - RSS growth under 0.5 GB over 4 h, with VmSwap about 0;
  - j/k p95 under 16 ms;
  - and, under `gc_soak = no`, the 24-hour soak.
- Close `sase-1nc`. File each target miss as a `PROPOSED FOLLOW-UP:`.

## Track 5 — publication backlog

Athena has about 3,360 quarantined agent-hood publication requests. About 2,840 of them
last failed on the unqualified 120 s drain timeout, and about 430 on
`agents sync lock is busy`. Apollo has 453 quarantined requests and 284 retired ones,
including 335 rebase-conflict rows on one replay commit. New requests still quarantine
within about 40 minutes. These causes are visible in the code but unmeasured:

- **Fixed cost per drain.** Every drain builds a whole-project hood inventory first. On
  athena that is about 19,000 git subprocesses, mostly `commit_record` lookups over
  9,400 commit markers. It also validates and renders every owner for each hood.
- **Budget charging.** A repo-wide failure charges an attempt to every active request
  (`_record_failure(logical_keys=None)` in
  `src/sase/agents_sync/commit_publication.py`). That contradicts
  `docs/reference/agents/hood-sync.md:330-333`.
- **Timeout side effects.** The SIGALRM timeout is swallowed inside per-hood publish. It
  orphans streaming `git push` children and leaves `prompts/` dirty.
- **Stale local commits.** On apollo, a stale local commit is replayed onto shared
  derived files that both machines render.

### Phase `pub-measure` (sase)

1. **Supersede the stalled epic.** Close `sase-11o.2`, then `sase-11o`, with
   `sase bead close <id> -R superseded --reason "<superseded by this epic's publication track; sase-11o.1's manifest slimming stays landed>"`.
2. **Permanent instrumentation:**
   - **One structured line per commit-triggered drain**, from
     `_drain_agent_publications` and
     `src/sase/agents_sync/commit_publication_transaction.py`. Write it to a bounded,
     rotated JSONL log that follows the existing `~/.sase/logs` conventions. It records
     the project, the active request count, distinct hoods, the outcome and error class,
     whether the alarm fired, and in which stage. It also records durations for:
     - lock wait;
     - payload clean;
     - pull and rebase;
     - the inventory build, with runs scanned, `commit_record` calls, and prompt and
       chat bytes read;
     - per-hood plan, validate, render, and apply;
     - prompt-archive preparation;
     - commit, push, verify, and cleanup.
   - **A sync-lock holder record.** When `bounded_agents_lock`
     (`src/sase/agents_sync/git_sync_ops.py`) acquires, write
     `{pid, operation, agent, started_at}` beside the flock file, and remove it on
     release. A lock-busy error then names the holder and its age. Ignore a record whose
     pid is gone.
   - **`state.agent_publication_outbox` doctor additions**
     (`src/sase/doctor/checks_agent_publication.py`):
     - a histogram of last-error classes;
     - the oldest active request's age;
     - hidden-clone state: commits ahead of and behind upstream, a rebase in progress,
       untracked `files/objects`, and dirty `prompts/` or `artifacts/`.
3. **Measure.** This step is read-only: no retries, resets, or `--retry-*` flags.
   - **Athena.** Add a profiling harness, `tools/agents_publication_profile`. It runs
     `build_project_hood_inventory` and `plan_hoods`, which write nothing, for one large
     hood (`research`) and one small hood against real state, and prints the stage
     breakdown. Run it with `SASE_SDD_GIT_SLOW_MS=0` to count git subprocesses.
   - **Apollo, over SSH.** Collect the doctor histogram. Inspect the hidden agents clone
     with `git status`, `git log @{u}..HEAD --name-status`, the rebase state, and its
     untracked `files/objects`. Identify the replay commit and which shared files
     conflict.
4. **Report.** Write it in the research sidecar (current month directory) and attach it
   to the phase bead. For each failure class (unqualified timeout, lock busy,
   `index.lock`, unstaged changes, rebase conflict, manifest mismatch), give:
   - the measured stage breakdown;
   - the root cause, with evidence;
   - the unique unpublished content that recovery must preserve.
5. **Tests** for the timing record, the holder record, and the doctor additions.
   **Docs:** update the outbox diagnostics in `docs/reference/agents/hood-sync.md` and
   `docs/contribute/agents-sidecar-sync.md`.

### Phase `pub-fix` (sase, and sase-core if needed; plans first)

Plan from the `pub-measure` report, then implement. It may author an epic if the
measured causes need one. It must cover:

1. **Failure accounting.** Repo-wide failures (lock busy, a timeout outside per-hood
   work, clone, pull, push, cleanup) stop charging every active request's attempt
   budget. Per-hood failures still charge their own rows. Update the tests that pin
   today's behavior (`tests/agents_sync/test_commit_publication_bounded_drain.py` and
   `tests/agents_sync/test_commit_publication_queue.py`).
2. **Timeout hygiene.**
   - `_PublicationDrainTimedOut` is no longer swallowed by the per-hood
     `except Exception`.
   - A timed-out streaming push or fetch is terminated with its process group in
     `src/sase/sdd/_git.py`.
   - Drain cleanup also cleans `prompts/` and `artifacts/`, as full sync does.
3. **Per-drain cost.** Bound the measured dominant cost. Examples: scope the inventory
   to the hoods being published, cache commit records, and validate or render only what
   changed. The proof is a representative, previously failing request that publishes
   inside the existing 120 s bound.
4. **Apollo's conflicts.** Resolve the stale local commit and shared-derived-file
   conflicts the report identifies, without losing unique unpublished content.
5. **Bounded retries.** Give `sase agent sync --retry-quarantined` a limit, and an
   error-class filter if the report justifies one. Keep the selection policy in
   sase-core's `select_publication_retries`.
6. **Acceptance**, from the audit:
   - a previously failing request publishes within the bound;
   - the remote manifest and its referenced files agree;
   - replay is idempotent;
   - an interrupted publication resumes;
   - newly completed agents publish.

### Phase `pub-recover` (operational, athena then apollo)

1. Confirm the installed sase on both hosts contains `pub-fix`. If not, request the
   update through `/sase_gate`.
2. **Back up first.** In each project's hidden agents clone, inventory the unpushed
   commits, untracked `files/objects`, and prompt docs. Back them up (for example, a
   `git bundle` of `@{u}..HEAD` plus a tarball of the untracked objects) under a dated
   directory in `~/.sase/` before any reset or conflict resolution. Then resolve
   apollo's wedged replay with `pub-fix`'s procedure.
3. **Bounded retries.** Retry quarantined requests per project (sase and bob-cli) with
   the new limit. Start at 25, and grow only after a batch publishes with no new
   quarantines. Between batches, use `/sase_monitor` and
   `sase doctor -C state.agent_publication_outbox -j` to confirm three things:
   - the quarantine count and the oldest active age are falling;
   - sampled agent pages resolve on GitHub;
   - the manifest and its files agree.

   Stop and record the evidence when a batch regresses. Never retry everything at once,
   and never drop requests. Retry retired requests only in classes whose cause `pub-fix`
   removed.

4. Confirm that an agent completed after the install publishes without manual help.
5. **Close or note.** When quarantine is near zero on both hosts, close `sase-10x` with
   the before and after counts and the cause. Otherwise, note what remains on `sase-10x`
   and add a `PROPOSED FOLLOW-UP:`.

## Track 6 — artifact-link endpoints converge

Every tick, `artifact_link_backfill` publishes all queued operations in one bead
projection call. One unresolvable endpoint fails that whole call, so none of them is
acknowledged. The endpoints failing today:

| Endpoint     | Store   | Cause                     |
| ------------ | ------- | ------------------------- |
| `sase-164`   | sase    | Removed (tombstoned)      |
| `sase-102`   | bob-cli | Exists in sase's store    |
| `bob-cli-8s` | tailnet | Exists in bob-cli's store |
| `bob-cli-5z` | apollo  | Unverified                |

Four mechanisms keep the failure going:

- `set_bead_link_projections` is all-or-nothing.
- `_published_operation_ids` applies one shared `bead_receipt` to every operation.
- The exception path skips finalize, so applied chunks stay uncommitted, and machine
  recovery stashes them every tick (207 stashes in bob-cli's hidden beads clone).
- Derived candidates are re-appended every tick (21,101 outbox entries for 221 operation
  ids on athena). `_swept_refs_after_chunk` then marks nothing swept for any chunk with
  an error.

Status says `ok` with `failed_projects=0`.

### Phase `links-core` (sase-core)

1. **Skip mode.** `set_bead_link_projections`
   (`crates/sase_core/src/bead/mutation/links.rs`) gains an optional request field
   `missing_holder: abort | skip`, defaulting to `abort`, which is today's
   all-or-nothing behavior.
   - Under `skip`, a request whose holder bead is absent is not applied, and the rest
     apply in the same single mutation, with the same idempotency receipts.
   - The response gains per-request outcomes:
     `{index, status: applied | unchanged | missing, missing_kind: absent | removed}`.
     `removed` means the holder's lineage stream ends in `issue_removed`.
   - Keep `link_projection_batch_is_transactional_on_invalid_middle_request` for
     `abort`, and add its `skip` counterpart. Update the doc comment.
2. **Per-endpoint receipts.** `ArtifactLinkPublicationEvidenceWire`
   (`crates/sase_core/src/artifact_link/ownership.rs`, which uses `deny_unknown_fields`)
   gains an optional `bead_endpoints` list.
   - Each entry is `{endpoint_ref, outcome}`, where `outcome` is
     `committed | removed | absent | foreign{owner_project} | transient{detail}`.
   - When present, it replaces the single `bead_receipt` boolean. When absent, behavior
     is exactly today's.
   - `artifact_link_publication_receipt` returns a per-operation `disposition`:
     - **`acknowledged`:** every bead endpoint is committed and the other requirements
       are met;
     - **`parked`:** no endpoint is transient, at least one is `removed`, `absent`, or
       `foreign`, and everything else is committed;
     - **`pending`:** anything else.
   - `pending_reasons` name each endpoint and its outcome, replacing the generic
     "artifact-link bead projection is not committed" in that case.
3. **Bindings.** Expose both in the beads and artifact-link binding domains of
   `crates/sase_core_py`, with round-trip tests.
4. Run `sase tool run check` and land the commit. `links-project` moves the pin.

### Phase `links-project` (sase)

1. Run `just ratchet-core-revision`, then `just rust-install`.
2. **Classify endpoints.** In `apply_events_to_beads`
   (`src/sase/sdd/_artifact_link_event_project.py`), route every bead endpoint before
   chunking. Use the existing router (`src/sase/bead/cross_project.py`
   `enabled_project_store_snapshots` and `src/sase/core/bead_target_routing_facade.py`).
   | Where the bead is | Outcome | | --- | --- | | This store | local | | Another
   enabled project's store | `foreign(<owner>)` | | Tombstoned | `removed` | | Nowhere
   relevant | `absent` | | Owning store unavailable | `transient` |
3. **Project.** Project only local endpoints, with `missing_holder=skip`. An endpoint
   routed local but missing from the hidden machine clone is `transient`; that is clone
   lag, like the one-off `bob-cli-87.3.1`.
4. **Exception safety.** When a chunk raises, finalize and commit the chunks that
   already applied (`_finalize_bead_projection`) instead of leaving residue for machine
   recovery to stash. Report the failing chunk's endpoints as `transient`.
5. **Per-operation evidence.** `_published_operation_ids`
   (`src/sase/sdd/_artifact_link_event_publish.py`) passes each operation its own
   `bead_endpoints`, built in `src/sase/sdd/_artifact_link_event_ownership.py`, instead
   of one shared boolean.
6. **Tests.** Build a mixed-batch fixture with two project stores. Reuse the `_store`
   helper in `tests/test_bead/test_cross_project.py` and extend
   `tests/sdd/_artifact_link_acceptance_helpers.py`.
   - A local endpoint is committed and acknowledged.
   - Foreign, removed, and absent endpoints are parked with typed reasons, and the
     foreign one names its owner.
   - A transient endpoint stays pending.
   - Replay is idempotent.
   - An exception in chunk 2 still commits chunk 1 and leaves no uncommitted residue.

   Change existing expectations in
   `tests/sdd/test_artifact_link_event_projection_batch.py` only where the per-operation
   semantics change on purpose.

### Phase `links-converge` (sase)

1. **Parked ledger.** Parked operations leave the active outbox for a replayable
   `artifact-link-outbox-parked.jsonl`, beside the existing dropped-entry audit file.
   Each record carries the operation, the typed per-endpoint reasons, and when it was
   parked. The immutable link event stays where it is.
   - The job re-checks parked entries at most once every 24 hours. An entry whose
     endpoints now classify as local and resolvable goes back to the queue.
   - Dropped stays final; parked is never dropped by this phase.
2. **Idempotent append.** `append_artifact_link_outbox_entry`
   (`src/sase/sdd/_artifact_link_outbox_io.py`) does nothing when the same operation id
   with an identical event is already queued. The next rewrite compacts the existing
   duplicates. The collision rejection for a different event under the same id is
   unchanged.
3. **Per-document sweep.** In `_swept_refs_after_chunk`
   (`src/sase/sdd/artifact_link_backfill.py`), a document counts as swept when each of
   its candidates is durable or parked. An error attributed to one document no longer
   holds back the rest of its chunk.
4. **Job status** (`src/sase/scripts/sase_chop_artifact_link_backfill.py`):
   - Add the counters `bead_projection_failed`, `parked_new`, `parked_total`, and
     `transient`.
   - Set `check_error` with reason `bead_projection_failed` when any project hit a hard
     projection failure this tick, even if traversal completed.
   - Log up to 5 newly parked operations with their reasons. Parked alone is not an
     error.
5. **Doctor.** Report parked counts by reason in
   `src/sase/doctor/checks_artifact_links.py`.
6. **Tests.**
   - Two ticks over the mixed fixture: tick 1 parks, tick 2 does no projection work for
     parked operations, and `sweep_remaining` reaches 0.
   - A duplicate append is a no-op.
   - A parked operation re-queues after its bead is created.
   - A hard failure gives `check_error`.
   - Doctor shows the parked counts.
7. **Docs.** Update every docs page that describes the artifact-link outbox or backfill
   job; find them with grep.
8. `sase-1c5` is closed by the land agent after host checks.

## Out of scope

Each of these becomes a follow-up through the land agent if not already tracked:

- Projecting foreign bead endpoints into their owning project's store with per-owner
  receipts. This epic parks them.
- Recording read links for a bead under the bead's owning project.
- Better owner attribution in `disk_footprint` (the "unattributed 192.7 GiB" row).
- The audit's "signals not promoted" items. `sase-1j1` already owns the gateway 504s.
- Reusing one cargo target across an agent session's turns.
- Blanket publication retries or longer timeouts.
- Deleting old `sase recovery` stashes in hidden bead clones.
- The doctor scan's `$(…)` execution.

## Landing (for the epic's land agent)

1. **Phase beads.** All 14 phase beads are closed.
2. **Host checks.** Get the landed code installed on athena and apollo, through
   `/sase_gate` for `sase update` if needed. Then verify on both hosts:
   - **Reaper.** With your own SSH session to apollo open, a dry-run `sase disk reap -j`
     no longer preserves dead-launch entries because of sshd. The next hourly
     `managed_tmp_reap` shows dead-launch removals above zero and `failed` near zero.
     Any remaining failures report `check_error` with the error lines. Free space
     recovers. Close `sase-1bw` and `sase-1ne` with the numbers.
   - **Artifact links.** The next `artifact_link_backfill` ticks stop repeating
     `Issue not found`, and park `sase-164`, `sase-102`, and `bob-cli-8s` (and
     `bob-cli-5z` on apollo, whatever its class) with typed reasons. `sweep_remaining`
     converges. Close `sase-1c5`.
   - **Doctor.** `sase doctor -j` prints one JSON document on both hosts.
3. **Task beads.** Re-verify and close any of these still open: `sase-1nb`, `sase-1na`,
   `sase-1f4`, `sase-1f3`, `sase-10x`, `sase-1nc`. For any that legitimately stays open,
   record why in the epic close note.
4. **Stale epics.** Add a note to `sase-11h` pointing at `reap-sase`, which made the
   disk-job statuses truthful. Add a note to `sase-12y` pointing at the links phases.
   Close neither; their remaining scope is unrelated.
5. **Follow-ups.** File each phase's `PROPOSED FOLLOW-UP:` notes and the out-of-scope
   items above through `/sase_new_task`, after its duplicate check.
6. Do not run `just check-full`.
