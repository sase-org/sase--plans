---
tier: tale
title: Retry transient git fetch failures during operational lease preparation
goal:
  A short GitHub-side SSH refusal or transport blip during operational lease preparation
  no longer fails the leased operation outright (for example, archiving an auto-approved
  plan). The lease fetch retries transient transport failures, re-checks one `Permission
  denied (publickey)` after a short delay, and can no longer hang forever.
size: small
proposed_by: bbugyi200.athena.0py
create_time: 2026-09-23 10:29:23
status: wip
---

# Plan: Retry transient git fetch failures in operational lease preparation

## Diagnosis (context only; no host or config change needed)

The user reported that `git pull` fails on the `apollo` machine "for any repo". A
host-by-host investigation found that apollo's configuration is healthy. The failure was
a GitHub-side SSH authentication blip that lasted a few minutes.

Evidence gathered:

- **Apollo credential is healthy.**
  - `~/.ssh/id_rsa` has no passphrase and authenticates as `bbugyi200` whether SSH uses
    the tmux-shell agent, the systemd/gpg-agent socket that `sase.service` inherits, or
    no agent at all.
  - The agent is empty, so no deploy key can be offered first.
  - `sase service status` shows no SSH credential warning.
- **Apollo git works now.**
  - 12 of 12 `ssh -T git@github.com` probes succeed in about 0.4 s each.
  - `git ls-remote` and `git pull` succeed for sase, sase-core, sase-github,
    sase-telegram, bob-cli, and chezmoi, including from an interactive TTY with the
    user's tmux environment.
- **The failure was one short window.**
  - Apollo's `~/.sase/logs/tui_git_ops.jsonl` shows about 1050 successful network git
    operations since 2026-09-19 and exactly one `Permission denied (publickey)`, at
    2026-09-23 14:01:20 UTC.
  - At 14:01:36 UTC agent `1k`'s auto-approved plan failed. `archive_approved_plan` →
    `operational_workspace_lease` → `prepare_from_primary_remote` did a single
    `git fetch` that GitHub refused. The agent was marked failed and its workspace was
    held.
  - The user's interactive `git pull` hung at 14:01:47 until Ctrl-C. Their `git push` at
    14:02:02 succeeded.
- **The cause is on GitHub's side.**
  - During the same minutes, athena got
    `git@ssh.github.com: Permission denied (publickey)` at 14:01:49, 14:01:59, 14:02:37,
    and 14:04:07 UTC, with successes in between. Athena uses a different key
    (`id_sase_service` with `IdentitiesOnly`) over a different route
    (`ssh.github.com:443`).
  - Two machines, two keys, and two routes flapped at once.
  - githubstatus.com has an open "Incident across several services" for 2026-09-23
    (database replicas detached). It may be related; this is not confirmed.
- **Not the same as athena's earlier outage.**
  - The athena problem fixed on 2026-09-22 was a dead captured SSH agent plus
    passphrase-protected keys. Plan `service_host_ssh_credential.md` fixed it with
    `id_sase_service`.
  - The lease error text points at that cause ("usually a missing or empty SSH agent")
    in every case, which made this blip look like a repeat.

Side effects of the diagnosis (for the record; the implementer has nothing to do here).
The reproduction attempts ran real `git pull`s on apollo, and all were plain
fast-forwards:

- the primary sase checkout: `24d5e20af` → `b3e31bb59`;
- `bob-cli`;
- the chezmoi source repo: `1a875285` → `7438df3e`. This was pulled only, not applied;
  apply it with `chezmoi apply` or `chez update`.

The real SASE defect is fragility. `src/sase/workspace_provider/_lease_git.py` runs the
lease-preparation `git fetch --quiet origin` exactly once, with no timeout and no retry.
It does not use the classifier-driven retries that SDD network git already has
(`run_sdd_network_git_with_retries` in `src/sase/sdd/_git.py`). So:

- a transient transport failure (connection reset, timeout, DNS) fails the whole leased
  operation;
- a GitHub-side publickey blip fails it just the same;
- a hung SSH handshake blocks the lease with no time limit.

## Changes

### 1. Bounded, classified retries for the lease fetch (`src/sase/workspace_provider/_lease_git.py`)

Keep `_run_git(args, checkout)` as the single-attempt execution seam, because existing
tests monkeypatch it. Add the retry policy around the `fetch` call in
`prepare_from_primary_remote` only. The later `checkout --force -B` is local, so it
stays single-shot.

- **Per-attempt timeout.**
  - Give `_run_git` an optional `timeout: float | None = None` that passes through to
    `subprocess.run`. Only the fetch sets it.
  - Use the same configured network timeout as SDD network git (`network_git_timeout()`
    in `sase.sdd._git`). If importing it creates a cycle, import it lazily inside the
    function.
  - Treat `subprocess.TimeoutExpired` as a retryable transport failure. If the final
    attempt also times out, raise
    `OperationalLeaseError("preparation", "git fetch timed out after <N>s")`, which
    carries no retryability verdict.
