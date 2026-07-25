---
tier: tale
title: 'Stop writing the `COMMIT: <sha>` note to beads on commit'
goal: 'Committing a bead-carrying change no longer overwrites the bead''s `notes`
  field with a stale `COMMIT: <sha>` value, while the post-commit bead-store amend
  safety net keeps working.'
create_time: 2026-07-25 09:53:27
status: done
---

- **PROMPT:** [202607/prompts/drop_bead_commit_note.md](prompts/drop_bead_commit_note.md)

# Plan

## Problem

`sase commit` currently stamps `COMMIT: <sha>` into the bead's `notes` field after every commit. This has two
independent defects, both confirmed by reading the code:

**1. The recorded sha is virtually always wrong.**

In `src/sase/vcs_provider/plugins/_git_commit_dispatch.py`, `_post_commit_bead_amend()` reads `HEAD` _before_ mutating
and amending:

```python
rev = self._run(["git", "rev-parse", "--short", "HEAD"], cwd)   # line 317
commit_hash = rev.stdout.strip() if rev.success else "unknown"

self._run(
    ["sase", "bead", "update", bead_id, "--notes", f"COMMIT: {commit_hash}"],  # line 321
    cwd,
)

changed_files = self._changed_bead_files(cwd)
if changed_files:
    add_out = self._run(["git", "add", "--"] + changed_files, cwd)
    if add_out.success:
        self._run(["git", "commit", "--amend", "--no-edit", "--quiet"], cwd)   # line 330 -- rewrites HEAD
```

The note write itself dirties the bead store, which triggers the `git commit --amend` two lines later. The amend
rewrites `HEAD`, so the sha just written into the note no longer exists on the branch. `vcs_create_commit()` then runs
`_sync_with_origin_after_commit()` (line 364), whose rebase can rewrite the sha _again_. The commit hash that
`vcs_create_commit` returns at line 372-373 — the one that lands in the ChangeSpec `COMMITS` entry — is re-read after
all of that, which is why the ChangeSpec hash is right and the bead note is wrong.

**2. It destroys notes an agent left behind.**

`notes` is a single `TEXT` column (`src/sase/bead/db.py:34`), and `sase bead update --notes` assigns rather than appends
(`src/sase/bead/cli_crud.py:137-138`: `fields["notes"] = args.notes`). Any working notes an agent recorded on the bead
during its run are silently replaced by the one-line `COMMIT:` stamp. `vcs_finalize_commit()` (line 438-442) calls the
same helper again on `sase commit --resume`, so a resumed commit clobbers a second time.

`docs/commit_workflows.md:256` describes step 6 as "Post-commit bead amend (append bead note)" — the docs claim an
append that the implementation never performed.

## Why the note can simply be deleted

Nothing reads it. A repo-wide search for `COMMIT:` consumers finds only the single write site in
`_git_commit_dispatch.py`; every other `COMMIT`-prefixed match is the unrelated ChangeSpec `COMMITS:` section parsing
(`src/sase/ace/changespec/parser.py`, `src/sase/workflows/commit_utils/*`,
`src/sase/workflows/commit/commit_tracking.py`). The historical `COMMIT: <sha>` strings living in
`sase/repos/plans/beads/events/streams/*.jsonl` are the persisted result of past writes, not consumers.

The bead-to-commit association is already recorded durably and correctly elsewhere:

- The ChangeSpec `COMMITS:` entry carries the real post-rebase hash.
- The bead itself carries `changespec_name` / `changespec_bug_id` (`src/sase/bead/model.py:63-64`), linking it to that
  ChangeSpec.

So removing the note loses no information.

The Rust core has no counterpart to remove: `crates/sase_core/src/bead/` models the `notes` field but never writes a
`COMMIT:` value, and commit dispatch lives entirely on the Python plugin side. This change is confined to this repo.

## What to keep

