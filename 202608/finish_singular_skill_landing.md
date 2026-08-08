---
tier: epic
title: Finish and land the singular xprompt skill namespace epic
goal: "Epic sase-hi closes only after a published core binding provides singular skill
  references, the latest primary tree adopts that release alongside post-start core and
  Python work, post-close Symvision is clean, and the linked plans are done.

  "
phases:
  - id: release_compatible_core_binding
    title: Publish and verify the compatible core binding
    depends_on: []
    size: medium
    description:
      "release_compatible_core_binding: publish the release-plz-managed binding that
      contains the singular skill contract and later compatible core work, then verify
      the exact distribution."
  - id: adopt_release_and_reconcile_primary
    title: Adopt the release and integrate the latest primary tree
    depends_on:
      - release_compatible_core_binding
    size: medium
    description:
      "adopt_release_and_reconcile_primary: require the published binding, reconcile
      later Patch and artifact-ref adoption, and prove the combined singular-skill
      contract with full repository gates."
  - id: land_sase_hi
    title: Verify, close, clean, and complete the epic
    depends_on:
      - adopt_release_and_reconcile_primary
    size: small
    description:
      "land_sase_hi: repeat the bead, commit, source, and follow-up audit; close
      sase-hi; run post-close Symvision cleanup; and mark the linked plans done."
proposed_by: bbugyi200.athena.sase-hi.land
parent_bead: sase-hi
create_time: 2026-08-08 14:50:28
status: wip
---

- **PROMPT:**
  [prompts/202608/finish_singular_skill_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/finish_singular_skill_landing.md)
