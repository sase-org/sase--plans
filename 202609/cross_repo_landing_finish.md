---
tier: tale
size: medium
title: Finish cross-repo landing and close sase-1cq
goal:
  Secure revision-pin writes against symlink escapes, finish the landed skill and plan
  closeout, verify the integrated epic, and close sase-1cq normally.
status: done
proposed_by: bbugyi200.athena.sase-1cq.land
bead: sase-1cq
---

- **PARENT:**
  [202609/cross_repo_landing.md](https://github.com/sase-org/sase--plans/blob/main/202609/cross_repo_landing.md)
- **BEAD:**
  [sase-1cq](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1cq/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1cq.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1cq.land.md)
- **COMMITS:**
  - [859140f](https://github.com/sase-org/sase/commit/859140f025decf1e11051412b9a8f6fac53f1252)
    — feat(finalizer): secure revision_pin against symlink escapes with doctor check and
    tests

# Finish cross-repo landing and close sase-1cq

Epic `sase-1cq` has three closed phases. Its plan is
`plan:202609/cross_repo_landing.md`. The land audit confirmed the core pin contains both
`0541387` and `1ad57ea`, `sase-1ck.4.1` and phase `sase-1ck.4` are closed, the
revision-pin finalizer and after-turn messaging are in source, and later commit
`8c38eb6a9a` resolved the unused bead-attachment symbols proposed by phases `.1` and
`.2`. The terminology fixture failure proposed by phases `.1` and `.3` is already
recorded on active parent attachment epic `sase-1ck` note 1; the current terminology
audit still reports 14 unclassified fixture tokens. The follow-up outcomes are recorded
on `sase-1cq` in a `LAND TRIAGE` note, so do not file duplicate tasks for these
proposals.

## Finish the epic-owned gaps

1. In `src/sase/finalizers/commit_revision_pin.py`, ensure a configured `revision_pin`
   cannot read or write through a symlinked path component outside the primary checkout.
   Reject or skip such a pin with `revision_pin` evidence and a diagnostic before
   creating directories or replacing files. Make the doctor check in
   `src/sase/doctor/checks_config_repos.py` report an escaping symlinked pin path. Keep
   legitimate relative pin paths working. Add focused tests using a temporary primary
   checkout and an external directory symlink; prove that the external file is
   untouched. Consider Windows drive-absolute spellings in the config normalizer as
   well. The contract is that pin files stay in the primary checkout; this is remaining
   `pin_follow` work, not a standalone task.
2. `plan:202609/note_cli.md` still has `status: wip`, though phase `sase-1cq.1` reported
   that it had marked it done. Open the plans repo with `sase repo open plans` and set
   this child plan to `status: done` after confirming the already closed `sase-1ck.4.1`
   and `sase-1ck.4` still satisfy it. Use `sase artifact read` for plan content and do
   not hand-edit the bead store.
3. The landed source `src/sase/xprompts/skills/sase_final.md` has the after-turn
   sentence, but the generated `sase_final` skill in chezmoi and the installed Codex
   skill lack it. Read `generated_skills.md`, open chezmoi with
   `sase repo open chezmoi`, and preview with the workspace
   `.venv/bin/sase skill init --diff` (the global `sase` executable renders an older
   installed source). Regenerate from the clean landed source without making a manual
   git commit; use the command's no-commit option so the host finalizer can commit the
   changed chezmoi repo, then apply the generated files to the home directory. Inspect
   all generated changes and confirm the `sase_final` sentence in the deployed copies.
   Follow chezmoi's apply instructions to the extent possible before the host-owned
   after-turn commit.

## Verify and land

4. Re-read `sase-1cq` and all three child notes, review the feature commits
   `c257a3f220`, `2fe7c6500f`, and `339a67306b` and the later master commits through
   current HEAD, and check that no later consumer needs integration with `revision_pin`
   or the after-turn message. Run the focused revision-pin and submit-output tests,
   `just fix`, and `sase tool run check` (the `just check` recipe). Do not run
   `just check-full`. At this audit's HEAD, `just symvision` also reports three
   unrelated unused symbols from later commits: `AltGroup`, `tool_run_duration_fit`, and
   `validate_sync_ceiling_seconds`. Record current outcomes accurately and triage any
   newly discovered unrelated issue through `/sase_new_task`; do not attribute
   pre-existing terminology or later-commit lint failures to the pin work. Resolve any
   actual epic-caused failure before closing.
5. Run `sase bead epic-symbols sase-1cq` immediately before close. Resolve every listed
   entry or re-key it only to a still-open later bead that needs the exemption. Then
   close the epic normally with
   `sase bead close sase-1cq --note "<verification of phases, later commits, tests, integration, and every follow-up proposal>"`.
   Do not use `--force` merely to bypass a descendant or stale symbol; finish the named
   issue and retry. Run `just symvision` after closing and report any unrelated failures
   while confirming no stale `sase-1cq` whitelist entry remains. Finally set
   `status: done` in `plan:202609/cross_repo_landing.md` through the opened plans repo.
   `sase-1cq` has no `parent_bead`, so this is the final landing step.
