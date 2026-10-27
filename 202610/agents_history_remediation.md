---
tier: epic
title: Rotate exposed credentials and purge agents-sidecar history
goal: Resolve the historical exposure tracked by sase-1ou.5 with private triage, owner-confirmed
  credential rotation, a rehearsed and explicitly approved purge, and evidence that
  all three hosts resume publication without restoring old history.
parent_bead: sase-1ou.5
phases:
- id: private-triage
  title: Classify findings and prepare the rotation checklist
  depends_on: []
  size: medium
  description: 'private-triage: account for every original finding and reachable historical
    variant without exposing values, prepare private purge inputs and a human rotation
    checklist, and preserve value-free evidence across credential rotation.'
- id: fleet-protocol
  title: Prepare and test the three-host maintenance procedure
  depends_on: []
  size: medium
  description: 'fleet-protocol: inventory every agents-sidecar writer and clone on
    athena, apollo and mac, implement a private command bundle for quiescence and
    clone recovery, and verify it on disposable fixtures without production mutations.'
- id: purge-rehearsal
  title: Rehearse the purge and assemble the execution package
  depends_on:
  - private-triage
  - fleet-protocol
  size: medium
  description: 'purge-rehearsal: bind the private removal manifest to frozen refs,
    rehearse the history transformation and fleet recovery on isolated copies, and
    prepare exact commands, count comparisons and stop conditions for human review.'
- id: review-execute
  title: Gate execution and verify the published result
  depends_on:
  - purge-rehearsal
  size: medium
  description: 'review-execute: obtain rotation confirmation and explicit approval
    of the concrete purge, execute through a durable gate turn, verify rewritten remote
    refs and every host, and record either completion or the owner''s deferral.'
proposed_by: bbugyi200.athena.sase-1ou.5
proposed_with: codex/gpt-6-astra
create_time: 2026-10-27 09:36:42
status: wip
bead_id: sase-1ou.5.1
---

