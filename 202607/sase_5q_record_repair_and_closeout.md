---
create_time: 2026-07-12 10:42:34
status: wip
prompt: 202607/prompts/sase_5q_record_repair_and_closeout.md
tier: tale
---
# Plan: Repair the split-SDD store record clobber, harden record handling, and close out sase-5q

## Product context / goal

Epic **sase-5q** ("Split the SDD companion repo into `--plans` and `--research` linked repos") is functionally complete:
all seven phase beads are closed, the code landed on master (`c13664dc6`/`5df88d7ca` (5q.1), `4c40d5af8` (5q.2),
`0bbd3cb50` (5q.3), `4976cdbd8` (5q.4), `75ee0fb6a` (5q.5), plus `eb624fa8e`/`d5cf97450` refactors and `sase-github`
plugin commits `5a2eb57`..`f9bd0b1`), the migration was executed on 2026-07-11 evening (`sase-org/sase--plans` and
`sase-org/sase--research` created public, content migrated, `sase-org/sase--sdd` archived on GitHub), and the system ran
correctly on companion storage for ~11 hours.

A post-migration review found the deployment was then **silently regressed by a stale process**, plus three code gaps
this epic should fix before the epic bead is closed. This plan (a) repairs the damaged state and salvages stranded data,
(b) hardens the code so this failure class cannot repeat, and (c) verifies and closes out the epic.

## Diagnosis (verified evidence)

**The incident.** At 2026-07-12 13:29:39Z a process still running **pre-split sase code** (the axe daemon/launch
pipeline had been up across the upgrade; it was only restarted at 13:33Z/14:00Z) performed SDD store materialization for
the sase project:

1. Pre-split `_load_sdd_store_record` does not recognize `storage: "companion_repos"` (not in its `_STORAGE_VALUES`), so
   it returned `None` — indistinguishable from a malformed/stale record.
2. `_materialize_sdd_store` therefore **deleted** the schema-v2 companion record at
   `/home/bryan/projects/github/sase-org/sase/.sase/sdd-store.json` and re-probed the provider.
3. The GitHub probe (`gh repo view --json name`) reported the **archived** `sase-org/sase--sdd` repo as `found` (it
   never checks `isArchived`), so a **schema-v1 `separate_repo` record** was written (current file contents, mtime
   2026-07-12 09:29:42 EDT) and a fresh legacy clone was re-created at `.sase/sdd`.
4. From then on all SDD writes routed to legacy clones, whose pushes **silently fail** against the read-only archived
   repo (`~/.sase/bead_push_logs/push-260712_09*.log`, `10*.log`: "This repository was archived so it is read-only").

**Stranded content inventory** (exists only in unpushable legacy clones):

- Primary checkout `.sase/sdd` (2 commits ahead of the archived origin): bead events closing the **sase-5m** family (5
  `issue_closed` events, 14:15:09Z) and re-closing sase-5q.6/sase-5q.7 (redundant — the plans companion already has both
  closed via different events; note the event sequence numbers collide, so these streams must never be file-merged).
