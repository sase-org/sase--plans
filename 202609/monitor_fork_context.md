---
tier: tale
title: Preserve monitor shell identity when expanding fork context
goal:
  Resume dotted agent families with bounded monitor evidence while preserving correct
  provider retry behavior.
size: medium
proposed_by: bbugyi200.athena.sase-x7.3.1.5.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.sase-x7.3.1.5.f0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-x7.3.1.5.f0.md)
  - [bbugyi200.athena.sase-xe.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xe.1/README.md)
- **COMMITS:**
  - [a45669b](https://github.com/sase-org/sase/commit/a45669b26fd3f6f313398e09f82160e2253fe168)
    — fix(agent): preserve monitor fork context
  - [8efecdd](https://github.com/sase-org/sase/commit/8efecdd7390a0103f66ce7a3f4b54376a2079a63)
    — feat(agent-listing): use bounded index snapshots

# Preserve monitor shell identity when expanding fork context

## Goal

Make a monitor continuation for a dotted numeric family such as `sase-x7.3.1.5` resume
the recorded agent family with a bounded command execution record. It must not inject
the full monitor log as an assistant conversation. Keep delayed retries for actual
transient provider errors and avoid retrying an unchanged request rejected for excessive
input length.

This is one bounded implementation tale. The family lookup repair, a small shared
output-bound primitive, and their integration tests can be completed by one coding
agent. Open `sase-core` through `sase repo open` before accessing that repository. Only
this scratch plan is authored during planning; implementation starts after approval.

## Confirmed diagnosis

The failed run is the concrete shell `sase-x7.3.1.5--1`, launched by monitor
`5hq6bd4hjza0`. The family selector in the incident report is `sase-x7.3.1.5`. Stable
local run evidence is under
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/06/20260906130712/`:

- `done.json` records the failure at 2026-09-06 13:08:41 EDT. Codex rejected
  `turn/start` with JSON-RPC code `-32602`, `input_error_code=input_too_large`,
  `max_chars=1048576`, and `actual_chars=1913445`.
- `raw_xprompt.md` and `submitted_xprompt.md` are each 24,153 characters. They contain
  the family fork directive and a monitor result with a 160-line tail.
- `workflow-tmp_260906_130821-main_prompt.md` is exactly 1,913,445 characters, matching
  the rejected request. Its inherited history occupies 1,889,378 characters. It begins
  with a monitor command as `User` and essentially the whole log as `Assistant`, under
  the singular `Previous Conversation` heading.
- `agent_meta.json` records `wait_for_fork_sources` as `kind=agent`, pointing at the
  monitor's artifact timestamp `20260906124509`, despite its metadata having
  `agent_family_role=monitor`, `agent_family=sase-x7.3.1.5`, and the monitor ID. The
  failed run has no retry/attempt files or retry metadata.
- The stable monitor chat
  `~/.sase/chats/202609/gh_sase_org__sase-ace_run-sase_x7_3_1_5__mon-20260906124509.md`
  is 1,889,459 characters across 20,665 lines. Its longest line is 3,155 characters. The
  160-line tail itself is about 22.5k characters. This incident was caused by full-log
  history injection, not an unusually long tail line.

The call chain is concrete:

1. `monitor/followup_prompt.py` emits `#fork:<family>`.
2. `agent/names/_lookup_groups.py::_find_agent_family_exact` rejects a name when the
   legacy `plan_chain.is_agent_family_member` helper says it is a member. That helper
   interprets the trailing `.5` as an old feedback suffix. It returns true for
   `sase-x7.3.1.5`, so family discovery returns `None` before reading its valid family
   metadata.
3. Fork source resolution falls through to the ordinary agent path. A workflow alias
   resolves to the completed monitor, whose `done.json` has `outcome=monitored` and a
   full-log `response_path`. The fallback fails to inspect the resolved artifact's
   monitor markers before choosing `kind=agent`.
4. `history/chat_fork/build.py` consequently calls `load_chat_for_resume` on the monitor
   transcript, bypassing the existing bounded `ForkProcInfo` rendering.
5. `axe/run_agent_exec_retry.py::handle_workflow_error` finds no retry pattern and
   returns `raise`. The current merged Codex configuration matches none of this error,
   as confirmed by executing the classifier against `done.json`.

The existing Rust identity facade correctly parses `sase-x7.3.1.5` with no shell member
role. A read-only experiment substituted that canonical classification in memory, using
the recorded starter and monitor artifacts. The resulting family contained an agent plus
a proc, and rendered 17,473 characters of inherited history while preserving the log
pointer and omitting the full-log prefix.

This denies the transient-provider hypothesis for this run. The stale rollout-path
messages precede the decisive input validation rejection; there is no evidence that
clearing those paths or waiting would repair this request. OpenAI documents a different,
retryable app-server overload response, `-32001`, with backoff in the
[official app-server protocol documentation](https://learn.chatgpt.com/docs/app-server#protocol).
The precise input limit above comes from the local failure record, not an assumed model
token-window size.

The monitor itself successfully launched its follow-up. Its command separately exited 1
at `selection-health`; this plan does not claim that `just check-full` passed or that
the fleet-deployment phase completed.

## Implementation

### 1. Use canonical identity for family discovery and resume selection

Repair the lookup paths in `src/sase/agent/names/_lookup_groups.py` and
`_lookup_resolution.py` to distinguish canonical `--<role>` shell members from dotted
family names using `core.agent_identity_facade.parse_agent_family_name`. The
implementation must consume the existing Rust classifier rather than add another Python
name parser. Recorded family metadata is the evidence that a canonical solo-shaped name
represents a family container.

Audit the corresponding exact-member lookup in `scripts/_agent_chat_from_name_common.py`
so a dotted family is not redirected to its apparent legacy parent. Keep the legacy
suffix helpers for their intended historical display/lookup uses; do not globally remove
`.plan`, `.code`, or numeric legacy support. When no matching family metadata exists,
preserve ordinary exact agent lookup. Preserve owner qualification, newest-generation
selection, parent-timestamp traversal, archived members, and exclusion of the current
shell.

`agent/fork_waits.py` must consequently bind the family and its root identity, rather
than label its monitor as a plain agent. Confirm the fixed behavior through its real
lookup dependency; do not merely mock `find_agent_family` in the regression.

### 2. Classify the resolved monitor artifact before loading any chat

In `scripts/_agent_chat_from_name_sources.py`, inspect the authoritative metadata of the
artifact returned by `resolve_resume_agent_name`, as well as the existing explicit
family-member path. Route real monitors through `resolve_monitor_fork_source` and the
existing `proc_info_from_monitor` projection before consulting a successful
`response_path`.

This must cover workflow aliases and incomplete/rootless family metadata, so a future
family lookup miss cannot reopen the full-log path. Check role and monitor markers
together using `is_real_monitor_member`: a starter agent may carry a `monitor_id`
without being the monitor shell. Preserve the starter's conversation in that case.
Missing monitor records should produce an actionable resolution error, never a silent
fallback to the full monitor chat. Align fallback fork-wait classification with the
resolved monitor identity where applicable.

Retain typed failed-command outcomes, untrusted-output framing, source attribution, and
`sase monitor show <id> --all-lines` access. Do not modify stored monitor logs, chats,
existing failed-run state, or agent names.

### 3. Bound direct shell output by characters as well as lines

The existing `shells/prompt.py::untrusted_output_section` only limits line count. Add a
small deterministic Rust operation for taking a tail under both a line and a
Unicode-character budget, expose it through `sase_core_py`, and consume it with a thin
SASE adapter. Use it from the shared monitor/gate output-section helper. This prevents a
single multi-megabyte line from independently overflowing the follow-up. Use a
12,000-character maximum for the selected raw tail, consistent with the existing fork
proc-log budget. Keep the omission notice separate and explicit about tail truncation;
count Unicode scalar values consistently with Python string length. Preserve ordinary
short output and existing line semantics.

Keep the full retained log retrievable, and preserve command, outcome, reason, workspace
explanation, routing, and the complete next action. Apply bounds before
choosing/widening Markdown fences, retain literal disabled regions, and ensure the
selected output cannot activate directives. `next-output=file` and `none` retain their
existing direct-output behavior. This is a bound on generated output excerpts, not
arbitrary truncation of user instructions or a promise that all possible conversation
histories fit every provider.

Add the Rust API and binding tests together. Follow release-plz ownership of crate
versions; do not manually bump Cargo versions. Build the local binding for SASE
verification and coordinate any required consumer dependency update with the normal core
release process before landing.

### 4. Verify the retry distinction without adding a futile retry rule

Do not add `input_too_large`, generic `turn/start failed`, `-32602`, or stale rollout
warnings to the transient retry patterns. Retrying this unchanged payload or wrapping it
in a fresh-shell fork would reproduce or enlarge the failure.

Add regressions using the recorded error shape, with harmless synthetic rollout paths,
proving that the default Codex config and workflow error handler do not sleep, spawn a
child, or fall back for this input rejection. Preserve the decisive error text and
counts in the failure evidence.

Also verify an actual configured transient error takes the delayed retry path, honors
retry limits and workspace preservation, and takes the fresh-shell path when
`spawn_new_agent=true`. Use mocked clocks and launchers, never real sleeps or paid
provider calls. Current defaults are three retries with 60/300/1800-second waits and
`spawn_new_agent=false`; fresh-shell retry is an existing opt-in, not a missing default
capability. No global retry-policy change is needed for this incident.

## Tests and acceptance

- Reproduce the original failure with a synthetic dotted family containing a completed
  starter, a settled monitor with a roughly 1.9 MB transcript, and the current
  successor. Exercise family lookup, fork-wait binding, source resolution, and the
  rendered/preprocessed provider prompt together. Before the fix, the family is rejected
  and monitor output becomes an assistant response. After the fix, the family includes
  the starter and a typed proc, excludes the successor, retains the requested next
  action and log pointer, and stays below 100,000 characters for this fixture. The test
  must fail on the original implementation.
- Cover plain and owner-qualified dotted numeric families, explicit `--mon` and
  `--mon-0` selectors, direct ordinary agent selectors, historical names, and generation
  selection. Existing archived/rootless-family tests must stay green.
- Cover a monitor reached only through an alias/fallback, a starter that merely has
  `monitor_id`, and a missing monitor record. Assert that a monitor never goes through
  the ordinary chat loader in these cases.
- Test shared output bounds with normal short output, zero lines, exact budget,
  over-budget multiline output, a multi-megabyte single line, non-ASCII text, repeated
  backticks, and directive-shaped text. Assert intact framing and next action. Check
  both monitor and gate consumers and the file/none policies.
- Add default retry and runner behavior regressions for the exact size-rejection carrier
  and representative existing transient errors; retain quota/fallback precedence tests.
  No broad expansion of the provider error catalogue is part of this repair.

Relevant existing suites include `tests/test_agent_names_lookup.py`,
`tests/test_agent_names_workflow.py`, `tests/test_agent_chat_from_name_family.py`,
`tests/test_agent_chat_from_name_family_monitor.py`, `tests/test_fork_workflow.py`,
`tests/monitor/test_monitor_followup_prompt.py`, gate follow-up prompt tests,
`tests/test_llm_provider_retry_defaults.py`, and the
`tests/test_axe_run_agent_exec_retry*` / retry-spawn suites. Locate the fork-wait tests
alongside them and extend the closest existing fixtures.

Run the core repository's `just check`, which includes PyO3 binding checks; do not
substitute `cargo test -p sase_core`. In SASE, read the current `lint_and_test` memory,
run `just install` when needed for the isolated environment, and run `just check` after
implementation. Use the required `TESTING`/`TESTED` monitor handoff for exhaustive
`just check-full` before landing or when selection broadens. For verification handoffs
while the installed runtime may still lack this fix, use a simple nonnumeric family name
and bounded/file output; do not rely on `--next-output=file` alone to repair the
existing full-history classification bug.

Report implementation and verification results separately from the pre-existing
fleet-deployment failure. Do not close or restart `sase-x7.3.1.5` as part of this coding
tale: the approved change should make a later explicit continuation safe, while that
phase retains its original verification obligations. No recovery run, live deployment,
provider cache cleanup, or historical artifact rewrite is needed to prove this repair.
