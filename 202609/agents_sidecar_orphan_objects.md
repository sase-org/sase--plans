---
tier: epic
title:
  Publish orphaned prompt-archive objects and stop sidecar dirt from wedging completion
goal: "Prompt-archive objects under files/objects are committed with the prompt that
  links them, the orphans already in the agents sidecar clones get published by that
  same path, broken or unpublished object links are reported, pre-existing dirt in a
  repository an agent is not committing no longer blocks or invalidates `sase final
  prepare`, and the raw-submit fallback can no longer commit the manifest template's
  placeholder message.

  "
phases:
  - id: archive-objects
    title: Commit prompt-archive objects with their prompts
    depends_on: []
    size: medium
    description: "archive-objects: stage files/objects in both prompt-archive commit
      paths, never clean it, sweep hash-valid pending objects (including today's
      orphans), and quarantine invalid ones. Pending objects must not break the
      pre-commit pull.

      "
  - id: archive-validation
    title: Report unpublished and dangling archive objects
    depends_on:
      - archive-objects
    size: medium
    description: "archive-validation: teach `sase agent prompts validate` about
      files/objects links (missing, untracked, digest, orphan) and add a bounded `sase
      doctor` check that reports dirt in agents sidecar clones.

      "
  - id: seal-scope-core
    title: Scope the completion seal to obligated repositories in sase-core
    depends_on: []
    size: medium
    description: "seal-scope-core: in the sase-core continuation seal and evaluator,
      apply the protected/foreign, completeness, and unknown-HEAD checks and compute the
      worktree fingerprint only over repositories that have a repository decision.
      Errors should name the repository.

      "
  - id: seal-scope-adopt
    title: Adopt the scoped seal in sase
    depends_on:
      - seal-scope-core
    size: small
    description: "seal-scope-adopt: move the sase-core revision pin, add Python
      regression tests for the incident shape, and document which repositories prepare
      inspects and fingerprints.

      "
  - id: submit-fallback-guard
    title: Guard the raw-submit fallback
    depends_on: []
    size: small
    description:
      "submit-fallback-guard: reject the manifest template's placeholder commit message
      in submit and prepare (sase-190). Let `sase final submit` accept a prepare
      wrapper, point prepare refusals at that one-command fallback, and fix the submit
      help example and the sase_final skill text that encourage the lossy rebuild."
proposed_by: bbugyi200.athena.0ry.f0
create_time: 2026-09-25 09:05:31
status: wip
---

- **PROMPT:**
  [prompts/202609/agents_sidecar_orphan_objects.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agents_sidecar_orphan_objects.md)

# Publish orphaned prompt-archive objects and stop sidecar dirt from wedging completion

## Context: what went wrong

Since about 2026-09-23, most agents' `sase final prepare` has been refused with
`protected path files/objects/sha256/40/40de62d8… in repo-f52723edcc8b makes the intent ineligible`.
`repo-f52723edcc8b` is the shared agents prompt-archive sidecar clone
(`~/.sase/projects/gh_sase-org__sase/repos/agents`; open it with
`sase repo open agents`). Many agents hit this, including phase agents sase-17o.2,
sase-17x.10, sase-17x.13.7, sase-17y.3, and sase-17m.5.1.6.3. That last one fell back to
a hand-built `sase final submit` manifest, dropped its real commit message, and landed
`ea130c678` as `feat(scope): describe the completed work` (sase-190). The orphans
themselves are tracked by sase-17u.

Two defects combine here.

**1. Prompt-archive objects are never committed.** Commit `341fff97a` (sase-js.5,
2026-08-11) moved pool-staged prompt artifacts from `artifacts/<month>/…` to the
content-addressed store `files/objects/sha256/<xx>/<sha256>`, via
`artifact_object_relpath` in `_publish_linked_artifacts`
(`src/sase/agents_sync/prompt_archive/preparation.py`). It did not update the path sets
that stage, commit, and clean the archive:

- `_ARCHIVE_PATHS = ("prompts", "artifacts")` in
  `src/sase/agents_sync/prompt_archive/git_ops.py`
- `_PROMPT_ARCHIVE_PATHS = ("prompts", "artifacts")` in
  `src/sase/agents_sync/git_sync_transaction.py`

As a result, every object written since then was left untracked, while the prompt
document that links to it (`../../files/objects/…`) was committed and pushed. Evidence
gathered 2026-09-25:

