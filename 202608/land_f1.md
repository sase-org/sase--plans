---
tier: tale
title: Land zero-friction model alias defaults
goal:
  Epic sase-f1 is acceptance-tested on the integrated tree, its follow-ups are dispositioned, and it is closed cleanly.
size: medium
proposed_by: bbugyi200.athena.sase-f1.land
bead: sase-f1
create_time: 2026-08-03 16:49:14
status: done
---

- **PARENT:**
  [202608/zero_friction_model_alias_defaults.md](https://github.com/sase-org/sase--plans/blob/main/202608/zero_friction_model_alias_defaults.md)
- **BEAD:** [sase-f1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-f1/README.md)

# Land epic sase-f1 after end-to-end acceptance verification

## Objective

Finish the active land phase of epic bead `sase-f1`, prove that edits to every shipped model-alias target and
description require no companion source or test edits, integrate post-epic-start changes, dispose of every child
`PROPOSED FOLLOW-UP:` note under the SASE task policy, and close the epic cleanly.

## Verified starting state

- `sase bead show sase-f1` links `@plan:202608/zero_friction_model_alias_defaults.md`. The epic itself has no notes.
- Children `sase-f1.1`, `.2`, and `.3` are closed as done; `.4` is the active land phase. Every child and child note has
  been read.
- The implementation commits are on `master`:
  - `2d87ba544` (`sase-f1.3`) de-hardcodes the doctor warning plus schema/default-config prose.
  - `5c76b3d4b` (`sase-f1.1`) splits and hardens the defaults parser, installs the frozen test defaults, re-pins the
    value-coupled tests, and adds real-file shape/negative-path coverage.
  - `568a96524` (`sase-f1.2`) generates the one surviving defaults table from YAML via `just fmt`, removes value-pinning
    prose, and deletes the docs freshness test.
- Source review confirmed the parser validates fallback references, selector grammar, terminal fallback chains, and
  cycles; the frozen test map is distinct from the shipped values while matching graph shape; the generated-docs tool is
  wired only into `fmt`/`fix`; and current docs contain shipped literals only in the generated block and model registry
  facts.
- All non-epic commits after `2d87ba544` were reviewed. Their only overlaps are prompt-duality removals in docs,
  `sase.schema.json`, and `default_config.yml`; they do not conflict with or duplicate the alias feature. The later
  `sase-f2.3`/`.4` changes removed both unused symbols named by the `sase-f1.4` Symvision proposal.
- The child proposals are:
  1. The same two Config Center visual snapshot failures proposed by `.1` and `.2`. They semantically match ready task
     `sase-bl`; post-start commit `376a3b1bb` changed the implicated fixture default, so rerun the exact nodes on the
     integrated tree before deciding whether new corroboration is still useful.
  2. `.2` reported a new watched-temp entry named `pytest-clean`. No task matches that exact symptom. It may have been a
     one-off/environmental scratch-root interaction; use the acceptance/full-check result and current source evidence to
     decide whether it is reproducible enough for a new task. If not, explicitly decline it as non-reproducible.
  3. `.3` reported the known full-suite lock-timeout flake. It is a semantic duplicate of in-progress task `sase-e2`
     (and ready duplicate `sase-dy`); `sase-e2` already contains a `+1 sw` entry from this alias-default verification,
     so do not double-count the same report.
  4. `.4` reported unused `load_xprompt_source_records` and `render_prompt_sections`. Both names are absent from current
     `src`, tests, and `Justfile`, removed by the causally related active epic `sase-f2`; no standalone task is needed.

## Implementation and verification

1. Reconfirm a clean primary worktree, then run `just install` as required for this ephemeral workspace.
2. Establish the integrated untouched-tree baseline. Run the exact two Config Center visual nodes and the previously
   listed non-visual baseline nodes as useful, then run the untouched full `just check`. Record every failure and rerun
   any load-sensitive node alone. Do not attribute unrelated failures to `sase-f1`.
3. Use `apply_patch` to change every `target:` and every `description:` in
   `src/sase/llm_provider/model_alias_defaults.yml` to valid values distinct from both the shipped values and
   `tests/_model_alias_defaults_fixture.py`; preserve alias graph shape. While that YAML is the only edited file, run
   the full `just check` and `just test-visual`. A failure caused by the perturbation is unfinished epic work: fix it,
   restore the perturbation, and repeat the full acceptance run.
4. With the perturbation still present, run `just fmt`. Confirm the only additional diff is the generated block in
   `docs/llms.md`, and a second `just fmt` is idempotent.
5. Exercise the packaged loader through `.venv/bin/sase doctor` with one temporary invalid YAML edit at a time: unknown
   fallback, two-alias fallback cycle, and malformed selector pool. Confirm each failure names the defaults resource and
   offending alias. Restore between cases.
6. Restore `model_alias_defaults.yml` and the generated docs block to committed content without destructive broad
   commands. Run `just fmt`, confirm the primary worktree is clean, and record the acceptance outcome on `sase-f1.4`.
7. Finish proposal triage under `/sase_new_task`: corroborate a true semantic duplicate at most once, attach a causally
   owned issue to its active epic, create a sized ready task only for a distinct reproducible issue, and explicitly
   document every declined proposal. Avoid a second `+1` for evidence already recorded by reporter `sw`.

## Final landing phase

1. Close phase `sase-f1.4` normally with a note summarizing the exact perturbation, untouched and perturbed full-check
   results, visual result, fmt-heal/idempotence proof, loader negative paths, post-start integration review, and every
   follow-up outcome.
2. Close the parent with `sase bead close sase-f1 --note "<complete verification and integration summary>"`. Do not use
   `--force` merely to bypass unfinished descendants; if closure names unfinished phases, finish or deliberately reopen
   them.
3. After the epic closes, run `just symvision`. Remove stale `sase-f1` whitelist entries and unused code it reports,
   then rerun `just symvision`. If source changes result, run `just install` and the required full `just check` before
   finishing.
4. Open the plans sidecar through `/sase_repo` if needed, set `status: done` in the frontmatter of
   `202608/zero_friction_model_alias_defaults.md`, and verify both the epic and plan status. Record any resulting file
   changes accurately; do not commit unless explicitly authorized.
