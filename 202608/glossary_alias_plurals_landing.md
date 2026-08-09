---
tier: tale
title: Finish and land glossary alias plural derivation
goal:
  The shipped glossary behavior is accurately documented, fully verified, and closed out
  with clean post-close state.
proposed_by: bbugyi200.athena.sase-i3.land
bead: sase-i3
create_time: 2026-08-09 09:35:21
status: done
---

- **PROMPT:**
  [prompts/202608/glossary_alias_plurals_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/glossary_alias_plurals_landing.md)
- **PARENT:**
  [202608/glossary_alias_plurals.md](https://github.com/sase-org/sase--plans/blob/main/202608/glossary_alias_plurals.md)
- **BEAD:**
  [sase-i3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-i3/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-i3.land](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-i3.land.md)
- **COMMITS:**
  - [8aaeb59](https://github.com/sase-org/sase--plans/commit/8aaeb593e283543a7a92eb804a02c32f183e3b0c)
    — docs(plan): mark glossary alias plural plan done

# Finish and land epic `sase-i3`

## Goal

Correct the one phase-2 release-documentation omission found by the land audit, then
close epic `sase-i3`, run its post-close Symvision cleanup, and mark its linked plan
done. Preserve the already-landed glossary behavior and the unrelated changes that
landed while the epic was in progress.

## Verified context

- `sase bead show sase-i3` reports three closed phases and links
  `plans:202608/glossary_alias_plurals.md`. The epic itself has no pre-existing notes.
- Every child and every child note was reviewed. The implementation reports correspond
  to core commit `5c555dcda69367e31b64edc57d487f0b4a464b5c`, core release commit
  `c416cd0b7db4fbf61be4523f3c9ecbe037361a9b` / tag `v0.21.2`, and sase commit
  `b73609337d9bd1e7be6184bd4cd97f16cb342683`.
- The current Rust source separates authored, display, and effective aliases; derives
  conservative plurals; keeps validation on authored aliases; serializes the defaulted
  `display_aliases` field without changing glossary schema version 1; and leaves LSP
  hover on configured aliases. The Python facade, LSP payload, memory renderer,
  generated instructions, docs, dependency floor, and regression tests consume that
  contract as planned.
- The generated glossary matches the acceptance table exactly: only Agent Hoods, Agent
  Instruction Files, Agent Neighbors, Repo, and xprompt Memory retain shortened
  `ALIASES:` lines. No generated file contains title-case `Aliases:`.
- A clean Python 3.12 environment installed `sase-core-rs==0.21.2` from PyPI and the
  published wheel returned `display_aliases: []`, derived `Widget Boxes`, and matched
  that plural. `cargo test --workspace glossary` passed the 10 core glossary tests plus
  3 LSP glossary tests and 1 config-glossary test. The 27 focused sase glossary tests
  passed, and `sase memory init --check` passed.
- Post-start integration was audited on the final tree. The ACE glossary underline
  commits, xprompt semantic-token documentation, generated `Glossary of Terms` title,
  bead-search work, new-task guidance, schema compatibility fix, BY_DATE fix, and sase
  release metadata all compose with the epic. The final docs retain underline and
  derived-plural guidance, the generated files retain the new title, direct
  `GlossaryEntry` fixtures carry `display_aliases`, and the dependency remains
  `sase-core-rs>=0.21.2,<0.22.0`. No additional main-repo integration edit is needed.

## Remaining epic defect

Phase `sase-i3.2` said the release-plz changelog entry covered the glossary change, but
the actual `v0.21.2` changelogs list only `*(bead)* add regex search support`. The tag
does contain the glossary commit, so runtime publication is correct; the missing release
note is still an unmet phase requirement and must be repaired as epic work.

## Phase 1: Correct and verify the released changelog

1. Use `/sase_repo` to open `sase-core` and use only the path it returns.
2. Reconfirm that `5c555dc` is an ancestor of `v0.21.2` and inspect the release snapshot
   before editing. Do not rewrite the tag, bump a version, or publish another release:
   this is a changelog correction for behavior already present in 0.21.2.
3. Under the existing `0.21.2` Added section in both `crates/sase_core/CHANGELOG.md` and
   `crates/sase_core_py/CHANGELOG.md`, add the missing glossary feature using the
   repository's release-plz style, e.g.
   `- *(glossary)* derive plural aliases for matching`. Keep the existing bead-search
   entry.
4. Inspect the diff and run the repository's formatting/check appropriate for these
   changelog-only edits. Re-run `cargo test --workspace glossary` to ensure the release
   correction is based on the still-valid shipped implementation.
5. From the sase repo, re-run the focused acceptance suite:

   ```bash
   just install
   .venv/bin/python -m pytest \
     tests/main/test_init_memory_glossary.py \
     tests/test_core_glossary_facade.py \
     tests/xprompt/test_glossary_catalog.py \
     tests/ace/tui/widgets/test_prompt_glossary.py -q
   PATH="$PWD/.venv/bin:$PATH" sase memory init --check
   ```

   Also confirm the dependency floor and the five retained generated alias lines did not
   drift. Do not regenerate or edit memory files unless `--check` reports a genuine
   mismatch caused by this work.

## Phase 2: Land and close (final phase)

1. Process the sole `PROPOSED FOLLOW-UP:` from `sase-i3.3` deliberately in the close
   record. Its two VCS-tag selector flakes are already tracked by in-progress task
   `sase-hk`, ready duplicate `sase-cw`, and active flake epic `sase-h8`; the plan
   approval node is already corroborated on umbrella task `sase-ct` and active epic
   `sase-h8`. The proposing phase observed only the selection-health baseline report,
   not an independent test failure. Current `just selection-health` dynamically
   classifies and suppresses all three nodes. Therefore create no task and add no +1:
   explain that this proposal was declined as duplicate, non-independent evidence that
   is already owned and automatically classified.
2. Record a complete close note covering: all bead/child notes reviewed; the three epic
   commits and actual source verified; published-wheel behavior; the generated alias
   table; post-start integration results; the corrected v0.21.2 changelogs; verification
   commands/results; and the follow-up disposition above.
3. Close normally, without force:

   ```bash
   sase bead close sase-i3 --note "<complete verification and follow-up outcome>"
   ```

   If close is rejected, inspect and deliberately finish or reopen the named child;
   never force a successful resolution merely to bypass validation.

4. Only after the close succeeds, run `just symvision`. Remove any now-expired `sase-i3`
   epic-symbol whitelist entries and unused code it reports. If that changes files in
   the sase repo, run `just check` as required by project instructions and include the
   cleanup in the landed change.
5. Use `/sase_repo` to open `plans`, edit the linked `202608/glossary_alias_plurals.md`,
   and change only its frontmatter status from `wip` to `done`. Confirm the epic is
   closed, the plan says `status: done`, all involved worktrees contain only the
   intended changes, and normal SASE commit/ publication finalizers have completed.

## Definition of done

- Both v0.21.2 changelogs accurately mention the shipped glossary plural-alias feature.
- The existing Rust/Python/LSP/memory behavior and all post-start integrations remain
  verified and unchanged.
- The unrelated flake proposal has an explicit duplicate/non-independent disposition in
  the close note, with no redundant task or +1.
- `sase-i3` is closed normally, post-close Symvision is clean, and
  `plans:202608/glossary_alias_plurals.md` has `status: done`.
