---
tier: tale
title: Gate shell creation, handoff, and settlement
goal:
  A gate whose request carries a shell block becomes a named non-LLM gate-shell member
  of its creator's family that owns the decision, kills its creator instead of blocking
  it, survives a bounded deadline enforced by a reclaim chop, and settles durably.
size: medium
proposed_by: bbugyi200.athena.sase-ud.3
bead: sase-ud.3
---

- **PARENT:** [202608/gate_shells.md](gate_shells.md)
- **BEAD:**
  [sase-ud.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ud/sase-ud.3.md)

# Gate shell creation, handoff, and settlement

## Goal

Complete phase `sase-ud.3`: add the additive `shell` block to the v3 gate request,
create the gate-shell family member with promotion and claim transfer, hand off and kill
the creator through `.sase_gate_pending`, run the ordered settlement, short-circuit
`%auto`, and bound pending gate shells with a required timeout plus a reclaim chop.

Python only. Read `sase/memory/cli_rules.md` and `sase/memory/symvision.md` before
touching the CLI or adding public symbols. Follow §2-§5, §7, §12 and R2 of the parent
epic plan; where this plan is more specific, this plan wins.

**Out of scope, and deliberately left to later phases:** Rust wire fields and the
`pending`-frees-the-runner-slot rule (`gate-core-rs`); `gate.log` streaming, the
`--detach` answer path, and `proc_id` during execution (`gate-exec`); every TUI surface,
glyph, lane, chip, and `is_monitor` filter-site decision (`gate-tui`); the composed
follow-up prompt and its output policy (`gate-followup`); `#fork` classification,
`sase gate list/cancel`, conformance, and the `/sase_gate` skill rewrite
(`gate-fork-cli`); every migration of an existing producer. This phase is verified
through `sase gate show`, artifact metadata, and tests — not through ACE.

## Implementation

1. **The `shell` block, additively within `schema_version: 3`.** Model it as a frozen
   `GateShellSpec` in a new `src/sase/notification_gates/model_shell.py`, beside
   `model_operations.py` and for the same documented reason: the block is new but the
   envelope stays v3, so `hashing.py`'s 2-or-3 check is untouched. Add `"shell"` to
   `GateSpec.from_mapping`'s `reject_unknown_fields` allowlist and a
   `shell: GateShellSpec | None` field to `GateSpec`; emit it from `_build_envelope`
   only when present, and include it in `_spec_fingerprint` so a replayed creation with
   a different shell block is a `request_id_conflict`. Keep the block out of
   `notification_gates`' dependency on any agent/axe machinery: this module models and
   validates, nothing else.

