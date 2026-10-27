---
tier: tale
size: medium
title: Finish agents-publication protection gaps and land sase-1oh.5.1
goal:
  Withheld hoods and runs no longer abort sync or get retired or charged as failures,
  agents git telemetry stops logging remote output, recovered objects are restored for
  the clean rebuild, and epic sase-1oh.5.1 and phase sase-1oh.5 close.
decisions:
  unverified_runs:
    ask:
      Should unverified runs and prompts retry every drain without being charged toward
      quarantine?
    choices:
      retry:
        Unverified runs and prompts stay pending uncharged, so a scanner outage never
        quarantines them
      charge:
        Unverified transcript runs are charged toward retryable quarantine, like today's
        prompt rule
    default: retry
    why:
      The contract says an inability to scan may be retried safely; quarantine needs a
      manual revive
    answer: retry
proposed_by: bbugyi200.athena.sase-1oh.5.1.land--3
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-27 06:06:36
status: wip
---

# Finish agents-publication protection gaps and land sase-1oh.5.1

## Context and authority

This tale is the remaining work of the land agent for child epic **sase-1oh.5.1**
(`plan:202610/agents_publication_secret_scan.md`). It is a child of phase **sase-1oh.5**
of epic **sase-1oh** (`plan:202610/backlog_epic_top_five_task_beads.md`). All five child
phases are closed and their commits are on master. The lander verified the shipped work
and recorded four remaining code gaps as note #1 on sase-1oh.5.1. Read that note first
with `sase bead read sase-1oh.5.1 -r "<why>"`. Also read the epic plan's "Shared
implementation contract" through `sase artifact read`. Its rules still bind this tale:
withheld means neither published nor falsely completed, blocked bytes never replay, and
remote output never reaches logs or state.

The landing has no other agent behind it. This tale must finish the landing itself (see
the final step). The human gates for this landing are already answered and recorded:

