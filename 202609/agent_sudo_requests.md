---
tier: epic
title: Agent sudo requests with terminal-handoff authentication
goal: "An agent on any machine can request that Bryan authenticate an exact, reviewed
  batch of sudo commands; authentication happens only on a real TTY handed to sudo/PAM
  (never through SASE), execution flows through an unprivileged hash-verified runner
  locally or over ssh -t remotely, a follow-up agent receives a structured ledger, and
  athena's NOPASSWD sudo policy is tightened so the reviewed approval becomes a real
  privilege boundary.

  "
phases:
  - id: core-runner
    title: Sudo manifest contracts and the TTY-attached runner in sase-core
    depends_on: []
    size: large
    description:
      "core-runner: add the sudo manifest/ledger/risk-badge wire contracts to sase_core
      and an unprivileged sase_sudo_runner binary (sudo -v, per-command sudo -n, sudo -k
      on the controlling TTY) shipped as a wheel console script."
  - id: sudo-gate
    title: Typed sudo gate kind with the sase sudo front doors
    depends_on: []
    size: large
    description:
      "sudo-gate: register the sudo gate kind, add the sase sudo
      request/answer/list/show CLI group with a questions-style single-turn handoff,
      requires_tty option enforcement, risk badges, the batch ledger follow-up prompt,
      and the agent_sudo_requests beta flag."
  - id: core-pin
    title: Ratchet the core pin and dependency floor past the runner surface
    depends_on:
      - core-runner
    size: small
    description:
      "core-pin: ratchet sase-core-revision.txt and the sase-core-rs floor to the
      release carrying the sudo runner and bindings, keeping binding validators green."
  - id: ace-review
    title: ACE review modal and the Authenticate terminal handoff
    depends_on:
      - sudo-gate
      - core-pin
    size: large
    description:
      "ace-review: add the 🔐 sudo review modal, the synchronous Approve-and-run suspend
      flow that hands the terminal to the runner, SUDO/SUDOED statuses,
      single-active-handoff locking, and completion toasts."
  - id: skill-guard
    title: The /sase_sudo generated skill and the raw-sudo PreToolUse guard
    depends_on:
      - sudo-gate
    size: medium
    description:
      "skill-guard: author the generated /sase_sudo skill teaching the request contract,
      add the chezmoi-managed PreToolUse deny for raw sudo in agent Bash calls, and
      revise the 'SASE never uses sudo' user-facing copy."
  - id: remote-sudo
    title: Machine-targeted and remote-raised sudo over ssh -t
    depends_on:
      - ace-review
    size: large
    description:
      "remote-sudo: add ssh targets to machine records, machine-targeted requests
      executed over ssh -t with a sealed manifest, remote-raised requests answered via
      the ssh -t relay from local ACE, and deny-only behavior on every non-TTY surface."
  - id: athena-policy
    title: Chezmoi sudo guards and the athena policy tightening
    depends_on:
      - ace-review
      - skill-guard
    size: medium
    description:
      "athena-policy: guard the chezmoi run_onchange scripts against password-required
      sudo, then live-tighten athena by replacing the NOPASSWD:ALL sudoers rule and
      setting ptrace_scope=1 through the shipped /sase_sudo flow itself."
  - id: acceptance
    title: Canary absence proof, live remote proof, and flag removal
    depends_on:
      - remote-sudo
      - athena-policy
    size: large
    description:
      "acceptance: land the canary-credential absence suite, run the live apollo
      machine-targeted proof with a real password prompt, remove the agent_sudo_requests
      flag, and finish docs and polish."
proposed_by: bbugyi200.athena.0kl
create_time: 2026-09-14 11:33:11
status: wip
---

- **PROMPT:**
  [prompts/202609/agent_sudo_requests.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_sudo_requests.md)

# Plan: Agent sudo requests with terminal-handoff authentication

## Context

Agents have no TTY, so `sudo` can never prompt inside an agent turn. Today athena
compensates with `bryan ALL=(ALL:ALL) NOPASSWD:ALL` (`/etc/sudoers:21`), which means any
agent can silently run root commands; apollo requires a password, so agents there simply
cannot do privileged work. This epic builds the feature that makes both problems
disappear: an agent submits a structured batch of exact argv commands plus a reason, its
turn ends through a gate-shell handoff, Bryan reviews and authenticates on a real
terminal, and a successor agent receives a per-command ledger. Once that flow is live,
athena's blanket NOPASSWD is removed so the reviewed approval becomes a real privilege
boundary.

