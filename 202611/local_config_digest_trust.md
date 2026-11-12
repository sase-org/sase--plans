---
tier: epic
title: Digest-pinned trust for project-local commands
goal: Require human acknowledgment of a project's current executable config before
  host-side hooks or config-defined workflows can run, while preserving ordinary local
  configuration.
parent_bead: sase-1sb.2
phases:
- id: trust-policy
  title: Define the Rust local-command trust contract
  size: medium
  depends_on: []
  description: 'trust-policy: Implement the pure Rust command-bearing projection,
    canonical

    digest, filtering and trust-decision contracts described in this plan. Add

    PyO3 bindings, round-trip tests and a thin Python adapter. Cover normalized

    legacy macro spellings, partial overrides of executable definitions, file

    hook template references, nested workflow steps and unchanged non-command

    keys. Do not activate enforcement yet. Publish core changes through host

    finalization and ensure the sase core revision pin contains the bindings.

    '
- id: trust-approval
  title: Persist trust and add human acknowledgment commands
  size: medium
  depends_on:
  - trust-policy
  description: 'trust-approval: Implement the Rust-owned machine-local trust store
    and the

    thin Python project-identity and filesystem adapters. Add sase config trust

    allow/status/revoke with the human-only acknowledgment contract in this plan.

    Trust must bind the reviewed digest to a host-resolved project identity;

    reject stale approvals and all agent/monitor acknowledgment attempts. Add

    atomicity, identity, failure and CLI tests. Document the trust commands and

    store. This phase does not yet change runtime command loading.

    '
- id: hook-enforcement
  title: Enforce trust for merged config and file hooks
  size: medium
  depends_on:
  - trust-approval
  description: 'hook-enforcement: Create the mandatory local_config_command_trust
    sunset

    flag using sase flag new, then filter local executable contributions before

    merging and before the independent file-hook loader consumes raw layers.

    Wire fresh trust checks into before/after commit hooks, explicit-project

    rebase gates and file-hook execution. Keep raw inspection and feature-flag

    bootstrap separate from runtime filtering. Preserve user/overlay commands,

    non-command local values and the explicit flag-Off compatibility branch.

    Add both-state and stale-cache/end-to-end no-execution regressions.

    '
- id: macro-enforcement
  title: Enforce trust for local macro workflows and finish integration
  size: medium
  depends_on:
  - hook-enforcement
  description: 'macro-enforcement: Apply the shared policy to config macro sources
    and the

    direct load_project_local_macros path, preserve provenance through parsing

    and namespacing, and revalidate executable local definitions before their

    bash/python steps run. Cover nested steps, references, inherited executable

    definitions, revocation and content changes. Finish runtime diagnostics,

    migration docs and the combined acceptance matrix. Run the required checks

    and leave completion evidence for this child epic''s land agent; do not close

    sase-1sb.2 or any ancestor from a child phase.'
proposed_by: bbugyi200.athena.sase-1sb.2
proposed_with: codex/gpt-6-astra
create_time: 2026-11-11 20:34:34
status: done
bead_id: sase-1sb.2.1
---

