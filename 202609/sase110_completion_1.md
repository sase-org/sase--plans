---
tier: tale
title: Complete and land epic sase-110 (agent sudo requests)
goal:
  Phases sase-110.7 and sase-110.8 are finished with live evidence, the
  agent_sudo_requests flag is removed, and the epic's land agent closes sase-110.
size: medium
proposed_by: bbugyi200.athena.0l9.f0
create_time: 2026-09-15 12:40:16
status: wip
---

# Complete and land epic sase-110 (agent sudo requests)

## Context

Epic `sase-110` (Agent sudo requests with terminal-handoff authentication) has phases
1–6 closed. Phases `sase-110.7` (athena-policy) and `sase-110.8` (acceptance) are
`in_progress` with no live workers (the .7 worker's sudo gate shell timed out on
2026-09-15; the .8 worker exited without doing work while waiting on .7). This plan
finishes the remaining phase work directly, closes both phase beads with evidence, and
recovers the epic's land flow so the land agent closes `sase-110` itself.

Verified ground truth (athena, 2026-09-15):

1. **Sudoers tightening is live.** `/etc/sudoers.d/00-bryan-password-required` exists
   (root:root 0440) and headless sudo fails for agents (agent 0l3.f0's chezmoi run died
   with "a terminal is required to read the password" on 2026-09-15). No temporary CWD
   NOPASSWD drop-in remains in `/etc/sudoers.d/` (only the tightening drop-in and
   README).
2. **`kernel.yama.ptrace_scope` is still 0** and `/etc/sysctl.d/` has no ptrace drop-in.
   This is the unfinished root-side half of phase 7.
3. **No pending sudo gates** (`sase sudo list` is empty against live host state).
4. **The chezmoi guard half of phase 7 was split out** (see the COORDINATION note on
   bead `sase-110`) into the separate tale plan `chezmoi_headless_sudo_guards`, which
   was still pending its plan-approval gate (raised by agent 0la.f0) when this plan was
   authored. Phase 7's remaining scope is the athena policy half only.
5. **Phase 8 has not started.** There is no canary absence suite
   (`grep -ril canary tests/` finds only unrelated files), no `docs/sudo.md`, and the
   `agent_sudo_requests` beta flag (bead `sase-111`, default off) is still registered.
   The flag surface is small: `src/sase/sudo/feature.py` plus the registry entry in
   `src/sase/feature_flags/registry.py`, referenced by `tests/test_sudo_gate.py`,
   `tests/ace/tui/test_notification_sudo.py`, and
   `tests/ace/tui/test_notification_custom_gate.py`.
6. **The remote-raised proof stays best-effort.** Epic `sase-xe.16` (apollo remote
   dispatch enrollment) is still `in_progress`, so per the epic plan the
   `%dispatch:apollo` remote-raised path is not gated on — only the machine-targeted
   proof from athena is required.
7. **Agent environments carry a launch-time `SASE_FEATURE_FLAGS` pin** that can be stale
   (it pins `agent_sudo_requests` off). When live host sudo state matters before the
   flag is removed, run the command with the pin dropped
   (`env -u SASE_FEATURE_FLAGS sase sudo ...`), as the approved
   `sudo_handoff_stale_flag_pin` plan established.

Binding background reading (audited reads, in order, before any file change):

- `sase artifact read plan:202609/agent_sudo_requests.md "<why>"` — the epic plan; its
  phase `athena-policy` and `acceptance` sections and the "Acceptance gates" table
  define this plan's completion criteria.
- `sase artifact read research:202609/agent_sudo_requests/agent_sudo_requests.md "<why>"`
  — binding for the canary suite (port acceptance list A minus root-runner cases) and
  the UX polish section.

Required memory reads (via `/sase_memory_read`): `lint_and_test.md` (mandatory before
finishing any turn that changed tracked files), `sase_flags.md` (before the flag
removal), `sase_beads.md` (before closing beads), `cli_rules.md` (before touching CLI
epilogs/help), `tui_perf.md` (before the 🔐 panel polish). Use `/sase_repo` before
touching the chezmoi repo, and `/sase_sudo` for every root-side change — never raw sudo.

## Work

Ordered; steps 4 and 6 end the current agent turn via a sudo gate handoff, so carry the
remaining checklist forward in each request's follow-up prompt, and make sure all
completed sase-repo work is declared (host-committed) before each handoff.

### 1. Canary credential absence suite (sase repo)

Per the epic plan's `acceptance` phase, first bullet: drive the full local flow with a
fake-sudo fixture fed a canary credential from the test TTY and assert the canary — and
its length and sha256 — appears nowhere: bundle files, response/journal/notification
stores, proc sidecars and logs, telemetry, argv (`/proc/<pid>/cmdline` during the run),
and environment. Port the research report's acceptance list A (minus root-runner cases).
Include:

- Tamper tests: manifest byte flip → runner refusal; command file swap →
  `hash_mismatch`.
- `tty_required` refusal on every headless entrypoint (durable answer proc, mobile
  bridge, fleet resolve, detached paths).
- The no-standing-privilege check (`/usr/bin/sudo -n true` fails immediately after a
  batch settles on a password host), fixture-based.

Reuse the existing fixtures and conventions in `tests/test_sudo_gate.py`,
`tests/test_sudo_runner.py`, and `tests/test_sudo_ssh.py`; do not build a parallel
harness. While the flag still exists, enable it the way the existing sudo tests do.

### 2. Docs and polish (sase repo)

