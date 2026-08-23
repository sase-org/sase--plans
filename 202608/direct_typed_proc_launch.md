---
tier: tale
title: Fix direct typed proc launches
goal:
  Direct ACE and sase run submissions execute enabled percent-proc and percent-if units
  through durable typed admission without creating empty agent shells.
size: medium
proposed_by: bbugyi200.athena.0bm
---

# Fix direct `%proc` / typed-unit launches

## Goal

Make enabled `%proc` and `%if` directives execute through the existing typed launch
planner and durable admission coordinator when a user submits them directly from ACE or
`sase run`. A direct user submission is already authorized and must not create a
`LaunchApproval`, but it must preserve the same immutable typed plan, digest, wait /
condition semantics, native proc dispatch, and recovery contract used after an approved
agent-initiated launch.

Keep the disabled feature state and ordinary all-agent direct launches behaviorally
unchanged. In particular, preserve forced-name-reuse handling, preplanned
`launch_units`, segment environment injection, legacy fanout behavior, prompt recovery,
and launch result deltas whenever the expanded prompt contains no active typed launch
directive.

## Root cause and current evidence

- The reported ACE submission `#gh:sase %proc("echo hello && sleep 20 && world")`
  creates an `AGENT SHELL`; directive extraction removes `%proc` and its body, leaving
  no model prompt, and the provider fails because it is invoked with empty input.
- The feature flag is enabled and the current Rust planner classifies that exact
  expanded prompt correctly as one `ProcUnit` with the expected Bash `CodeValue`. Native
  `xprompt-proc` dispatch and proc-shell presentation are therefore downstream
  capabilities that the failing request never reaches.
- `src/sase/agent/launch_request.py` attaches a typed plan only while constructing a
  `LaunchApproval`. By contrast, the non-agent-context branch in
  `src/sase/main/query_handler/_launch.py` sends ACE and direct `sase run` requests
  straight to `launch_agents_from_cwd()`, the legacy agent-only path.
- `docs/configuration.md` and `docs/xprompt.md` explicitly describe that direct-path
  limitation, but it conflicts with the `sase-s6` acceptance contract requiring the
  enabled feature to work end to end from ACE and CLI launch surfaces.
- The root-cause evidence has been recorded as a `DISCOVERED ISSUE:` note on the active
  `sase-s6` epic. Do not create a duplicate task bead or close the epic; its land agent
  owns closure.

## Implementation

### 1. Share typed planning and select it without regressing legacy launches

- Extract a public, side-effect-free typed-plan preparation helper from the private
  LaunchApproval-only code. It should consume the already expanded/normalized preview
  prompt, call the Rust `plan_typed_launch_units` facade, translate diagnostics to the
  existing launch error shape, and return the serialized `LaunchPlan` plus digest.
- After recursive xprompt expansion, detect active `%if` / `%proc` syntax using the
  shared directive/fence contract (including parenthesized and fenced forms and
  excluding fenced, inline-literal, and `%xprompts_enabled:false` text). Do not add a
  second ad-hoc regex grammar. Route a direct launch through typed admission only when
  such a directive is actually present; otherwise retain the existing legacy path
  byte-for-byte.
- Resolve the plan's selected project from the expanded prompt's explicit known VCS ref
  before falling back to the current non-home launch context. Store the canonical
  project identity for workspace lookup, while rendering the configured project display
  name in user-facing previews/messages. Do not let an implicit/current `home` context
  override an explicit `#gh:` / `#git:` target, and do not invent workspace 0.
- Reuse this project-selection and typed-plan helper from LaunchApproval construction as
  well, so direct and approved launches cannot classify the same prompt differently.
  Keep grammar, graph validation, and unit classification in `sase-core`; no duplicated
  Python backend logic is acceptable.

### 2. Give direct typed launches a durable, coordinator-readable request

- Add a SASE-owned direct-launch bundle/request helper that writes the exact expanded
  plan, content digest, submitted prompt/source cwd, selected project, safe input
  context, request id, and source surface atomically before any unit is dispatched. It
  must not publish a notification gate or masquerade as a pending `LaunchApproval`.
- Make the admission coordinator's request reader accept both existing neutral
  LaunchApproval bundles and the new direct bundle through one validated adapter. The
  detached coordinator must be able to reopen the direct request after the short-lived
  `sase run`/ACE launch proc exits, resume waits idempotently, and retain its receipt
  and per-unit journal under that bundle.
