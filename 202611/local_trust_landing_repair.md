---
tier: tale
title: Repair trust snapshot propagation and finish sase-1sb.2.1
goal:
  Local commands retain their reviewed snapshot through execution, revocation blocks
  every in-scope path, and the trust epic and delegated parent phase close with
  verification.
size: medium
proposed_by: bbugyi200.athena.sase-1sb.2.1.land
proposed_with: codex/gpt-6-astra
bead: sase-1sb.2.1
create_time: 2026-11-12 05:33:28
status: done
---

- **PARENT:**
  [202611/local_config_digest_trust.md](https://github.com/sase-org/sase--plans/blob/main/202611/local_config_digest_trust.md)
- **BEAD:**
  [sase-1sb.2.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1sb/sase-1sb.2.1.md)

# Repair trust snapshot propagation and finish sase-1sb.2.1

Complete the remaining work in `sase-1sb.2.1`, then close its delegated parent phase
`sase-1sb.2`. This is one bounded integration repair: the Rust trust policy, store,
human acknowledgment CLI, and initial consumers already exist. Keep the approved
`local_config_commands = digest_trust` decision. Preserve the sunset flag's current Off
behavior, ordinary local values, and user/plugin/overlay commands. Do not add a second
trust policy, store, approval UI, or permanent switch. No memory changes.

Read `sase bead read sase-1sb.2.1 -r "Need landing audit and triage outcomes"`, all four
child phases and their notes, and
`sase artifact read plan:202611/local_config_digest_trust.md "Need the approved trust contract"`.
The landing audit below supplies the remaining scope; do not redo completed policy or
store implementation. This tale owns implementation, verification, and the entire
closeout. There is no returning land agent after the tale.

## Verified starting point and remaining defects

The lander reviewed every child note, all five sase implementation commits `2e7ce5cea1`,
`181b213e8c`, `7710dd0bc6`, `021cd73021`, and `ea4121387b`, and core commits `e7647bf7`
and `233cb41d`. The core pin is `233cb41d6bb7186a711e75999394570e9ad9943e`, containing
the policy and store bindings. The four focused trust suites passed together: 51 tests.
These passes do not cover the following reproduced gaps:

1. `load_workflows_from_config()` in `macro/workflow_loader_sources.py` filters the
   config in `load_macros_by_source()`, then rereads the file to obtain a descriptor.
   Loading approved command A, replacing the file and acknowledgment with B between
   those reads, yields a workflow containing A with B's descriptor; the dispatch guard
   accepts it.
2. `load_after_rebase_gate_settings()` reads the merged command, then reevaluates trust.
   Revoke between those reads and it returns the old local command with `trust=None` and
   no warning. `run_after_rebase_gate()` reaches `execute_gate` after revocation.
3. File-hook models retain `source_layer` but not their original trust snapshot.
   `_batch_payload()` attaches a new ambient-cwd descriptor at dispatch. The runner's
   `_local_run_blocked()` also accepts an explicitly local run with no descriptor. The
   latter is reproduced on an unacknowledged local project. Loader evaluation exceptions
   return `None`, which also permits raw local hooks through the loader.
4. `validate_trust_descriptor()` returns True for a retained `enabled=False` without
   checking current policy. A local workflow loaded while Off passes the dispatch guard
   after switching On, with no acknowledgment.
5. Embedded execution drops the child's trust: `_execute_embedded_workflow_steps()`
   passes child steps to script methods that check `self.workflow` (the parent).
   `EmbeddedWorkflowInfo` retains only steps/context/name. A revoked local child is
   correctly blocked directly, but its pre-step executes a temporary-file sentinel
   through a user-origin parent. Cover retained post-steps and parallel embedding too.
6. `load_merged_config()` returns its old cached mapping after acknowledgment is
   revoked; its key has no synchronous trust input. The reproduction returns the same
   object with the revoked local `commit_hooks.before` command.
7. Acknowledging a sparse local input-default override of a user executable macro still
   produces the user workflow with the user default and no local descriptor.
   `load_workflows_from_config()` classifies each layer but parses its sparse value
   without composing it with the lower executable definition. The direct project loader
   also partitions without the lower context. Complete the approved composition and
   provenance contract, not merely the untrusted filtering half.

The immutable reproduction artifact is `file:explicit:f7ff68994727a64108f4cdd2`. It
contains scripts and observed output. Read it with
`sase artifact read file:explicit:f7ff68994727a64108f4cdd2 "Need landing reproductions"`.
All probes use temporary state and harmless sentinel execution or mocked dispatch, never
the user's trust store. Convert the probes into maintained behavioral regressions.

The lander fetched origin: `origin/master` and HEAD were `ea4121387b`. The two non-child
commits since the first trust commit are `d0dc942c16` (sibling prompt provenance) and
`1e4bcb5f38` (documentation refresh). Preserve their prompt rendering, script quoting,
and explicit-repository trust behavior. Embedded workflow provenance is the integration
gap in those shared paths. Refresh the drift review on your tree.

## 1. Carry one evaluated snapshot from loading to execution

Use the existing Rust projection/filter/decision APIs and existing merge semantics. Have
config consumers receive the effective contribution and its original evaluation as one
result; never build the descriptor from a subsequent read. Keep raw
`load_config_layers()` available for inspection and flag bootstrap.

Update `config/core.py`, `config/macro_sources.py`, `config/file_hooks.py`, macro
loaders, `workflows/commit/rebase_gate_settings.py`, and file-hook dispatch as needed.
The local data actually parsed into commands and its descriptor must share identity,
selected logical path, digest, and policy version. Preserve origins through lower-layer
composition, namespacing, templates, and caches. File-hook dispatch must use the hook's
retained origin instead of ambient cwd. A second read may invalidate a retained
snapshot, never authorize its old command under a new digest.

Refresh executable eligibility synchronously even on a merged-config cache hit; cache
parsing only when actual content and current trust state permit it. Do not rely on the
asynchronous stat token or mtime/size. `_trusted_yaml_loader()` must freeze absent or
unreadable local snapshots too, rather than handing the raw loader a second chance to
read unchecked local commands after its first read returned no mapping. Preserve safe
lower-layer and ordinary local settings whenever the policy can classify them.

Make the rebase path retain provenance when trust disappears between load and retry;
never reinterpret a previously local command as user-origin. Legitimate lower-layer
hooks may still run, with the blocked-local diagnostic retained. Before/after commit
hooks must continue to use their explicit repository and the same evaluated snapshot.

## 2. Enforce current trust through all dispatch paths

Consult the current authoritative flag at dispatch. An Off snapshot is not a permanent
permission after the flag becomes On. Under On, an explicitly local definition with
missing, malformed, unavailable, or mismatched provenance fails closed. An error while
loading/evaluating local file hooks must omit those entries, not make them look
non-local. Detached runners independently validate the retained identity, current
selected config, digest/policy, and acknowledgment. Preserve the current Off branch
without creating approval records. Do not trust a payload's `enabled` boolean.

Carry the child's trust descriptor through embedded pre/post step records and every
parallel/embedded execution path. Check the definition that supplied each executable
step, including inherited steps; checking only the containing user workflow is
insufficient. Missing provenance on an explicitly local workflow also fails closed.
Retain origin across namespacing, copying, and wrapping. Keep the sibling provenance
rendering and shell/Python quoting changes intact.

Shared policy, composition decisions, or new cross-frontend wire behavior belong in
`sase-core`; open it with `/sase_repo` before reads/edits, follow its AGENTS.md, expose
bindings and tests as needed, and ratchet the pin through host finalization if changed.
Python filesystem/cache/executor glue can stay here. Do not recreate policy in a Python
shape-based fallback; a missing required Rust binding must not silently classify
ambiguous entries as safe.

## 3. Finish executable macro composition and symbol cleanup

Compose an acknowledged sparse local executable definition with its lower layers using
established config composition, keeping the local descriptor on the resulting
executable. Cover input/default and metadata-only overrides, list precedence, references
and both ordinary and namespaced project loaders. The untrusted case must still block an
explicit request rather than execute a lower-priority namesake. Prompt-only local
definitions remain available. Use existing Rust composition rather than a parallel
Python merge policy.

The lander ran `sase bead epic-symbols sase-1sb.2.1`: six exemptions remain in the
Justfile, all keyed to this epic. Their disposition is:

- `LocalCommandTrustError`, `LocalTrustDecision`, and `LocalTrustFilterResult` have only
  in-module production consumers: privatize them and update annotations/in-module uses
  and affected tests.
- `digest_local_commands` and `LocalCommandDigest` are an unused Python wrapper and its
  return type: remove them and their wrapper-only tests. Retain meaningful digest
  behavior coverage through the consumed `project_local_commands()` API.
- `local_command_trust_wire_schema_version` is an unused Python wrapper: remove it;
  retain the direct Rust-binding version assertion and capability validator coverage.

Remove the six corresponding Justfile lines and stale exports/imports. Do not delete
public Rust bindings (they are an existing wire contract), add test/Markdown pragmas, or
re-key these exemptions to the parent just to make closing succeed. If fresh source
review finds a real external consumer, verify it through `/sase_repo` and use the
Symvision policy's supported non-test pragma instead.

## 4. Verify the combined behavior

Read `lint_and_test.md` and `symvision.md`. Add deterministic regressions at real
load/dispatch boundaries, with mocked inter-read changes and sentinel commands so the
assertions prove no execution. Required new cases include all seven findings above,
file-hook snapshot A retained while B becomes current, an explicit other-project cwd,
missing descriptors and evaluation failures, Off-to-On queues, embedded pre/post and
parallel children after revoke/change, and trusted sparse composition followed by
revocation. Check lower-layer hooks still execute and non-command local settings still
merge. Cover canonical/fallback selection changes, same-size/mtime edits, and supplied
candidate overrides without a global disable in conftest.

Run the existing four trust suites plus affected config, file-hook, commit/rebase,
macro/embedded-workflow and prompt-authorship suites. Run `just fix` and
`sase tool run check` (`just check`); use `/sase_monitor` for long verification. Run the
linked core's `sase tool run check` if core changes. Do not run `just check-full`. A
pass of check plus a check-full-only failure is test infrastructure follow-up, not
remaining trust work. Keep the approved contract and concise docs accurate; read
`docs_writing.md` and `docs/contribute/writing-docs.md` before documentation edits.

The lander completed follow-up triage before proposing this tale: child `sase-1sb.2.1.3`
note #1's edit-lock report belongs to the active parent security epic `sase-1sb`, whose
phase `.5` authored the defective locking code. A deterministic empty-file publication
race admits two owners. Evidence and a DISCOVERED ISSUE note are on that parent; no new
task or quarantine/retry workaround is warranted. Include this disposition in the close
note. Do not pull lock repairs into this trust tale or close the containing security
epic.

## 5. Finish the landing in this same implementation turn

After fixes and required verification, reread this epic, all descendants/notes, and its
linked plan. Verify every approved requirement and note; record concrete test and drift
evidence. If additional unrelated follow-ups appear, use `/sase_new_task` and record
every outcome, including declined proposals. The known trust defects above are epic work
and cannot be deferred to tasks.

Run `sase bead epic-symbols sase-1sb.2.1` again. Resolve every remaining entry or re-key
only to a verified still-open later consumer bead. Then close normally:

```bash
sase bead close sase-1sb.2.1 --note "<verified fixes, accepted decisions, descendant readiness, integration/drift evidence, check result, symbol cleanup, and follow-up disposition>"
just symvision
```

Open the plans sidecar with `/sase_repo`, use the PLAN path returned by the bead read,
and set `status: done` in `202611/local_config_digest_trust.md` frontmatter. Reads of
the plan stay audited through `sase artifact read`. If closure rejects leftover symbols,
finish their cleanup and retry. Never use `--force` to advance successful nested
landing; finish or reopen an incomplete phase. Canceled/superseded closure is only
appropriate for a deliberately abandoned outcome with an explicit reason.

Read `sase bead read sase-1sb.2.1 -r "Need the parent link"` and
`sase bead read sase-1sb.2 -r "Verify the delegated parent phase is complete"`. Its
parent is phase `sase-1sb.2`. Verify this child fulfilled that phase's digest-trust
scope, then close only that phase normally with
`sase bead close sase-1sb.2 --note "<verified child completion and delegated trust scope>"`.
Leave containing epic `sase-1sb` and its plan to its already-waiting land agent. If this
phase has a new blocker, note it and leave it open.

Do not wait for this turn's own commit SHA, push, or CI result to perform closeout.
Closing in the same turn as the final code is the required path. Finish through
`/sase_final`, covering every edited repository, so the host commits the final repair
and plan-status change after the turn ends.
