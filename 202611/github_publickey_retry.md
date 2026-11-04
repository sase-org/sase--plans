---
tier: epic
title: Retry transient GitHub SSH publickey denials on git network operations
goal: 'A GitHub-side burst of `Permission denied (publickey)` rejections of valid
  SSH keys no longer aborts agent launches or background git jobs. Launch-blocking
  and background git network operations retry such denials within a bounded, deadline-aware
  budget. Genuine credential failures still surface promptly as permanent authentication
  errors that keep the original stderr: deploy-key refusals, a host with no local
  SSH credential at all, and a denial that outlasts the budget.

  '
decisions:
  launch_budget:
    ask: How long should a launch-blocking git operation keep retrying SSH publickey
      denials?
    choices:
      two_minutes: ~110s over 6 attempts (5/10/20/30/45s); a wrong key fails a launch
        ~2 min late
      five_minutes: ~5 min over 9 attempts (up to 60s apart); rides out longer clusters,
        slower real failures
      thirty_seconds: ~30s over 4 attempts (5/10/15s); fails fast, but 7-denial clusters
        still kill launches
    default: two_minutes
    why: 2026-11-04 clusters cleared within minutes; 6 tries at ~50% denial fail under
      2% of launches
    answer: two_minutes
  deploy_key_marker:
    ask: Also classify `denied to deploy key` refusals as permanent authentication
      (sase-167's classifier half)?
    default: true
    why: same marker table; the precedence test needs a defined verdict for deploy-key
      text
    answer: true
phases:
- id: core-verdict
  title: sase-core credential_unconfirmed verdict for SSH publickey denials
  depends_on: []
  size: medium
  description: 'core-verdict: in sase-core, classify SSH publickey denials on git
    and git_clone operations as a new non-retryable `credential_unconfirmed` verdict
    whose reason keeps the `authentication` category. Deploy-key refusals stay permanent,
    the retryability module becomes a facade, and table and parity tests cover the
    new verdict.'
- id: credential-retry-policy
  title: sase pin bump and shared credential-confirmation retry policy
  depends_on:
  - core-verdict
  size: medium
  description: 'credential-retry-policy: move sase''s sase-core CI pin past the classifier
    change and add the Python wire constant. Add a shared, deadline-aware credential-retry
    tracker with launch and background budgets, an env kill switch, a per-host exhaustion
    memo, and a conservative local-SSH-credential precheck that fails fast when no
    credential exists.'
- id: sdd-clone-adopt
  title: Adopt credential retries in the SDD store clone
  depends_on:
  - credential-retry-policy
  size: medium
  description: 'sdd-clone-adopt: give `clone_sdd_store_to_path` a credential-retry
    budget separate from its transient schedule, using the launch budget, never holding
    the clone admission permit while sleeping. Prove it with an SSH-shim fake GitHub
    remote that denies N times and then serves a real bare repo.'
- id: network-git-and-lease-adopt
  title: Adopt credential retries in SDD network git and workspace lease fetch
  depends_on:
  - credential-retry-policy
  size: medium
  description: 'network-git-and-lease-adopt: route publickey denials in `run_sdd_network_git_with_retries`
    (agent publication) and the operational-lease `git fetch` through the shared tracker,
    replacing the lease''s bespoke one-shot confirmation. Keep `is_credential_failure`,
    the ssh-agent hint, and the external mirror''s `auth_error` label intact.'
- id: docs-and-followups
  title: Document the auth-retry behavior and record follow-ups
  depends_on:
  - sdd-clone-adopt
  - network-git-and-lease-adopt
  size: small
  description: 'docs-and-followups: document the retry budgets, kill switch, and fast-fail
    cases in the service-host and SDD docs. Note the deploy-key marker on sase-167,
    record proposed follow-ups for network git sites that never consult the classifier,
    and run the shim end to end against both adopted paths.'
proposed_by: bbugyi200.athena.sase-1r5
proposed_with: claude/opus
parent_bead: sase-1r5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-04 06:54:51
status: wip
bead_id: sase-1r5.1
---

- **PROMPT:** [prompts/202611/github_publickey_retry.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/github_publickey_retry.md)
- **BEAD:** [sase-1r5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1r5/sase-1r5.1.md)

# Plan: Retry transient GitHub SSH publickey denials

## Context

From about 09:37 UTC on 2026-11-04, GitHub intermittently rejected valid, registered SSH
keys with `Permission denied (publickey)`. The same key flipped between success and
denial within minutes on two hosts with different keys and different routes
(`github.com:22` and `ssh.github.com:443`). Each rejection killed an agent launch in
workspace preparation:

```
run_agent_runner_launch._prepare_workspace_and_repos
  -> axe.runner_workspace_protection.prepare_launch_workspace_repos   (strict for workspace > 1)
  -> sdd.store.ensure_workspace_sdd_clone -> ... -> sdd._store_clone_remote.clone_sdd_store_to_path
  -> sdd._store_clone_common.handle_failed_sdd_clone  => SddMaterializationError (permanent)
```

`clone_sdd_store_to_path` already retries transient failures (4 attempts, 0.25s/1s/2s
backoff). But its `is_transient_remote_clone_failure` delegates to sase-core's
`classify_failure_retryability`, and that function puts `permission denied (publickey)`
in `AUTH_MARKERS`. It returns
`verdict="permanent", reason="authentication: authentication or authorization failed"`,
so the first denial is fatal. The AXE `agent_publication` and `external_issue_mirror[*]`
jobs failed the same way.

### Design: a new non-retryable verdict, not a changed `permanent`

The classifier cannot tell a revoked key from a GitHub-side auth outage from a single
observation. So the classifier reports exactly that ambiguity, and Python call sites
that can afford to wait opt in to a bounded confirmation budget:

- New verdict `credential_unconfirmed` (wire constant
  `RETRYABILITY_VERDICT_CREDENTIAL_UNCONFIRMED`). It comes with `retryable: false`,
  `retry_after_seconds: null`, and a reason that keeps the stable category prefix
  `authentication:` (for example
  `authentication: ssh publickey denied; GitHub may be rejecting a valid key transiently`).
- **Why `retryable: false`:** every generic consumer that only reads `.retryable`
  (`github_cli`, `_git_runner` telemetry, `is_retryable_git_clone_failure`, and any
  plugin) keeps today's fail-fast behavior. Only call sites that explicitly check the
  new verdict retry, and they bring their own budget. Auth failures stay terminal unless
  a call site deliberately opts in.
- **Why keep the `authentication` category:**
  `workspace_provider._lease_model.is_credential_verdict` splits `reason` on `:` and
  compares the category, so the ssh-agent remediation hint,
  `OperationalLeaseError.is_credential_failure`, and the external mirror's `auth_error`
  label all keep working when the budget runs out.
- **Wire compatibility:** the change is additive. The schema version stays 1, and
  Python's `retryability_verdict_from_dict` does not validate verdict values. Old sase
  with a new core sees a non-retryable authentication verdict, which matches today. New
  sase with an old core never sees the new verdict and so never retries, which also
  matches today. Neither direction rejects the wire, so this is not a breaking change
  under sase-core's minor-window rule.
- Rejected alternatives: making publickey `retryable_transient` or
  `retryable_after_delay` would make every `.retryable` consumer retry on its own short
  schedule. It would also break the `authentication` category, or overload it. Keeping
  `permanent` and adding a side-channel hint field would leave a verdict that says
  "permanent" on an observation that sase-1oe.4's 2026-10-30 note and the incident both
  showed passing on retry.

### Inventory of classifier consumers and adoption decisions

| Call site                                                                                                                                | Consults classifier                                      | Decision                                                                                       |
| ---------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `sdd/_store_clone_remote.py` `clone_sdd_store_to_path` (launch SDD clone)                                                                | via `is_retryable_git_clone_failure`                     | **Adopt**, launch budget (phase `sdd-clone-adopt`)                                             |
| `sdd/_git_network.py` `run_sdd_network_git_with_retries` (used by `agents_sync/git.py`: agent publication fetch/push in AXE)             | yes                                                      | **Adopt**, background budget (phase `network-git-and-lease-adopt`)                             |
| `workspace_provider/_lease_git.py` `_fetch_origin_with_retries` (operational lease; external issue mirror, `launch_condition_workspace`) | yes, with a bespoke one-shot 10s credential confirmation | **Adopt**: replace the bespoke logic with the shared tracker (same phase)                      |
| `workspace_provider/_lease_git.py` `_preparation_error`                                                                                  | yes, for the hint                                        | Unchanged; the verdict keeps the `authentication` category                                     |
| `external_mirror/issues.py` (`error.is_credential_failure` → `auth_error`)                                                               | indirectly                                               | Unchanged; add a regression test                                                               |
| `sdd/_git_runner.py` `_classify_retryability` (telemetry fields)                                                                         | yes                                                      | Unchanged; the new verdict shows up in `classifier_verdict` automatically                      |
| `github_cli.py` (`gh` retries)                                                                                                           | yes, `gh` kind                                           | Unchanged; `gh` uses token auth, and the new verdict is only emitted for `git` and `git_clone` |
| `service/ssh_agent.py` readiness probes, `bead/_sync_logs.py` log labels                                                                 | **no**, they use their own string matchers               | Must stay untouched, so they keep reporting real credential problems promptly                  |
| Bead sync worker, sidecar git, host finalizer push, AXE sidecar runner                                                                   | no                                                       | Out of scope; `docs-and-followups` records proposed follow-ups                                 |

## Phase core-verdict: sase-core classifier (sase-core repo)

Work in the linked sase-core checkout (`sase repo open sase-core`) and read its
`AGENTS.md` first. Do not edit any `CHANGELOG.md` or version field, because release-plz
owns them. Use a Conventional Commit subject such as `feat(retryability): ...`. This is
not a breaking change.

1. Make `crates/sase_core/src/retryability/mod.rs` a facade (`mod`/`pub use` only), as
   the sase-core conventions require. Move the classifier body into
   `retryability/classify.rs` and its unit tests into `retryability/tests.rs`. Keep
   every public path (`sase_core::retryability::*`) and the `lib.rs` re-exports
   unchanged. Do not add new root re-exports or `core_*` prelude aliases.
2. In `wire.rs`, add
   `pub const RETRYABILITY_VERDICT_CREDENTIAL_UNCONFIRMED: &str = "credential_unconfirmed";`
   and re-export it from the facade. Keep `RETRYABILITY_WIRE_SCHEMA_VERSION = 1`.
3. In the classifier, after the rate-limit check:
   - > [!decision] deploy_key_marker
     >
     > First, `denied to deploy key` returns `permanent("authentication", ...)`, before
     > any publickey handling. This is the classifier half of sase-167; leave its
     > `_sync_logs.py` and `ssh_agent.py` wording work alone.
   - > [!decision] deploy_key_marker = no
     >
     > Skip the marker. Deploy-key text stays `unclassified` permanent, and the
     > precedence rule is that `credential_unconfirmed` requires the absence of
     > `denied to deploy key`.
   - Then, for `OperationKind::Git | OperationKind::GitClone` only, the lowercase prefix
     `permission denied (publickey` returns `credential_unconfirmed` with
     `retryable: false`, `retry_after_seconds: None`, and a reason starting with
     `authentication: `. The prefix match also covers
     `(publickey,keyboard-interactive)`, the same way `service/ssh_agent.py` matches it.
     For `gh` and other kinds, publickey text still falls through to the existing
     `AUTH_MARKERS` permanent verdict, so drop `permission denied (publickey)` from
     `AUTH_MARKERS` only if the `gh` path keeps it permanent.
   - All other `AUTH_MARKERS` (`authentication failed`, `bad credentials`, HTTP 401,
     ...) stay `permanent` / `authentication`.
4. Tests:
   - Classifier table tests in `retryability/tests.rs`:
     - `git_clone` and `git` publickey give `credential_unconfirmed`, `!retryable`, and
       a reason category of `authentication`.
     - A `gh` publickey and an HTTPS `Authentication failed for 'https://...'` stay
       `permanent`.
     - A deploy-key refusal stays `permanent` with an `authentication` category, even
       when the same stderr also contains publickey text.
     - The publickey stderr quoted in the incident
       (`git@github.com: Permission denied (publickey).\nfatal: Could not read from remote repository.`)
       is `credential_unconfirmed`.
     - Transport markers and rate limits are unchanged.
   - Add rows to `tests/integration/retryability_parity.rs`, including a serialization
     assertion for the new verdict's Python dict shape.
5. Verify with `sase tool run check` in sase-core. The commit must land on sase-core
   master before the next phase can pin it.

## Phase credential-retry-policy: pin bump and shared tracker (sase repo)

1. Move `sase-core-revision.txt` to the sase-core commit that contains `core-verdict`.
   Follow the pin procedure in `docs/rust_backend.md` and
   `docs/contribute/rust-backend.md#the-ci-source-revision-pin`. Make sure the local
   `sase_core_rs` binding used by tests is rebuilt from that revision.
2. In `src/sase/core/retryability_wire.py`, add
   `RETRYABILITY_VERDICT_CREDENTIAL_UNCONFIRMED = "credential_unconfirmed"` to the
   constants and `__all__`. Add parity rows to `tests/test_core_retryability.py`.
3. In `src/sase/service/ssh_agent.py`, add a public, never-raising
   `local_ssh_credential_state(env, host) -> Literal["present", "absent", "unknown"]`.
   It reuses `is_live_ssh_agent_socket` and `_probe_ssh_agent` without changing
   `probe_git_remote_auth`, `ssh_agent_readiness_warnings`, or their wording:
   - `unknown` when `GIT_SSH_COMMAND` or `GIT_SSH` is set, or when `host` is None.
   - `unknown` when `ssh` or `ssh -G <host>` fails or times out (2s cap).
   - `present` when the agent (`SSH_AUTH_SOCK`, or `identityagent` from `ssh -G` unless
     it is `none`) is live and `ssh-add -l` reports identities, or when any
     `identityfile` path printed by `ssh -G <host>` exists after `~` expansion.
   - `absent` only when every check positively failed.
4. Add `src/sase/git_credential_retry.py`, the only place that owns credential-retry
   policy:
   - `CredentialRetryBudget(label, delays)` with:
     - > [!decision] launch_budget = two_minutes
       >
       > `LAUNCH_CREDENTIAL_RETRY_BUDGET` with delays `(5, 10, 20, 30, 45)`: 6 attempts
       > and ~110s of sleep.
     - > [!decision] launch_budget = five_minutes
       >
       > `LAUNCH_CREDENTIAL_RETRY_BUDGET` with delays `(5, 10, 20, 30, 45, 60, 60, 60)`:
       > 9 attempts and ~290s of sleep.
     - > [!decision] launch_budget = thirty_seconds
       >
       > `LAUNCH_CREDENTIAL_RETRY_BUDGET` with delays `(5, 10, 15)`: 4 attempts and ~30s
       > of sleep.
     - `BACKGROUND_CREDENTIAL_RETRY_BUDGET = (10,)`: one confirmation retry. This
       preserves the lease fetch's current behavior and stays cheap for AXE jobs that
       rerun on the next tick.
   - Env kill switch `SASE_GIT_CREDENTIAL_RETRY_MAX_SECONDS` (float seconds). It caps
     the total sleep of every budget, and `0` disables credential retries. Name it in
     the module docstring and, in phase `docs-and-followups`, in the docs.
   - A `CredentialRetry` tracker, constructed per logical operation with `budget`,
     `deadline` (monotonic or None), `env`, and injectable `sleep`, `monotonic`, and
     `credential_probe`. Its `retry_after(verdict, detail) -> bool` method:
     1. Returns False for any verdict other than `credential_unconfirmed`.
     2. Parses the SSH host from `detail` (`<user>@<host>: Permission denied`; None if
        absent).
     3. Returns False when the budget or env cap is exhausted, or when the next delay
        would not fit before `deadline`.
     4. Returns False on the first denial when
        `credential_probe(env, host) == "absent"`, recording the reason
        `no local SSH credential`.
     5. Returns False when the process-local exhaustion memo marks the host as exhausted
        within the last 10 minutes. A later success clears it, so consecutive sidecar
        clones in one launch don't each burn a full budget against a truly bad
        credential.
     6. Otherwise logs a warning that names the attempt, the delay, and the original
        stderr, sleeps, and returns True.

     Also expose `note_success()` (logs "transient SSH publickey denial cleared after N
     retries" and clears the memo), `attempts`, `first_detail`, and
     `failure_suffix() -> str`. The suffix is a one-line explanation (for example
     `GitHub refused SSH publickey authentication on 6 attempts over 110s; see docs/reference/agents/service-host.md`),
     which callers append after the original stderr and never use to replace it.

5. Unit tests in `tests/test_git_credential_retry.py`, using a fake clock, sleep, and
   probe, with no real sleeps:
   - Retries up to the budget, then stops.
   - Respects the deadline.
   - The env cap of 0 disables retries.
   - `absent` fails fast.
   - The exhaustion memo short-circuits and is cleared by success.
   - Non-publickey verdicts never retry.
   - The suffix wording.

   Extend `tests/service/test_service_effective_env.py` (or a sibling) for
   `local_ssh_credential_state` using a fake `ssh` and `ssh-add` on PATH.

## Phase sdd-clone-adopt: SDD store clone

1. In `src/sase/sdd/_store_clone_remote.py`, add a credential-retry path to
   `clone_sdd_store_to_path` that is separate from `_REMOTE_CLONE_RETRY_DELAYS`:
   - Classify the failure detail once with
     `classify_failure_retryability(RETRY_OPERATION_GIT_CLONE, ...)`. Keep
     `is_transient_remote_clone_failure` for the transient branch.
   - On `credential_unconfirmed`, call
     `CredentialRetry(LAUNCH_CREDENTIAL_RETRY_BUDGET, deadline=deadline, env=clone_env)`.
     When it returns True, remove the partial clone and run another attempt. Credential
     attempts don't consume transient attempts, and the reference-retry rule is
     unchanged.
   - Sleep only outside `remote_clone_admission`, so a waiting clone never holds a host
     clone permit.
   - When the tracker says stop, call `handle_failed_sdd_clone(..., transient=False)`.
     The message is `failed to clone SDD store <url> into <path>: <original stderr>`
     followed by the tracker's `failure_suffix()`, so the launch still records a
     permanent materialization failure with the original stderr. On success after
     credential retries, call `note_success()`.
   - Add credential-attempt fields (`credential_retry_attempt`,
     `credential_retry_budget`) to the per-attempt telemetry dict.
2. Tests in `tests/sdd_store/test_sidecar_clone_retry_credentials.py`, reusing
   `tests/sdd_store/_clone_retry_helpers.py`:
   - **SSH-shim fake GitHub remote.** Set `GIT_SSH_COMMAND` to a temp script with a
     counter file. For the first N invocations it prints
     `git@github.com: Permission denied (publickey).` to stderr and exits 255. After
     that it runs `git-upload-pack` against a local bare repo. Clone
     `git@github.com:org/repo.git` and assert:
     - With N=2, the clone succeeds and there are 3 shim invocations.
     - With N larger than the budget, the clone fails strictly with a non-transient
       `SddMaterializationError` whose message contains the original stderr and the
       suffix, within the (patched-small) budget.
     - With a deploy-key refusal from the shim, the clone fails after one invocation.
   - Patch the tracker's sleep (or set the env cap) so the tests run in well under a
     second.
   - A deadline shorter than the next delay stops early.
   - Transient-transport retry behavior is unchanged: existing tests in
     `test_sidecar_clone_retry_*.py` still pass untouched.

## Phase network-git-and-lease-adopt: SDD network git and workspace lease

1. In `src/sase/sdd/_git_network.py` `run_sdd_network_git_with_retries`:
   - Add
     `credential_budget: CredentialRetryBudget | None = BACKGROUND_CREDENTIAL_RETRY_BUDGET`
     (None disables it).
   - On a `credential_unconfirmed` verdict, consult one tracker per call (with the
     call's `deadline` and `env`) instead of `max_attempts`. Credential retries don't
     count against transient attempts.
   - A final failure returns or raises the last result unchanged, so callers keep the
     original stderr.
   - `agents_sync/git.py` keeps the default.
2. In `src/sase/workspace_provider/_lease_git.py`:
   - Delete `_CREDENTIAL_CONFIRMATION_DELAY_SECONDS` and the `credential_retry_used` /
     `first_credential_stderr` bookkeeping.
   - Route credential verdicts through a `CredentialRetry` whose budget is threaded from
     `acquire_operational_lease`. Scheduled AXE jobs (external issue mirror) keep the
     background budget. Callers that block an agent launch (check
     `agent/launch_condition_workspace.py`) get the launch budget.
   - Keep the "transient remote credential refusal cleared on retry" warning, through
     `note_success()`.
   - Keep `_preparation_error` and `_with_ssh_agent_hint` unchanged. Confirm that the
     exhausted `credential_unconfirmed` verdict still yields
     `is_credential_failure is True` and the hint.
3. Tests:
   - Extend `tests/workspace_provider/test_workspace_lease_prepare.py`. Its existing
     publickey tests, including `test_fetch_publickey_denial_then_success_retries_once`,
     must still pass, adjusted only where the budget is now explicit. Add a
     launch-budget case and an `absent`-credential fast-fail case.
   - Add `run_sdd_network_git_with_retries` cases in the existing network-git tests:
     deny then succeed, persistent denial returns the original stderr, and
     `gh`/transport paths are unchanged.
   - Add an external-mirror regression test: an exhausted denial still reports
     `degraded="auth_error"`.
   - Run `tests/service` and the `bead/_sync_logs` tests unchanged to show those
     matchers are untouched.

## Phase docs-and-followups

1. Read the `docs_writing` reference memory, then update
   `docs/reference/agents/service-host.md` (its publickey remediation section) and the
   relevant SDD doc section. Cover:
   - SASE retries GitHub SSH publickey denials within the launch and background budgets.
   - It fails fast with no local credential, on deploy-key refusals, and after an
     exhausted budget, always keeping git's original stderr.
   - The `SASE_GIT_CREDENTIAL_RETRY_MAX_SECONDS` kill switch.
2. Add a note to sase-167 recording which sase-core commit added the deploy-key marker
   (only under `deploy_key_marker = true`). Its `_sync_logs.py` and ssh-agent wording
   work stays with sase-167.
3. Record `PROPOSED FOLLOW-UP:` notes on this phase's bead (epic phases must not create
   beads) for network git sites that never consult the classifier. These include the
   bead sync worker, `sdd/_sidecar_git.py`, host finalizer pushes in
   `vcs_provider/plugins/_git_*`, and `axe/runner_workspace_sidecar.py`. For each one,
   state whether it already retries elsewhere and whether incident-style denials would
   abort user-visible work.
4. Final verification: run the SSH-shim tests from both adopted paths together, then run
   `just check` through `sase tool run` per the `lint_and_test` memory.

## Verification summary

- Run sase-core: `sase tool run check`.
- Run sase: `sase tool run check`, which covers the new tracker, SDD clone shim, lease,
  network-git, and external-mirror tests, plus the untouched `ssh_agent` and
  `_sync_logs` suites.
- Acceptance:
  - A shim that denies twice and then serves the repo lets a strict launch-time SDD
    clone succeed.
  - A persistent denial fails as a permanent `SddMaterializationError` within the
    budget, with git's original stderr first.
  - A deploy-key refusal, or a host with no local SSH credential, fails on the first
    attempt.
