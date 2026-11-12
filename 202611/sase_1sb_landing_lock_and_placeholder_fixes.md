---
tier: tale
size: small
title: Finish and land epic sase-1sb (edit-lock race and prompt placeholder fixes)
goal:
  Fix the two remaining defects the sase-1sb epic introduced, verify them, and close the
  epic
proposed_by: bbugyi200.athena.sase-1sb.land
proposed_with: claude/opus
bead: sase-1sb
create_time: 2026-11-12 17:30:35
status: wip
---

- **PARENT:**
  [202611/security_bug_audit_fixes.md](https://github.com/sase-org/sase--plans/blob/main/202611/security_bug_audit_fixes.md)
- **BEAD:**
  [sase-1sb](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1sb/README.md)

# Finish and land epic sase-1sb

## Context

Epic `sase-1sb` ("Security and robustness audit fixes") has every phase closed,
including nested child epic `sase-1sb.2.1`. Its land agent audited all phases,
decisions, commits, and post-epic drift; the full record is note #2 on `sase-1sb`
(`sase bead read sase-1sb -r "<why>"`). That audit left two defects the epic itself
introduced. This tale fixes both and then closes the epic. Nothing resumes the landing
afterward, so the closeout at the end of this plan is mandatory, in the same turn as the
code.

Follow-up triage is already finished and recorded on the epic. Do not create task beads
for these items:

- `sase-1sg`: TUI lifecycle flake (+1 added)
- `sase-1sm`: sase-core migration guard
- `sase-1sn`: structured workflow output still reaching host `$(cmd)`
- `sase-1sa`: sase-github `_run_shell` audit

`sase bead epic-symbols sase-1sb` is currently empty. `sase-1sb` has no parent bead.

## Defect 1: edit-lock ownership race (from phase sase-1sb.5)

File: `src/sase/ace/patch/locking.py`. Tests: `tests/ace/changespec/test_edit_lock.py`.

`acquire_edit_lock` creates the lock with `os.open(O_CREAT | O_EXCL)` and fills it with
`os.write` afterwards. Between those two calls the file exists but is empty. A second
contender's `_read_edit_lock` returns `None` for the empty file,
`_remove_stale_edit_lock(project_file, None)` sees content `None == expected` and
unlinks the first owner's lock, and both contenders return as owners.

Deterministic reproduction on master: patch `locking.os.write` so the first thread
blocks inside its write until a second thread's `acquire_edit_lock` returns. The outcome
is `['first:owner', 'second:owner']`. This race is what makes
`test_concurrent_edit_lock_acquisition_has_one_owner` flaky. Fix the race; do not
quarantine or retry the test.

`release_edit_lock` has a related ownership hole: it unlinks the lock unconditionally. A
process whose lock was reclaimed can therefore delete its successor's lock.

### Required changes

1. **Atomic publication.** Write `f"{pid}\n{start_token or ''}\n"` to a unique temp file
   in the lock's directory (`tempfile.mkstemp(dir=..., prefix=...)`), then publish it
   with `os.link(temp, lock_file)`. The lock pathname must never be visible empty or
   partial. `FileExistsError` from `os.link` takes the existing-lock path: read it,
   raise `FileExistsError` if it is active, otherwise reclaim it as stale and retry.
   Always unlink the temp file (`finally`).
   - If `os.link` fails because the filesystem does not support hard links (`EPERM`,
     `ENOTSUP`/`EOPNOTSUPP`, `EXDEV`, `EMLINK`, `ENOSYS`), fall back to the existing
     `O_CREAT | O_EXCL` + write path.
2. **Unparseable locks get a grace window.** Distinguish "lock file missing" from "lock
   file present but unparseable/empty". A dedicated sentinel or result type is fine;
   `None` must no longer mean both.
   - An unparseable lock whose mtime is younger than a short module-level grace constant
     (for example 5 seconds) is treated as held: `acquire_edit_lock` raises
     `FileExistsError` and `is_edit_locked` returns `True`. It may be an in-flight
     fallback publication or a write by an older sase version.
   - An older unparseable lock is stale and is reclaimed through
     `_remove_stale_edit_lock`'s existing flock plus inode/content recheck.
3. **Owner-checked release.** `release_edit_lock` unlinks only when the lock records
   this process's PID, and its start token when one is recorded. Use the same flock plus
   inode/content recheck as `_remove_stale_edit_lock`. Otherwise leave the file in place
   (debug log). Missing files stay a no-op.
   - Both production callers acquire and release in one process:
     `src/sase/ace/tui/actions/patch/_core.py` and
     `src/sase/ace/tui/modals/project_management_actions.py`. Confirm this still holds.
4. Keep the public API (`acquire_edit_lock`, `release_edit_lock`, `is_edit_locked`,
   `wait_for_edit_lock_release`, `patch_lock`) and the `changespec` re-exports
   unchanged. `patch_lock`'s timeout fix from sase-1sb.5 is correct; leave it alone.

### Required tests (in `tests/ace/changespec/test_edit_lock.py`)

- **Deterministic interleaving regression.** Make the first contender block inside its
  lock-content write until the second contender's `acquire_edit_lock` has returned or
  raised. Assert exactly one owner and one `FileExistsError`, and that the surviving
  lock file names the owner.
  - Also cover a pause between temp write and `os.link`.
  - Use events/barriers with bounded timeouts, not sleeps.
- **Fallback path.** With `os.link` forced to raise `EPERM` and the first writer paused
  mid-write, the second contender gets `FileExistsError`: a fresh empty lock is treated
  as held.
- **Unparseable locks.** A fresh unparseable lock blocks acquisition. An unparseable
  lock aged past the grace window (`os.utime`) is reclaimed. Update
  `test_acquire_reclaims_malformed_edit_lock` to age its file. This is an intended
  semantic change, so say so in the test docstring.
- **Release ownership.** `release_edit_lock` leaves a lock owned by another PID/token in
  place and still removes its own.
- `test_concurrent_edit_lock_acquisition_has_one_owner` and every other existing test in
  the file still pass. Rerun the file 10 times in a shell loop to show it is stable.

## Defect 2: repeated model-output placeholders (from phase sase-1sb.1)

File: `src/sase/macro/workflow_executor_utils.py::render_prompt_template`. Tests:
`tests/test_prompt_authorship_safety.py`.

`render_prompt_template` swaps each `__SASE_MODEL_OUTPUT_<nonce>_<n>__` placeholder for
its `wrap_disabled_region(...)` block with a single `rendered.find(placeholder)`. Only
the first occurrence is replaced. A prompt template that interpolates the same value
twice (e.g. `A {{ step._raw }} B {{ step._raw }}`) leaves a literal placeholder token in
the prompt the agent receives. Verified on master.

### Required changes

- Replace every occurrence of each placeholder, applying the same newline padding at
  each occurrence: prefix gets a trailing `\n` when non-empty and not already
  newline-terminated, and the suffix gets a leading `\n` likewise. The nonce makes
  collisions with protected text impossible; keep it.
- Extend the docstring: model output (`_raw`/`_response`) in prompt templates is
  interpolation-only. Jinja conditionals and transforming filters operate on an opaque
  placeholder, which keeps the fail-closed property. Branch on model output in an `if:`
  step condition or a bash/python step, and point to task `sase-1sn` for the broader
  design.

### Required tests

- Interpolating `{{ step._raw }}` twice yields two protected regions, no
  `__SASE_MODEL_OUTPUT_` substring, and the literal model text in both.
- Pass that render through `preprocess_prompt_late(..., agent_authored=False)` with a
  `$(...)` payload that would create a marker file in `tmp_path`. The marker must not
  exist and the literal `$(...)` text must survive. Follow the existing tests in
  `tests/test_prompt_authorship_safety.py`.

## Verification

1. Run the focused files: `tests/ace/changespec/test_edit_lock.py`,
   `tests/test_prompt_authorship_safety.py`, and the other workflow render tests that
   import `render_prompt_template` (`grep -rln render_prompt_template tests`).
2. Run `just check` through `sase tool run check`. Do not run `just check-full`; this
   landing does not require it. A failure in an area these two diffs do not touch, which
   passes in isolation, is a flake. Triage it through `/sase_new_task` rather than
   treating it as landing work.

## Closeout (mandatory, same turn as the code)

1. Run `sase bead epic-symbols sase-1sb`. For every listed `--epic-symbol` entry,
   resolve the symbol per the Symvision epic-whitelist policy (wire it up, privatize it,
   add a non-test pragma, or delete it). Re-key the Justfile line only if a still-open
   later bead needs the exemption. The list is empty today; add no new entries keyed to
   `sase-1sb`.
2. Close the epic normally:

   ```bash
   sase bead close sase-1sb --note "<verification>"
   ```

   The note states what was verified:
   - the landing audit in epic note #2 (all phases, DECISIONS honored, integration
     reviewed)
   - both fixes, with their regression tests and the passing `sase tool run check` ID
   - the triage outcomes: sase-1sg +1; new tasks sase-1sm and sase-1sn; sase-1sa
     pre-existing; edit-lock flake fixed here
   - epic-symbols empty

   Never use `--force`. If the close is rejected, fix the cause and close again.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file:
   `plan:202611/security_bug_audit_fixes.md`, at the PLAN path printed by
   `sase bead read sase-1sb`, in the plans sidecar. It currently says `status: wip`.
5. `sase-1sb` has no `parent_bead`, so there is no parent to close.