The consolidated research report is binding background reading for every phase: read
`research:202609/agent_sudo_requests/agent_sudo_requests.md` via `sase artifact read`
before starting. Its two central resolutions are adopted unchanged: authentication is a
**terminal handoff** to `/usr/bin/sudo`/PAM (SASE never renders a password field, never
holds the credential in any process, pipe, file, digest, or log), and execution is an
**unprivileged per-command Rust runner** (sudoers evaluates the real commands; no
root-owned binary is installed anywhere).

Verified ground truth on the current trees (sase `16ee9c2336`, athena, 2026-09-14):

1. **Gate kinds are a data-driven registry.** `_ADAPTERS` in
   `src/sase/notification_gates/adapters.py:349-497` registers eleven kinds; mobile and
   Telegram derive their action maps from the registry rather than per-surface
   allowlists (`src/sase/integrations/_mobile_notification_actions.py:23-27`,
   `docs/notifications.md`). Adding a kind means an adapter entry, a `kind_validation/`
   module plus a dispatch clause in `src/sase/notification_gates/validation.py:225-242`,
   a spec builder (model: `src/sase/user_question_actions.py`), an ACE modal branch in
   `src/sase/ace/tui/actions/agents/_notification_modal_flow.py:186-215`, and the
   hand-kept presentation tables (`notification_modal_constants.py`,
   `notification_modal_tags.py`, `notifications/priority.py`, `_toasts.py:347-350`,
   `gate_debug_modal.py:338-346`).
2. **The questions front door is the handoff precedent.**
   `src/sase/main/questions_command_handler.py` validates loudly while the agent still
   runs, writes a pending marker (`src/sase/agent/pending_handoff.py:5-19`), and
   SIGTERMs the runner group (`src/sase/main/utils.py:60-90`); the runner adopts the
   marker in `src/sase/axe/run_agent_exec.py:146-198` and creates the gate shell. The
   working handoff helpers are `maybe_handoff_shell_from_agent` /
   `will_handoff_shell_to_agent_runner` in `src/sase/shells/handoff.py:21-57`. Note:
   `src/sase/gate_shell/__init__.py:35-45` lazily re-exports
   `maybe_handoff_gate_from_agent` and two sibling names that do not exist in
   `gate_shell/handoff.py`, yet `src/sase/main/gate_handler.py:86-87` and two other call
   sites import them — verify and repair (or route around) this in `sudo-gate` rather
   than building on it.
3. **The bundle trust model already fits.** Gate bundles under
   `~/.sase/interaction_requests/<kind>/<id>/` are journal-first, hash-bound
   (`src/sase/notification_gates/service.py:94-160`), and executed through
   `run_owned_command` which re-hashes and execs `/proc/self/fd/N`
   (`src/sase/notification_gates/command_runner.py:34-121`). The executor's
   `partial_attempt` + `retry=resume|restart` journal semantics
   (`src/sase/notification_gates/executor.py:95-125`) map directly onto stop-on-failure
   batches and resume-at-command-N.
4. **Every existing answer path is headless.** ACE submits gate answers through a
   durable proc (`_notification_gate_execution.py:41-114`), remote gate answers execute
   through the gateway's notification bridge as a child process with pipes for stdio
   (`sase-core crates/sase_gateway/src/host_bridge.rs:628-650` → `run_owned_command`),
   and `run_owned_command` itself allocates no TTY. Sudo authentication therefore CANNOT
   ride any existing answer transport; the approve branch needs a new TTY-attached path,
   and every headless surface must degrade to deny/display.
5. **Remote attention plumbing is live and reusable.** Remote gates already surface
   locally (`src/sase/dispatch/attention_inbox.py`), carry their origin installation id
   in their stable identity (`sase-core crates/sase_core/src/fleet_attention.rs:88`),
   and settle with dedup and durable intent journals (`src/sase/dispatch/attention.py`,
   `attention_intent.py`). What is missing for sudo: `MachineRecord`
   (`src/sase/dispatch/models.py:90-115`) has **no SSH field** — reachability is an
   HTTPS gateway endpoint only — and there is no deny-only surface primitive.
6. **The runner has a packaging precedent.** `crates/sase_gateway` ships two thin
   `[[bin]]` shims re-exported through `crates/sase_core_py` as wheel console scripts
   (`sase_gateway`, `sase_federation_worker`), so every `uv tool install sase` machine
   gets them without PATH changes; the dispatch resolver already looks in the installed
   venv bin dir.