- `sase--agents` has never tracked any file under `files/`.
- Four tracked prompt documents link to objects:
  - `prompts/202608/bbugyi200.athena.0g6.md` → `40de62…`, untracked on athena,
    2026-08-29
  - `prompts/202609/0gr.md` → `5bd2b6…`, untracked, 2026-09-06
  - `prompts/202609/0lb.md` → `414a92…`, untracked, 2026-09-15
  - `prompts/202609/bbugyi200.apollo.2.md` → `412ed4ed…`, missing on athena; it came
    from apollo
- The bob-cli agents sidecar (`~/.sase/projects/gh_bobs-org__bob-cli/repos/agents`) has
  two more untracked objects, `ce18b4…` and `d8bdfd…`. Its tracked prompts
  `202609/01k.md` and `202609/02e.md` link to them.
- Every orphan's sha256 matches its filename, and each is a tale-plan snapshot. Their
  bytes differ from the archived copies in `sase--plans`, so these objects are the only
  copy of the exact prompt-time text. The linked prompts are already public on GitHub,
  and the same plans (later revisions) are public in `sase--plans`.
- `sase agent prompts validate` passes anyway because `_local_artifact_path` in
  `src/sase/agents_sync/prompt_archive/validation.py` only inspects the legacy
  `artifacts/` tree. The tests in `tests/agents_sync/test_prompt_archive.py` check that
  objects are copied but never that they are committed.

**2. Prepare treats unrelated sidecar dirt as disqualifying.** `sase final prepare`
(`src/sase/finalizers/prepare.py`, `observe_completion_repositories` →
`_observation_repos`) observes every run-start baseline repository, which always
includes the agents prompt-archive sidecar. The sase-core seal
(`crates/sase_core/src/continuation/completion.rs`, `seal_conditional_completion` →
`reject_protected_or_foreign`) rejects the whole intent if any observed path in any
observed repository is protected, meaning dirty at run start and unchanged since. This
applies even to repositories the declaration does not commit. The submit path already
handles protected paths by excluding them from the stitch
(`protected_paths_for_decision` in `src/sase/finalizers/commit_dispatch_support.py`).

The same over-broad scope affects evaluation. The seal's worktree fingerprint
(`worktree_fingerprint`) hashes every observation, including HEAD. The evaluator
(`crates/sase_core/src/continuation/completion_eval.rs`) invalidates the intent with
`stale_worktree_fingerprint` whenever it changes. The agents sidecar received 124
archive commits on 2026-09-24 alone, so once the protected-path wedge is removed, any
verification that runs for minutes would usually go stale and launch a recovery
successor instead of completing on the host.

In September, only 8 intents were ever sealed. All came from runs that happened to have
no run-start baseline for the sidecar. Seven of their verification monitors failed, and
the eighth hit an unrelated `SASE_AGENT_TIMESTAMP` recovery. The fingerprint problem has
therefore not shown up yet, but the code path makes it certain.

## Immediate relief (optional, user-run, before this epic lands)

The blocker is the three untracked objects, and the right fix is to publish them, not
delete them:

- Tracked, pushed prompt documents link to them, so deleting them would leave those
  links permanently dangling.
- Their hashes are verified.
- They publish nothing beyond what is already public.

Agents cannot commit these files themselves:

- Completion is host-owned.
- A path that was dirty at run start is protected and excluded from any agent's commit.

Phase `archive-objects` publishes them automatically: after it lands, the next
prompt-archive publication on athena commits and pushes them. To unblock agents right
away instead, the user can run:

```bash
git -C ~/.sase/projects/gh_sase-org__sase/repos/agents add -- files/objects
git -C ~/.sase/projects/gh_sase-org__sase/repos/agents commit -m "chore(agents): publish orphaned prompt-archive objects"
git -C ~/.sase/projects/gh_bobs-org__bob-cli/repos/agents add -- files/objects
git -C ~/.sase/projects/gh_bobs-org__bob-cli/repos/agents commit -m "chore(agents): publish orphaned prompt-archive objects"
```

No manual push is needed. The next prompt-archive publication in each clone rebases and
pushes, because it pushes whenever the clone is ahead of origin. Phase agents must not
run these commands.

## Phase archive-objects: Commit prompt-archive objects with their prompts

Only the `sase` repo changes. Do not touch the live sidecar clones.

