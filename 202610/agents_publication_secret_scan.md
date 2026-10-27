---
tier: epic
title: Protect agents-sidecar publication from secrets and recover from GH013
goal: Withhold unsafe transcript and prompt bytes, keep clean publications moving,
  and provide private audit evidence and reviewed rollout gates.
parent_bead: sase-1oh.5
phases:
- id: protection-policy
  title: Define core publication protection and durable blocked-content state
  size: medium
  depends_on: []
  description: 'protection-policy: reuse the attachment scanner, implement Rust-owned
    publication protection and rejection classification, expose tested bindings, and
    add private durable state plus doctor reporting.'
- id: transcript-publication
  title: Withhold unsafe run files without breaking snapshots or clean runs
  size: medium
  depends_on:
  - protection-policy
  description: 'transcript-publication: enforce the protection contract in full and
    targeted hood publication, including richer-file fallback, rendering, manifests,
    and staging; prove clean runs still publish.'
- id: prompt-publication
  title: Protect prompt documents, captured objects, and deferred restoration
  size: medium
  depends_on:
  - protection-policy
  - transcript-publication
  description: 'prompt-publication: protect direct and deferred prompt archives and
    pending text objects before staging, preserve private recoverable copies, and
    keep blocked obligations out of retry and acknowledgment loops.'
- id: rejection-recovery
  title: Recover all publisher paths from GH013 without replaying blocked bytes
  size: medium
  depends_on:
  - transcript-publication
  - prompt-publication
  description: 'rejection-recovery: wire core rejection classification into all pushes,
    isolate offending publications, preserve local-only objects, reset only the managed
    publication clone, and prove unrelated publication continues.'
- id: private-audit
  title: Audit published content privately and prepare reviewed rollout actions
  size: medium
  depends_on:
  - rejection-recovery
  description: 'private-audit: scan the current agents sidecar through the core scanner,
    attach a paths-and-rule-ids report privately to sase-1oh.5, and prepare exact
    remediation and repository-setting gate material for the child land agent. Do
    not change GitHub settings or rewrite history.'
proposed_by: bbugyi200.athena.sase-1oh.5
proposed_with: codex/gpt-6-astra
create_time: 2026-10-26 14:45:01
status: done
bead_id: sase-1oh.5.1
---

