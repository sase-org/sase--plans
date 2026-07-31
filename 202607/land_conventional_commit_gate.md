---
tier: tale
title: Finish and land the Conventional Commit subject gate
goal: Epic sase-bj is fully integrated with concurrent changes, verified, closed normally, and recorded as done.
bead: sase-bj
create_time: 2026-07-31 09:43:56
status: done
---

- **PROMPT:** [202607/prompts/land_conventional_commit_gate.md](prompts/land_conventional_commit_gate.md)
- **PARENT:**
  [202607/conventional_commit_subject_gate.md](https://github.com/sase-org/sase--plans/blob/main/202607/conventional_commit_subject_gate.md)
- **BEAD:** [sase-bj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-bj/README.md)

# Finish and land epic `sase-bj`

## Goal

Complete the remaining integration work for the Conventional Commit subject gate, verify the result against the current
canonical branches, disposition every `PROPOSED FOLLOW-UP:` entry from the four closed phase beads, and land epic
`sase-bj` without forcing its close.

The land audit already confirmed that all four phase beads are closed with `resolution=done` and that their source work
exists in the canonical histories:

- sase-core `d8cb1e2` adds the parser and Python binding; `6b52cb0` (also tagged to `sase-bj.2`) fixes the planned
  `fix:`/`empty_description` edge case.
- sase `748b617c0` adds the facade, config/schema, validation presenter, and init-memory reuse; `84721922e` gates all
  three commit methods before cwd lookup or any workflow side effect; `2f565d0be` adds docs, generated-skill source
  guidance, and the PR-title type drift guard.
- Focused epic coverage is green (72 tests), the entire Rust workspace passes, Rust fmt and Clippy pass, and
  `sase plan links validate` passes.

Do not redo those implementations. Address the concrete remaining integration gaps below.

## Phase 1: Reconcile concurrent changes and finish deferred deployment

1. Run `just install` before repository checks. Preserve unrelated user changes if any have appeared.
2. Read `generated_skills.md` through the `sase_memory_read` skill, and use the `sase_repo` skill before accessing the
   linked chezmoi checkout. From the clean, canonical sase source, preview `sase skill init --diff` and confirm the
   expected drift is limited to the provider renderings of `sase_git_commit` and `sase_beads`. Then run
   `sase skill init --force`; this is the deferred post-merge deployment required by the generated-skills workflow and
   performs its own chezmoi commit/push/apply sequence. Verify `sase skill init --check` exits successfully and the
   chezmoi checkout is clean and synchronized. Do not use `--allow-dirty`.
3. Reconcile the visual corpus with changes that landed while the epic was in progress. Current baseline from
   `just test-visual` is **53 failed, 339 passed, 1 skipped**. Representative artifacts under
   `.pytest_cache/sase-visual/` show intentional UI changes from the concurrent commits: the Admin Center alternate-
   section footer (`0002a0590`), promoted-family status (`1273c78a9`), project display-name handling (`22e78f792`), and
   model alias presentation/state (`e5361f4de`, `90d8c3ac6`). Inspect at least one expected/actual/diff/source artifact
   from each affected family before accepting it. Use `just update-visual-snapshots` only after confirming the frames
   are intentional, review the resulting PNG-only diff, and rerun `just test-visual` to exact green. Do not update a
   frame whose difference cannot be traced to the current source/history.
4. Re-run the focused commit-subject suites:

   ```bash
   .venv/bin/pytest -q \
     tests/core/test_commit_subject_facade.py \
     tests/workflows/test_commit_message_validation.py \
     tests/workflows/test_commit_workflow_message_gate.py \
     tests/main/test_init_memory_commit_message.py \
     tests/test_pr_title_type_drift.py \
     tests/main/test_init_skills_sources.py
   ```

5. Run `just check` after the PNG changes. Keep the known model-label unit failure below out of this epic unless current
   evidence shows it is required for visual convergence; it is a separate follow-up, not part of the commit gate.

## Phase 2: Disposition follow-ups and record final evidence

Collect the current notes again with `sase bead show sase-bj.1` through `.4`, then handle their proposals as follows.
Avoid duplicate task beads by searching first.

1. File one ready task bead for the duplicated phase 1/2 performance proposal. Suggested title:
   `Cache agent-name registry validation during bead-page association rendering`. Explain that
   `HostedLinkResolver.agent_url()` reaches `lane_ref_for_lane_name()` → `get_reserved_family_names()` once per
   association, and outside `name_registry_load_session()` every cached registry access recomputes `_source_signature()`
   across artifact/dismissed-bundle paths. Cite both `sase-bj.1` and `sase-bj.2`, request a bounded load session or
   equivalent cached snapshot plus a regression/performance test, and mark the task `ready`.
2. File one ready task bead for the still-reproducing model label proposal. Suggested title:
   `Reconcile Codex provider-label casing in model completion metadata`. Cite `sase-bj.2`, `sase-bj.3`, and `sase-bj.4`.
   Record that
   `tests/test_xprompt_model_completion.py::test_model_completion_catalog_reflects_real_builtin_model_metadata`
   currently gets `codex (gpt53spark)` while expecting `Codex (gpt53spark)`, whereas the targeted model completion and
   Models-panel visual tests currently pass. Ask the task worker to choose and enforce one human-facing provider-label
   contract rather than merely weakening the assertion. Mark the task `ready`.
3. File one ready task bead for the additional audit finding that `sase/memory/generated_skills.md` still describes
   runtime-specific `sase_hg_commit` availability, contradicting the current Tier-1 `Uniform Agent Runtimes` rule. State
   that editing the memory note itself requires explicit user approval. Search first and cite this land audit as
   provenance.
4. Do **not** file duplicate plan-link tasks: `cf2aba89a` fixed the approval race, the durable plan and prompt now have
   reciprocal links, and `sase plan links validate` passes.
5. Do **not** file a separate visual-snapshot task after Phase 1 makes the corpus green. Explain in the close note that
   the 53 stale frames were integrated directly because they reflected commits landing during the epic.
6. Do **not** file a generated-skill deployment task after Phase 1 completes the canonical deploy. If deployment fails
   for a genuine external reason, stop and report it rather than closing the epic or papering over it with a task.

Before closing, assemble a concise evidence note naming the verified main/core commits, focused and Rust results, visual
reconciliation, generated-skill deployment/check, plan-link validation, `just check`, the ready task IDs, and the
reasons the other proposals were not filed.

## Phase 3: Land the epic (final phase)

1. Re-run `sase bead show sase-bj` and confirm every descendant is still closed. Close normally, without `--force`:

   ```bash
   sase bead close sase-bj --note "<the evidence assembled in Phase 2>"
   ```

   If the close is rejected, finish or reopen the named work. Never force a done resolution merely to succeed.

2. After the close, use `sase_memory_read` to read `symvision.md` before changing anything Symvision reports. Run
   `just symvision`; remove only stale `sase-bj` epic-symbol whitelist entries and genuinely unused code it identifies,
   then rerun Symvision. If this changes the sase checkout, rerun `just check` as required by repository instructions.
3. Use `sase_repo` to open the plans sidecar and change the frontmatter of `202607/conventional_commit_subject_gate.md`
   from `status: wip` to `status: done` **after** the bead close. Preserve the reciprocal PROMPT/BEAD links and make no
   unrelated plan edits.
4. Verify the final state: `sase bead show sase-bj` reports `closed`/`done`, each new task bead reports `ready`, the
   durable epic plan reports `status: done`, `just symvision` is green, `sase skill init --check` is green, and every
   modified checkout contains only intentional changes. Report any known unrelated test failure explicitly rather than
   claiming a globally green suite.