- **PROMPT:** [prompts/202611/local_config_digest_trust.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/local_config_digest_trust.md)
- **PARENT:** [202611/security_bug_audit_fixes.md](https://github.com/sase-org/sase--plans/blob/main/202611/security_bug_audit_fixes.md)
- **BEAD:** [sase-1sb.2.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1sb/sase-1sb.2.1.md)

# Digest-pinned trust for project-local commands

## Scope and accepted decisions

This is a child epic of `sase-1sb.2`, implementing the final
`local_config_commands = digest_trust` decision from
`plan:202611/security_bug_audit_fixes.md`. Do not revisit that choice. The parent also
chose prompt substitution provenance and automatic bash interpolation quoting; those
belong to sibling phases, not this implementation.

The assigned phase is in progress and has no recorded earlier implementation or
remaining-work notes. Inspection found no existing local-config trust policy in the
current loaders. Four medium implementation units are appropriate: shared policy,
durable human acknowledgment, hooks, and macros each have their own integration and
failure tests. Dependencies are deliberately serial because each unit consumes the
preceding unit's contract and the enforcement phases share config loading.

This plan edits no SASE memory notes. It changes `sase` and the linked `sase-core`
repository. Open the latter through `/sase_repo` and follow its `AGENTS.md`; never
reimplement shared policy or durable-state behavior in Python. Host finalizers own
commits. New Python bindings must be covered by `sase-core-revision.txt`: when both
repositories change, the host's linked-repo finalizer writes the pushed core SHA; when
consuming already-landed bindings, use the documented revision ratchet.

## Existing paths that must converge

The bead's original line numbers predate a config-module split. Relevant current entry
points are:

- `src/sase/config/core.py`: cached `load_merged_config()`, explicit-project
  `load_merged_config_for_local_path()` including `local_config_override`,
  `load_macros_by_source()`, and raw `load_config_layers()`.
- `src/sase/config/loading.py`: `merge_config_sources()` normalizes each layer and
  deep-merges the local contribution last.
- `src/sase/config/file_hooks.py`: `_effective_raw_file_hooks()` consumes raw layers
  independently and `_load_file_hooks()` has its own cache. Template `use:` entries can
  introduce executable commands without a literal local `command:` field.
- `src/sase/config/macro_sources.py`: loads config macros independently.
- `src/sase/macro/loader_sources.py`: `load_project_local_macros()` directly reads
  another project's config, bypasses `_include_local_config`, and adds a project
  namespace. Filtering only the ordinary merged config misses this path.
- `src/sase/workflows/commit/command_hooks.py`, `rebase_gate_settings.py`,
  `src/sase/file_hooks/runner.py`, and
  `src/sase/macro/workflow_executor_steps_script.py`: execution boundaries.
- `src/sase/feature_flags/snapshot.py` bootstraps flags from raw config layers. Its
  resolver already rejects local flag overrides; preserve that protection and avoid
  recursive flag resolution when adding trust filtering.

Use canonical `sase/sase.yml` and the existing root `sase.yml` fallback/collision rules.
Do not create a second config discovery algorithm.

## Trust contract

### Identity and persistence

Store versioned trust records under `sase_home()/local_config_trust/`, outside all
repository checkouts and config overlays. Use a fixed path helper with the existing
pytest sandbox guard; no project-config option may redirect the store. Records are
machine-local and are not synchronized into a sidecar or committed to a project.

Key records by a versioned project identity. For registered projects, derive this
identity from the host's project record and canonical primary checkout, and accept
managed checkouts only after checking host workspace-registry membership. A repo's
display name, remote URL, or `.sase/checkout.json` alone is insufficient evidence. For
an unregistered checkout, use its canonical checkout root. Ambiguous or missing
membership must fall back to checkout-scoped trust rather than inheriting another
project's permission. A linked repository has its own identity, never its parent's.
Canonicalize both sides of comparisons so normal macOS path aliases work.

An acknowledgment stores the identity, policy version, SHA-256 digest, timestamp, and
human audit identity. Keep one current digest per project; acknowledging a new digest
replaces the old one, and revoke removes it. No wildcard, trust-all-projects, permanent
directory allow, or approval inferred from VCS cleanliness is supported. The digest
includes the logical selected config location, so changing between the canonical path
and compatibility fallback requires review.

Rust owns record validation, read/modify/write semantics and decision logic. Reuse
existing locked atomic-state patterns: private directory/file permissions, unique
temporary files, atomic replace, bounded locking, and no writes through symlinked record
files. A missing, malformed, unsupported, unreadable or unsafe store is untrusted with a
diagnostic. Reading status does not create or repair the store. Concurrent allow/revoke
operations must not lose unrelated project records.

This restores SASE's application-level boundary for repo-file edits and supported agent
commands. It does not claim an OS sandbox against a same-user process that can rewrite
SASE's own binaries or private host state.

### What is acknowledged

Add a pure Rust policy in the existing config domain, with additive, versioned wire
records and PyO3 exports. The Python adapter supplies decoded local data, lower-layer
context when needed, resolved identity and trust facts. Rust returns the normalized
command-bearing projection, digest, effective local contribution, blocked paths and a
structured status/reason. Do not distribute separate key deny-lists among loaders.

Normalize legacy macro spellings before classification and hashing, using the existing
Rust macro-normalization contract. Canonical serialization sorts mapping keys
recursively, preserves list order and scalar types, and uses a domain/version prefix
before SHA-256. YAML whitespace, comments and mapping order alone should not invalidate
acknowledgment.

The projection includes:

- The complete local `commit_hooks` contribution, including auxiliary values that
  influence execution. Remove that whole local contribution when untrusted, so it cannot
  override or disable trusted user hooks with malformed/null values.
- Complete local file-hook entries, including templates (`use:`), inputs, filters,
  enablement, working-directory and timeout fields. Removing only `command` permits
  template bypasses or leaves misleading half-valid hooks; omit blocked entries.
- Complete config-defined executable macro/workflow definitions, not just bash or python
  source strings. Include inputs/defaults, nested/parallel/conditional steps, embedded
  definitions, referenced executable step declarations and other fields that affect
  execution. A local sparse override of a lower-layer executable definition is
  command-bearing even when it supplies only inputs or metadata. Conservative
  classification of a structured workflow reference is appropriate when its safety
  cannot be established without running it. Keep recognized prompt-only definitions
  available; never execute code to classify a definition.

Include every local contribution that changes the executable definition after
composition, so adding a parameter cannot reuse approval for a different command. Reject
or conservatively block ambiguous executable shapes. Ordinary local config such as
project display values and aliases must still merge; edits solely to those values must
not change the executable projection's digest. Preserve lower-layer values and existing
list precedence after removing local blocked contributions.

The digest covers configuration, not the transitive contents of arbitrary scripts,
binaries or standalone macro files it invokes. Document that limit explicitly.
Standalone repo macro files, other command-bearing subsystems such as service/tool
catalogs, and prompt substitution are outside this phase's enumerated trust scope;
record concrete newly discovered gaps as `PROPOSED FOLLOW-UP:` notes, not new beads.

### Human acknowledgment UX

Implement these sorted CLI subcommands under `sase config trust`:

```text
sase config trust allow [PATH]
sase config trust revoke [PATH]
sase config trust status [PATH] [-j|--json]
```

`PATH` defaults to cwd and selects a checkout or its config, with the same identity and
path resolution as runtime loading. Bare `trust` prints help. `status` is read only and
reports project identity, selected config path, current/acknowledged digest, policy
status, blocked key paths and any reason for refusal. JSON is suitable for tests and
future frontends. It must remain usable when runtime commands are blocked.

`allow` shows the exact canonical command-bearing projection, project identity and
digest, then requires an explicit confirmation from a human terminal. Render text
literally, escaping terminal controls; never expand macros, run hooks or evaluate
templates while previewing. There is no `--yes`, stdin answer-file, caller-supplied
`human` bit, or environment bypass. Reject agents and monitors using the shared
`agent_marker_present()`/identity-discovery contract, fail closed on discovery errors,
and require a controlling terminal. Apply this guard at the mutation adapter as well as
the argparse handler, so alternate public entry points cannot skip it. Automated plan
approval is not local-command acknowledgment.

After confirmation, reread identity, config and projection; if the digest or selected
path changed, refuse with “configuration changed; review again” and write nothing. Only
the reviewed snapshot can be acknowledged. An already-current acknowledgment is an
idempotent no-op. Revoke uses the same human-only mutation lane and immediately
invalidates existing readers. `allow`/`revoke` return nonzero on refusal or storage
failure; status distinguishes untrusted from operational errors in its JSON.

This CLI-only acknowledgment keeps the implementation bounded. No new notification gate
kind or TUI interaction is needed. Human-only here follows SASE's existing
caller-provenance contract; terminal availability alone is never proof of authorship.

### Runtime filtering, freshness and diagnostics

Keep `load_config_layers()` raw for provenance inspection and flag bootstrap. Introduce
an explicit effective-local adapter and use it before all executable consumers,
including the independent file-hook and direct project-macro readers. Config inventory
should distinguish authored values from blocked effective values; the user must be able
to see why a hook is absent without granting it permission.

Trust and config changes are security inputs, not eventual display-cache inputs.
Evaluate the command-bearing snapshot synchronously whenever loading executable config.
Do not rely solely on `current_config_token()`'s asynchronous stat refresh, mtime, file
size, or a prior file-hook cache entry. Cache parsing if useful, but key execution
eligibility by actual projection digest and current acknowledgment state. Use the same
decoded snapshot for digesting and filtering; do not hash one read and execute a second
unchecked read. `local_config_override` must be checked against its supplied candidate
data, not trusted merely because the on-disk file was approved.

Carry a small trust descriptor with local executable definitions (identity, path,
digest/policy version). Before dispatch, confirm that the descriptor still matches the
current acknowledged snapshot. This matters for a rebase retry holding settings, a
queued file-hook plan and a parsed workflow retained in memory. Changed config or
revocation blocks further local execution until acknowledged again. A command already
started is not terminated. Never relabel a retained local command as user-origin.

Before/after commit hooks must select configuration for their explicit `cwd` repo, as
the rebase gate already does, rather than borrowing a trusted ambient cwd. Host
finalizer subprocesses must independently resolve trust; an agent-supplied boolean or
prefiltered mapping is not authority.

When untrusted, omit local automatic hooks and continue with trusted user/overlay hooks
and ordinary non-command settings. Surface a warning that local commands were skipped,
identifying config path and `sase config trust allow PATH`. For an explicitly requested
blocked macro, give a specific trust error rather than a vague unknown-macro message or
silently executing a lower-priority namesake. Ensure blocked rebase hooks are shown as
skipped, never reported as successful verification. Deduplicate warnings by
project/digest/reason in a process while retaining structured diagnostics for each
relevant operation. Loading config never prompts or starts a gate.

## Migration flag and staged implementation

The hook-enforcement phase must first read `sase_flags.md` and create
`local_config_command_trust` with `sase flag new`, kind `sunset`. This is the required
flag lifecycle operation expressly called for by the assigned phase, not permission to
create unrelated task beads. Paste the generated registry entry; never hand-author a
flag record or removal bead.

- Enabled (default): local command-bearing contributions require current digest trust.
- Disabled: retain today's local-command behavior, visibly marked as compatibility mode
  in status. Do not create trust records or imply the content was acknowledged.
- Remove when users have migrated local command configuration and acknowledgment,
  revocation and refresh behavior are established; removal deletes the Off branch.

The existing resolver's refusal of local feature-flag overrides remains mandatory. Both
states use the same discovery and merge order. Avoid a new permanent config switch. If
implementation adds a real config field, update `default_config.yml` and schema, but the
proposed trust store and flag need no such field.

Phases one and two expose additive policy/management APIs without changing command
execution. Phase three activates hook protection with the sunset flag; phase four
extends the same flag to all in-scope macro paths. Do not declare the assigned phase
finished while any of these paths retains an unguarded enabled branch.

## Verification and documentation

Each phase adds behavioral tests with temporary config/state, without touching actual
user trust or executing project scripts. Use sentinel commands or mocked spawn points to
prove absence of execution; asserting only that a key disappeared is insufficient for
final integration coverage. Preserve existing loader patch points where practical.

Required matrix:

1. Flag On/Off; no acknowledgment, current acknowledgment, changed digest, revoked
   acknowledgment; unchanged non-command config merges in every case.
2. Before/after commit hooks, after-rebase, literal and template file hooks, ordinary
   and namespaced local macro loaders, bash and python execution, nested workflows,
   partial overrides of lower-layer executable definitions and prompt-only macros.
3. User/plugin/selected-overlay hooks remain available; non-authoritative local flag
   overrides do not disable trust; raw-layer flag bootstrap never recurses.
4. Canonical and fallback config paths, explicit-project loads from another cwd,
   candidate overrides, registered managed checkouts, unrelated repos with identical
   text, unregistered roots and spoofed checkout markers.
5. Same-size/mtime-preserving config edits, long-lived caches, queued commands and
   workflows, allow/revoke in another process, and changes between preview and allow.
6. Missing/corrupt/unsupported/unreadable store, symlink records, interrupted atomic
   writes and concurrent different-project updates. All fail closed for local execution,
   preserving safe config and actionable diagnostics.
7. Human confirmation succeeds; agent env/name, metadata-only identity, monitor
   identity, discovery failure, no terminal, declined confirmation and direct adapter
   calls cannot acknowledge. No hook or macro runs during preview/status/revoke.

Rust tests own policy, serialization and store concurrency. PyO3 round-trip tests prove
exported names and wire shape. Python tests exercise actual loader/runner glue; extend
`test_config_local.py`, `test_config_layers.py`, file-hook, macro-loader, commit-hook
and rebase tests beside the new focused trust suites. Account for existing fixtures that
intentionally execute local config by explicitly acknowledging their temporary snapshots
or selecting the compatibility branch, never by globally disabling the default
protection in conftest.

Read `lint_and_test.md` before finishing any implementation phase. Run `just fix` and
the required `just check` through `sase tool run check`; use `/sase_monitor` for long
builds/checks and wait for staging to exit. Core changes also need the linked repo's
full `sase tool run check`, beyond targeted `just test` runs. Do not run bare cargo,
globally install SASE, or substitute `just check-full` for the agent check.

Read `docs_writing.md` and `docs/contribute/writing-docs.md` before documentation edits.
Update configuration file/merge behavior in
`docs/reference/config/files-and-merging.md`, migration details in
`docs/reference/upgrading/configuration.md`, and regenerate CLI/flag references with
`just fix`. Describe the review command, digest inputs, checkout identity, skip
diagnostics, compatibility flag and referenced-script limit. Keep within page budgets;
use a concise adjacent Reference page if needed. No memory changes are authorized.

## Completion ownership

Every child phase closes only its own assigned bead after resolving its own
`sase bead epic-symbols` entries. Put newly discovered work in `PROPOSED FOLLOW-UP:`
notes on the current phase. A baseline-identical check failure is evidence for such a
note, not a reason to leave a finished phase open.

This child epic's land agent integrates all four phases, verifies that the current core
revision contains the bindings, reruns the combined required checks, and records
evidence for the full acceptance matrix. It resolves or re-keys any remaining
`--epic-symbol` entries before closing. Once its descendants and its own epic are
settled, that land agent completes and closes the delegated phase `sase-1sb.2` with a
verification note, as authorized by this child-epic delegation. It must not close
`sase-1sb` or another ancestor of the delegated phase. Until then, `sase-1sb.2` remains
in progress; a plan proposal alone is not implementation completion.
