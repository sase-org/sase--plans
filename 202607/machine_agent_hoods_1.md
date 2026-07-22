---
tier: tale
title: Machine agent hoods end to end
goal: Qualify every new durable local agent identity with the configured machine hood
  while preserving bare local display and lookup behavior, legacy-name equivalence,
  foreign-machine visibility, and machine-aware commit provenance.
bead: sase-8k.3
parent: sase/repos/plans/202607/agents_sidecar_repo.md
create_time: 2026-07-22 14:04:37
status: done
---

- **PROMPT:** [202607/prompts/machine_agent_hoods_1.md](prompts/machine_agent_hoods_1.md)

# Machine agent hoods end to end (`sase-8k.3`)

## Goal

Make a configured `machine_name` the durable top-level hood for every newly created local agent while preserving today's
user experience on the owning machine. New names, metadata, environment values, chat identity, registry entries, and
commit footers must use `<machine_name>.<local-name>`; local UI, CLI, prompt, completion, and lookup surfaces must
continue to show and accept the bare local name. Imported/foreign machine-qualified names must remain fully visible.
Existing unqualified names remain in place and are treated as local for collision and lookup purposes.

The prerequisite config work (`sase-8k.1`) is already present in this repo and the prerequisite Rust helpers
(`sase-8k.2`) are already present in `sase-core`. This tale integrates those helpers in the Python application; it does
not need another Rust-core change unless verification exposes a binding mismatch.

## Behavioral contract

- When `machine_name` is unset, every path behaves exactly as it does today.
- With local machine `athena`, both `foo` and `athena.foo` identify the same local agent. New durable state uses
  `athena.foo`; a legacy stored `foo` still resolves and prevents reuse of `athena.foo`.
- A known foreign prefix such as `zeus.foo` is never stripped on `athena` and cannot be used to launch a new local
  agent.
- Qualification applies once at the root: `athena.clan.member` and `athena.family--code`, never
  `athena.clan.athena.member`.
- Raw qualified names remain available to persistence, identity, and mutation code. Stripping happens only at lookup
  normalization or final presentation.
- No existing artifacts or registry entries are mass-renamed.

## Implementation

1. Add `src/sase/core/machine_hood_facade.py` as the single Python boundary to the Rust `validate_machine_name`,
   `qualify_machine_agent_name`, `strip_machine_agent_name`, and `machine_hood_of` bindings. Build small application
   helpers on top for optional local qualification/stripping, known-foreign detection, canonical local-equivalence keys,
   and lookup candidates. Source identity from `get_machine_name()` and known overlays from `discover_machine_names()`,
   but allow callers processing a loaded batch to pass a resolved identity snapshot so Agents-tab
   render/group/completion loops do not perform selector/config I/O per row. Keep the no-config path a strict no-op.

2. Make name validation, allocation, reservation, and launch persistence hood aware.
   - In `src/sase/agent/launch_validation.py`, compare explicit requests, reserved names, clan containers, and family
     containers by canonical local-equivalence key. Accept an explicitly local-qualified spelling, reject a prefix
     matching another known machine with an actionable error, and keep forced-reuse/container diagnostics expressed with
     the user's local spelling.
   - In `src/sase/agent/names/_registry.py`, `_claim.py`, `_auto.py`, and `_templates.py`, qualify new local
     claims/reservations and make exact-name plus dotted-namespace availability checks symmetric across `foo` and
     `athena.foo`. Preserve the actual stored key and owner record, so legacy entries are not rewritten. Ensure release,
     lookup, suggestion, template matching, and registry rebuild paths use the same equivalence rules rather than
     bypassing them.
   - Update `PlannedNameAllocator` and its single/multi-prompt launch callers to choose template/auto tokens in the bare
     namespace, qualify the selected concrete name before durable reservation and `SASE_AGENT_PLANNED_NAME` publication,
     and retain the visible `0`, `1`, ... sequence. Normalize template reference matching against stored qualified names
     while returning the durable identity used by the child.
   - At the child-runner identity choke point in `src/sase/axe/run_agent_directive_identity.py`, qualify any name that
     could not be planned in the parent before claim, `SASE_AGENT_NAME` publication, and `build_agent_meta()`. Audit
     follow-up/retry/resume/revive/rename and dismissed-name flows so newly written `name`/`workflow_name` values remain
     qualified without corrupting legacy names.
   - Qualify clan containers and members together in `multi_prompt_launch_plan.py` / `clan_membership.py`, and compose
     family children from the already-qualified family root in `plan_chain.py` and the family attach/promotion paths.
     Persist qualified clan/family metadata and validate membership after both sides use the same durable form.