1. **One definition of the archive path sets.** In
   `src/sase/agents_sync/prompt_archive/git_ops.py`, split `_ARCHIVE_PATHS` into two
   sets:
   - Regenerable roots: `prompts` and `artifacts`. `clean_prompt_archive_worktree` keeps
     resetting, restoring, and cleaning these.
   - Published roots: the regenerable roots plus `files/objects`.
     `commit_prompt_archive_if_dirty` stages these.

   `src/sase/agents_sync/git_sync_transaction.py` must import the published set instead
   of keeping its own `_PROMPT_ARCHIVE_PATHS` copy. `files/objects` is append-only and
   content-addressed, and an object is not reliably regenerable because its pool copy
   lives in an ephemeral workspace. Nothing may `git clean`, `git checkout`, or delete
   under it.

2. **Pin the object root to the Rust layout.** Add a test asserting that
   `sase_core_rs.artifact_object_relpath(<digest>)` starts with the published object
   root. If the layout moves again, this fails a test instead of silently orphaning
   objects.

3. **Sweep hash-valid pending objects.** Add a helper that lists untracked paths under
   `files/objects`. Use one `git ls-files --others --exclude-standard -- files/objects`
   call.
   - A candidate is valid only if all of these hold:
     - Its path matches `files/objects/sha256/<2 hex>/<64 hex>`.
     - The 2-hex directory equals the digest prefix.
     - It is a regular file, not a symlink.
     - The sha256 of its bytes equals its name.
   - Valid pending objects are committed. This includes pre-existing orphans such as
     today's, so the next publication after this lands publishes them.
   - Invalid candidates are moved, never deleted, into a quarantine directory under the
     sidecar's git dir. Use a path such as
     `<git-dir>/sase-quarantine/objects/<utc-ts>/…` and log it, so a bad file can never
     wedge the worktree and stays recoverable.
   - Unreferenced valid objects, for example from a transaction that failed after the
     copy, are still committed. They are harmless, and phase `archive-validation`
     reports them as orphan warnings.

4. **Pending objects must not break the pull.** Both publication paths run clean →
   `git pull --rebase` → write → commit → push:
   - `src/sase/agents_sync/prompt_archive/publish.py`
   - the full sync in `git_sync_transaction.py`

   A locally untracked object whose path the remote already tracks, for example the same
   object published from another machine, makes `git pull --rebase` refuse to overwrite
   it. Fix this with a sweep before the pull: while holding the agents sync lock, commit
   the valid pending objects first. Use a SASE-authored
   `chore(agents): publish pending prompt-archive objects` commit with the same identity
   and `apply_auto_commit_type_tag(…, AGENTS_SYNC_AUTO_COMMIT_TYPE)` tagging as the
   archive commit. An identical add/add then rebases cleanly. The post-write archive
   commit also stages `files/objects` for objects written in this transaction.

5. **Tests.** Extend `tests/agents_sync/test_prompt_archive*.py` and the full-sync
   transaction tests:
   - Publishing a prompt with a pool-staged artifact leaves
     `git status --porcelain -uall` empty, and HEAD tracks the object.
   - The incident shape: three valid untracked objects linked from already-tracked
     prompt documents are committed by the next publication, with no other prompt
     content needed beyond an ordinary new archive.
   - An invalid object (digest mismatch, or a stray non-canonical file under
     `files/objects`) is quarantined, not committed, and the worktree ends clean.
   - The full-sync path stages objects, and its clean steps do not delete them.
   - A two-clone case: the remote already tracks an object that is untracked locally,
     and publication still succeeds.
   - The layout-pin test from step 2.

6. Verify with `just check`, run through `sase tool run` as the repo's lint/test memory
   requires.

## Phase archive-validation: Report unpublished and dangling archive objects

1. **`sase agent prompts validate`**
   (`src/sase/agents_sync/prompt_archive/validation.py`, CLI in
   `src/sase/agents/cli_prompts.py`):
   - Accept link targets that resolve under `files/objects/`, alongside the legacy
     `artifacts/`.
   - Report these codes:
     - `artifact-missing` (error): the object is absent.
     - `artifact-untracked` (error, new code): the object is present but not tracked by
       git. Use a single `git ls-files -- files/objects` call, not one per file.
     - `artifact-digest` (error): the sha256 of its bytes differs from its filename.
     - `artifact-orphan` (warning): a tracked object that no prompt links to.
   - Keep the legacy `artifacts/` checks unchanged.
   - Update any docs that list the validator's issue codes (grep `docs/`).