- Publisher update gate `custom-7063d7d4-b8c2-479d-a898-66bce1310f75` = **update**. It
  is verified: the installed tool imports the recovery modules, and the primary checkout
  contains cecfbbab2f (note #3 on sase-1oh.5.1).
- Push-protection gate `custom-85522f5d-db96-465b-bb01-542a31f7ea2f` = **enabled and
  verified on sase-org/sase--agents only**. sase--beads, sase--plans and sase--research
  were left unchanged, so remote enforcement stays deferred there (note #4 on
  sase-1oh.5.1).
- Historical-remediation gate `custom-57bc7a62-1331-4a3b-900d-a1b2d4612e5d` = **approve,
  scope both (credential rotation and history purge)**. The gate command recorded the
  request as note #5 on sase-1oh.5. Nothing was rotated, rewritten or force-pushed. That
  work belongs to the original sase-1oh lander as separately reviewed human work, and
  this tale must not plan or perform it.

Follow-up triage is done: tasks sase-1or (unattributed GH013 isolation) and sase-1os
(prompt migration bypassing scanning) were filed, and flake sase-1ny was corroborated
(note #2 on sase-1oh.5.1). Do not create new task beads for those. Do not start another
backlog epic.

Hard rules:

- Never close sase-1oh or sase-1dp.
- Never use `--force` to make a close succeed.
- Never run `just check-full` or a global install.
- Do not rotate credentials, rewrite history or force-push.
- Do not weaken assertions or add skips, xfails or symbol suppressions.

Bead notes and plan files are public. Never write finding paths, rule ids, credential
names or values into them.

Every gap below was rechecked on current master during planning. Confirm each one still
reproduces before you fix it.

## Gap A: one withheld hood aborts full sync

`publication_snapshot.py::build_hood_snapshot` raises when no current run in a hood is
publishable. It raises `AgentsSyncFormatError` when every run is blocked and
`RuntimeError` when every run is unverified. `publication_planning.py::plan_hoods` does
not catch either, so `publication.py::reconcile_agent_hoods` (full sync) aborts for the
whole project. `plan_targeted_hoods` has the same uncaught path.

- Replace the raise with a typed, structured withheld-hood outcome. Use either a
  dedicated exception that carries the hood, the reason (`blocked` or `unverified`) and
  the withheld run global names, or an equivalent return shape.
  - It must not subclass `AgentsSyncFormatError`. The commit drain treats that error as
    terminal.
  - Keep the separate "no publishable runs" raise for a genuinely empty inventory
    (`not current and previous is None`), and the one in `publication.py`. Queue
    retirement relies on that wording for absent sources.
- In `plan_hoods` and `plan_targeted_hoods`, skip a fully withheld hood:
  - Write no new snapshot or run files for it.
  - Leave its existing manifest entry and on-disk snapshot exactly as they are. Do not
    restore blocked candidates from upstream, and do not claim history was purged.
  - Add a value-free diagnostic with the hood name and reason only.
  - Keep planning every other hood.
  - Exclude the skipped hood from the published/refreshed/unchanged counts.
- Surface the withheld runs in structured form to callers. A suggested shape is a
  `withheld_runs` field on `V2PublicationCounts` with a safe default, holding
  `(global_name, reason)` pairs for every withheld run, whether the whole hood was
  withheld or only some runs. Check every construction and consumer of
  `V2PublicationCounts` (CLI output, JSON, tests) before adding fields. Build the field
  from `_GatedRun.withheld_reason`, never by parsing note strings.

## Gap B: drain accounting treats withheld runs generically

`commit_publication_transaction.py` has two generic paths:

- `_acknowledge_completed` sends a blocked or unverified run whose clean siblings
  published through `publication_request_fulfilled`. The page is absent, so the request
  is retired terminally with "did not materialize during commit drain".
- The per-hood publish loop charges every request in a fully withheld hood. A blocked
  hood is retired terminally through the `AgentsSyncFormatError` branch, and an
  unverified hood is charged toward quarantine.

Required behavior (epic contract items 4–5; sase-1oh.5.1.2 follow-up #2):

- Collect the structured withheld runs from Gap A for every hood published in the drain.
  Use `hooks.publish_hood`'s returned counts, or thread them through
  `PublicationTransactionHooks` if the hook currently drops them. A fully withheld hood
  no longer raises, so its requests reach the same settlement.
- Before `publication_request_fulfilled`, settle a request whose `global_agent` is a
  withheld run as follows. For an agent-session request, use the hood when the whole
  hood was withheld.
  - **blocked**: park it with a value-free error that says the content is withheld by
    publication protection. Mirror the prompt path's `PROMPT_PROTECTION_BLOCKED`
    handling: `increment_attempts=False`, no quarantine threshold, no `terminal_reason`,
    not acknowledged. Generic retries and re-enqueue must not publish it. The ledger
    already guarantees that, so add a test.
  - **unverified**: keep it pending with a value-free unverified error. It is not
    acknowledged and not retired.

> [!decision] unverified_runs = retry Do not charge unverified runs
> (`increment_attempts=False`, no quarantine threshold). Apply the same rule to the
> prompt path: make `should_charge_prompt_protection_failure` return False for
> `PROMPT_PROTECTION_UNVERIFIED` as well as `PROMPT_PROTECTION_BLOCKED`, and update its
> docstring and any test that encodes the old rule. This is a deliberate policy change,
> not a weakened assertion. Both callers, `git_sync.py` and
> `commit_publication_transaction.py`, pick it up.

> [!decision] unverified_runs = charge Charge unverified transcript runs once per drain
> toward the existing retryable quarantine (`increment_attempts=True`, the configured
> threshold, no `terminal_reason`). Leave the prompt-path rule unchanged. Unverified
> runs must still never be retired terminally.

- Clean sibling requests in the same hood must still be acknowledged in that drain.
  Requests in other hoods must remain unaffected.
- Keep this accounting in the Python adapters, consistent with the prompt path, unless a
  core function such as `decide_publication_request_completion` or
  `agent_publication_recovery` already owns the exact decision. If it does, extend core
  in the linked sase-core repo (open it with `sase repo open sase-core`, test with
  `just test -p <crate> <filter>`), land it through host finalization, and ratchet
  `sase-core-revision.txt` past the new commit as `docs/rust_backend.md` describes.

## Gap C: agents git telemetry logs raw remote output

`src/sase/sdd/_git_runner.py::_log_git_operation` writes `stdout_preview` and
`stderr_preview` (500 characters each) for every push or fetch, every failure and every
slow command. It writes them to the local git-ops JSONL through
`sase.logs.log_tui_git_operation`. Agents-sidecar git calls go through
`src/sase/agents_sync/git.py` (`run_git` and `run_git_bytes`, which call `run_sdd_git`
and `run_sdd_network_git`). So GH013 rejection text, and for slow `git show` calls even
transcript bytes, reach the log.

- Add an explicit opt-out for output previews, such as an `output_previews: bool = True`
  keyword, to `run_sdd_git`. Thread it through `_run_streaming_sdd_git`,
  `run_sdd_network_git` and `run_sdd_network_git_with_retries` into
  `_log_git_operation`. When it is off, omit both preview fields. Keep `status`,
  `returncode`, timing, `cmd` and `cwd`.
- Keep the retryability classifier fields only if `classifier_reason` is a fixed
  vocabulary string that never echoes output. Verify that in the core facade. If it can
  echo output, drop it when previews are off.
- Pass the opt-out from `run_git` and `run_git_bytes` for every agents-sidecar call,
  local and network.
- Enumerate every other git invocation that operates on the agents sidecar (grep
  `src/sase/agents_sync/` including `prompt_archive/`, for direct `run_sdd_git*` or
  `subprocess` git calls) and route each through the same opt-out.
- Also check `_git_network.py`'s retry loop and any `_logger` calls on these paths for
  raw output.
- Callers outside agents-sidecar publication keep their previews unchanged.
- Existing local log lines are size-rotated by the JSONL sink. Do not rewrite them; say
  so in the evidence note.

## Gap E: privately preserved objects are never restored

`publication_recovery.py::_recover_secret_scanning_rejection` preserves outgoing
`files/objects/...` files into private storage through
`prompt_archive/protection.py::preserve_private_publication_object`, at
`<sase_home>/private/agents-publication-recovery/<project>/<xx>/<sha256>`. It then
resets the managed clone to the upstream tip. Nothing ever copies a clean preserved
object back. A clean prompt whose ephemeral pool copy is gone therefore loses its object
in the rebuild.

- Add a restore helper next to `preserve_private_publication_object`, for example
  `restore_private_publication_object(*, project_key, sha256, destination)`. It must:
  - validate the project and digest the same way the preserve helper does;
  - refuse symlinks;
  - verify the private copy's digest;
  - write atomically into the managed clone;
  - re-verify the written digest;
  - never overwrite an existing different file;
  - leave the private copy in place.
- After `_reset_managed_clone_to_tip` succeeds, and before returning
  `RECOVERY_REBUILD_CLEAN`, restore each preserved path that meets all of these:
  - Its digest comes from
    `prompt_archive/archive_objects.py::canonical_archive_object_digest` (canonical
    `files/objects/sha256/<xx>/<sha256>` paths only).
  - It is absent from the reset worktree.
  - It is not among this rejection's blocked paths.
  - `publication_content_retry_allowed` allows its destination and digest.

  Restore it as an untracked pending object. The existing pending-object guard
  (`prompt_archive/git_ops.py::publish_pending_archive_objects`) then scans it by
  content and commits it or withholds it. Never commit it here.

- Record the restored paths on `PublicationPushRecovery` next to `preserved_paths`.
- A failed restore for one object is value-free and non-fatal:
  - the object stays private and is not published;
  - the rebuild continues for clean units;
  - the error message carries no bytes.
- Restoration must not loosen bounded recovery. Unattributed rejections remain
  sase-1or's scope.

## Tests

Add focused regression tests with synthetic sensitive values. Assert that those values
are absent from every newly published blob, log record and ledger entry. Extend these
fixtures:

- `tests/agents_sync/test_transcript_publication_protection.py`
- `tests/agents_sync/test_publication_recovery.py`
- `tests/agents_sync/test_commit_publication_drain_accounting.py` and
  `commit_publication_fixtures.py`
- `tests/agents_sync/test_prompt_archive_protection.py`
- the existing `tests/sdd` git-runner tests

Cases:

- **A**:
  - Full sync with one fully blocked hood, one fully unverified hood (scanner made
    unavailable for that hood) and one clean hood. The clean hood publishes, the
    withheld hoods' manifest entries and snapshots are unchanged, the diagnostics are
    value-free, and nothing raises.
  - The same three hoods through targeted planning.
  - An empty inventory still raises the existing "no publishable runs" error.
- **B**:
  - A drain with a clean run and a blocked run in one hood. The clean request is
    acknowledged. The blocked request is not retired, not charged and not acknowledged,
    and its error is not "did not materialize".
  - A second drain and a generic retry or re-enqueue still do not publish the blocked
    bytes.
  - A fully blocked hood is parked, not retired.
  - Fully unverified hood and unverified sibling cases follow the chosen
    `unverified_runs` branch, including the prompt-path rule under `retry`.
- **C**:
  - A failing agents push through `run_git(..., network=True)` with fake stderr
    containing GH013 wording and a synthetic secret. Redirect the git-ops JSONL path to
    a temp file. The record has no preview fields and no synthetic bytes.
  - A slow agents `run_git_bytes` call likewise.
  - A non-agents `run_sdd_git` call still records previews.
- **E**: use a real temporary git repo with a remote and a fake rejection runner, or a
  rejecting hook.
  - A rejected batch with one clean canonical object and one blocked file. After
    recovery the clean object is restored untracked at its canonical path with the right
    digest, and the blocked content is not restored.
  - The next pending-object publish pushes the clean object, and the blocked bytes never
    reach the remote.
  - A private copy whose digest does not match is skipped value-free.
  - An object that the ledger marks blocked is not restored.

Verification:

1. Read `lint_and_test` through `/sase_memory_read`.
2. Run the focused tests with `.venv/bin/python -m pytest <files>`.
3. Run `sase tool run check`; never `just check-full`. A failure that reproduces
   identically on a clean base, verified with `git stash`, is not this tale's work.
   Record it in the evidence note with the command and its tracking bead, if any.
4. If sase-core changed, run its tests and checks too, and move the pin.

## Final step: close out this landing

Do this in the same turn as the code. Never wait for or order this after this tale's own
commit, SHA, push or CI result. Host finalization lands the commit.

1. **Evidence for the original lander.** Run `sase bead note sase-1oh.5 "<evidence>"`,
   using sanitized aggregate wording only. The note must contain:
   - the base sase master revision (`git rev-parse HEAD` before your changes), and that
     this tale's A/B/C/E fixes land through host finalization;
   - the pinned sase-core revision from `sase-core-revision.txt`;
   - the tests and checks you ran, with results;
   - the private report reference: attachment audit-report.md,
     sha256:53a690592ee5dd1488af5287bc4be8af05a491b20f15e9be8ceb9e9259a766a6, on
     sase-1oh.5. Aggregates only: tip 5066d26a, rules v1, 138525 files, 138497 clean, 27
     hits, 1 skipped, 0 errors;
   - the gate outcomes:
     - publisher-update `custom-7063d7d4-b8c2-479d-a898-66bce1310f75` = update,
       verified;
     - push-protection `custom-85522f5d-db96-465b-bb01-542a31f7ea2f` = enabled and
       verified on sase-org/sase--agents only, with sase--beads, sase--plans and
       sase--research unchanged and enforcement deferred there;
     - historical-remediation `custom-57bc7a62-1331-4a3b-900d-a1b2d4612e5d` = approve,
       scope credential rotation plus history purge, recorded as note #5 on sase-1oh.5,
       nothing rotated or rewritten;
   - that sase-1dp is left for the sase-1oh lander;
   - that rotation, history purge and push protection on the other three sidecars were
     deferred or requested, not completed;
   - that pre-fix local git-ops log lines were left to size rotation, not rewritten.
2. **Epic symbols.** Run `sase bead epic-symbols sase-1oh.5.1` and resolve every entry
   you or anyone added: wire it up, privatize it, add a non-test pragma, or delete it
   per the Symvision epic-whitelist policy. Re-key a Justfile line only to a still-open
   later bead that genuinely needs the exemption. At planning time it reported no
   entries.
3. **Close the epic.** Run `sase bead close sase-1oh.5.1 --note "<verification>"`. The
   note must cover the five phases verified on master, the A/B/C/E fixes and their
   tests, the check results, the integration finding (aa97e544b0 `--retry-key` shares
   the protected drain), the follow-up triage (sase-1or and sase-1os filed, sase-1ny
   corroborated, the other proposals declined as non-reproducible or already fixed), and
   the three gate ids and outcomes above. If the close is refused for leftover
   epic-symbol entries, finish step 2 and close again. Never use `--force`.
4. Run `just symvision` and confirm the whitelist is clean.
5. Set `status: done` in the frontmatter of the epic plan file. Its path is the PLAN
   path shown by `sase bead read sase-1oh.5.1`, which is
   `202610/agents_publication_secret_scan.md` in the plans sidecar.
6. **Parent phase sase-1oh.5.** Read it with `sase bead read sase-1oh.5 -r "<why>"`.
   Verify that its required work is complete:
   - scan enforcement for transcripts, prompts and objects;
   - GH013 recovery on every agents push path;
   - the private audit attached privately;
   - the settings and remediation proposals raised as gates, with outcomes recorded.

   Then run `sase bead epic-symbols sase-1oh.5` and resolve any entries. Close only this
   phase with `sase bead close sase-1oh.5 --note "<what was verified>"`. Leave epic
   sase-1oh to its waiting land agent. Never close sase-1oh or sase-1dp, and never use
   `--force`. If the phase is incomplete or ambiguous, do not close it. Record the
   blocker with `sase bead note sase-1oh.5 "..."` and report it.
