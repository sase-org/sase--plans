---
tier: tale
size: medium
title: Finish and land the on-demand memory web rosters epic (sase-1v7)
goal: The on-demand roster epic's leftover defects are fixed, its skill text is live
  in every provider, `sase tool run check` passes, and epic sase-1v7 is closed with
  its plan file marked done.
decisions:
  decisions_memory:
    ask: Finish the epic's decisions-web memory edits (supersession mark, full record,
      preamble line)?
    memory:
    - decisions
    - decisions:rosters-load-on-demand
    - decisions:webs-render-in-their-own-section
    default: true
    corrects: web descriptor always renders its full roster in generated instructions
      is retired.
    answer: true
proposed_by: bbugyi200.athena.sase-1v7.land
proposed_with: claude/opus
bead: sase-1v7
decided_by: auto
status: done
---

- **PARENT:**
  [202611/on_demand_web_rosters.md](https://github.com/sase-org/sase--plans/blob/main/202611/on_demand_web_rosters.md)
- **BEAD:**
  [sase-1v7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1v7/README.md)

# Finish and land on-demand memory web rosters (epic sase-1v7)

## Context

Epic `sase-1v7` (plan `plan:202611/on_demand_web_rosters.md`; run
`sase bead read sase-1v7 -r "<why>"` for its notes) added `roster_loading: on_demand`,
the generated roster pointer, the audited `sase memory read roster:<web>` read, the
bare-web guard (`-a/--all-strands`), TUI surfaces, and the adoption by the sase and
bob-cli decisions webs. All five phases are closed, and the code is on master in commits
`e489558571`, `1a468da521`, `47cec6ea49`, and `8e6818d0a6`. bob-cli's adoption is landed
in bob-cli at `d13f8d1d`. The land agent verified the epic and found the gaps below. The
land agent recorded the full triage in its `LAND TRIAGE` note on `sase-1v7`. This tale
fixes those gaps and then closes the epic. Nothing else is in scope. bob-cli's
pre-existing test failures belong to task `sase-1v9`, not to this tale.

Governing decisions: `decisions:rosters-load-on-demand`,
`decisions:webs-render-in-their-own-section`, `decisions:readable-agents-md`,
`decisions:check-full-is-explicit`, and `decisions:guarded-recipes`. Read the
`generated_skills.md`, `lint_and_test.md`, and `docs_writing.md` memory notes with
`/sase_memory_read` before the steps that touch those areas. Use `/sase_memory_write`
before editing any memory note.

## Steps

### 1. Deploy the epic's skill text

Phase `sase-1v7.4` reported a skill deploy, but it ran the global `sase`. That tool is
an editable install of the primary sase checkout, which sat at `e489558571` (phase 1).
So the chezmoi skill manifest (`~/.local/share/chezmoi/home/.sase-skills-manifest.json`)
records `source_commit: e489558571…`. The deployed `sase_memory_read` and `sase_plan`
skills therefore lack the `roster:<web>` text that phase 2 landed in
`src/sase/macros/skills/`. Those sources are already on master, so deploy them from this
workspace before editing anything:

1. `just install-venv` if `.venv` is missing or stale.
2. Run `.venv/bin/sase skill init --diff`. It will probably also show **deletions** of
   the `mermaid_diagrams` and `sase_spotify` skills for every provider. This workspace's
   venv lacks the plugins that provide them. The global tool has them as editable
   installs. Find their source paths in
   `~/.local/share/uv/tools/sase/lib/python*/site-packages/_editable_impl_sase_{spotify,research_artifacts}.pth`.
   Install them into this workspace's venv only:
   `uv pip install --python .venv/bin/python --no-deps -e <spotify src parent> -e <research-artifacts src parent>`.
   Pass each project root, the directory that holds its `pyproject.toml`, not its
   `src/`. If an import fails for a missing dependency, install that dependency by name.
   Never let a plugin install replace the workspace's editable `sase`.
3. Rerun `.venv/bin/sase skill init --diff`. Proceed only if every change is an
   overwrite of a `sase_memory_read/SKILL.md` or `sase_plan/SKILL.md` that adds the
   `roster:` text, with no deletions or other skills. Otherwise do not deploy. Instead,
   record a `sase bead note sase-1v7` naming the exact diff and the deploy command, and
   continue.
4. Deploy with `.venv/bin/sase skill init -y`. If the provenance-manifest guard refuses,
   check `git merge-base --is-ancestor <manifest source_commit> HEAD`. If it succeeds,
   the deploy only moves forward, so rerun with `--force` as `generated_skills.md`
   prescribes. Otherwise stop and note it as in sub-step 3. Never pass `--allow-dirty`.
5. Verify: `.venv/bin/sase skill init --diff` is empty, the manifest's `source_commit`
   is this workspace's `HEAD`, and `~/.claude/skills/sase_plan/SKILL.md` contains
   `roster:<web>`.
6. Restore the venv: run
   `uv pip uninstall --python .venv/bin/python sase-spotify sase-research-artifacts`,
   then `.venv/bin/python tools/setup_required_plugins`. This restores the pinned
   `sase-research-artifacts`, so later completion-spec and test runs match CI.

### 2. Fix the on-demand "empty authored body" blocker

`_roster_loading_blockers` in `src/sase/memory/web/validation.py` checks
`strip_managed_roster_markers(web.body).strip()`. That helper removes only the marker
lines and keeps the roster entries. So the blocker fires only when the managed region is
empty. A real descriptor whose body is nothing but a populated `<!-- sase:strands -->`
region passes validation, and the pointer then has no authored prose. The land agent
reproduced this with a one-strand fixture: no blockers, no warnings.

- Add a helper in `src/sase/memory/web/roster.py` that returns the body with the whole
  marker-delimited region removed. It must remove the markers and everything between
  them, and return the body unchanged when no markers are present. Use it in the
  blocker. Keep the helper public only if Symvision requires it.
  `render_web_projection_body` already slices the region the same way, so share the
  logic if that stays simple.
- Add a test to `tests/memory/test_memory_web_roster_loading.py`. Give an on-demand
  fixture web a body that holds only a **populated** region, and assert the blocker
  fires. Keep the existing empty-region test. Also assert that the default `_seed` body
  (preamble plus region) raises no such blocker.

### 3. Regenerate the CLI completion snapshot

Phase 2 added `-a/--all-strands` and new `roster:<web>` help text, but never synced the
snapshot. So `tests/completion/test_snapshot.py::test_checked_in_snapshot_has_no_drift`
and `::test_current_structural_view_matches_checked_in_snapshot` fail on master. Run
`just sync-completion-spec` after step 1's venv restore. It is not part of `just fix`.
Commit the updated `tests/completion/snapshots/cli_spec.json`. Confirm that the diff
touches only the `memory read`/`memory show` options and help digests.

### 4. Memory fixes (decision `decisions_memory`)

Use `/sase_memory_write` first. Edit only these three notes.

1. `sase/memory/decisions/webs-render-in-their-own-section.md`:
   - Change `superseded_by: decisions:rosters-load-on-demand` to
     `superseded_by: decisions/rosters-load-on-demand`, the `decisions/<slug>` form
     every other record uses. The `web:keyword` form breaks
     `tests/test_render_design_decisions.py::test_real_decisions_web_renders_every_record`
     on master. It also renders as `decisions:rosters-load-on-demand` instead of a slug
     in the roster's supersession marker.
   - Replace the back-link blockquote with the house style, which states what is
     retired. The current text calls "every web descriptor always renders its full
     roster" this record's claim, but the record never says that. What is retired is the
     Cost sentence "there is no longer a way to make one read-on-demand", for rosters
     only. For example:
     `> _Superseded in part:_ the Cost sentence "there is no longer a way to make one read-on-demand" is retired for rosters: a web may now load its roster on demand ([[decisions/rosters-load-on-demand]]). The descriptor body itself still always renders in the Memory Webs section.`
   - Edit nothing else in that record.
2. `sase/memory/decisions/rosters-load-on-demand.md`: phase 4 wrote a thin record. Bring
   it up to the epic plan's spec (its `adopt-sase` step 2). Keep the slug, the keyword
   `Rosters Load On Demand`, and the existing claim. Keep the record concise.
   - **Why:** the measurements at `df9ce4ef55`:
     - the sase decisions section was 8,408 B, 38% of the root bundle
     - the bob-cli section was 25,641 B, 69% of its bundle
     - bob-cli's Codex bundle was 4.5 KB over its 32,768 B cap, so compaction ran every
       turn
     - researchers make 78% of decision reads
     - a bare `sase memory read decisions` printed every body by accident (about 69 KB
       in sase)

     Add the outcome. sase's AGENTS.md went from 21,076 B to 13,868 B. bob-cli's
     decisions section went from 19,095 B to 1,775 B, and its Codex bundle from 30,954 B
     (compacted) to 14,031 B (uncompacted). Add the structural reason too:
     per-invocation bundles are frozen at spawn, so the host cannot know whether an
     agent will plan, and the pointer must be standing text.

   - **Rejected alternatives:** relying on the compaction ladder; a titles-only roster;
     reviving `type:` on descriptors; `when:` audience-scoped bundles; a flat reference
     note as the index; semantic search. Also list the research's rejected command
     shapes if
     `sase artifact read research:202611/on_demand_web_rosters/on_demand_web_rosters.md "<why>"`
     resolves. If it does not, omit them; the land agent could not resolve that ref from
     its workspace.
   - **Cost:** ambient discovery is lost for unplanned work; compliance is social in v1
     because nothing enforces the read; `roster` becomes a reserved web slug; there is
     one more divergence between the on-disk descriptor and its projection.
   - **Reopens when:** planner compliance measured from `memory_reads.jsonl` stays poor.
     The fix then is the deferred `sase plan validate` roster check or a titles-only
     `roster_loading: compact` mode. It also reopens if the host gains a reliable
     spawn-time fact that an agent will plan.
   - **Evidence:** cite the research ref as plain text, and link
     `[[decisions/webs-render-in-their-own-section]]`.

3. The `decisions` web descriptor: rewrite only the preamble's last sentence so it no
   longer duplicates the pointer's strand-read line. Drop the
   ``Read one on demand with `sase memory read decisions:<keyword> -r "<why>"`;`` clause
   and keep "each record states the claim, why it was chosen over the credible
   alternatives, what it costs, and the condition that would reopen it", recapitalized.
   Do not touch the frontmatter or the managed region.

Then run `sase memory init`. It rewrites the managed roster region, the memory README,
AGENTS.md, and CLAUDE.md.

### 5. Memory panel docs

`docs/reference/tui/prompt-completion-panels.md` (the Memory panel section, at the
sentence beginning "A web row instead shows `WEB` plus `EXPANDED`/`COLLAPSED`") does not
describe what phase 3 added. Add one or two sentences covering:

- the `ON DEMAND` badge
- the `Roster` row reading `list · on demand`
- the `Read when` trigger rows
- the `Last roster read` row (agent · reason · age, after an agent reads `roster:<web>`)
- the rail row's trailing `· on demand`

Stay within the page's docs budget.

### 6. Verify

1. Run `just fix`. It regenerates `docs/reference/design-decisions.md`, the glossary
   docs, and CLI help.
2. Run the targeted tests:
   `.venv/bin/python -m pytest -q tests/completion/test_snapshot.py tests/test_render_design_decisions.py tests/memory/test_memory_web_roster_loading.py tests/memory/test_memory_selector_roster.py tests/main/test_init_memory_memory_webs.py`.
   All must pass.
3. Run `sase memory init --check`. It must be clean.
4. Run `sase memory show roster:decisions`. The superseded entry for
   `webs-render-in-their-own-section` must read
   `[partly superseded by `rosters-load-on-demand`]`.
5. Run `sase tool run check`. Never run `just check-full`. Treat UNKNOWN failures as
   yours.

### 7. Close the epic (final step)

1. Run `sase bead epic-symbols sase-1v7`. It was empty at landing time. If any
   `--epic-symbol` entries appear, resolve each per `symvision.md`: wire the symbol up,
   privatize it, add a non-test pragma, or delete it. No later open bead needs these
   exemptions.
2. Run
   `sase bead close sase-1v7 --note "<what you verified: the steps above, test and check results, deploy outcome>"`.
   Never use `--force`. If the close is refused, fix the cause and close again.
3. Run `just symvision` and confirm the whitelist is clean.
4. In the epic's plan file (`plan:202611/on_demand_web_rosters.md`; the PLAN path shown
   by `sase bead read sase-1v7`), change the frontmatter `status: wip` to
   `status: done`.
5. `sase-1v7` has no `parent_bead`, so nothing further is needed. Do not wait for this
   work's own commit, push, or CI before closing. Closing in the same turn as the final
   code is the normal landing.
