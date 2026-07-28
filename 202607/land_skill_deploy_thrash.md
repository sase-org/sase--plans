---
tier: tale
title: Land epic sase-ae with its missing ABA regression test
goal: The ABA revert that epic sase-ae was created to stop is blocked by an automated
  test that drives the real `sase skill init` handler, and epic sase-ae is closed
  with its plan file marked done.
bead: sase-ae
create_time: 2026-07-28 10:04:44
status: done
---

- **PROMPT:** [202607/prompts/land_skill_deploy_thrash.md](prompts/land_skill_deploy_thrash.md)
- **PARENT:** [202607/skill_deploy_thrash.md](https://github.com/sase-org/sase--plans/blob/main/202607/skill_deploy_thrash.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ae.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ae.land.md#member-code)
  - [bbugyi200.athena.sase-ae.land--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ae.land.md#member-plan)
- **COMMITS:**
  - [7d85188](https://github.com/sase-org/sase/commit/7d85188c18080e4e986e8fd65394144c8ae9ce2f) — test(skills): cover backwards manifest ABA refusal (sase-ae)

# Plan: Land epic sase-ae with its missing ABA regression test

## Why This Plan Exists

Epic `sase-ae` ("Stop `sase skill init` skill-deployment thrashing", plan `plans:202607/skill_deploy_thrash.md`) is
substantively complete. A land-agent audit confirmed every phase against the source, the commits, and the live chezmoi
repo, with one exception.

The `converge` phase (`sase-ae.5`) named one check as **"the regression test for the whole epic — reproduce the original
ABA against the fixed code and show it is now blocked."** That bead closed with no note, no close reason, and no commit
in the sase repo. No such test exists anywhere in `tests/` or `smoke/`. If the ABA reproduction was run at all, it was a
one-off manual check that leaves nothing behind to protect the invariant.

This plan lands that test and then closes the epic.

## What Is Already Verified — Do Not Redo

The audit confirmed all of the following. Treat them as done and do not re-litigate them:

- **`sase-ae.1` guard** — `src/sase/main/_init_skills_source_integrity.py` refuses a chezmoi deploy on dirty
  `src/sase/xprompts/` or a HEAD that is not an ancestor of the canonical branch; `-D/--allow-dirty` is the escape
  hatch. Handler-level tests cover refusal, `--allow-dirty`, `--diff`, `--check`, and non-chezmoi mode.
- **`sase-ae.2` manifest** — `src/sase/main/_init_skills_manifest.py` writes `.sase-skills-manifest.json` with
  `source_commit`, `xprompt_set_sha256`, and `deployed_at`, and refuses backwards or divergent sources.
  `tests/main/test_init_skills_manifest.py` covers all seven relation cases at the `prepare_skill_manifest` unit level.
- **`sase-ae.3` serialize** — `deploy_to_chezmoi` holds a bounded `flock` for the whole read-compare-write-commit-push
  span, reused by the memory and config init deploys; the direct and deferred paths both emit `SASE_SOURCE_REVISION`,
  `SASE_WORKSPACE`, and `SASE_AGENT` trailers. Verified live in the chezmoi deploy commit.
- **`sase-ae.4` reconcile** — the canonical `src/sase/xprompts/skills/sase_beads.md` carries the `### history`,
  `### note`, and `### dep` sections; every documented option (`history --lost-notes --restore`, `note --author`,
  `dep tree --direction`, `dep rm`, `close --force --resolution`) exists in the CLI.
- **`sase-ae.5` convergence (partial)** — `sase skill init --diff` from a clean `origin/master` workspace produces
  **no** output, so the deployed skill set is byte-reproducible from master. The manifest records a source commit that
  is an ancestor of master, and the orphaned `home/dot_gemini/skills/` tree is gone from chezmoi.
- **`sase-ae.6` docs** — the commit-then-deploy workflow is documented in `sase/memory/generated_skills.md`,
  `sase skill init --help`, `docs/init.md`, `docs/xprompt.md`, and `docs/configuration.md`.
- **Integration** — no commit landed since the epic started touches the skill-deploy path. Every other chezmoi writer
  (`sase memory init`, `sase config init`, bare `sase init`) already funnels through the shared `deploy_to_chezmoi`
  lock. The doctor's next-step guidance matches the new documented workflow. CI is unaffected because `use_chezmoi`
  defaults to `false`, and CI is green on master.

The only gap is the missing end-to-end refusal test described below.

## The Gap

`tests/main/test_init_skills_handler.py` covers the handler wiring for the **source-integrity** refusal
(`test_handler_dirty_chezmoi_source_is_refused_before_write`) but there is no equivalent for the **manifest** refusal.
`tests/main/test_init_skills_manifest.py` exercises `prepare_skill_manifest` in isolation, never through
`run_init_skills`. So nothing verifies that a stale workspace actually fails to overwrite a newer deployment: the
handler could write files before consulting the manifest, drop the refusal, or leak the deploy lock, and every existing
test would still pass.

## Work

### 1. Share the manifest git stub

`tests/main/test_init_skills_manifest.py` has a private `_stub_git(...)` helper that fakes `run_git` and
`get_sase_package_xprompts_dir` on the `sase.main._init_skills_manifest` module, driving `rev-parse HEAD` and
`merge-base --is-ancestor` from an explicit ancestor set.

Promote it to `tests/main/init_skills_handler_helpers.py` as a public `stub_manifest_git(...)` and import it from
`test_init_skills_manifest.py` so there is one stub, not two. Keep its behavior identical; this is a move, not a
rewrite.

### 2. Add the ABA regression test

Add to `tests/main/test_init_skills_handler.py`:

`test_handler_backwards_manifest_refuses_and_leaves_newer_deployment_intact`

Set up the exact shape of the original incident — a stale workspace deploying over a newer deployment:

- `stub_skill_source(tmp_path, monkeypatch)`, `get_use_chezmoi` → `True`, `CHEZMOI_HOME` → a `tmp_path` chezmoi home.
- Pre-write the rendered target `SKILL.md` with **different, newer** content than this workspace renders, standing in
  for another agent's deployment.
- Pre-write `.sase-skills-manifest.json` recording a `source_commit` that is a **descendant** of the stubbed incoming
  HEAD.
- Use `stub_manifest_git` so the real `prepare_skill_manifest` resolves the recorded commit as newer — the `backwards`
  relation. Do **not** monkeypatch `prepare_skill_manifest` itself; the point is to exercise the real decision logic
  through the real handler.
- Monkeypatch `init_skills_handler._deploy_to_chezmoi` with a `MagicMock`.

Assert all of:

- exit code is `1`;
- the pre-written `SKILL.md` bytes are **unchanged** (this is the no-revert assertion and the heart of the test);
- `.sase-skills-manifest.json` still records the newer commit;
- `_deploy_to_chezmoi` was never called;
- stderr names both SHAs and mentions `--force`.

`test_handler_force_overrides_backwards_manifest_and_records_incoming`

Same setup with `make_args(force=True)`. Assert exit code `0`, the `SKILL.md` is overwritten with the rendered content,
the manifest now records the incoming commit, and `_deploy_to_chezmoi` was called.

Name the incident in a short docstring on the first test so a future reader knows what it protects: a later run
restoring a strictly older `sase_beads/SKILL.md` and discarding two prior deployments.

### 3. Verify

- `just install` first — workspaces are ephemeral, so this is not optional.
- Run the focused suite:
  `pytest tests/main/test_init_skills_manifest.py tests/main/test_init_skills_handler.py tests/main/test_init_skills_sources.py tests/main/test_init_skills_deploy.py tests/main/test_init_skills_plan.py`
  (88 tests pass today; expect 90 after this change).
- Confirm the new test genuinely fails against the pre-guard behavior: temporarily make `prepare_skill_manifest` return
  `(write, None)` for the backwards relation, watch the test fail, then revert. Do not commit that mutation.
- Run `just check`. Its SASE validation stage is expected to stop on pre-existing plans-sidecar artifact-link errors
  unrelated to this change; every other stage must pass.
- Commit with your `/sase_git_commit` skill, referencing `sase-ae.5`.

## Land

Do this only after the work above is committed.

1. `sase bead close sase-ae`. If the close is rejected, the named phases were not complete — finish or reopen them.
   Never force merely to make the command succeed.
2. Run `just symvision`. Epic-symbol whitelist entries for `sase-ae` expire at close; remove any stale entries and
   unused code it reports. The audit found no `sase-ae` whitelist pragmas in `src/` or `tests/`, so a clean run is
   expected — but run it and act on whatever it reports.
3. Set `status: done` in the frontmatter of `plans:202607/skill_deploy_thrash.md`.

## Out Of Scope

Noted during the audit, deliberately not included:

- Teaching `sase doctor` to report manifest provenance drift (for example, chezmoi deployed from a commit that is not an
  ancestor of your HEAD). A reasonable follow-up, but new feature work the epic never promised.
- Extending the chezmoi deploy lock to the ACE prompt-bar xprompt save path. That path commits a single user xprompt
  rather than doing a read-compare-write over a generated set, so it is a different failure mode.
- The manifest bumping on every deploy from a newer source commit even when the rendered skill set is byte-identical.
  That is the recorded-provenance behavior the epic chose on purpose.