- **Transient transport failures.**
  - Classify each non-zero fetch with the existing
    `classify_failure_retryability(RETRY_OPERATION_GIT, ...)`. Do not duplicate the
    classifier's signatures.
  - The classifier already marks connection reset, connection timeout, and DNS failure
    as `retryable=True`. In that case, retry up to 3 total attempts, which matches
    `DEFAULT_NETWORK_GIT_MAX_ATTEMPTS`.
  - Sleep `verdict.retry_after_seconds` when it is set, otherwise the next value from
    `_FETCH_RETRY_DELAYS_SECONDS = (1.0, 5.0)`.
- **Credential refusals get exactly one confirmation retry.**
  - When the verdict is an authentication verdict (`is_credential_verdict(verdict)`),
    sleep `_CREDENTIAL_CONFIRMATION_DELAY_SECONDS = 10.0`, then fetch once more.
  - If that attempt succeeds, continue normally. Log one `warning` saying a transient
    remote credential refusal cleared on retry, and include the first attempt's stderr.
  - If it fails again, raise exactly as today. The existing `_with_ssh_agent_hint` text
    and message shape stay unchanged: a refusal that survives a delayed re-check is when
    the hint applies.
  - Only one confirmation retry, never more. The delay stays short because chop/routine
    callers such as toobig_split units and the external mirror call leases repeatedly
    while a credential is really broken.
- **Everything else.** Any non-retryable, non-credential failure (for example
  `couldn't find remote ref main`) raises immediately on the first attempt, as today.
- **Test seam.** Sleep through a module-level seam (`_sleep = time.sleep`) so tests can
  patch it without real delays.
- **Size.** Keep the loop small and local. Do not change the Rust classifier, and do not
  make authentication verdicts retryable globally; other callers must keep their current
  semantics.

### 2. Tests (`tests/workspace_provider/test_workspace_lease_prepare.py`)

- Update the `_fake_git` helper to also patch
  `sase.workspace_provider._lease_git._sleep` to a recorder, so no test sleeps for real.
  It must accept the new optional `timeout` keyword.
- Let the fake return a sequence of fetch results (per-attempt stderr, or success), so
  retries can be tested.
- Add cases:
  1. Publickey denial then success: no error, exactly 2 fetch attempts, one sleep of
     `10.0`, and one warning logged.
  2. Persistent publickey denial: raises. The message still equals the existing
     `test_publickey_denial_message_shape_is_unchanged` expectation. Exactly 2 fetch
     attempts. `is_credential_failure` is still true.
  3. Retryable transport failure (for example
     `kex_exchange_identification: read: Connection reset by peer`) then success: no
     error, 2 attempts, first sleep `1.0`.
  4. Persistent retryable transport failure: raises after exactly 3 attempts, with
     sleeps `1.0` then `5.0`. It is not marked as a credential failure.
  5. Non-retryable, non-credential failure (`fatal: couldn't find remote ref main`):
     raises after exactly 1 attempt with no sleep.
  6. Fetch `TimeoutExpired` on every attempt: raises a preparation error that mentions
     the timeout, after 3 attempts.
  7. A checkout-step publickey denial is still not retried: 1 checkout attempt, and the
     hint is still appended.
- Every existing test in the file must keep passing unchanged, apart from the helper
  update.

### 3. Docs (`docs/init.md`)

In the paragraph that starts "While the credential is refused, host chops that write
beads cannot fetch their operational workspace", add one or two sentences:

- Operational lease preparation retries a transient transport failure (connection reset,
  timeout, DNS) up to three attempts.
- It re-checks a `Permission denied (publickey)` once after about 10 seconds before
  failing. So a brief remote-side refusal, such as during a GitHub incident, no longer
  fails the leased operation.
- A refusal that survives the re-check is reported with the existing remediation.

Keep the rest of the section unchanged.

## Verification

- Read the `lint_and_test.md` reference memory, then run the project's standard
  verification recipe that it names. Do not run `check-full` unless explicitly told to.
- Run the focused test module `tests/workspace_provider/test_workspace_lease_prepare.py`
  and the other lease tests under `tests/workspace_provider/` that touch preparation,
  such as `test_workspace_lease_acquire.py`.

## Out of scope

- No change to any machine's SSH config, keys, agents, or `sase service` environment.
  Apollo and athena credentials were verified healthy.
- No change to the Rust retryability classifier or its verdicts.
- No change to SDD network git retries (`src/sase/sdd/_git.py`) or to other
  `classify_failure_retryability` callers.
- Do not relaunch or clean up the failed `1k` agent or its held workspace. That is the
  user's call.