7. **athena's policy surface is small.** The only NOPASSWD rule is `/etc/sudoers:21`;
   `/etc/sudoers.d/` holds only the README; `kernel.yama.ptrace_scope=0`; the zsh config
   aliases `sudo` to `sudo -E` (any runner must call `/usr/bin/sudo` by absolute path).
   No cron entries or systemd user units use sudo. The only headless passwordless-sudo
   dependents are two chezmoi scripts:
   `home/.chezmoiscripts/run_onchange_install_lazygit.tmpl` (weekly, unconditional
   `sudo install`) and `run_onchange_install_luarocks.tmpl` (monthly, sudo only when
   lua5.1/luarocks are missing).
8. **Related but independent work.** Task `sase-10z` (ready) fixes the generic
   `secret: true` gate-input leaks; this design bypasses `option_inputs` entirely, so
   neither work item blocks the other. Epic `sase-xe.16.11` (in progress) is completing
   remote dispatch enrollment; this epic deliberately avoids depending on it (see
   Decisions). `sase memory read tailnet.md` documents the chezmoi-managed SSH aliases
   (`athena`, `apollo`, `mac`) every tailnet machine already has.

## Decisions this plan adopts

- **The abstraction is "request a privileged execution", never "ask for a password."**
  No schema, transport, journal, or UI in this epic ever carries a password, its length,
  its hash, or its attempt count. UI language says **Authenticate** (PAM may be
  OTP/U2F/NOPASSWD), and the only password prompt any surface shows is the genuine one
  sudo prints on a real TTY.
- **Terminal handoff, not password collection.** On approve, the reviewing terminal is
  handed to the runner, which lets sudo/PAM converse directly with the TTY. With
  `kernel.yama.ptrace_scope=0` and Python's unzeroizable strings, an in-SASE password
  field cannot be hardened; the handoff design has nothing to harden.
- **Unprivileged per-command runner.** `sase_sudo_runner` invokes `/usr/bin/sudo -v`,
  then `/usr/bin/sudo -n -u <run_as> -D <cwd> -- <argv…>` per command, then
  `/usr/bin/sudo -k`. sudoers evaluates the real commands; the bounded TTY-scoped
  timestamp window between `-v` and `-k` is the accepted cost. No standing privilege:
  never `timestamp_type=global`, no warm-credential reuse, no `sudo -K`.
- **`requires_tty` is the one deny-only mechanism.** Rather than per-surface allowlists
  (which `docs/notifications.md` forbids), the sudo approve branch is marked
  `requires_tty` and the shared executor refuses to run such an option without a
  controlling TTY. Every headless surface — ACE's durable answer proc, Telegram, mobile,
  the fleet notification bridge, detached procs — mechanically degrades to
  display-plus-deny, with the `sase sudo answer <id>` hint. Deny stays answerable
  everywhere because denying runs no privileged command.
- **Remote authentication rides `ssh -t`, never the fleet gateway.** The fleet
  protocol's execution path is pipe-fed and TTY-less by design, and relaying a live PAM
  conversation through it would put SASE back in the credential path. The controller
  suspends ACE and runs `ssh -t <target> …`; ssh's encrypted channel carries the PAM
  conversation directly between Bryan's terminal and the target's sudo. This also
  decouples the epic from the in-flight sase-xe.16.11 enrollment work: machine-targeted
  sudo needs only SSH reachability plus an installed sase, not a completed fleet
  enrollment.
- **Exact-command review is the security boundary.** Commands are argv arrays rendered
  verbatim, hash-bound from review to execution via the existing bundle model. Scripts
  living in agent-writable locations are snapshotted into the bundle or refused.
  Approval requires explicit activation even on NOPASSWD hosts, so the experience is
  identical fleet-wide and tightening athena changes nothing about the flow.
- **Version-one exclusions** (from the research, adopted): no interactive root programs
  (stdin is `/dev/null`; no `visudo`, `$EDITOR`, `-i` shells), no nested
  `sudo`/`su`/`pkexec`, no askpass helpers, shell strings only behind an explicit
  `"shell": true` with a ⚠ badge and the full program text shown.
- **`agent_sudo_requests` is epic scaffolding.** A `beta` flag created with
  `sase flag new` in `sudo-gate` keeps partially-landed surfaces dark; `acceptance`
  deletes the Off branch and closes the flag bead before the epic lands.