2. **Creation-time validation, in `GateShellSpec.from_mapping`,** given the already
   normalized branches and option ids. Every `branches` key is either a compiled branch
   (that branch's option ids `+`-joined in canonical query order) or one of the reserved
   `timeout` / `stopped` / `failed`; `pending_status` / `settled_status` / per-branch
   `status` clamp to 20 characters through the substrate's status clamp; `accent` and
   per-branch `accent` are `#RRGGBB`; `workspace` is `inherit | release`; `next.fork` is
   `family | shell | none`; `next.output` is one of `none | results | tail | file` or a
   list of them; `suffix`, when given, is a `--`-prefixed family suffix. A typo raises
   `GateError` at `sase gate create`, never at settle time. Defaults: `suffix`
   allocated, `pending_status` `GATE`, `settled_status` `GATED`, accent hashed,
   `workspace` `inherit`, `next.prompt` `null`, `next.output` `["results"]`, `next.fork`
   `family`, `next.model` inherited.

3. **A required-or-defaulted deadline (R2, not deferrable).** When a request carries a
   `shell` block and omits `gate_timeout_seconds`, `GateSpec.from_mapping` defaults it
   to 24 hours, so the envelope, the fingerprint, and every consumer see a real
   deadline. A non-shell gate's timeout semantics are unchanged.

4. **Family suffix taxonomy in `plan_chain.py`.** Add `PLAN_CHAIN_GATE_SUFFIX`
   (`--gate`), register it in `_KNOWN_SUFFIXES` and `_PHASE_SUFFIX_ROLES` (role `gate`),
   add `gate` to `_EXPLICIT_FAMILY_ROLES`, and add a `_GATE_SEQUENCE_SUFFIX_RE` beside
   `_MONITOR_SEQUENCE_SUFFIX_RE` wired into **both** `canonical_plan_chain_suffix`'s
   helper and `_parse_plan_chain_suffix`. Without it `--gate-0` falls through to the
   phase-question split and the row silently drops out of the roster; cover that with a
   direct regression test.

5. **A new `sase.gate_shell` package** built on `sase.shells`, holding everything that
   needs agent/axe machinery so `notification_gates` never imports it eagerly: state
   buckets and role predicates (`MONITOR_STATE_BUCKETS` plus a leading
   `"pending": "Stopped"`, via `ShellStateConfig`); the `GATE`/`GATED` status pair via
   the substrate's status helpers; suffix allocation (`--gate`, then `--gate-0`, ...)
   via `allocate_shell_suffix`; the member creator via `create_family_shell_member`; the
   `ace-gate` claim-workflow label in its own dependency-free leaf, mirroring
   `sase.monitor.claims`; the handoff marker helpers; an index-backed store; the ordered
   creation transaction; settlement; and the reclaim pass. Do not add a monitor/gate
   branch to any substrate function — parameterize it.

6. **Member metadata.** `shell_kind: "gate"`, `agent_family_role: "gate"`, `proc_id` and
   `pid` `None` while pending, plus flat `gate_*` fields mirroring the generic half of
   the `monitor_*` block that `gate-core-rs` will put on the wire: id, kind, state,
   status pair, accent, label, reason, creator agent, bundle and notification identity,
   timeout, request fingerprint, workspace policy, and the follow-up slots
   (agent/outcome/error/degraded-reason/prompt-path/next action/output/model) that
   `gate-followup` fills. The branch policy itself is **not** duplicated into metadata:
   the hash-verified envelope's `shell` block is its single source of truth, and
   settlement reads it from there.

7. **The ordered creation transaction**, `create_gate_shell(request)`, implementing §4's
   rule that the creator is never killed until another durable owner has acknowledged
   the gate and no actionable notification is published without one:
   1. Validate the whole request, including the `shell` block, before creating anything.
   2. Resolve the creator from `SASE_ARTIFACTS_DIR` (falling back to `SASE_AGENT_NAME`),
      promote a bare creator through `promote_agent_to_family` so `acme` becomes
      `acme--0` plus container `acme`, allocate the suffix, and create the gate-shell
      member in `pending`. Reuse an existing member carrying the same `gate_id` instead
      of creating a second one, so a replayed creation stays idempotent.
   3. Move the creator's workspace claim to the gate shell: transfer it to the
      `ace-gate` workflow under the member's own artifacts timestamp for
      `workspace: inherit`, or release it for `workspace: release`. A creator with no
      claimable workspace (`workspace_num` unset or `0`) skips the move.
   4. Call the existing `create_gate`, which materializes the bundle and publishes its
      notification atomically under its own journal and compensation.
   5. On any failure in 3 or 4, tear the member down as `failed`/settled and give the
      creator its claim back exactly as it held it — never into the free pool — then
      re-raise.
   6. Record the notification id and bundle path on the member.

   Failure after publication is compensated the other way: if the pending marker cannot
   be written, **cancel the gate**, settle the shell `stopped`, and restore the claim,
   rather than let the creator and the gate shell own one lane concurrently.

8. **Handoff and the killed-runner adoption.** Add `GATE_PENDING_MARKER`
   (`.sase_gate_pending`) to `agent/pending_handoff.py` and `PENDING_HANDOFF_MARKERS`;
   confirm it lands in `run_agent_runner_signals._NON_MONITOR_HANDOFF_MARKERS`, whose
   early return is exactly right once the claim has already moved. Add
   `will_handoff_gate_to_agent_runner()` carrying the same `NoReturn`-ordering docstring
   `will_handoff_monitor_to_agent_runner` already has, and
   `maybe_handoff_gate_from_agent()` over the substrate's handoff helper. In
   `run_agent_exec._handle_killed_iteration`, read-and-delete the gate marker beside the
   existing four (including in the `has_user_kill_intent` branch) and dispatch to a new
   `handle_gate_marker`, modelled on `run_agent_exec_monitor.py`: normalize and finalize
   the creator's handoff artifacts, adopt the promoted suffix, save the creator's
   transcript with a gate-handoff response naming the gate id, shell name, decision
   title and `sase gate show` pointer, and terminalize the creator as `DONE` with
   outcome `"gated"`.

9. **`sase gate create --shell` and the descriptor-before-handoff rule.** Add the §5
   flag surface to `parser_gate.py`, alphabetically ordered with short aliases per
   `cli_rules.md`: `-n/--next`, `-f/--next-fork`, `-m/--next-model`, `-N/--next-output`,
   `-G/--shell`, `-g/--shell-status`, `-E/--shell-stop-status`. Everything else in the
   block stays expressible in JSON so `/sase_gate` remains declarative; CLI values merge
   into the request's `shell` block the way the existing presentation flags already
   merge. In `gate_handler._handle_gate_create`, route to `create_gate_shell` when the
   merged request carries a `shell` block, **print the creation descriptor first**, and
   only then hand off — `kill_agent_runner_group()` is `NoReturn`, so nothing after it
   and nothing conditioned on its return value ever runs.

10. **The `%auto` short-circuit (§7).** Always create the gate shell, for one uniform
    family shape and audit trail, but hand off only if the gate is still pending after
    `create_gate` returns. An auto-resolved gate settles its shell in place and the CLI
    exits normally without writing a marker or killing anything, so `%auto` still costs
    exactly one agent; the in-process successor continuation is `gate-followup`'s work.

11. **Settlement, one function, for every terminal state**, implementing §3's ordering
    rule with no `gate_settled` field — a terminal `gate_state` implies every required
    side effect is already durable: set `gate_state` to `settling`; retain what the
    bundle recorded; release or transfer the workspace claim and launch, degrade,
    suppress, or durably reject the follow-up through the substrate's
    `settle_shell_claim_and_followup` with a gate-specific `ShellSettlementConfig` (the
    launcher hook stays `None` in this phase and `gate-followup` fills it); write member
    metadata, the done marker, and a decision-record chat file (title, branches with the
    selection marked, reviewer note, per-option results — `gate-exec` later extends it
    with the output tail); finalize workflow state and pulse artifact watchers; then set
    the terminal `gate_state` with its termination reason. Assert the invariant directly
    in tests.

12. **Bounded pending shells (R2, required here).** Add a `sase_chop_gate_shell_reclaim`
    builtin chop, registered in `default_config.yml`'s hourly group beside
    `bead_stale_cleanup` and in `src/sase/config/sase.schema.json` with its
    `gate.shell.*` settings and typed accessors. Because a pending gate shell is
    processless, nothing else enforces its deadline. Each pass, over every enabled
    project's non-terminal gate-shell members: settle `lost` when the bundle is
    unreachable; reconcile a bundle that already carries a response or cancellation to
    the matching terminal state (a surface answered it without settling); cancel with
    reason `timeout` and settle `timeout` once the envelope's own deadline has passed;
    and force-settle `lost`, releasing the claim, once a shell has been pending past a
    configurable grace beyond that deadline. Emit a chop summary and never let one
    unreadable project stop the pass.

13. **Do not let a pending shell's claim be reaped or leak.** In
    `ace/scheduler/stale_running_cleanup.py`, add a `_gate_claim_is_releasable` guard
    beside `_monitor_claim_is_releasable`, keyed on the `ace-gate` workflow, reading the
    owning member's `gate_state` and failing closed: a dead creator pid alone is exactly
    the state every pending gate shell is in, so only a terminal member releases.

14. **`sase gate wait` refuses a shell gate under `SASE_AGENT`.** In
    `notifications/cli_wait.py`, detect a `shell` block on the envelope and, when
    `SASE_AGENT` is set, exit with an actionable error naming `sase gate create --shell`
    and explaining that the gate shell publishes the decision instead. `wait_for_gate`
    itself, non-shell gates, and non-agent callers are untouched, which is what keeps
    `tests/gate_conformance/` and `sase-telegram` working.

15. **Tests**, under a new `tests/gate_shell/` package plus focused additions beside the
    existing gate suites: `shell`-block validation (unknown field, bad branch key, bad
    enum, over-long status, bad accent, defaults, the 24h timeout default); v3
    additivity (envelope round-trip and fingerprint sensitivity); the `--gate` /
    `--gate-0` suffix canonicalization regression; promotion of a bare creator and
    second-gate suffix allocation; the creation ordering with both compensations
    (publication failure restores the creator's claim and tears the member down;
    marker-write failure cancels the gate); the `%auto` short-circuit creating a settled
    shell with no marker and no kill; marker adoption terminalizing the creator as
    `DONE` with its transcript saved; settlement for each terminal state plus the
    terminal-implies-durable invariant; the claim guard in both directions; each
    reclaim-chop case; the `sase gate wait` refusal and its non-regression; and the CLI
    flag merge.

16. **Symvision.** Prefer a real in-phase consumer for every new public symbol. Only for
    symbols a later phase of this epic will consume, add
    `--epic-symbol "sase-ud(<symbol>)"` entries to the `_lint-symvision` recipe in the
    `Justfile`, and record them so the phase's `sase bead epic-symbols` check is clean
    at close.

## Verification

- Run the new `tests/gate_shell/` suite plus `tests/gate_conformance/`,
  `tests/test_notification_gates.py`, `tests/test_gate_wait_cli.py`, `tests/monitor/`,
  and `tests/shells/` while iterating.
- Run `just install` first if the workspace is stale, then `just check` after each
  meaningful change, addressing only failures this phase caused.
- Run the required exhaustive `just check-full` through `/sase_monitor` with the
  `TESTING` / `TESTED` status pair before closing the phase.
- Run `sase bead epic-symbols sase-ud.3`; resolve every leftover symbol or re-key its
  `Justfile` line to the still-open parent epic or a later phase.
- Close only `sase-ud.3`, with a note naming the ordering, compensation, `%auto`,
  settlement-invariant, reclaim, and full-verification evidence. Do not close `sase-ud`
  or any ancestor. Record out-of-scope discoveries only as `PROPOSED FOLLOW-UP:` notes
  on this phase bead.