Keep the post-commit amend of changed bead files. Once the note write is gone it is a no-op on the normal
`vcs_create_commit` path (bead changes are already staged pre-commit by `_stage_bead_dirs`, line 348), but it remains a
real safety net on the `vcs_finalize_commit` resume path: after a human hand-resolves a rebase conflict, straggler bead
files can be left in the working tree, and folding them into `HEAD` keeps the committed bead store consistent.

Also keep the existing `bead_id` guard, so the amend still runs only for bead-carrying commits. Widening it to every
commit would be a behavior change beyond the scope of this fix.

## Implementation

All source changes are in `src/sase/vcs_provider/plugins/_git_commit_dispatch.py`.

1. **Delete the note write.** In `_post_commit_bead_amend()` (lines 311-330), remove the `git rev-parse --short HEAD`
   call, the `commit_hash` local, and the `sase bead update ... --notes` invocation. Keep the `bead_id` guard and the
   `_changed_bead_files` / `git add` / `git commit --amend --no-edit --quiet` block.

2. **Rename the helper to match its narrowed purpose.** Rename `_post_commit_bead_amend` to `_amend_bead_changes` and
   update both call sites (line 362 in `vcs_create_commit`, line 442 in `vcs_finalize_commit`). Keep the
   `_skip_bead_amend` payload key as-is — it is documented in `docs/commit_workflows.md:220`, is consumed by
   `create_proposal` delegation, and still reads correctly.

3. **Rewrite the docstring** so it states the remaining contract: fold any bead-store changes that appear after the
   commit into that commit; explicitly note that it no longer writes bead notes, and why (the sha would be stale after
   the amend/rebase, and `--notes` overwrites rather than appends).

4. **Update the docs.**
   - `docs/commit_workflows.md:256`: change step 6 from "Post-commit bead amend (append bead note)" to wording that
     describes folding straggler bead-store changes into the commit — drop the bead-note claim entirely.
   - `docs/commit_workflows.md:220`: reword the `_skip_bead_amend` row's Purpose so it no longer implies a note write.
   - Verify the remaining "bead amend, push with retry" phrasings (`docs/commit_workflows.md:355`,
     `docs/commit_workflows.md:401`, `docs/vcs.md:49`) still read correctly; they describe the amend, not the note, so
     they should need no change. Confirm rather than assume.

## Tests

Add regression coverage to `tests/test_vcs_provider_bare_git_plugin.py`, next to
`test_vcs_create_commit_stages_concrete_bead_files` (line 140), which already builds the fixture you need:

1. **`vcs_create_commit` writes no bead note.** Reuse that test's `handler`-based `subprocess` mock with
   `bead_id="sase-1"` in the payload, then assert no recorded command starts with `["sase", "bead", "update"]` and that
   no command contains `--notes`.

2. **`vcs_finalize_commit` writes no bead note either.** Same assertion against the resume path, since it was the second
   clobber site.

3. **The amend safety net survives.** Assert that when `git ls-files` reports changed bead files after the commit, the
   `git add -- <bead files>` plus `git commit --amend --no-edit --quiet` pair still runs. This is what stops the fix
   from being over-applied into a full removal of the helper.

`test_vcs_create_commit_stages_concrete_bead_files` itself should stay green unchanged: it asserts only on `git add`
shapes and the returned hash, never on `sase bead update`.

## Out of scope

Beads whose notes were already overwritten by past commits are **not** repaired here. The prior note values are still
recoverable from the `issue_updated` events in `sase/repos/plans/beads/events/streams/*.jsonl`, so a backfill remains
possible as separate work if the user wants it. Do not attempt it as part of this change.

Do not change `sase bead update --notes` assignment semantics into an append. Replace-on-write is the right behavior for
an explicit user-facing field edit; the bug was the automated caller, not the CLI.

## Verification

This repo requires `just check` after file changes, and workspaces are ephemeral, so install first:

```bash
just install
just check
```

Then confirm the behavior directly:

```bash
grep -rn "COMMIT: " --include=*.py src/    # must return no matches
```