2. **`sase doctor` check.** Add a default-group, bounded, read-only check such as
   `project.agents_sidecar_dirt`. Model it on the existing
   `project.primary_sidecar_link_dirt` check.
   - For each enabled project's agents sidecar clone, run one bounded
     `git status --porcelain -uall` and report WARN with the dirty paths.
   - Classify each path:
     - Hash-valid pending objects will be published by the next prompt-archive
       publication. The next step is `sase agent sync -p <project>`.
     - Anything else is unexpected dirt that needs a human look.
   - A clean sidecar is OK.

3. **Tests** for the new codes, including the incident shape: tracked prompts linking
   one untracked object and one missing object. Also add doctor-check tests for clean,
   pending-object, and unexpected-dirt sidecars.

4. Verify with `just check` via `sase tool run`.

## Phase seal-scope-core: Scope the completion seal to obligated repositories in sase-core

Open the linked checkout with `sase repo open sase-core -r "<why>"`, read its
`AGENTS.md`, and work only in the printed path. Only `sase-core` changes in this phase.

1. **Rule.** The repositories that matter to a conditional-completion intent are the
   ones with a repository decision, which `validate_obligation_coverage` already
   guarantees cover every obligation. Only those repositories are committed by host
   completion. Pre-existing or concurrently changing state elsewhere cannot be harmed by
   host completion and must not disqualify or invalidate the intent. New dirt in a
   previously undecided repository is still caught, through the evaluator's existing
   `new_repository_obligation` reasons.

2. **Seal** (`crates/sase_core/src/continuation/completion.rs`, in
   `seal_conditional_completion`):
   - Apply these checks only to observations whose `repo_id` has a repository decision:
     - the `!repo.complete` ("incomplete or unstable repository observation") check
     - the unknown-HEAD check
     - `reject_protected_or_foreign`
   - Keep recording the full observation list in the intent, so the evidence and wire
     shape stay unchanged.

3. **Fingerprint.** Compute `worktree_fingerprint` over the decided repositories'
   observations only, ordered deterministically by `repo_id`. Use it identically in
   three places:
   - at seal time
   - at intent load/validation (the `intent.seal.worktree_fingerprint` comparison in
     `completion.rs`)
   - in `evaluate_conditional_completion`
     (`crates/sase_core/src/continuation/completion_eval.rs`), taking the decided set
     from `intent.repository_decisions`

   The evaluator's `incomplete_observations` and `unknown_head` reasons are likewise
   limited to decided repositories. A decided repository that is missing from the
   current observations must make the fingerprint differ (stale), not be skipped.

4. **Actionable errors.** Protected/foreign rejections name the observation's `name`
   alongside `repo_id`, and say the path was already dirty before this run. For example:
   `protected path P in repo-X (main) was dirty before this run and is unchanged; prepare cannot seal a repository with pre-existing dirt — clean or commit it, or finish with sase final submit`.

5. **Compatibility.** Intents sealed by an older core carry an all-repository
   fingerprint. After upgrade they must fail closed through the existing conflict/stale
   path, which ends in ordinary recovery, and never panic. Add a test. The wire is
   unchanged, so this is a `fix(continuation): …` commit, not a breaking change.

6. **Rust tests** beside the module:
   - An undecided repository with protected and foreign paths → the seal succeeds.
   - A decided repository with a protected path → still rejected, and the message names
     the repository.
   - An undecided repository with an incomplete observation or unknown HEAD → the seal
     succeeds.
   - Evaluation where an undecided repository's HEAD or paths changed → no
     `stale_worktree_fingerprint`.
   - Evaluation where a decided repository changed → `stale_worktree_fingerprint`.
   - New dirt in an undecided repository → a `new_repository_obligation:*` reason.
   - The old-fingerprint compatibility case.

7. Verify with `sase tool run check` in sase-core, as its `AGENTS.md` describes; allow
   more than 10 minutes. Also rebuild the local extension as `docs/rust_backend.md`
   describes, and run sase's `tests/test_final_prepare.py` and
   `tests/monitor/test_monitor_host_completion*.py` against it, to catch Python callers
   that assumed all-repository scope.

## Phase seal-scope-adopt: Adopt the scoped seal in sase

1. Once phase `seal-scope-core`'s commit is on sase-core's remote default branch, move
   `sase-core-revision.txt` with `just ratchet-core-revision`. See "The CI source
   revision pin" in `docs/rust_backend.md`.