- **PARENT:**
  [202608/xprompt_skill_singular_namespace.md](https://github.com/sase-org/sase--plans/blob/main/202608/xprompt_skill_singular_namespace.md)

# Finish and land the singular xprompt skill namespace epic

## Goal

Finish epic bead `sase-hi` only after a released `sase-core-rs` binding actually
provides the singular skill-reference contract, the primary repository consumes that
released contract on top of all work that landed after the epic began, and the original
epic is closed with every child note and proposed follow-up accounted for.

## Verified starting state

The original epic plan is `202608/xprompt_skill_singular_namespace.md`. Its three phase
beads are closed, and their implementation commits are:

- `sase-hi.1`: `sase-core` commit `8a0db5999a9f4dd3a64031cf31ca994151535fc8` splits the
  plural physical `skills/` directory contract from singular `skill/` xprompt references
  and moves the package locator to `package:xprompts/skills`.
- `sase-hi.2`: primary-repository commit `92f0ff3774ca867ee971cedb092045d2a4824262`
  moves bundled Markdown and the frame template under `src/sase/xprompts/skills/` while
  retaining Python modules in `src/sase/skills/`.
- `sase-hi.3`: primary-repository commit `54c1436cd27fdcd8015ea33faa745bf42c2e5883` cuts
  CLI, ACE, editor, docs, and fixtures over to singular references and adds provider
  context to inline skill expansion.

The source and tests contain the intended implementation, but the released dependency
does not. Tag `sase-core` `v0.20.1` is the parent of `8a0db599`; the primary repository
still requires and locks `sase-core-rs>=0.20.1,<0.21.0`; and an exact installed 0.20.1
wheel reports content-layout schema 3, returns `skills/foo` from `skill_reference_name`,
accepts `#skills/sase_plan`, and rejects `#skill/sase_plan`. This is an epic-caused
release/adoption blocker, not a follow-up task.

Two more unreleased core commits landed after `sase-hi.1` and must be integrated rather
than lost or duplicated: `8344869` establishes the compatible Patch/stitch core contract
for active epic `sase-hn`, and `4071bf0` adds the artifact-ref contract for active epic
`sase-ho` and advances the content-layout schema from 4 to 5. The latter's Python
adoption phase is `sase-ho.2`; the former's is `sase-hn.2`. Start every phase from
current remote state and reconcile work those epics land while this continuation runs.

The four primary-repository commits interleaved between `92f0ff377` and `54c1436cd` were
audited: `8037b9496` changes agent refresh/publication, `47cad6a02` repairs post-epic
plan-link assertions, `38fd25afd` replaces a timed contract oracle with a manifest
budget, and `6de3ff745` mechanically splits xprompt argument-assist tests. None
currently duplicates or conflicts with the singular skill contract, but the final
integration audit must include them and every newer commit.

The child proposals already have these preliminary outcomes:

- `sase-hi.1`'s invalid `research-highlights` file-hook proposal was fixed by chezmoi
  commit `ed0b708c15305f175dd4e0081604bb1d41579710`; live `sase file-hook list --json`
  now returns the hook with nested schema-3 `filters`. No task is warranted.
- `sase-hi.2`'s compatible-binding proposal is the epic-caused work in phases 1 and 2
  below, not a separate task.
- `sase-hi.3`'s stale deterministic-failure classification was duplicate-corroborated on
  ready task `sase-hl`; its three broad-xdist/pass-serial failures were
  duplicate-corroborated on in-progress umbrella task `sase-ct`; and both causal
  outcomes were recorded as a `DISCOVERED ISSUE` note on active epic `sase-h8`. No new
  task bead was needed.

## Phase 1 — `release_compatible_core_binding`

**Size:** medium  
**Depends on:** nothing in this continuation

Open `sase-core` through the `sase_repo` skill and refresh it to current remote state.
Re-read the bead-tagged commits and actual source. Confirm that the release candidate
contains `8a0db599` and every later core commit already on the canonical branch,
including `8344869` and `4071bf0`; preserve the singular skill reference namespace,
plural external source directories, `package:xprompts/skills`, and schema-5-or-newer
fields introduced by the later work.

Run the core repository's formatting, strict Clippy, full workspace tests, and focused
content-layout, catalog, editor, gateway, LSP, and PyO3 tests. Do not hand-edit Cargo
versions: versions are release-plz-owned, and the `BREAKING CHANGE` metadata on
`8a0db599` must determine the compatible released series. Use the repository's normal
release-plz workflow and wait for its tag, GitHub release, sdist, and supported wheels
to publish. If a compatible release has already appeared, verify it instead of
triggering a duplicate release.

Install the exact published wheel in a disposable environment with no linked-checkout
override. Verify its tag contains all expected core commits and exercise the installed
API directly:

- global/project reference generation yields `skill/foo` and `app/skill/foo`, and the
  inverse accepts those singular shapes while rejecting `skills/foo`;
- `sase_content_layout` exposes the current canonical schema and retains plural
  project/home/plugin skill paths plus `package:xprompts/skills`;
- the packaged/catalog contract loads bundled skills from `xprompts/skills`, exposes
  singular hash references, and retains bare provider skill names for slash invocation;
- later Patch/stitch and artifact-ref wire additions remain present and their focused
  tests pass.

Do not close `sase-hi` in this phase. Record the exact released version, tag, commit
ancestry, package provenance, verification commands, and results on the continuation
phase bead.

## Phase 2 — `adopt_release_and_reconcile_primary`

**Size:** medium  
**Depends on:** `release_compatible_core_binding`

Start from the latest primary-repository canonical branch, not the historical
`54c1436cd` tree. Inspect the current status and landed commits for active epics
`sase-hn` and `sase-ho`, especially `sase-hn.2` and `sase-ho.2`, before editing. Reuse
their Python domain/content-layout adoption if it has landed; do not recreate or
overwrite in-flight work. Re-audit every primary commit since `92f0ff377`, excluding the
two `sase-hi` commits, for new skill fixtures, catalog consumers, dependency
constraints, schema assertions, or source paths that should adopt the completed singular
contract.

Update `pyproject.toml` and `uv.lock` to require the actual compatible published binding
series (expected from the current breaking batch to be 0.21.x, but use the release-plz
result rather than guessing). Update version-floor smoke assertions and content-layout
schema/wire tests to the current released contract. If artifact-ref adoption has landed,
preserve its schema-5-or-newer fields; if it is still active, coordinate the minimal
non-overlapping dependency/schema change and verify against its latest canonical result
before completing this phase.

Confirm the complete source contract in the combined tree:

- external project, home, project-home, and plugin authoring remains in plural `skills/`
  directories;
- bundled Markdown and `SKILL.frame.template.md` ship only under
  `src/sase/xprompts/skills/`, while `src/sase/skills/` contains only Python command
  code;
- built-in, project, and home skills expand through `#skill/<name>` or
  `#<project>/skill/<name>` and resolve to the same source used by `/<name>`;
- `#skills/<name>` neither expands nor completes, and the remaining plural-reference
  string is only an explicit negative regression;
- provider rendering still receives its declared template context and generated provider
  installation targets remain unchanged;
- wheels include the nested package resources and ordinary xprompt scanning does not
  double-load them.

Run `just install` before repository checks as required. Run focused Rust-binding,
content-layout, package-resource, skill-loader, expansion, CLI, editor, LSP-facing, and
ACE regression tests; installed-wheel smoke against the exact published distribution;
the singular/legacy stale-string audit; `just docs-check`; `just build-check`;
`just test-visual`; and `just check-full`. Distinguish host-local selection-health or
parallel-flake failures already tracked by `sase-hl`/`sase-ct` from real regressions,
but do not waive an active failure. Resolve every issue caused by this epic before
handing off, and route genuinely external new work through `sase_new_task` with the
proposing bead identified.

Do not close `sase-hi` in this phase. Record the exact dependency range, lockfile
version and hashes, installed-package provenance, post-start commit audit, focused/full
gate results, and any follow-up routing on the continuation phase bead.

## Phase 3 — `land_sase_hi`

**Size:** small  
**Depends on:** `adopt_release_and_reconcile_primary`

This is the required final landing phase. Refresh all involved repositories and verify
the combined canonical tree rather than trusting earlier notes. Run
`sase bead show sase-hi`, review the epic's complete history and linked plan, then show
every child and review every child note/history again. Re-read the actual implementation
source and all commits tagged with `sase-hi*` in both repositories. Re-run the
since-start integration audit, including the interleaved primary commits, the
post-`sase-hi.1` core commits, and anything newer. Confirm phases 1 and 2 addressed the
installed 0.20.1 counterexample and that no epic-caused issue remains.

Reconcile every `PROPOSED FOLLOW-UP:` entry and include every outcome in the close note:
the resolved chezmoi hook migration, the now-published/adopted binding, the `sase-hl`
and `sase-ct` duplicate corroborations, the `sase-h8` causal note, and any later
proposal or task outcome. Use `sase_new_task` before recording any genuinely new
external task.

Close the original epic without force:

```bash
sase bead close sase-hi --note "<detailed verification, integration, gates, and every follow-up outcome>"
```

If close is rejected, finish or deliberately reopen the named descendant; never force
merely to succeed. Immediately after a successful close, run `just symvision` if the
recipe exists because `sase-hi` whitelist entries have just expired. Before fixing any
reported Symvision issue, use the audited `sase_memory_read` workflow for
`symvision.md`. Remove stale `sase-hi` whitelist entries and genuinely unused code, then
run `just check` for any primary-repository file changes (or `just check-full` when the
broadening set requires it).

Open the plans sidecar through `sase_repo`. Set `status: done` in the frontmatter of the
original `202608/xprompt_skill_singular_namespace.md` and in this continuation's durable
linked plan file, preserving all other provenance. Make those final plan edits durable
through the normal SASE finalizer/commit workflow. Finish only after the epic is closed,
post-close Symvision is clean, both plan files say `done`, and all required verification
results and follow-up outcomes are recorded.

## Completion criteria

The work is complete only when a clean install from the published binding distribution
uses singular inline skill references, plural external source directories and nested
builtin package resources remain correct, the legacy plural xprompt namespace is
rejected, all later core/primary work is integrated, the proportionate full gates pass,
`sase-hi` is closed without force, post-close Symvision is clean, and both linked plan
files are marked done.