- Numbered-workspace legacy `.sase/sdd` clones (discover via `git rev-list --count @{u}..HEAD` in each
  `<workspaces-root>/sase_*/`): three clones each hold 2 unpushed commits ("Add SDD files for X" / "Complete SDD plan
  for X") for `tui_pump_starvation_freeze` (its agent is still running), `fix_agent_deltas_commit_provenance`, and
  `prompt_stash_panel_leader_keymap` — plan + prompt-snapshot files under `plans/202607/` that never reached
  `sase--plans`.
- This plan file itself will be proposed while the record is still broken, so its own plan/prompt files will land in a
  legacy clone and must be salvaged by Phase 1 like the rest.

**Code gaps (in-scope epic defects):**

- **G1 — forward-compat clobber (this repo).** Current `_load_sdd_store_record`
  (`src/sase/sdd/_store_records.py:114-116`) still returns `None` for unrecognized `storage` values, and
  `_materialize_sdd_store` (`src/sase/sdd/store.py:303-307`) still deletes such records and re-probes. The next
  record-schema evolution will reproduce this exact disaster on any not-yet-restarted daemon or older install.
- **G2 — archived-repo adoption (sase-github plugin).** `_probe_github_repo_detail`
  (`src/sase_github/workspace_plugin.py`) queries only `name`; archived repos probe as `found` and get adopted,
  producing silent push black holes. Also affects a fresh machine bootstrapping a migrated project with no local record:
  it would rediscover and adopt the archived `--sdd`.
- **G3 — doctor blind spot (this repo).** `src/sase/doctor/checks_config_sdd.py` flags companion-record problems
  (`missing-*-companion-clone`, `legacy-sdd-after-migration`) but has no check for the **reverse** mixed state we are in
  now: a `separate_repo`/missing record while split companion clones exist for the project. Nothing surfaced the
  regression for five hours.

## Phase 1: State repair and data salvage (no code changes)

Objective: restore companion routing, salvage all stranded SDD data, retire resurrected legacy clones.

1. **Salvage first, while writes are still few.** For each legacy clone with unpushed commits (primary + numbered
   workspaces; re-enumerate at execution time since agents are running):
   - Copy stranded `plans/202607/<name>.md` and `plans/202607/prompts/<name>.md` into the plans companion clone at
     `202607/<name>.md` / `202607/prompts/<name>.md`, rewriting `prompt:`/`plan:` frontmatter links to the post-split
     form (drop the `[.sase/]sdd/plans/` prefix — same rewrite `sase.sdd.migrate` applies).
   - Do NOT copy `beads/` files (event-ID collisions); bead state is replayed via the CLI in step 3.
   - Commit to the plans companion with a clear message (e.g. "Restore SDD files stranded by store-record regression")
     and push. Salvage the still-running agent's plan files early so its finalizer finds them in the companion store
     once the record is repaired.
2. **Restore the companion record.** Prefer the supported path: `sase sdd init` for the sase project (with
   `--check`/plan preview first) — both companions exist on GitHub so this is pure adoption (no repo creation, no typed
   confirmation). Before running it, move the resurrected legacy clone aside (`.sase/sdd` → `.sase/sdd.clobber-260712`)
   so no legacy-import path re-ingests it. If init insists on anything interactive, fall back to writing the schema-v2
   record with `sase.sdd.write_sdd_store_record` (companions: `sase-org/sase--plans` / `sase-org/sase--research`,
   matching `_record_to_json`'s shape exactly).
3. **Replay stranded bead state via the CLI** (now routed to the plans companion): close `sase-5m.1`–`sase-5m.4` and
   `sase-5m` (their closes at 14:15:09Z exist only in the stranded clone). sase-5q.6/.7 need no replay (already closed
   in the companion store).
4. **Retire legacy clones.** After confirming each has no remaining unique commits: tar the retired clones into
   `~/.sase/backups/` as a safety net, then delete the primary `.sase/sdd.clobber-260712` copy and every numbered
   workspace's `.sase/sdd` (one workspace also carries a months-stale clone with a half-staged historical migration in
   its index — same treatment). Remove stale per-workspace `.sase/sdd-store.json` files (only the primary record is
   authoritative). Leave the pre-migration `.sase/sdd.post-split-legacy` backup at the primary in place.
5. **Verify:** `sase bead show sase-5q` works from a numbered workspace; `sase sdd path plans|beads|research` resolve
   into companion clones; `sase bead list` sane; a bead touch (e.g. the step-3 closes) auto-commits and pushes to
   `sase--plans` with a clean push log; `sase validate` and `sase doctor` green (modulo the known pre-existing
   warnings).

## Phase 2: Record forward-compat hardening + doctor check (this repo)

Objective: make unrecognized store records inert instead of deletable, and make the regressed state visible.

- **G1 fix.** Distinguish three record-file states in `sase.sdd._store_records`: (a) recognized record → current
  behavior; (b) genuine stale negative cache (parses, recognized storage, `discovery: not_found`) → still replaceable;
  (c) **foreign record** (valid JSON dict whose `storage` is unrecognized, or `schema_version` above the highest this
  build knows, or unparseable content) → materialization must NOT delete or overwrite it; raise
  `SddMaterializationError` with an actionable message ("SDD store record was written by a newer/unknown sase version —
  upgrade/restart sase; refusing to touch it"). Wire the guard into `_materialize_sdd_store`'s delete/re-probe sites;
  resolution (`resolve_sdd_store`) should surface the same clear failure rather than silently falling back to in-tree
  paths.
- **G3 fix.** New doctor diagnostic in `checks_config_sdd.py`: when the effective record is missing or `separate_repo`
  but split companion clones (`<repo>--plans` with a `beads/` dir, `<repo>--research`) exist in the host checkout's
  linked-repo clone area, emit an error-level "sdd-record-regressed" issue explaining the likely clobber and pointing at
  `sase sdd init`. Filesystem-only, no network.
- Tests for both: foreign-record variants (unknown storage, higher schema_version, junk JSON) are preserved untouched
  and fail loudly; stale `not_found` records still get replaced; doctor flags the reverse-mixed state and stays quiet on
  legacy-only and healthy-companion layouts.
- `just check` gates the phase.

## Phase 3: Archived-repo probe hardening (sase-github linked repo)

Objective: archived GitHub repos must never be silently adopted as writable SDD stores.

- Extend `_probe_github_repo_detail` to request `isArchived` alongside `name`; report archived repos as
  `("unavailable", "<repo> is archived and read-only; if this project migrated to split companions run `sase sdd
  init`, otherwise unarchive the repo")` so both legacy `--sdd` adoption and companion adopt/create preflights fail with
  a clear message instead of adopting a black hole.
- Tests: archived probe result classification, adoption/preflight surfacing of the message.
- Open the plugin with `sase workspace open -p sase-github -r "<reason>" <workspace_num>`; run that repo's checks and
  commit through the standard sase commit workflow.

## Phase 4: Epic verification and close-out

Objective: re-verify the Phase-6 exit criteria end-to-end on the repaired system, then close the epic.

- End-to-end: from a numbered workspace confirm launch prep auto-clones `sase--plans`; propose + approve (or simply land
  via this plan's own flow) files under `202607/` with prompt snapshots in `sase--plans`; bead create/close round-trip
  pushes cleanly; `sase sdd path research --ensure` materializes/syncs the research clone; `sase validate` +
  `sase doctor` green in this repo's context.
- Update the epic plan file frontmatter (`202607/sdd_split_into_plans_and_research_repos.md` in `sase--plans`) from
  `status: wip` to `status: done`, following the existing "Complete SDD plan for X" commit convention.
- Fix the epic bead's stale design path (it points into an ephemeral numbered-workspace legacy clone):
  `sase bead update sase-5q --design <primary plans clone>/202607/sdd_split_into_plans_and_research_repos.md`.
- Close the epic: `sase bead update sase-5q --status closed`.

## Risks / notes

- **A live agent is writing SDD state during the repair.** Mitigation: salvage its stranded plan files before repairing
  the record (step 1 ordering), never touch its workspace clone until it finishes or its clone shows no unique commits;
  every `sase` CLI invocation re-reads the record, so the route flips atomically per command.
- **Bead event streams must never be file-merged** across the divergence (sequence-number collisions in
  `sase-5m`/`sase-5q` streams); all bead repair goes through the `sase bead` CLI so new events are minted.
- The pre-split code already shipped inside the uv-tool site-packages (currently shadowed by the dev editable override);
  recommend refreshing the installed tool after this lands so no stale copy can ever run again.
- Phase 2's foreign-record guard intentionally trades silent self-healing for loud failure; the doctor check plus the
  error message keep the operator unblocked.
- No Rust core changes: record handling, doctor, and the GitHub probe are pure Python (bead facades still receive
  resolved paths).