2. **Python regression tests** in `tests/test_final_prepare.py` and
   `tests/monitor/test_monitor_host_completion.py`:
   - The incident shape: the agents prompt-archive sidecar is in the run-start baseline
     with protected untracked files, and the main repository has agent-authored changes.
     `sase final prepare` succeeds.
   - An unrelated commit in that sidecar between prepare and evaluation does not make
     the intent stale.
   - Protected dirt in the main repository still refuses prepare, and the message names
     the repository.

3. **Docs.** In `docs/monitors.md` ("Prepared host completion"), state that prepare
   records every relevant repository but only repositories with a repository decision
   decide eligibility and the staleness fingerprint, and say why. No Python logic change
   is expected in `src/sase/finalizers/prepare.py`. If the tests show one is needed,
   keep it a thin adapter change.

4. Verify with `just check` via `sase tool run`.

## Phase submit-fallback-guard: Guard the raw-submit fallback

This phase resolves bead sase-190. Read it with `sase bead read sase-190 -r "<why>"`.

1. **Reject the placeholder.** Hoist the template message in
   `src/sase/finalizers/declaration_manifest.py` (`manifest_template`, currently the
   literal `feat(scope): describe the completed work`) into one shared constant, so the
   template and the check cannot drift. `_validate_commit_decision` rejects a commit
   decision whose message equals that constant, after whitespace normalization, with a
   stable error code such as `commit_message_placeholder`. The message tells the agent
   to describe the work in that repository. The `sase final prepare` validation path
   applies the same check.

2. **One-command fallback.** Let `sase final submit` accept a prepare wrapper file
   directly: a top-level object with `declaration` and `verification` and no `payloads`.
   It submits that wrapper's `declaration` unchanged, so the agent never rebuilds the
   manifest.
   - A plain manifest behaves exactly as today.
   - Stale-context handling is unchanged.
   - Update the parser help in `src/sase/main/parser_final.py`. Its current example,
     `sase final context -f json | jq '.manifest_template' | sase final submit -`,
     submits the unedited template, which after this phase is a guaranteed rejection.
     Replace it with an example that edits the template first, or with the wrapper form.

3. **Refusal hint.** When `sase final prepare` is refused for eligibility, its error
   output ends with this fallback: run the verification command inline, then
   `sase final submit <same-wrapper-file>`.

4. **Skill text.** In `src/sase/xprompts/skills/sase_final.md` ("Prepared Monitor
   Completion"), add: if prepare is refused, run the verification inline and pass the
   same wrapper file to `sase final submit`; never rebuild the manifest from
   `manifest_template`, which drops the message you wrote. Do not deploy generated
   skills. Deployment happens later from the landed tree (see the generated-skills
   memory).

5. **Tests.** Cover each of these:
   - The placeholder is rejected by submit and by prepare.
   - A real message is accepted.
   - A wrapper file is accepted by submit and yields exactly its declaration.
   - A malformed wrapper is rejected with a clear error.

6. Verify with `just check` via `sase tool run`.

## Landing and rollout verification

- After landing, the host's editable install (the primary checkout) runs the new code.
  After the next prompt-archive publication on athena, confirm:
  - `git -C ~/.sase/projects/gh_sase-org__sase/repos/agents status --porcelain -uall` is
    empty.
  - `git ls-files files/objects` lists `40de62…`, `414a92…`, and `5bd2b6…`.
  - `sase agent prompts validate` reports no `artifact-untracked`.
- The only expected `artifact-missing` is `412ed4ed…` (linked from
  `prompts/202609/bbugyi200.apollo.2.md`). It self-heals if apollo still has the object
  once apollo runs the fixed sase. Otherwise, report it rather than inventing content.
- bob-cli's two objects publish on the next bob-cli prompt publication, or on
  `sase agent sync -p bob-cli`. `sase doctor` should flag them until then.
- Close sase-17u and sase-190, with resolution notes pointing at this epic's commits.
- Leave the three `feat(scope): describe the completed work` commits (`1c4ebe76a`,
  `1230ed8da`, `ea130c678`) alone. Rewriting master history is out of scope.

## Out of scope

- A discovered anomaly for the land agent to file through `/sase_new_task` if nothing
  already tracks it: the eight runs that sealed intents in September have no `run_start`
  records in `finalizer_baseline.json`, only `opened_repo`. Example: `sase-17d.6--code`,
  artifacts dir `…/ace-run/202609/24/20260924083707`. Without those records,
  pre-existing dirt in those runs' repositories is not protected on the submit path
  either. Investigate whether this is intended for `--code`/`--plan`/`.land`
  continuations.
- Letting prepared completion accept deferrals, which the sase-17x.10 follow-up
  mentions.