- Write `docs/sudo.md` (flat file beside the other docs pages): agent contract, review
  UX, remote flow, policy rationale, and recovery (reopening a pending gate,
  resume-at-N, the lockout escape hatch). Link it from the sudo CLI epilogs.
- Final polish pass over help text, toasts, badges, and the 🔐 panel against the CLI
  rules memory and the research report's UX section. Keep the zero-cost-when-unused
  acceptance gate intact (no new startup/refresh/keystroke work with no sudo gate
  pending).

### 3. Verify and declare the code batch

Run the full verification flow from `lint_and_test.md` (at minimum `just check` green)
and let the host finalizer commit this batch before any gate handoff.

### 4. Phase 7 root-side: ptrace_scope=1 via the shipped sudo flow

Stage a world-readable sysctl drop-in (content `kernel.yama.ptrace_scope = 1`) and raise
one `/sase_sudo` request with exact argv commands to (a) install it as
`/etc/sysctl.d/99-sase-ptrace-scope.conf` (mode 0644, root-owned) and (b) apply it
immediately (`sysctl` load/apply of that setting). This request **is** the epic's "Local
flow live" acceptance evidence: request → ACE review → Authenticate & run → real PAM
prompt → ledger → follow-up agent. Watch for the CWD preflight issue recorded as
PROPOSED FOLLOW-UP #1 on `sase-110.7`; if the runner refuses for a sudoers CWD reason,
record the failure on the bead rather than working around it with raw sudo.

After settlement, in the follow-up agent: verify `sysctl kernel.yama.ptrace_scope`
prints 1 and the drop-in exists, then record command-output evidence (not claims) on
`sase-110.7` via `sase bead note`, including the ledger reference.

### 5. Phase 7 chezmoi-side verification, then close sase-110.7

- Confirm the `chezmoi_headless_sudo_guards` tale has landed: its plan gate is answered
  and the guard commits are in the chezmoi repo (open it with `/sase_repo`). If it is
  still pending review, wait for it with `/sase_monitor` (never busy-wait, never answer
  Bryan's gate).
- Once landed, verify a headless chezmoi apply/update is green end-to-end with the guard
  skip notices (or cite that tale's own recorded verification evidence if it already
  proved exactly this), honoring the chezmoi project's apply-after-commit rule.
- Close `sase-110.7` with `sase bead close sase-110.7 --note "..."` recording: the
  sudoers drop-in evidence, the accepted root-side evidence in lieu of the rejected
  `visudo -cf` proof (per PROPOSED FOLLOW-UP #2 on the bead: live headless-sudo
  failures, the drop-in listing, and the ptrace ledger showing password-gated sudo
  working), ptrace verification, and the headless chezmoi result. Leave the two PROPOSED
  FOLLOW-UP notes untouched for the land agent to triage.

### 6. Phase 8 live remote proof (apollo)

From athena, raise a machine-targeted `/sase_sudo` request with `machine: apollo` and a
harmless exact-argv root command (for example `/usr/bin/id -u`). Bryan reviews in ACE,
chooses Authenticate & run, and types the real password into apollo's genuine PAM prompt
over `ssh -t`; the ledger returns and launches the follow-up agent. Record the ledger
evidence on `sase-110.8`. Because `sase-xe.16` enrollment is not live, do not attempt
the `%dispatch:apollo` remote-raised path; record a note on `sase-110.8` pointing at
`sase-xe.16` for that follow-up, per the epic plan (never a gate on it).

### 7. Remove the agent_sudo_requests flag

Per the flags memory removal rule, in one change: delete the Off branch (the
`require_sudo_requests_enabled` guard path in `src/sase/sudo/feature.py` and its call
sites become unconditional), remove the registry entry, convert the both-state tests to
unconditional-On coverage, and close flag bead `sase-111` in that same change (`--note`
naming the removal commit). `tools/check_feature_flags` and the full `lint_and_test.md`
flow must pass. This is the last sase-repo code change.

### 8. Close sase-110.8, then recover the land flow

- Close `sase-110.8` with an evidence note: canary suite landed and green, apollo ledger
  evidence, flag removed and `sase-111` closed, docs and polish landed.
- Run `sase bead work sase-110`. With every phase closed it reassigns and launches only
  the land agent, which performs land verification, triages the PROPOSED FOLLOW-UP notes
  into task beads, and closes epic `sase-110`. Do **not** close `sase-110` directly —
  the land agent owns that close.

## Verification

- `just check` green after each sase-repo code batch (steps 3 and 7), per
  `lint_and_test.md`.
- The canary suite passes and its absence assertions cover value, length, and sha256
  across bundles, journals, notification stores, proc sidecars, logs, telemetry, argv,
  and env.
- Live evidence recorded on the phase beads as command output: ptrace_scope=1 on athena,
  the local ledger, the apollo ledger with the real PAM prompt.
- `sase flag list` no longer shows `agent_sudo_requests`; `sase-111` is closed.
- `sase bead show sase-110` ends with all eight phases closed and the epic closed by its
  land agent.

## Non-goals

- Implementing or reviewing the `chezmoi_headless_sudo_guards` tale itself (it has its
  own plan and coder); this plan only consumes its landing.
- The `%dispatch:apollo` remote-raised proof (blocked on `sase-xe.16` enrollment).
- Deciding the fate of the PROPOSED FOLLOW-UP notes on `sase-110.7` (land agent's
  triage) and removing the stale shadowed `/usr/local/bin/{lazygit,luarocks}` copies
  (explicitly deferred by the COORDINATION note on `sase-110`).