- **PROMPT:** [prompts/202610/agents_publication_secret_scan.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/agents_publication_secret_scan.md)
- **PARENT:** [202610/backlog_epic_top_five_task_beads.md](https://github.com/sase-org/sase--plans/blob/main/202610/backlog_epic_top_five_task_beads.md)
- **BEAD:** [sase-1oh.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1oh/sase-1oh.5.1.md)

# Protect agents-sidecar publication

## Scope and authority

This is a child epic of phase **sase-1oh.5** in
`plan:202610/backlog_epic_top_five_task_beads.md`. Its final decision
`secret_scan = build` is binding. Implement it without re-asking that decision. The
original task is **sase-1dp**. Read both beads and the parent design through
`sase bead read` and `sase artifact read` before implementing.

The assigned phase is in progress and has no earlier implementation notes at this
planning point. Inspection found the gap still present: the run and prompt publishers
have no scanner call or GH013 branch. No product files were changed by this planning
turn. `sase bead epic-symbols sase-1oh.5` reported no entries.

Use an epic because the work crosses the Rust binding boundary, three publication
transactions, persistent retry state, irreversible object preservation, and a private
operational audit. Each phase above is bounded direct implementation work. Dependencies
are deliberately sequential after the core contract because the publication phases share
staging and completion behavior.

The current assignment says to close only sase-1oh.5. This overrides the parent design's
instruction for this worker to close sase-1dp. Child phase workers close only their
assigned child phase; the child land agent closes its own child epic and then
**sase-1oh.5**, once all requirements below are satisfied. Leave evidence for the
original epic's land agent to settle sase-1dp. Never close **sase-1oh** or another
ancestor plan bead. Do not start the next backlog epic from this child.

No memory edits are authorized. Do not create task beads. Record discoveries as
`PROPOSED FOLLOW-UP: <summary — detail>` notes on the assigned phase, and copy important
integration evidence to sase-1oh.5.

## Findings that constrain the implementation

- `src/sase/agents_sync/publication_snapshot.py::_published_run` creates prompt, chat,
  metadata, workflow, and prompt-step payloads. `_merge_richer_run_files` can restore
  older bytes from the sidecar when the new file is absent or shorter.
- `publication_planning.py` builds full and targeted plans, file references, owner
  manifests, and rendered browsing pages. Removing a payload entry only after references
  are built would leave invalid snapshots or broken links.
- `prompt_archive/preparation.py` renders prompts, copies captured artifacts to
  `files/objects`, writes documents/indexes, and produces reference updates.
  `prompt_archive/deferred.py` runs during both commit drains and full sync.
- `prompt_archive/git_ops.py` stages whole archive roots. Its
  `publish_pending_archive_objects` commits pending hash-valid objects before pulling.
  Hash validity is not evidence that content is safe to publish. `files/objects` can
  hold the only surviving copy after an ephemeral source goes away; preserve those bytes
  before any reset or removal from publication.
- `git_sync_transaction.py`, `commit_publication_transaction.py`, and
  `prompt_archive/publish.py` each push. A failed push leaves local commits ahead of
  upstream. Looking only at the final tree misses rejected content in earlier commits
  that a later push would send.
- Outbox schema 7 has retryable quarantine and terminal retirement, both of which can be
  revived by existing commands. A content-security block must survive those generic
  retries and re-enqueue operations. Whole-project sync must also honor the block when
  it reconstructs inventory without an active row.
- The Rust `note_attachment/scanner.rs` and `attachment_scan_file` binding already
  return only outcome, rule id/kind, line, byte count, and rules version. They return
  `skipped` for unreadable, over-limit, NUL-containing, or non-UTF-8 files. They
  currently collect known values for each call. Reuse this scanner; do not copy its
  credential patterns to Python.
- Core `agent_publication_recovery` owns retry selection, accounting, and completion
  policy. Core `sidecar_publication.rs` currently distinguishes remote divergence from
  unknown failure. Extend the appropriate domain contracts; Python owns Git subprocesses
  and adapters.
- The attachment precedent in `src/sase/bead/attachments/upload/secret_scan.py` blocks
  and resets a store. Its plumbing recognizes GH013 and repository-rule rejection text.
  Use that behavior as a precedent, but do not copy raw rejection output into
  diagnostics.

## Shared implementation contract

1. **Scan the bytes actually being published.** Reuse `attachment_scan_file` against
   stable, private staged copies, and verify the digest of the bytes consumed by the
   write/stage path. Cover transcript and prompt text, textual captured objects, and
   generated text/JSON that can repeat their content. Avoid extension-only detection for
   hash-named objects. Do not store candidate bytes in tracked scratch files or print
   them. A missing binding, scan error, or `skipped` result is not a clean scan for
   required text; withhold it with a value-free reason. Keep existing nontext media
   handling, explicitly separate from a successful text scan, and document that the text
   scanner cannot inspect binary content. Do not silently raise scanner limits to
   publish oversized text.
2. **Withhold and preserve.** Omit unsafe text and its derived references; publish safe
   run metadata and other clean files where the schema permits. If a required structural
   file is unsafe, withhold the smallest internally valid publication unit. A public
   page may say that content was withheld, but must not contain detector evidence or
   private source paths. Preserve the original local artifacts. Preserve any sole-copy
   object privately before removing it from the publication worktree or rejected
   commits.
3. **Durable private state.** Keep a project-scoped protection ledger alongside the
   existing local outbox state, outside the published tree and outside reset targets.
   Key content decisions by repository identity, relative destination, and content
   digest, with affected request identities where available. Records contain status,
   rule ids/kinds, optional line, rules version, and timestamps; never matched bytes,
   environment values, stdout/stderr excerpts, or surrounding text. Keep private content
   recovery storage separate from this metadata. Core owns the schema, matching, and
   transitions; adapters use existing locking and atomic filesystem infrastructure.
4. **Permanent means no automatic replay.** Exact content blocked by a scanner hit or
   remote rejection remains blocked across restart, re-enqueue, full sync, deferred
   restoration, and generic retired/quarantined retries. New or sanitized content is a
   new digest and must pass scanning; retain the old evidence. A transient inability to
   scan is visibly unverified and may be retried safely, without publishing bytes.
   Neither blocked nor unverified means published.
5. **Doctor and completion agree.** Surface protection counts and safe path/rule details
   through `sase doctor`, including JSON output. A blocked prompt must not be mistaken
   for an absent source, falsely acknowledged as fully published, or charged on every
   drain until the whole hood becomes quarantined. Other requests must remain eligible.
   Use structured core decisions, not parsing a human error string to reconstruct
   security state.
6. **Remote output is untrusted and may contain secrets.** Classify GH013,
   `push cannot contain secrets`, and repository-rule rejection before generic
   logging/error conversion. Return sanitized structured classifications, with only
   validated relative paths/commit identities when usable for attribution. Distinguish a
   generic rule rejection from a confirmed secret; never fabricate a scanner rule. Never
   execute or publish a GitHub bypass URL.
7. **No settings or history mutation as an implementation shortcut.** No global
   reinstall, repository settings PATCH, credential rotation, force push, or
   published-history rewrite in a coding phase. The lander's gates below are required by
   the approved parent design.

## Phase protection-policy

Open `sase-core` with `/sase_repo` and read its `AGENTS.md`. Implement a focused
publication-protection domain, or extend the existing publication recovery domain where
ownership is clear, without migrating unrelated existing policy.

- Define typed scan-disposition, durable content-block, and sanitized push rejection
  contracts. Keep attachment scanner semantics intact. Add scanner API surface only if
  the file binding cannot support the stable-copy adapter.
- Define idempotent block/upsert and digest-change behavior. Ensure generic publication
  retry selection cannot clear a security block. Preserve old outbox reads with explicit
  defaults/migration if an additive field is needed. A separate content ledger must also
  protect publishers that have no queued row.
- Add Python binding round-trip and registration coverage and thin sase adapters for
  scanning, locked private persistence, and the existing doctor check family in
  `src/sase/doctor/checks_agent_publication*`. Do not put scanner patterns or new domain
  transition policy into Python.
- Prefer no persistent clean-result cache: known values can change independently of file
  digests. Keep scans bounded and avoid collecting secrets repeatedly within one batch
  if an existing scanner facility allows safe reuse; do not invent a cross-run cache or
  a broad optimization project.
- Test clean/hit/skipped/error, all three existing detector families using synthetic
  values, deduplication, restart persistence, changed digest, schema compatibility,
  retry refusal for exact blocked bytes, and value-free doctor output. Prove GH013
  classification takes precedence over generic retry hints.

Run core and sase checks. Land core through host finalization before dependent sase code
relies on new bindings, then ratchet `sase-core-revision.txt` to a landed commit
containing them. Follow `docs/contribute/rust-backend.md`; recheck the pin against
concurrent work rather than lowering it. Do not change crate versions or changelogs.
Keep the core API exercised through real adapters and tests; do not add lint
suppressions for phased development.

## Phase transcript-publication

Integrate the protection adapter before constructing run file references in
`publication_snapshot.py` and at the full/targeted publication boundary.

- Build file references, capabilities, snapshot digests, and rendered links from
  accepted files. Preserve clean siblings in the same hood and unrelated hoods.
- Make `_merge_richer_run_files` consult protection decisions so it cannot put a
  withheld old prompt/chat back into the payload. Apply the same rule to prior records
  copied from upstream when they would be re-published.
- Check generated metadata, workflow/prompt-step JSON, browsing pages, and manifests
  that contain source-derived text. If required metadata itself hits, exclude its
  publication unit consistently rather than leaving dangling entries.
- Integrate a final pre-stage verification for the agents payload paths in
  `git_sync_ops.py`; scan exact candidate blobs, not every historical upstream file on
  each drain. Existing dirty/pending content cannot evade the guard. Keep the reusable
  protection boundary available to the next phase's archive staging without imposing
  archive-specific behavior here.
- Reconcile withheld updates with previously published content honestly: do not restore
  a blocked local candidate from upstream as a fallback or claim that withholding purged
  already-published history. Existing historical remediation belongs to the audit gate.

Test full sync, targeted commit publication, same-hood clean siblings, previous richer
content, correct manifest validation, no broken generated links, stale dirty worktree
input, and digest change between scanning and writing. Use the existing
`tests/agents_sync/test_publication.py`, targeted-publication and snapshot/I/O fixtures.
A synthetic sensitive byte sequence must be absent from every newly staged/published
blob, not merely from the named transcript file.

## Phase prompt-publication

Protect `prompt_archive/preparation.py`, `publish.py`, `deferred.py`, `git_ops.py`, and
`archive_objects.py` through the shared adapter.

- Scan raw and rendered prompts and each textual captured object before public
  writes/staging. Filtering must remove references and index entries that would point to
  withheld files. Do not enqueue reference updates or artifact-index records that imply
  a withheld prompt/object was published.
- Guard pending objects before their early standalone commit. Scan by actual content
  rather than suffix; cover staged/tracked modifications as well as untracked pending
  objects. Preserve hash checking as an independent invariant.
- Preserve sole-copy rejected/withheld objects in private durable storage before any
  cleanup. A failed private preservation step stops that mutation and keeps the source
  recoverable; it never authorizes publication.
- Propagate structured blocked/unverified prompt outcomes through direct publish,
  deferred regeneration, and completion/accounting. A fresh drain and generic retry must
  not regenerate the same unsafe bytes or falsely acknowledge them.
- Keep clean archive publication and normal binary assets working under the explicit
  shared text-scan contract. Do not weaken existing object-preservation guarantees to
  make cleanup easier.

Test direct/deferred/full-sync routes, unsafe raw and rendered prompts, a secret inside
a hash-valid text object, a pre-existing pending object with no surviving source pool,
clean siblings, private-copy failures, restart/retry behavior, and prompt completion.
Use existing prompt-archive, deferred, object, and publication outbox fixture families,
extending them with meaningful regression scenarios.

## Phase rejection-recovery

Use the core rejection classification at every initial and retry push in
`git_sync_transaction.py`, `commit_publication_transaction.py`, and
`prompt_archive/publish.py`. Enumerate all agents-sidecar push call sites before
editing; include any additional writer that can send these same local commits.

Implement one shared recovery adapter used by all three transaction paths:

1. Under the existing publication lock, identify rejected candidate content and its
   source units from the prepared payload/provenance and validated remote locations.
   Record permanent blocks before any replay. Do not store raw remote output. If
   locations are absent or ambiguous, reset the rejected batch and replay independently
   valid units with bounded attempts to isolate failures; never blame an arbitrary clean
   row or retire the whole project indefinitely.
2. Preserve every local-only object and source required to reconstruct clean units,
   including an earlier pending-object commit. Verify private backup completeness. The
   recovery boundary is the current remote upstream tip, obtained under lock, not
   blindly `HEAD^` or a hard-coded `main` branch.
3. Reset only the verified managed machine publication clone, never a primary checkout
   or arbitrary user repo. Fail safely if the path/ref identity, fetch, backup, or reset
   cannot be verified. Interrupted recovery must be idempotent and resume from durable
   records. Preserve recoverable work; do not use broad untracked cleanup that deletes
   the only object copy.
4. Rebuild safe publication units from preserved sources and current upstream, honoring
   protection state on every reconstruction. Ordinary clean publication should continue
   in this drain or the next bounded drain. Keep existing non-fast-forward behavior and
   avoid charging unrelated requests for a repository-level failure.
5. Prevent a later push from resending blocked bytes in older local unpushed commits.
   Inspect/reconcile the full outgoing commit range, not only the tip tree. Never
   rewrite already-published remote history or force push.

Use real temporary git repositories and a fake rejection runner or local rejecting hook.
Cover a mixed clean/blocked batch, no-location rejection, multiple offending units, all
three push routes, a pre-existing ahead commit, an early object commit, concurrent
upstream movement, interrupted recovery, fetch/reset failure, and a subsequent fresh
process. Verify clean bytes reach the remote, rejected bytes do not occur in its newly
reachable history, private recovery retains the original objects, manifests validate,
and no remote diagnostic bytes leak into logs or state. A generic repository rule that
forbids all pushes must surface a sanitized repository problem; do not promise
publication against a remote that rejects all content or spin indefinitely.

## Phase private-audit

The earlier task reports **17 of 21,467** files with hits. That is historical evidence,
not an expected current count and not proof that credentials are live.

- Open the configured agents sidecar using `/sase_repo`; use only its returned checkout.
  Read metadata/context artifacts with `sase artifact read`. For the authorized bulk
  scan, pass local file paths to the scanner without displaying file contents. Do not
  use raw GitHub file downloads or scan live state through an unrecorded clone.
- Scan the published tip's transcripts, prompt documents, textual captured objects, and
  relevant replicas with the same core scanner and text limits. Enumerate tracked files
  only; record the tip SHA, scanner rules version, total/clean/hit/skipped/error counts,
  and coverage. This is a current-tree audit, not an exhaustive history scan. Send a
  long scan through `/sase_monitor`.
- Produce a private report with relative paths and rule ids only for findings; include
  aggregate coverage metadata and sanitized skip/error reasons. Never emit matched
  values, snippets, environment names/values, or credential bypass links. Do not infer
  that all reported hits are valid live credentials.
- Attach it privately to **sase-1oh.5** using `sase bead note -K` or the private
  attachment command. Use a generic filename and public note prose because filenames and
  note text remain public. Verify private placement; if private publication is
  unavailable, retain the report locally and accurately report that limitation instead
  of publishing it publicly.
- Prepare the child lander's gate material: the private report reference, current
  sidecar inventory and visibility, exact supported GitHub settings JSON, safe branch
  outcomes, and verification commands. Open `sase-github` with `/sase_repo` and read
  `docs/configuration.md`. Its `sdd_secret_scanning` provider option only affects newly
  created public repos; existing repos need the documented PATCH. List exact eligible
  public sidecars and current settings via read-only API calls; do not change settings,
  create sidecars, or include private repositories.

Keep this operational evidence private where required. Store the gate material in a
durable private attachment if it includes finding paths; no secret values are needed
anywhere. Record verification and artifact identifiers for the lander, then close only
this child phase.

## Verification and phase completion

Every implementation phase reads `lint_and_test.md` through `/sase_memory_read` and runs
`sase tool run check` in every repository it changes, plus the targeted regression cases
above. Rust uses the repo's `just test -p <crate> <filter>` wrapper, never bare cargo.
Python targeted tests use `.venv/bin/python -m pytest`. Use `/sase_monitor` for
known-long commands, with formatting first and an explicit follow-up; do not start a
background command and end the turn. Do not run `check-full`, global install recipes, or
unrequested benchmark suites.

Do not weaken assertions, add skips/xfails, increase budgets to mask failures, or add
symbol suppressions. A failure reproduced identically on a clean base is recorded as a
`PROPOSED FOLLOW-UP:` note with command, evidence, and any existing tracking bead; it
does not strand a completed phase. Before closing any assigned phase, run
`sase bead epic-symbols <phase-id>` and resolve or re-key every entry to a still-open
owner. Close with a note naming the actual verification results.

## Child land agent

1. Verify all five child phases landed and re-read their evidence. Check the current
   master and pinned core revision contain all protections. Re-run the focused
   scan/withhold/GH013 integration cases on that integrated tree and run
   `sase tool run check`. Confirm doctor sees the durable private block records and that
   no report, local ledger, or raw secret was added to tracked files.
2. Only after scan enforcement and GH013 recovery are on master, use `/sase_gate` to
   propose enabling secret scanning and push protection on the exact existing public
   sidecars inventoried by the audit. Read current settings again first. Confirm the
   active installed publisher also contains the recovery code before enabling remote
   enforcement; code on master alone does not update a running installation. If it is
   stale, propose the required human update through a gate and verify it in the
   continuation before applying settings. A declined update leaves remote enforcement
   deferred, with that outcome recorded. Use the supported
   `security_and_analysis.secret_scanning.status=enabled` and
   `secret_scanning_push_protection.status=enabled` PATCH, then GET to verify each
   result. Never execute the PATCH directly in the agent. The gate must have explicit
   accept and defer/decline outcomes and a follow-up for every outcome that records
   results and resumes this landing checklist; do not rely on the creating turn
   remaining alive.
3. Raise a separate `/sase_gate` for the historical findings, linking only the private
   report and showing aggregate counts in public text. Ask whether the owner wants
   credential rotation and history-purge remediation. Do not put guessed credentials or
   a broad destructive command in this gate. Its approve command records the requested
   remediation scope for the original epic lander; decline/defer records the outcome.
   Every branch resumes this checklist. Rotation and history rewriting need concrete,
   separately reviewed work and are human follow-up, not actions authorized by this
   epic. Creation of the requested gate and an honest record of its outcome satisfy this
   epic's proposal requirement. Do not start another plan before finishing this landing.
4. Settings/remediation gates end their creating turns. Carry this checklist, bead
   identities, evidence refs, and the next pending gate in each explicit continuation.
   Do not leave an in-progress child phase relying on a future wake-up. If the owner
   declines the settings change or historical remediation, retain the approved code and
   report the declined operation; do not label unexecuted operations completed.
5. Once code, private audit, and required proposals are complete, append concise
   evidence to sase-1oh.5 for the original epic's lander, including core/sase revisions,
   tests, private report reference, and gate identifiers/outcomes. Leave sase-1dp for
   that original lander under the current close-only constraint.
6. Resolve the child epic's remaining symbol exemptions, close this child epic after all
   its phases are closed, then run `sase bead epic-symbols sase-1oh.5` and close **only
   sase-1oh.5** among the pre-existing beads, with the verified evidence note. Do not
   close sase-1oh or another ancestor plan bead. Use `/sase_final` for normal
   completion; host-owned finalizers land commits.