- Dispatch the prepared direct plan with `dispatch_typed_launch_request`. Immediate
  eligible procs must reserve native `proc-shell` rows with origin `xprompt-proc`;
  blocked plans must detach the existing coordinator and return successfully as accepted
  work. Never create an agent directory, runner claim, LLM invocation, `done.json`,
  family, or finalizer for a proc unit.
- Add a defense-in-depth invariant at the legacy agent boundary: if an enabled typed
  directive somehow reaches agent-only execution, fail with a targeted typed-admission
  error before allocating or invoking an LLM. A missed caller must never degrade into an
  empty agent prompt again.

### 3. Return typed launch outcomes correctly to CLI and ACE

- Extend the durable `run.launch` success payload for typed dispatch with plan digest,
  `admission_complete`, the six admission counts, per-unit outcomes/identities where
  available, and serialized `AgentLaunchResult` rows only for actual agent units.
- Treat a proc-only launch with zero agent results as valid. Use unit/admission language
  in CLI and ACE messages (for example, accepted/launched launch units), rather than
  reporting zero agents or `agent launch produced no results`. An unresolved wait is an
  accepted background admission, not a failed launch; immediate condition/launch errors
  should use the coordinator's existing summary semantics.
- Keep VCS MRU recording, unresolved-reference warnings, prompt-stash recovery, and
  operation result writing intact. Request a coalesced Agents refresh after immediate
  dispatch so the top-level proc-shell projection appears, and rely on the existing proc
  observer/settlement notifications for later coordinator progress without fabricating
  an agent delta.

### 4. Cover the real entry points and update the contract docs

- Add focused query-handler tests proving the exact reported ACE/direct `sase run`
  prompt calls typed admission, yields a `ProcUnit`, and never calls
  `launch_agents_from_cwd` or the LLM path.
- Cover positional, named, and fenced `%proc`; direct `%if` agent admission; mixed
  Agent/Proc waits; an initially blocked plan that resumes from its durable direct
  bundle; invalid typed syntax; feature-off rejection; explicit VCS target precedence;
  and proc-only durable results with no `AgentLaunchResult` entries.
- Add compatibility tests showing that feature-on prompts without active typed
  directives still take the legacy path, including forced-name reuse and supplied
  `launch_units`, and that directive-looking text in literal/disabled regions does not
  opt into typed admission.
- Add an integration test with an isolated `SASE_HOME` that submits a harmless direct
  Bash proc, waits for settlement, verifies `origin=xprompt-proc` and
  `lifecycle=proc-shell`, checks its output, and proves no agent artifact or provider
  invocation was created.
- Update `docs/configuration.md` and the experimental typed-launch section of
  `docs/xprompt.md`: user-initiated ACE/CLI submissions execute directly through durable
  typed admission, while only agent-initiated launches require `LaunchApproval`. Remove
  the obsolete warnings that direct launches merely strip `%if` / `%proc`, and keep the
  feature flagged as beta.

## Verification and epic handoff

- Run `just install` before repository checks, then run the focused planner,
  query-handler, launch-admission, proc-runtime, durable-operation, and ACE launch
  completion tests covering the changes.
- Run `just check`. Because this changes the launch broadening path, run
  `just check-full` only through `/sase_monitor` with the required `TESTING` / `TESTED`
  statuses and a follow-up that inspects the result; repair and rerun until clean.
- Manually exercise the reported feature-enabled ACE prompt with a short harmless body
  and confirm that ACE shows a top-level proc shell, the command settles successfully,
  and no failed/empty agent shell is created. Also exercise a normal agent prompt with
  the flag enabled to confirm the legacy direct launch remains unchanged.
- Append a final verification note to `sase-s6` with the landed root-cause fix and test
  / manual evidence so the running land agent can account for it. Do not close or
  otherwise rewrite the epic.

## Acceptance criteria

- With `typed_launch_units` enabled, direct ACE and `sase run` `%proc` submissions reach
  native durable proc dispatch and never invoke an LLM or create an agent shell.
- Direct `%if`, typed waits, mixed units, digest validation, restart/replay, workspace
  policy, and summaries use the same planner/coordinator semantics as an approved
  launch.
- User-initiated direct launches create no approval notification; agent-initiated
  launches continue to require and honor `LaunchApproval`.
- Feature-off behavior and feature-on all-agent legacy launches remain compatible,
  including forced reuse and preplanned launch-unit payloads.
- Proc-only operation results are successful and proc-aware despite containing no agent
  result rows, ACE refreshes to the real proc-shell projection, and invalid routing
  fails before provider invocation.
- Focused tests, `just check`, monitored `just check-full`, the direct ACE smoke test,
  and the final `sase-s6` handoff note all complete successfully.
