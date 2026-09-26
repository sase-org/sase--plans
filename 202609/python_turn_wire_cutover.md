---
tier: tale
title: Python persistence and wire cutover for sase turns
goal: "SASE writes turn and named-proc spellings to its Python-owned durable records and
  core requests, while pre-rename records and the still-legacy core output remain
  readable. The additive sase-core bindings are pinned and used, and the phase check
  passes without changing runtime syntax or TUI presentation.

  "
size: medium
proposed_by: bbugyi200.athena.sase-1ab.2
bead: sase-1ab.2
create_time: 2026-09-26 03:14:00
status: wip
---

- **PARENT:**
  [202609/sase_turn_rename.md](https://github.com/sase-org/sase--plans/blob/main/202609/sase_turn_rename.md)
- **BEAD:**
  [sase-1ab.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ab/sase-1ab.2.md)

# Plan: Python persistence and wire cutover for sase turns

This implements phase `sase-1ab.2` of the archived `sase_turn_rename` epic. Work only in
the sase checkout. The preceding sase-core additive rename is already landed; its wire
output deliberately retains legacy spellings until the later contract flip. Follow the
epic's vocabulary and compatibility policy: a new writer emits only the new spelling,
readers prefer it and fall back to all old spellings, and rewriting a record removes old
keys. Do not change published `sase-core-rs` version bounds, the runtime CLI and syntax,
TUI labels and sections, or core schema versions here.

## 1. Pin and shared member accessors

- Move `sase-core-revision.txt` past the landed additive core commit with
  `just ratchet-core-revision`; run `just install` to rebuild the extension. Switch
  Python callers to `find_gate_turn_by_gate_id` and
  `validate_standalone_named_proc_name`, including binding checks in
  `tools/validate_sase_core_rs` and `tools/check_sase_core_rs_bindings`. Fix demo seeds
  that construct the renamed request fields.
- In `src/sase/plan_chain.py`, make `AGENT_SESSION_TURN_KEY` and `TURN_KIND_KEY`
  canonical; name the older `agent_session_shell`, `family_shell`, and `shell_kind`
  constants as legacy. Put preferred-new-then-legacy lookup and `proc` to `monitor`
  value normalization in shared accessors, and make writes drop the old keys.
- Use those accessors throughout agent metadata and done marker readers, including
  `core/wire.py`, `core/agent_scan_wire_markers.py`, `core/wait_dependency_resolution/`,
  and the existing TUI metadata loaders. Update `shells/member.py`, `monitor/member.py`,
  `monitor/start.py`, and `gate_shell/member.py` to write `turn_kind` with `gate` or
  `monitor`. Keep the runtime package and TUI-owned names for their later phases.

## 2. Preserve old Python-owned records while writing new ones

- Rename plan-gate metadata keys and files from `plan_shell_*` to `plan_gate_turn_*` in
  `plan_shell/create.py`, `plan_shell/followup.py`, and inherited-prompt lookup. A plan
  gate created before upgrade must still complete using its old metadata and prompt
  files; a new gate creates only new names.
- Have gate envelopes write the `turn` block, `gate_turn` continuation mode,
  `gate_next_fork: turn`, and renamed cancel source, error-code, and intent-marker
  values. Keep explicit old-value readers in `notification_gates/`, `sudo/gate.py`, and
  `agent/gate_intent.py`. Verify hashes against the stored envelope as stored; do not
  normalize an old bundle before checking its request hash.
- Rename proc model and request fields to `proc_name` and `proc_role`, lifecycle and
  origin to `named-proc`, and the concurrency prefix to `named-proc:`. Cover `procs/`,
  `service/host_spawn.py`, and `sudo/detach_approve.py`. `Proc.from_dict` must accept
  old row fields and values. The duplicate-name check must treat a live legacy
  `shell:<name>` and new `named-proc:<name>` as the same name.
- Write `agent_session_turn_{kind,id,state}` in runner-slot requests and `named_proc` in
  hold candidates. Rename the dismissed-proc module and store to `dismissed_procs.py`
  and `dismissed_procs.json`; read the old file if no new one exists, then remove the
  old file only after a successful new-file write. Update background-command and
  startup-prune callers. Rename `AgentType.PROC_SHELL` to `NAMED_PROC`, adding a legacy
  value reader if investigation finds persisted data.

## 3. Hydrate either core wire spelling

- Rename `core/agent_scan_wire_agent_session_shell.py` and its `AgentSessionShell*Wire`
  classes and conversion function to turn names. Update importers and the agent/fleet
  node and row mirrors for `row_kind`, `turn_id`, `historical_turn`, and
  `shell_start_status` replacement. Hydrate both new and legacy spellings, including
  core-expand output, without a Python fallback for core-owned behavior.
- Audit schema-version comparisons in Python, `core/health.py`, and
  `tools/validate_sase_core_rs`. A later core output version must cause the documented
  fallback or cache rebuild, not an exception; record any remaining contract-flip hard
  failure on `sase-1ab.2` for that later phase.
- Change `Agent` model references to renamed core fields. Leave TUI-owned modules, text,
  section ids, and row names, and runtime package names outside `core/` and `procs/`,
  for their already assigned phases.

## 4. Verify and close this phase

- Add realistic legacy-input tests for agent metadata and done markers, a pending old
  plan gate, a hashed old gate bundle, old proc rows with a live `shell:`/`named-proc:`
  conflict, and an old dismissed file. Assert every new writer omits the legacy keys and
  values. Exercise both wire spellings and round trips through the new Rust bindings.
  Rename the two phase-owned test modules
  `test_core_agent_scan_wire_agent_session_shells.py` and
  `test_dismissed_proc_shells.py`, keeping explicitly named legacy fixtures.
- Read the required SASE lint/test and TUI memory before editing those areas. Run
  focused tests while iterating, then `sase tool run check` in sase. Do not run
  `just check-full`. Classify remaining shell references in this phase's scope as
  deliberately later-phase work, unrelated Unix/UI meanings, or named legacy
  compatibility code.
- Before closing, run `sase bead epic-symbols sase-1ab.2`; resolve any entries or re-key
  their Justfile lines to an open bead. Put out-of-scope discoveries or an identical
  clean-base check failure in a `PROPOSED FOLLOW-UP:` note on this phase. Close only
  `sase-1ab.2` with `sase bead close sase-1ab.2 --note` recording what passed; leave
  `sase-1ab` and all other phases open.