- **Auth failure is an authentication outcome, never a command failure.** Wrong password
  (after sudo's own `passwd_tries`), Ctrl-C, or a missing TTY leaves the gate pending
  and re-answerable with "nothing was run" (or the honest partial ledger); nothing is
  auto-retried.

## Contracts declared by this plan (build against these, not each other's WIP)

- **Request schema** (`sase sudo request`, JSON on stdin): `reason` (string, rendered as
  untrusted); `commands`: list of `{id, argv, why, timeout_seconds?, shell?: false}`;
  `run_as` (default `root`); `cwd` (default `/`); `machine` (optional enrolled-machine
  alias or SSH target; absent = local host); `env` (optional reviewed allowlist; `LD_*`,
  `PATH`, `SUDO_*` rejected); `stop_on_failure` (default true); `output_to_agent`
  (`none|tail|full`, default `tail`); `next` (`{prompt}` follow-up block, same shape as
  other gate-shell nexts).
- **Approved manifest** (bundle resource, produced at review settlement, consumed by the
  runner): the reviewed subset of commands in review order plus
  `{request_id, host, run_as, cwd, env, stop_on_failure, resume_from}` — hash-bound
  under the bundle's existing `hashes` envelope so the runner can refuse a manifest that
  differs from what was reviewed.
- **Ledger** (runner → settlement → follow-up prompt): `outcome`
  (`completed|auth_failed|cancelled|tty_unavailable|runner_error`), per command
  `{id, status: ran|failed|skipped, exit_code, duration_seconds, output_tail}` with
  output bounded per the reviewed `output_to_agent` policy and the existing
  `GATE_RESULT_MAX_CHARS` conventions. A sudo gate settles approved **only** with a
  runner-produced receipt; an approve answer without one is refused.
- **Runner invocation**: `sase_sudo_runner` runs unprivileged, attached to the
  controlling TTY, takes the manifest path (or fd), calls `/usr/bin/sudo` by absolute
  path with a clean environment, stdin `/dev/null` per command, process-group timeouts,
  streamed bounded output, and emits the ledger as JSON. Exit status distinguishes
  authentication failure from command failure. If `sudo -n` fails after a successful
  `-v` (e.g. `timestamp_timeout=0`), fall back to plain per-command `sudo` on the same
  TTY. Sets `PR_SET_DUMPABLE(0)` on Linux as cheap hygiene even though it never holds a
  credential.
- **Option metadata**: gate option branches gain an optional `requires_tty: true` flag.
  The shared executor (`executor.py`) refuses to execute such an option without a
  controlling TTY, returning a structured `tty_required` execution error that leaves the
  gate answerable; surfaces render such options disabled with the
  `sase sudo answer <id>` hint instead of offering a button that would fail.
- **`ssh_target`**: `MachineRecord` and the `dispatch.machines.<alias>` config schema
  gain an optional `ssh_target` field (default: the machine alias itself, which resolves
  through the chezmoi-managed `~/.ssh/tailnet.conf` convention; discovery may suggest
  the tailnet MagicDNS name). It is used only by the sudo ssh relay in this epic; the
  fleet gateway path is unchanged.

## Constraints and obligations for every phase

- Read `research:202609/agent_sudo_requests/agent_sudo_requests.md` via
  `sase artifact read` before starting.
- Read the lint/test memory before finishing; run `just check` after changes (via
  `/sase_monitor` when slow); `just check-full` through `/sase_monitor` only. Read the
  CLI rules memory for any phase adding or changing CLI surfaces (sorted subcommands,
  short aliases for every public long option, options never required, bare group
  delegates to `list`). Read the TUI perf memory before `ace-review` and `remote-sudo`.
  Read the flags memory before touching the `agent_sudo_requests` flag. Read the
  generated-skills memory before `skill-guard`.
- Open sase-core and chezmoi only with the `/sase_repo` skill. Two-repo changes keep
  `tools/validate_sase_core_rs` green. Never hand-edit sase-core versions or path-dep
  pins (release-plz owns them).
- **Credential hygiene is absolute.** No password, passphrase, or PAM exchange — nor any
  length, digest, timing, or attempt-count derivative — may appear in any bundle,
  journal, sidecar, proc log, notification, telemetry event, argv, environment, or test
  fixture. Tests assert absence, not redaction. The strings `sudo`/`SUDO` in logs are
  fine; credential material is not.
- Always invoke `/usr/bin/sudo` by absolute path (the interactive shell aliases `sudo`
  to `sudo -E`); never set `SUDO_ASKPASS`; never pass `-E`.
- The zero-pending state stays free: no new startup, refresh, or keystroke work in ACE
  when no sudo gate is pending (the TUI perf memory's demand rules bind).
- Phase workers never create beads: record `PROPOSED FOLLOW-UP:` notes on your own phase
  bead. Do not edit files under `sase/memory/`; feature documentation for memory routes
  through a `memory` task bead filed by the land agent (the remote-dispatch precedent).
- Respect the single-turn and gates-never-block decisions: `sase sudo request` always
  ends the calling agent's turn through the shell handoff; nothing ever waits on a human
  in-process.

## Phase details

### Phase `core-runner` (sase-core)

- Wire contracts in `crates/sase_core`: `SudoManifestWire`, `SudoLedgerWire`, and risk
  badge derivation (`shell`, `network`, `package-manager`, `system-path-write`,
  `service-restart` — flag sshd/tailscaled restarts as lock-out-prone when the manifest
  host is remote) as pure functions over the manifest, so every surface renders
  identical badges. Property tests for canonicalization and hash stability.
- The runner: a new thin `[[bin]] sase_sudo_runner` in `crates/sase_gateway` following
  the `sase_federation_worker` model (logic in `lib.rs`, shim
  `src/sudo_runner_main.rs`), implementing the invocation contract above. Use `libc`
  (already a workspace dep) for `isatty`, process groups, and `PR_SET_DUMPABLE`.
  Behavior tests run against a fake `sudo` fixture binary (argv recording, prompt
  simulation, `-n` failure injection) — never real sudo in CI.
- PyO3/wheel plumbing in `crates/sase_core_py`: `py_sudo_runner_main` re-export, a
  `sase_sudo_runner` console script beside the gateway/worker ones, bindings for
  manifest validation/hashing and badge derivation, and an installed-wheel smoke test
  that the script resolves and answers `--help`.
- Fleet capability hardening (small, rides along since this is the core phase): where
  the gateway computes per-request attention capabilities
  (`crates/sase_gateway/src/routes.rs` around `evaluate_attention_precondition`),
  exclude the approve capability for sudo-kind pending actions so a remote approve
  refuses as `capability_missing` even if a client misbehaves; deny/read capabilities
  unchanged.
- Verification: `scripts/check.sh` plus the wheel smoke, per the repo's CI shape.

### Phase `sudo-gate` (sase)

- First step: create the flag with
  `sase flag new agent_sudo_requests -k beta --when-enabled "..." --when-disabled "..." --remove-when "..."`
  (read the flags memory first). Gate the new CLI front doors and notification surfaces
  behind it; both flag states tested.
- Register the `sudo` gate kind: adapter entry in
  `src/sase/notification_gates/adapters.py` (`action="SudoRequest"`, sender `sudo`,
  `branch_actionable=True`, bespoke — not `generic_form`), a `kind_validation/sudo.py`
  module plus its dispatch clause in `validation.py`, spec builder + command entrypoints
  modeled on `src/sase/user_question_actions.py`, the `_KIND_NEXT_ACTIONS` hook in
  `src/sase/gate_shell/kind_next_action.py`, and every hand-kept presentation table
  listed in Context item 1 (badge `SUDO`, icon 🔐, priority alongside questions).
  Validation enforces the v1 exclusions (argv arrays, no nested privilege tools,
  `shell: true` only with full program text, agent-writable scripts snapshotted into the
  bundle or refused) and computes badges through the core binding — behind an injected
  seam until `core-pin` lands, with a requires-real-binding integration test.
- `requires_tty` option metadata per the declared contract: model field, executor
  enforcement in `src/sase/notification_gates/executor.py` (structured `tty_required`
  refusal that leaves the gate answerable), and CLI rendering in `cli_show.py` /
  `cli_answer.py` (disabled option with the hint). This single change makes Telegram,
  mobile, the fleet bridge, and detached ACE procs deny-only for sudo with no
  per-surface code.
- `sase sudo` CLI group (bare group delegates to `list` via the central
  `_default_list_subcommands` convention; subcommands sorted; short aliases):
  - `request` — JSON on stdin, loud validation while the agent still runs, bundle + gate
    shell creation with `pending_status: SUDO`, `settled_status: SUDOED`, then the
    questions-style handoff via `maybe_handoff_shell_from_agent` /
    `kill_agent_runner_group`. Notification copy:
    `🔐 <agent> requests root on <host> (N commands)`.
  - `answer` — the one TTY execution entrypoint used by terminal users, ACE's suspend
    flow, and the ssh relay: renders the review (commands verbatim, verified facts,
    untrusted reason, badges), collects the approve/deny/select decision when not
    pre-supplied by flags, then for approve materializes the hash-bound manifest, hands
    the TTY to `sase_sudo_runner` (resolved from the installed venv bin like the
    federation worker), ingests the ledger, and settles through the executor journal so
    `partial_attempt`/resume-at-command-N works. Approve without a runner receipt is
    refused; deny settles with optional feedback anywhere.
  - `list` / `show` — consistent with `sase gate list/show`, plus badges and ledger.
- Follow-up prompt: register the sudo hook so the successor receives the fenced,
  untrusted ledger block (outcome line, per-command table, bounded tails per
  `output_to_agent`) via `compose_gate_followup_prompt` conventions; the authentication
  outcome renders distinctly from command failure.
- While wiring the front door, verify the `gate_shell/__init__.py` lazy re-export issue
  from Context item 2; repair it (or migrate the three call sites to
  `sase.shells.handoff`) with a regression test rather than copying the broken names.
- Tests: front-door validation loudness, handoff marker adoption, requires_tty refusal
  on every headless path (`gate answer` proc, mobile bridge entrypoint), resume-at-N
  after a mid-batch failure, ledger prompt bounds, both flag states.

### Phase `core-pin` (sase)

- Ratchet `sase-core-revision.txt` with `just ratchet-core-revision` once sase-core's
  remote HEAD contains `core-runner`; bump the `pyproject.toml` `sase-core-rs` floor to
  the published release carrying the runner surface. The floor bump requires the PyPI
  wheel to exist: monitor the release with `/sase_monitor`; if the release-plz PR needs
  a human merge, ask via `/sase_questions`; if still unpublished after a reasonable
  wait, land the SHA ratchet alone and record the floor bump as a `PROPOSED FOLLOW-UP:`.
- Flip `sudo-gate`'s seam-marked integration tests to the real bindings; keep
  `tools/check_sase_core_rs_bindings` and `tools/validate_sase_core_rs` green.

### Phase `ace-review` (sase)

- Review modal (bespoke `SudoRequestModal`, dispatched from
  `_notification_modal_flow.py`): checklist of commands (all selected, Space toggles),
  verified facts (host, run-as, cwd, env policy, requesting agent/project, expiry), the
  agent's reason visibly labeled untrusted, risk badges, `v` for full argv/env/snapshot
  detail, `d` for deny with feedback. Primary action: **Authenticate & run** — explicit
  activation always, including NOPASSWD hosts.
- The synchronous handoff (this is deliberately NOT `submit_gate_execution_task`, which
  is a headless durable proc): modal confirm → `with self.app.suspend():` (catch
  `SuspendNotSupported` like `project_management_actions.py:199-214`) → print a
  host-authored banner
  (`SASE sudo request <id> on <host> — N commands as <run_as> (Ctrl-C cancels)`) → run
  `sase sudo answer <id> --run …` attached to the terminal → restore ACE with a
  completion toast (`N/M root commands succeeded on <host>`) and a refreshed 🔐 panel.
  Wrong password after sudo's retries, Ctrl-C, or timeout restores ACE with the gate
  still pending and an honest toast; never auto-retry.
- Statuses: the shell block's `SUDO`/`SUDOED` plus branch statuses (mirroring
  `question_shell/create.py`), phase-label wiring in `_agent_display_content.py`,
  `_agent_gate_section.py`, and `_running_listing_common.py` so the agents list and
  Focus header read SUDO while pending.
- Single active authentication handoff at a time (lock under the sase home; a second
  approve attempt gets a clear toast naming the active one).
- Guardrail from the research: custom gates whose text mentions a sudo/root password get
  a standing warning banner ("Never enter your system password here") in the custom gate
  modal.
- Perf obligations: modal data loads off-pump (`asyncio.to_thread`, matching
  `_notification_custom_gate.py`), zero-pending state adds no refresh work, and the
  suspend flow leaves no stalled pump callbacks (stall watchdog clean in a manual
  check). PNG snapshot coverage for the modal and 🔐 panel per the visual suite
  conventions.

### Phase `skill-guard` (sase + chezmoi)

- Author `src/sase/xprompts/skills/sase_sudo.md` (generated-skills flow: preview with
  `sase skill init --diff`, commit first, deploy from the landed tree). The skill
  teaches: never ask for a password anywhere; never run raw `sudo`; do all non-root prep
  first and batch every root step into one request with a one-line `why` per command;
  prefer non-root alternatives and say why root is unavoidable; set
  `output_to_agent: "none"` for secret-adjacent commands; `machine:` for remote targets;
  the turn ends at `sase sudo request` and the successor gets the ledger.
- PreToolUse guard (chezmoi repo, via `/sase_repo`): a `Bash`-matcher hook in the
  chezmoi-managed Claude settings (`home/dot_claude/settings.json` plus a small hook
  script) that denies commands whose simple-command position resolves to
  `sudo`/`doas`/`pkexec` and surfaces `/sase_sudo` in the deny message. Guidance, not a
  boundary (Claude-only, per the provider-hooks research); ship it in the same phase as
  the skill so athena agents never lose the capability without the replacement being
  present. Keep `sase sudo …` itself unmatched.
- Revise the "SASE never uses sudo" user-facing copy
  (`src/sase/agent_clis/operations.py:345`, `src/sase/main/parser_agent_cli.py:42`) to
  say SASE never runs sudo without a reviewed request.

### Phase `remote-sudo` (sase)

- `ssh_target` on `MachineRecord.from_config`/`to_config`, the `dispatch.machines`
  schema, `sase machine add/init` plumbing, and `sase machine show/list` rendering, per
  the declared contract. Discovery may suggest the MagicDNS name; nothing else consumes
  the field.
- Machine-targeted requests (`machine` set on a request raised locally): review happens
  locally as usual; on Authenticate & run, the suspend flow runs
  `ssh -t <ssh_target> sase sudo exec -` with the sealed manifest on stdin; the
  target-side internal `exec` subcommand re-verifies the manifest hash, refuses without
  a TTY, runs the installed runner attached to the ssh TTY (PAM converses straight
  through ssh's channel), and returns the ledger on stdout for local settlement. Target
  resolution order: enrolled machine's `ssh_target`, else the request's `machine` value
  as an ssh destination with an explicit "unenrolled ssh target" badge at review time.
  Clean, honest failures for unreachable hosts, missing sase on the target, and version
  skew (probe `sase sudo exec --contract` first).
- Remote-raised requests (a dispatched agent runs `sase sudo request` on its own
  machine): the bundle lives on the origin host and already surfaces locally through the
  attention inventory with its origin identity. Add the sudo rendering to the remote
  attention flow: Deny rides the existing `sase machine attention approve/answer`
  transport (headless-safe); **Authenticate over SSH** suspends and runs
  `ssh -t <origin ssh_target> sase sudo answer <id> --run`, then re-polls the inventory
  so settlement, dedup, and the follow-up launch on the origin host all flow through
  existing plumbing. The `requires_tty` enforcement from `sudo-gate` plus the
  `core-runner` capability exclusion guarantee no fleet path can execute the approve
  branch.
- Telegram/mobile posture verified by tests, not new per-surface code: sudo requests
  project as display-plus-deny with the `ssh -t`/`sase sudo answer` hint (Telegram
  plugin changes, if its projection needs the hint text, go through `/sase_repo`; keep
  them minimal).
- Fixture-first tests (offline fake ssh + fake target covering manifest transfer, hash
  refusal, tty refusal, ledger round-trip, unreachable host); the live proof is
  `acceptance`'s job.

### Phase `athena-policy` (chezmoi + live athena operations)

- Chezmoi guards first (via `/sase_repo`): add a `chez::sudo_available` helper to
  `lib/chezmoi_utils.sh` (true when `/usr/bin/sudo -n true` succeeds or a TTY is
  attached), and gate the privileged steps of `run_onchange_install_lazygit.tmpl` and
  `run_onchange_install_luarocks.tmpl` on it — skipping with a loud `chez::log` notice
  naming `/sase_sudo` as the agent-era path when passwordless sudo is gone and no TTY is
  present. Headless `chezmoi apply` must stay green end-to-end. Honor the chezmoi
  project's apply-after-commit rule.
- Then tighten athena, applying each root change through the shipped `/sase_sudo` flow
  itself (this is the local live acceptance): replace `/etc/sudoers:21` with a
  password-required rule via a `visudo -cf`-validated `/etc/sudoers.d/` drop-in (keep
  `mail_badpass`; no NOPASSWD remnant), and set `kernel.yama.ptrace_scope=1` via a
  persistent `sysctl.d` drop-in plus immediate apply. Keep a logged-in root escape hatch
  for the transition window (an open root shell in tmux while verifying) so a sudoers
  mistake cannot lock the machine out; verify with `/usr/bin/sudo -n -l` failing and an
  interactive `sudo -v` prompting before closing it.
- Post-tightening verification on athena: headless `chezmoi apply` green with the skip
  notices, an agent-raised sudo request end-to-end with a real password prompt,
  `sudo -n` failing for agents, and evidence (command output, not claims) on the phase
  bead.

### Phase `acceptance` (sase, live operations)

- **Canary absence suite**: drive the full local flow with a fake-sudo fixture fed a
  canary credential from the test TTY and assert the canary — and its length and sha256
  — appears nowhere: bundle files, response/journal/notification stores, proc sidecars
  and logs, telemetry, argv (`/proc/<pid>/cmdline` during the run), or environment. Port
  the research report A acceptance list (minus root-runner cases); include tamper tests
  (manifest byte flip → runner refusal; command file swap → `hash_mismatch`), the
  `tty_required` refusal on every headless entrypoint, and the no-standing-privilege
  check (`/usr/bin/sudo -n true` fails immediately after a batch settles on a password
  host).
- **Live remote proof** (record evidence on the phase bead): from athena, a
  machine-targeted request with `machine: apollo` — review in ACE, Authenticate & run,
  type the real password into apollo's genuine PAM prompt over `ssh -t`, ledger back,
  follow-up agent launched with it. Best-effort (never a gate): if sase-xe.16's apollo
  enrollment is live by then, also prove the remote-raised path with a
  `%dispatch:apollo` agent; otherwise record the fixture evidence and a
  `PROPOSED FOLLOW-UP:` pointing at the enrollment epic.
- Remove the `agent_sudo_requests` flag per the flags memory (delete the Off branch,
  make On unconditional, remove the registry entry, close the flag bead in the same
  change).
- Docs: `docs/sudo.md` (agent contract, review UX, remote flow, policy rationale,
  recovery: reopening a pending gate, resume-at-N, lockout escape hatch), linked from
  the CLI epilogs; note in the land handoff that feature memory documentation routes
  through a `memory` task bead.
- Final polish pass: help text, toasts, badges, and the 🔐 panel reviewed against the
  CLI rules memory and the research report's UX section.

## Acceptance gates

| Gate                            | Required evidence                                                                                                                                     |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Credential absence              | The canary suite proves absence (not redaction) across bundles, journals, sidecars, logs, telemetry, argv, and env — value, length, and digest.       |
| Exact-bytes boundary            | Manifest and command tamper tests fail closed; the runner refuses an unreviewed manifest; approve without a runner receipt is refused.                |
| Headless surfaces are deny-only | `tty_required` refusal verified on the durable answer proc, mobile bridge, fleet resolve, and detached paths; deny works from every surface.          |
| Local flow live                 | On athena post-tightening: request → review → Authenticate & run → real PAM prompt → ledger → follow-up agent, with `sudo -n` failing for agents.     |
| Remote flow live                | Machine-targeted apollo proof over `ssh -t` with the real password prompt and a returned ledger.                                                      |
| No standing privilege           | `sudo -k` after every batch; `sudo -n true` fails immediately after settlement on a password host; no timestamp reuse across batches.                 |
| Athena tightened safely         | `/etc/sudoers` NOPASSWD rule gone (visudo-validated), `ptrace_scope=1`, headless `chezmoi apply` green with guard notices, no lockout.                |
| Zero-cost when unused           | No new startup/refresh/keystroke work with no sudo gate pending; j/k p95 unchanged per the perf memory's benches.                                     |
| Gate semantics hold             | Deny, Ctrl-C, timeout, and auth failure leave the gate pending/answerable; resume-at-N re-runs nothing already `ran`; single active handoff enforced. |
| Flag lifecycle                  | `agent_sudo_requests` created with both-state tests and removed (Off branch deleted, bead closed) before the epic lands.                              |

## Non-goals and reopeners

- **Allowlist promotion** (approve the same normalized argv N times → offer a reviewed
  narrow sudoers rule) is deliberately out: no demand corpus exists yet
  (corpus-before-mechanism). Reopen when repeated identical approvals are actually
  observed in the ledger history.
- **Fleet-relayed password entry** (authenticating a remote request without ssh) is out
  permanently under this design; it would put SASE back in the credential path.
- **Generic secret-input substrate fixes** are `sase-10z`, not this epic; this design
  bypasses `option_inputs` entirely.
- **Interactive root programs and nested privilege tools** stay excluded until a
  concrete need appears; requests that want them are refused with the reason at
  validation time, while the agent is still running.