- **PROMPT:** [prompts/202610/agents_history_remediation.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/agents_history_remediation.md)
- **PARENT:** [202610/backlog_epic_2_top_five_task_beads.md](https://github.com/sase-org/sase--plans/blob/main/202610/backlog_epic_2_top_five_task_beads.md)
- **BEAD:** [sase-1ou.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ou/sase-1ou.5.1.md)

# Plan: Complete the historical agents-sidecar remediation

## Scope and observed state

This is a child epic for phase `sase-1ou.5`, derived from
`plan:202610/backlog_epic_2_top_five_task_beads.md`, phase `agents-secret-purge`. The
source task is `sase-1ot`. The assigned phase had no earlier work notes when read on
2026-10-27. This needs separate bounded implementation units: evidence and owner
classification, fleet recovery commands, a real rehearsal, and reviewed execution. A
plan approval is not approval to force-push or reset production clones.

At planning time the sase checkout was `13df005a9840645bb999ba561389d479ad8153ed`. An
audited `sase repo open agents` resolved the shared machine publication clone; its
observed tip was `cdf4e8cf60`. The original audited tip
`5066d26a4b234769f48be023e92038d9d9727c0f` remained its ancestor. The old exposure has
therefore not been removed from that clone's history. These are observations, not an
execution-time remote snapshot: refresh all identities before acting.

The original private attachments are on `sase-1oh.5`:

- `audit-report.md`, SHA-256
  `53a690592ee5dd1488af5287bc4be8af05a491b20f15e9be8ceb9e9259a766a6`;
- `rollout-gate-material.md`, SHA-256
  `e20f23a3cd69be332b03da0cf767281c81edd8207f0be29e0e891e7cb7ff978f`.

Both attachment checksums were verified. Fetch their private views using
`sase bead attachment path sase-1oh.5 <name>`; never publish their contents in plan
prose, terminal summaries, or public notes. Read the parent plan through
`sase artifact read`, and beads through `sase bead read ... -r "<reason>"`.

The report scanned **one tree**, not all history: 138,525 tracked files, 138,497 clean,
27 hits, one skipped, zero errors, rules v1, 16,777,216-byte limit. A hit counts a file,
not a distinct credential; the scanner returns only its first finding. The report alone
cannot prove that any credential is live or that every historical copy has been found.
Do not describe all 27 as confirmed credentials.

Gate `custom-57bc7a62-1331-4a3b-900d-a1b2d4612e5d` approved the intended scope `both`
(rotation and purge), but its recorded result explicitly performed neither.
New-publication protection and agents-sidecar push protection landed through
`sase-1oh.5.1`; verify the installed versions and current settings before relying on
them. The rollout attachment's earlier disabled-settings observations are stale.

The inherited `dormant_flakes = close` decision is final and irrelevant to this work.
There are no new product or memory decisions. Do not edit `sase/memory/**`.

## Boundaries for every phase

- Never print, commit, attach, put in argv, or send to an LLM a matched credential
  value, raw transcript excerpt, raw environment, or raw push diagnostic. Use an
  allowlist of metadata fields in reports and subprocess summaries. Private reports
  contain paths, rule IDs, blob IDs, service labels and verdicts only.
- Rotation belongs to Bryan or the identified service owner. Do not authenticate with an
  exposed credential to test it. An unknown verdict stays unresolved until the owner
  classifies or rotates it; lack of evidence is not a false-positive verdict.
- Read `sase_beads`, `sase_artifacts` and relevant `tailnet` memory with
  `/sase_memory_read`. Use `/sase_repo` for every repository, including remote-host
  inventories, and use only its resolved paths. Never guess primary clone paths. The
  `agents` role is machine-shared across numbered checkouts; it is not scratch.
- Keep detailed work products and execution resources owner-private. Attach them with
  `sase bead attach -K <own-phase> <files...>` after checking CLI help and the private
  store's placement. Public note text contains only aggregates and attachment
  identities. Each phase records the immutable attachment IDs needed by successors;
  successors must not depend on temporary paths from a prior run.
- Prefer one-operation scripts packaged as reviewed gate command resources, using
  existing scanner, protection and queue APIs. Do not create a general migration
  framework or duplicate Rust policy in Python. Any necessary shared behavior change
  belongs in `sase-core`, with bindings, tests and the sase revision pin.
- No production rewrite, force-push, clone reset, repository-setting change or service
  stop happens before the concrete `/sase_gate` approval. Do not stop publishers while
  waiting for the owner. The explicitly requested local rehearsal operates only on an
  isolated disposable copy, with production push disabled.
- No broad deployment or global install. If a deployment is needed, include exact
  commands in a separate `/sase_gate` action. Use `/sase_sudo` if privilege is needed.
- Use `/sase_monitor` before long audits, rehearsals or verification commands; preserve
  machine-readable private progress and resume through its successor. Gates use
  `/sase_gate` turn handoff with `auto: false`, durable resources and explicit
  follow-ups. Never end a normal turn with a background command running.
- Every worker closes only its assigned child phase, after
  `sase bead epic-symbols <own-id>` has no entries. Resolve or re-key leftovers to an
  open owner. Do not close `sase-1ou`, `sase-1ou.5`, or `sase-1ot` from a child phase.
  The child lander owns the delegated phase closeout described below.
- Never create task beads. Append `PROPOSED FOLLOW-UP: <summary — evidence>` to the
  worker's phase. Keep the existing `sase-1or` and `sase-1os` defect ownership.
- For tracked code changes, read `lint_and_test`, run focused tests and
  `sase tool run check` in each changed repository. Do not run `check-full`. Read the
  appropriate memory before editing `tools/`, `docs/` or adding CLI commands. A failure
  identical on a clean base gets a documented follow-up, with its existing owning bead,
  and does not prevent phase closure.

## Phase private-triage

Deliver a private, reproducible finding inventory and rotation checklist. Do not execute
a rotation or create a human gate in this phase; the final phase brings one complete
execution package to the owner.

1. Re-read `sase-1ot` and the original private report. Assign stable opaque finding IDs
   covering all 27 rows, and associate replicas by blob ID without claiming that
   distinct files mean distinct credentials. Record each rule kind (known-value,
   credential-pattern, environment dump), scope and original blob ID.
2. Build a bounded private audit driver around the existing
   `require_rust_binding("attachment_scan_file")` API. See
   `src/sase/agents_sync/publication_protection.py::_scan_file` and `sase-core`'s
   `crates/sase_core/src/note_attachment/scanner.rs`. Preserve the scanner's size limit
   and fail-closed treatment of skipped/error results. Record version, enumerated
   objects, processed objects and outcomes; never emit source bytes even on exceptions.
   Test with synthetic fixtures.
3. Snapshot advertised refs and enumerate reachable unique blobs, including alternate
   refs, moved paths, prompt copies and captured objects. Scan a stable snapshot, not a
   live worktree during publication. Account separately for the original tree, the
   current tip and historical objects. Track commit/tag metadata, LFS and
   non-text/oversized objects as explicit coverage gaps requiring an identified
   resolution; do not quietly label them clean. Use the original report's known-bad
   path/blob coverage even if a later scanner returns clean.
4. Have a local metadata-only classifier or the owner supply evidence for each verdict:
   confirmed credential, test/placeholder, false positive, or unresolved. Do not open
   raw chats in the agent transcript for classification. A first-hit rule cannot rule
   out other credentials in the same file. Treat uncertain files conservatively in the
   purge candidate list, and flag any uncertain service ownership for the owner's
   checklist.
5. Capture the removal manifest before asking the owner to rotate. The scanner's
   known-value set comes from its environment and configured secret files; after
   rotation, a clean scan alone cannot prove old bytes are gone. Record blob IDs and
   affected paths/versions so later verification is independent of knowing the old
   value. Do not save old credentials in a matcher or replacement file.
6. Prepare `triage-report.md`, a machine-readable removal manifest, and
   `rotation-checklist.md` as private attachments. The checklist names service, owner,
   affected finding IDs, required revoke/rotate action, dependent host configurations
   that the owner must update, and an attestation field. It must separate
   already-invalid/test values from live or unresolved credentials.

Acceptance: all original rows have evidence or explicit unresolved owner questions;
every discovered historical candidate has a disposition; counts reconcile; verification
can still detect the inventoried bad blobs after credentials change. Unknowns are
permitted as explicit inputs to the final review, not as a completed security claim.
Attach the driver and its synthetic-test results for reuse.

## Phase fleet-protocol

Deliver an exact, tested maintenance command bundle and read-only inventory. This phase
may run alongside private-triage because it needs no finding values.

1. Read `tailnet` memory. Inventory athena and apollo as `bryan`, and `mac` as `bbugyi`,
   using the configured SSH aliases. Record accessibility, OS, installed sase/core
   versions, service definitions and the agents remote identity. On each host use the
   SASE repository inventory to enumerate hidden publication, primary, worktree and
   other managed clones, canonicalizing shared git dirs. Identify local-only commits and
   dirty work without showing content.
2. Trace every writer, not just the scheduled sync job: full sync, commit drains,
   direct/deferred prompt publication, migration, referenced-by/link publication,
   workspace preparation and generic sidecar publish/rescue. Useful entry points:
   `agents_sync/git_sync.py`, `_commit_publication_drain.py`,
   `prompt_archive/publish.py`, `prompt_archive/migration.py`,
   `referenced_by_publication.py`, and `axe/runner_workspace_protection.py`. Identify
   scheduler/service and manual entry points in the installed version.
3. Specify the narrow maintenance barrier and implement its command resources. Stop
   every discovered automatic writer through its actual service control, prevent manual
   writes during the window, wait for in-flight writes and take the existing publication
   locks in a fixed order. Merely stopping the scheduler or holding one git lock is not
   sufficient unless all writers were proved covered. Locks inside `.git` need special
   care if the clone is replaced: preserve the barrier across inode/path changes and
   restarts. A worker must never kill the gate or monitor process that owns this
   maintenance transaction.
4. Prepare clone recovery against the verified new remote refs. Prefer clean
   reprovisioning through supported repo lifecycle paths; an in-place reset must also
   deal explicitly with stale local branches, tags, remote refs and reflogs. Never merge
   or rebase contaminated history back onto the rewritten remote. Preserve local-only
   work privately before destructive changes and rescan any proposed recovery. Do not
   delete a human's untracked work to simplify recovery.
5. Describe exact queue/ledger treatment using existing APIs:
   `agents-publication-protection.json`, `agents-publication-outbox.json`, referenced-by
   requests, deferred prompts, pending captured objects and publication
   revision/completion state. Preserve security blocks and unrelated pending work. Do
   not clear ledgers, blanket-revive stopped rows, or change primary source-repository
   revision IDs when only sidecar commits changed. Build a ledger/manifest mapping for
   content that could be regenerated after rotation; new protection scans must not be
   the sole protection for old values.
6. Re-read `sase-1or` and `sase-1os`. Until their fixes are verified on each host, keep
   prompt migration outside the resumed writer set and use a bounded clean canary drain
   with a stop-on-GH013 policy. Do not retry an unattributed rejection repeatedly or
   bypass push protection. Any additional replay source must remain disabled or be
   covered by a reviewed containment step.
7. Test the maintenance commands with temporary repositories and fake host/service
   runners: one inaccessible host, changed remote refs, dirty clone, local-only objects,
   lock contention, partial host reset, repeated execution and command failure. Assert
   that production mutation cannot start with incomplete preflight and that post-push
   failures keep unsafe writers paused. Never use real secrets.

Acceptance: attach `fleet-inventory.md`, `maintenance-runbook.md`, the executable
command bundle and test evidence privately. All hosts must have a named reset and resume
strategy. An offline Mac is a blocker to execution until it reconnects or the owner
approves and the bundle verifies a durable isolation mechanism that prevents its stale
clone from publishing. A timeout is not proof of isolation. No real services or clones
are changed in this phase.

## Phase purge-rehearsal

Deliver an immutable execution package with precise impact and executable checks.

1. Consume both predecessor attachment sets. Refresh refs/settings read-only and
   preserve the full old ref-to-SHA map privately. The target is only
   `sase-org/sase--agents`; do not rewrite other sidecars or the source repository.
   Verify no producer is relying on an unsupported scanner build.
2. Default to removing inventoried affected file paths through their full history,
   including previous names, plus identified duplicate blobs at other paths. This avoids
   writing secret-bearing replacement rules. Preserve unrelated content. Explain every
   deletion, dependency and broken artifact link in private review material;
   whole-repository history truncation is outside this plan.
3. Resolve an isolated disposable copy through the repository skill/workflow. If that
   workflow resolves to the machine-shared clone, stop and obtain a supported isolated
   workspace; never run filtering there. Do not discover or clone repositories outside
   `/sase_repo`. Use an offline rehearsal with pushes disabled. Pin and verify the
   filtering tool/version; author a manifest-based `git-filter-repo` invocation without
   values. The parent explicitly requires this disposable local rehearsal before
   requesting production execution.
4. Compare the old and transformed trees by path, mode, type and blob identity:
   unaffected entries must match; every removal must be in the approved candidate set.
   Record changed refs/commits, file/object counts, first changed commits, affected PR
   refs and any LFS caveats. Verify removal throughout the rewritten reachable object
   graph, not only the tip. Run the same private audit driver over both the transformed
   tip and the relevant reachable history.
5. Rehearse the fleet protocol against fixture clones of the old and transformed graphs,
   with old queued requests and protected objects. Prove clean content publishes,
   affected bytes remain withheld, and no old history is merged back. Verify crashes
   before push, after push and midway through clone recovery.
6. Bind commands to the removal-manifest hash, tooling versions, reviewed ref map,
   expected new ref map and fleet inventory. Production execution must recheck these
   immediately after quiescence. Use explicit ref targets and expected-old SHA leases,
   or an equally strong guarded operation; no unguarded mirror push. New refs,
   unexplained content changes, stale inventory or newly unresolved findings stop
   execution and require an updated rehearsal/review. Never silently widen a previously
   approved purge.
7. Package commands for preflight, quiesce, backup verification, transformation, ref
   publication, per-host recovery, audit and bounded resume. Commands return
   schema-validated JSON metadata with sanitized diagnostics. Each mutation is scoped,
   idempotent where possible, and journaled privately. On failure before publishing,
   verified unchanged clones may resume; after publishing, use forward recovery with
   unsafe writers paused. Never roll back by republishing bad history.

Acceptance: private `purge-rehearsal.md`, manifests and command bundle identify the
exact reviewed operation and pass fixture/rehearsal checks. A clean report must
explicitly account for skips, errors and unresolved findings. No production history,
service or setting has changed.

GitHub's current guidance calls for rotation first, coordinated clone cleanup, and
special treatment for cached views, forks and PR refs; filtering changes commit IDs and
can invalidate signatures. Include a support-request draft and the relevant metadata for
Bryan, with the limits of what the rewrite proves. Support assistance is conditional,
and other users' copies cannot be erased by this operation. Do not send support requests
or messages on the owner's behalf. See
[GitHub's sensitive-data removal guidance](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository).

## Phase review-execute

1. Re-read the predecessor evidence and validate that every required command is
   concrete, reviewed and ready. Use `/sase_gate` with `auto: false` and a durable turn
   continuation. The gate must show the affected repository, all hosts, deletion counts,
   ref changes, preservation strategy and stop/recovery behavior. Detailed manifests
   remain private; do not inline finding paths or values into the public notification.
   Give the owner access to the private attachment views.
2. Collect owner attestations for every live or unresolved checklist entry: the service,
   rotation/revocation completed, dependent configurations updated, and resolution for
   any non-credential finding. Never ask for the new or old value. The execution branch
   requires all applicable attestations; selecting a button alone is not proof of
   rotation. Separate a rotation-only outcome if the owner needs time, with a durable
   follow-up that preserves the pending purge.
3. Approval runs the exact bundle, not an open-ended shell. Establish the barrier on
   every host before the guarded push. Validate all refs and the manifest again; changes
   outside the approved operation fail closed. Re-audit before publishing and after
   fetching the rewritten remote. Restore repository protections if an explicitly
   approved temporary setting change was required.
4. Repair every inventoried clone under the maintenance barrier. Verify no managed ref
   reaches the removed blobs; preserve/quarantine backups without push access. Verify
   ledger and pending-object consistency, then resume a bounded clean publication canary
   on each eligible host. Compare each resulting sidecar update against the removed blob
   set and audit it. Resume automatic publication only when the canary and protection
   checks pass. Record any deliberately disabled migration entry point and its existing
   owning bead.
5. Produce a private post-purge report and a public aggregate closeout: actual gate IDs,
   old/new tip identities, tree and reachable-history coverage, clean/hit/ skipped/error
   counts, unresolved count, confirmed rotations and per-host status. Report remaining
   documented test/placeholder findings separately. Zero live hits with unresolved
   candidates or unexplained skipped objects is insufficient.
6. Define explicit successor policies for approve, rotation-only, defer, reject,
   timeout, stopped and failed. Defer/reject only counts as an owner deferral when the
   owner actually chooses it. Record that answer and a `PROPOSED FOLLOW-UP:` on the
   assigned phase; do not claim completion of the purge. A technical failure, timeout or
   unreachable host is not a deferral: continue via the durable gate/monitor workflow
   and leave security work pending.

Acceptance for success: owner attestation covers every applicable finding, approved
remote refs are verified, affected history is unreachable from managed refs, all hosts
are reset or explicitly isolated, protection state is preserved, and clean publication
resumes without replay. Acceptance for owner decline/defer: the answer is recorded, no
unsupported success is claimed, and the remaining remediation is explicit. Close only
this child phase after symbol cleanup.

## Child epic landing and delegated phase closeout

The child lander verifies all child phase evidence and gate results; it does not repeat
an already successful purge. Recheck current remote identities and the post-resume
audit, not just rehearsal output. Resolve any remaining technical failure through a
concrete gate or monitor continuation, rather than leaving a phase open expecting an
automatic relaunch.

On success, attach the final private reports to `sase-1ou.5` and record aggregate
completion evidence. On an explicit owner decline/defer, carry that answer and a
`PROPOSED FOLLOW-UP:` to `sase-1ou.5`; leave `sase-1ot` open, as the outer plan
requires. Prepare evidence for the outer `sase-1ou` lander to close `sase-1ot` on the
success path; this child plan does not authorize phase workers to close it.

After all descendants are closed and symbol checks are clear, the child epic's land
agent closes its own epic and then the delegated phase `sase-1ou.5`, citing the verified
success or the owner's explicit deferral. This is the delegation requested by the user.
It must not close outer epic `sase-1ou` or any other ancestor. Child phase workers
supply evidence only; they never perform this step.

Run `sase bead epic-symbols sase-1ou.5` immediately before its close. Any entries must
be resolved or re-keyed to a still-open owner first. Use
`sase bead close sase-1ou.5 --note "<verified evidence and actual gate outcome>"`. Use
`/sase_final` as the last action before any normal turn-ending response; successful
plan, gate and monitor handoffs end through their mechanical workflow.