3. Normalize every agent-name read and prompt reference without changing the underlying durable identity.
   - Teach the central lookup stack under `src/sase/agent/names/_lookup*.py` and template/resume/wait helpers to try the
     exact spelling plus its local qualified/legacy equivalent. Apply the same rule to exact agents, workflows,
     families, clans, dismissed bundles, `@name`, `%wait`, and `#fork` resolution; prefer an exact durable match when
     both forms exist.
   - Audit secondary prompt consumers (family attach, output-variable context, wait-status maps, and template rewrites)
     so keys presented to a local user are bare while stored relationships continue to use raw qualified names.
   - Keep foreign qualified identities exact and unmodified throughout lookup.

4. Strip the local hood at final presentation surfaces while retaining full foreign hoods.
   - In `Agent.refresh_presented_agent_name()`, derive the family reference first and then strip the local hood. Update
     the Agents-tab render-only call sites that still interpolate `agent_name` directly (list/detail, clan/family
     rosters, logs, cleanup/revive/kill labels, clipboard and context labels) to use the presented value, while
     operational actions keep using the raw value.
   - Base `agent_name_key`, `agent_hood`, descendant/ancestor indexing, and `_agent_hood_chain` on the precomputed
     presented identity for local rows. This prevents `athena` from becoming a visible local kinship group while
     allowing `zeus` to group foreign rows normally. Do not add config reads, stats, globs, or subprocesses to
     render/navigation paths.
   - Make agent completion candidates, labels, member lists, and wait maps insert/display bare local names and full
     foreign names from the already loaded Agent snapshot. Preserve raw identity as a search alias where that helps an
     explicitly qualified local spelling resolve.
   - Ensure chat writers receive the qualified durable identity in filenames, headers, and metadata. Project local hood
     stripping into `sase chat list` pretty/JSON output and agent selectors without rewriting transcript files.
     Grep-audit all remaining display-name sites and classify each raw use as intentional persistence/identity or
     convert it to the presentation helper.

5. Make commit provenance machine-aware in `src/sase/workflows/commit/runtime_tags.py`. `SASE_AGENT` must be qualified
   (including defensive qualification of a legacy env/meta value during the transition), and `SASE_MACHINE` must prefer
   configured `machine_name` with the current hostname/`HOSTNAME` behavior retained only as the compatibility fallback.
   Existing legacy footer parsing remains unchanged.

## Tests and validation

- Add facade tests for all four Rust calls plus configured/unconfigured local helpers, idempotence, nested/family names,
  known foreign classification, and lookup/equivalence candidates.
- Extend registry/template/launch tests for both legacy-collision directions, namespace collisions, explicit bare and
  local-qualified names, known-foreign rejection, unchanged behavior without config, auto/template token order, and
  parent/child reservation cleanup.
- Add launch-level coverage proving single, multi-prompt, clan, family, retry/resume, metadata, `SASE_AGENT_NAME`, chat,
  and workflow relationships persist exactly one local machine hood.
- Extend lookup/chat tests so bare and local-qualified selectors find both new qualified and legacy agents, while
  foreign identities stay qualified in results and `sase chat list` output strips only the local hood.
- Extend Agents-tab model/grouping/completion tests with local `athena.*`, legacy bare, and foreign `zeus.*` fixtures.
  Assert presented names, family roots, hood chains, descendant counts, inserted completions, labels, and raw identity
  preservation. Verify the implementation performs no per-row config or filesystem reads in render/navigation/completion
  paths.
- Extend runtime-tag tests for configured machine precedence, hostname fallback, qualified agent footers, idempotence,
  and legacy env/meta qualification.
- Run `just install` before checks, then focused tests while iterating and the repository-mandated `just check` for
  final validation. Re-run affected TUI visual tests if any rendered fixture changes; inspect rather than accepting
  unrelated golden drift.

## Completion

After all checks pass, update only `sase-8k.3` to `closed` and re-read it plus parent `sase-8k` to verify the phase is
closed while the epic remains open/in progress. Do not create beads, close the parent epic, edit SASE memory files or
generated instruction shims, or perform the later agents-sidecar/sync phases.
