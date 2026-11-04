---
tier: tale
title: Stop workspace retention from breaking agent kill/dismiss cleanup
goal:
  TUI kill and dismiss cleanup procs submit and persist again when any roster agent has
  a retained workspace, and a test guard stops any future non-JSON Agent bundle field
  from landing unclassified.
size: small
proposed_by: bbugyi200.athena.18b
proposed_with: claude/opus
create_time: 2026-11-04 15:27:12
status: wip
---

# Plan: Fix "Cleanup task failed: Object of type AgentWorkspaceRetention is not JSON serializable"

## Symptom

Killing or dismissing agents in the ACE TUI (single `x`, bulk `X` cleanup, clan
kill/dismiss) shows the usual "Dismissed N agents → Archive" / "Killed agent" toast,
then an error toast:

> Cleanup task failed: Object of type AgentWorkspaceRetention is not JSON serializable

`~/.sase/logs/tui_toasts.jsonl` has five of these today across separate TUI sessions.
The matching `tui.log` tracebacks all end in
`sase.procs.submission._write_operation_request` → `sase.ops.io.write_private_json` →
`json.dumps` →
`TypeError: Object of type AgentWorkspaceRetention is not JSON serializable`.

## Root cause

Commit `31a28d9618` ("feat(stranded-work): retain held workspace as stranded unless live
successor owns", Nov 1) added a presentation field to the Agent model:

- `src/sase/ace/tui/models/_agent_state_core.py`:
  `workspace_retention: AgentWorkspaceRetention | None = field(default=None, compare=False, repr=False)`.
- Meta enrichment (`_loaders/_meta_enrichment_wire.py`,
  `_loaders/_meta_enrichment_filesystem_impl.py`) fills it with a frozen
  `AgentWorkspaceRetention` dataclass for every terminal agent whose
  `diagnostics/workspace_retention.json` record is still unresolved. These are the
  agents behind the header's "N stranded" count.

The field is `init=True` and was **not** added to `_RUNTIME_ONLY_BUNDLE_FIELDS` in
`src/sase/ace/tui/models/agent_bundle.py`. So `agent_state_to_bundle_dict()` copies the
raw dataclass into the dict. It has no encoder branch for it. Every kill/dismiss path
builds its durable request through `cleanup_payload.serialize_agents()`, which calls
`agent_state_to_bundle_dict()`:

- `actions/agents/_dismissing.py` (bulk + single dismiss)
- `actions/agents/_kill_procs.py` (bulk + single kill)
- `actions/agents/_marking.py`

The request is JSON-encoded in the durable submit worker
(`_proc_action_submission.py::_submit` → `submit_durable_proc_request`). The worker's
generic `except Exception` turns the `TypeError` into a failed `TrackedProcResult`.
`CleanupProcMixin._on_cleanup_proc_complete` then shows it as "Cleanup task failed: …".

**Why it hits almost every cleanup, not just retained agents:** each payload also
carries `"agents_with_children": serialize_agents(agents_with_children_snapshot)`. That
is the **whole roster** (`app._agents_with_children`). One retained-workspace agent
anywhere in the list poisons every kill and dismiss, even for unrelated agents. A
scratch repro confirmed this. Dismissing a plain agent fails when only a bystander in
the `agents_with_children` snapshot has `workspace_retention` set.

**Consequence of each failure:** the persist-cleanup proc never starts. The UI has
already tombstoned the rows and sent SIGTERM. But no dismissed bundle or dismissed-index
entry is written, no artifact/workspace cleanup runs, and kill escalation for agents
that ignore SIGTERM is skipped. Rows hidden this way come back after a TUI restart.

This is the third bug of this kind. The same failure happened for
`wait_epic_follows`/`EpicFollowView` (fixed in `682e52fc71`) and for the clan-wave
projection (`clan_waves`/`clan_wave_slot`, now runtime-only). The existing guard,
`test_bundle_dict_is_json_serializable_for_populated_agent`, only covers fields its
author remembered to populate. So a new dataclass-typed field slips through.

## Fix

### 1. Treat `workspace_retention` as runtime-only bundle state

In `src/sase/ace/tui/models/agent_bundle.py`, add `"workspace_retention"` to
`_RUNTIME_ONLY_BUNDLE_FIELDS` (keep the set sorted the way it is grouped now). Add a
short comment on why it is safe to drop:

- It is warm presentation state. Meta enrichment rebuilds it from the on-disk retention
  record on every load (`load_agent_workspace_retention`).
- The cleanup subprocess does not read it. Dismissal resolves retention from disk in
  `_dismiss_persistence._release_held_workspace_claims` →
  `dismiss_terminal_workspace_retention(project_file, artifacts_dir)`.

This one change fixes all three places that use the skip set:

- the cleanup archive DTO (`serialize_agent`)
- dismissed-bundle writes (`dismissed_agents_bundles.save_dismissed_bundle` →
  `to_bundle_dict`)
- `from_bundle_dict`, which already ignores runtime-only keys, so a stray
  `workspace_retention` key in an old payload is harmless.

Do **not** add an encoder/decoder pair. Persisting the retention view would duplicate
the authoritative on-disk record and go stale after settlement.

### 2. Regression tests for this bug

In `tests/ace/tui/actions/test_cleanup_payload.py` (follow the existing
`test_cleanup_archive_omits_runtime_clan_wave_projection` pattern):

- `test_cleanup_archive_omits_workspace_retention`: build `_local_agent(...)` and set
  `agent.workspace_retention = AgentWorkspaceRetention(record=WorkspaceRetentionRecord(path=..., payload={...}), held_claim_ttl_days=7)`.
  Check that `"workspace_retention" not in serialize_agent(agent)` and that
  `json.dumps(payload, allow_nan=False)` succeeds.
- `test_cleanup_dismiss_payload_encodes_with_retained_bystander`: copy the production
  failure shape. A dismiss payload whose `agents` is one plain agent and whose
  `agents_with_children` is `serialize_agents([plain, retained_bystander])` must
  round-trip through `json.loads(json.dumps(...))`.

In `tests/test_agent_model_bundle_roundtrip.py`, next to
`test_bundle_skips_wait_display_source`, add `test_bundle_skips_workspace_retention`:

- `to_bundle_dict()` omits the key and is JSON-serializable.
- `Agent.from_bundle_dict({...bundle, "workspace_retention": {"path": "x"}})` restores
  `workspace_retention is None`.

Re-export it from the `tests/test_agent_model_bundle.py` facade, including `__all__`,
like its siblings.

### 3. Structural guard so the next dataclass field cannot regress silently

Add `tests/ace/tui/models/test_agent_bundle_field_types.py` with a test that audits
Agent field annotations instead of relying on hand-populated fixtures:

- Find the persisted field names by behavior, not by importing the private skip set:
  `persisted = set(minimal_agent.to_bundle_dict())`.
- For each `dataclasses.fields(Agent)` entry in `persisted`, read its annotation. With
  `from __future__ import annotations` this is a string, e.g.
  `"AgentWorkspaceRetention | None"`. If it is not a string, fall back to a readable
  name.
- Collect the identifier tokens, after collapsing `Literal[...]` contents. Require each
  token to be in a JSON-native allowlist: `str`, `int`, `float`, `bool`, `None`, `list`,
  `dict`, `tuple`, `Literal`, `Any`, `datetime`, `AgentType`, and `VarValue` (a JSON
  type alias in `sase.core.output_variable_values`).
- Any field that fails must be in an explicit `_ENCODED_BUNDLE_FIELDS` set in the test,
  naming fields that have a dedicated encoder in `agent_state_to_bundle_dict` and a
  decoder in `from_bundle_dict`. Today that set is `finalizer_status`, `linked_repos`,
  `wait_epic_follows`, `wait_turn_bindings`, and `imported_source_owner`. A scratch
  audit of the current tree shows these, plus `output_variables` (covered by `VarValue`)
  and the buggy `workspace_retention`, are the only non-native persisted annotations
  among ~282 persisted fields.
- The failure message must name the field and its annotation. It should tell the author
  to either add the field to `_RUNTIME_ONLY_BUNDLE_FIELDS` in `agent_bundle.py` or add
  an encoder/decoder pair and list the field in `_ENCODED_BUNDLE_FIELDS`.
- Add a second assertion that `_ENCODED_BUNDLE_FIELDS` has no stale entries. Each name
  must still be persisted and still have a non-native annotation, so the allowlist
  cannot quietly grow.

Check that the guard actually works: temporarily remove the step-1 change and confirm
the new test fails naming `workspace_retention`, then restore it. Mention this check in
the commit body. Do not commit the reverted state.

### 4. File the failure-path follow-up (do not fix it here)

Use `/sase_new_task` to dedupe-check and, if it is new, file a `bug` task bead covering
this gap. When the durable cleanup proc fails **before it starts** (asynchronous submit
worker failure in `_proc_action_submission._submit`, as opposed to the synchronous
rejection handled by `_submit_cleanup_proc` returning False):

- `CleanupProcMixin._on_cleanup_proc_complete` only toasts.
- Tombstoned or dismissed rows are not resurfaced, because the completion has no
  `resurface_identities` payload.
- `_escalate_kills_without_cleanup_proc` never runs.

So any future submit-time failure again silently half-applies a kill or dismiss. Point
the bead at `actions/agents/_cleanup_procs.py`, `_kill_procs.py`, `_dismissing.py`, and
`_proc_action_submission.py`. Use your judgment on size per the bead skill; `large` is
the default when unsure.

## Out of scope

- Changing what `agents_with_children` carries (the whole roster snapshot). Shrinking it
  is a separate performance/robustness topic. With the guard test, a single field can no
  longer poison it.
- The unrelated "The selected clan row has no agent reference" toast in the same
  screenshot. That is the clipboard/copy action (`actions/clipboard/_agents.py`)
  rejecting a clan row, which is expected behavior.
- Making `agent_state_to_bundle_dict` silently drop unknown values at runtime. The
  module deliberately fails loudly at the JSON boundary. The new test moves that failure
  to CI instead.

## Verification

- New and updated tests pass: `tests/ace/tui/actions/test_cleanup_payload.py`,
  `tests/test_agent_model_bundle_roundtrip.py` (via the facade too), and
  `tests/ace/tui/models/test_agent_bundle_field_types.py`.
- Existing `test_cleanup_archive_dto_fields_match_bundle_writer` still passes; the DTO
  and the writer share the skip set.
- Run `just check` as the `lint_and_test` memory note directs.

## Notes for the user after landing

- Restart `sase tui` on each machine (athena and apollo) so the running process loads
  the fix.
- Rows whose cleanup failed were only hidden for that TUI session. They reappear after
  the restart and need to be dismissed again. Agents killed during a failed cleanup got
  SIGTERM but no escalation or archive persistence.
