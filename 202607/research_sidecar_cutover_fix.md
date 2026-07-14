---
create_time: 2026-07-14 13:02:09
status: done
prompt: 202607/prompts/research_sidecar_cutover_fix.md
tier: tale
---
# Fix Research Sidecar Remote Cutover, Remediate Project State, and Land Epic sase-60

## Context

Landing verification for epic bead sase-60 ("Retire `sase sdd`, Make Sidecar Repos First-Class, and Generalize Repo
Config") confirmed all five phases landed (commits e7411b8a8, 3d103bd06, 5db03cb12, 78057dd22, 8c716fa74) and the
cross-repo cutover (chezmoi global config, xprompts, deployed skills, actstat/bob-cli project configs, sidecar README
templates and deployed sidecar READMEs) is complete — with one defect: **the Phase 5 shared-research remote cutover
silently failed for actstat and bob-cli**, the only two projects whose research remote actually changed (for the sase
project the old and new remotes coincide, so the bug was invisible there).

### Root cause (verified by tracing a live `sase repo init` run on actstat)

The global config pins the shared research sidecar
(`repos.sidecar: [{name: research, repo: sase-org/sase--research, ...}]`), but the resolution and init pipeline lets the
_old_ per-project remote leak back in:

1. `_configured_sidecar_specs()` in `src/sase/main/repo_init_handler.py` builds `SidecarInitSpec` with `repo=` from the
   config pin but never populates `remote_url`, even though `_sidecar_repo_identity()` already resolved the correct
   expected remote.
2. `_create_or_adopt_sidecar()` in `src/sase/sdd/_sidecar_init.py` then backfills the missing option from the existing
   SDD store record: `options.setdefault("sdd_remote_url", existing_sidecar.remote_url)`. It does this **even when the
   resolved `sdd_repo` (the new pin) differs from the record's repo**, pairing the new repo name with the old remote
   URL.
3. The sase-github provider's `_sdd_repo_target_from_options()` (in the `sase-github` linked repo,
   `src/sase_github/workspace_plugin.py`) trusts a provided `sdd_remote_url` without checking that it corresponds to
   `sdd_repo`, probes the _new_ repo name, finds it, and returns a store record with `repo: sase-org/sase--research` but
   `remote_url: git@github.com:<owner>/<project>--research.git`.
4. `initialize_sidecars()` then runs `ensure_sidecar_sdd_clone(root, <old remote>)`, so the existing clone is adopted
   as-is, and `_seed_sidecars()` commits and pushes guide refreshes **to the old, to-be-archived research repo**.
5. The mismatched record is written to `.sase/sdd-store.json` and poisons every later resolution:
   `_sidecar_repo_identity()` in `src/sase/_linked_repo_config.py` trusts `store_remote` whenever
   `_repo_basename(store_repo) == slug` — which now matches — so inventory, `sase repo path research --ensure`, and
   repeat `repo init` runs all resolve research to the old remote. The remote-cutover replacement logic
   (`_materialize_remote_identified_sidecar()` in `src/sase/linked_repos.py`, added by Phase 5) never fires because the
   _expected_ remote itself is already wrong.

### Current bad state on this machine (athena)

- `~/projects/github/bbugyi200/actstat`: `.sase/sdd-store.json` research half is mismatched
  (`repo: sase-org/sase--research`, `remote_url: git@github.com:bbugyi200/actstat--research.git`); the
  `sase/repos/research` clone still has origin `bbugyi200/actstat--research`; a guide-refresh commit was pushed to that
  old repo.
- `~/projects/github/bobs-org/bob-cli`: identical shape (`bobs-org/bob-cli--research`).
- Both research clones are clean and fully pushed; no local content is at risk.
- The sase project's own record, clones, and the shared `sase-org/sase--research` repo are correct.

Everything else about the epic is verified done, so this plan fixes the cutover defect, remediates the two projects, and
finishes the epic landing as its final phase.

## Phases

Phases 1 and 2 are independent; Phase 3 depends on Phase 1 (and benefits from 2); Phase 4 depends on Phase 3.

### Phase 1 — Fix cutover resolution and init in this repo

Scope (sase repo only):

- `src/sase/sdd/_sidecar_init.py` — in `_create_or_adopt_sidecar()`, only backfill `sdd_repo`/`sdd_remote_url` from
  `existing_sidecar` when the record's repo agrees with the already-resolved `sdd_repo` option (compare full
  `owner/repo` names, tolerating the option being unset). When the configured pin names a different repo, pass no legacy
  remote so the provider derives the remote from the pin.
- `src/sase/main/repo_init_handler.py` — populate `SidecarInitSpec.remote_url` from the config-resolved identity
  (`_SIDECAR_REMOTE_URL_KEY` on the normalized entry) so the spec carries one consistent (repo, remote) pair end to end.
- `src/sase/_linked_repo_config.py` — harden `_sidecar_repo_identity()`: only adopt `store_remote` when the remote URL
  itself identifies the same repo as `store_repo` (reuse/port the remote-identity comparison used by
  `_clone_origin_matches()` in `src/sase/linked_repos.py`). An inconsistent record half is ignored so a poisoned record
  self-heals to the config-derived remote instead of perpetuating itself.
- Consider (optional, if it fits naturally): a doctor drift warning when a store record half's `repo` and `remote_url`
  disagree, pointing at `sase repo init`.
- Tests (`tests/sdd_store/`, `tests/main/test_repo_init_*`, `tests/test_linked_repo_resolution.py`): cover the
  changed-remote cutover — existing record + clean clone on the old `<project>--research` remote, config pin to the
  shared repo → identity resolves to the pinned remote, `repo init` re-points the clone, writes a consistent record, and
  seeds/pushes against the new remote; a pre-poisoned record (repo=new, remote=old) self-heals; the dirty-clone refusal
  path stays intact.

Acceptance: with a record and clone on the legacy remote and a pinned `repos.sidecar` research entry, `sase repo init`
and `sase repo path research --ensure` both converge clone + record on the pinned remote; a mismatched record never
survives a resolution round-trip.

### Phase 2 — Harden the sase-github provider pairing

Scope (`sase-github` linked repo; open via `/sase_repo`):

- `src/sase_github/workspace_plugin.py` — in `_sdd_repo_target_from_options()`, when both `sdd_repo` and
  `sdd_remote_url` are provided but the remote URL does not identify `sdd_repo` (owner/repo comparison across ssh/https
  forms), ignore the provided remote and derive the URL from `sdd_repo`/`sdd_host` instead, so the returned store record
  is always internally consistent regardless of caller bugs.
- Port/extend that repo's existing tests for `ws_create_sdd_remote`/`ws_preflight_sdd_sidecar` with the mismatched-pair
  case.

Acceptance: the provider can no longer return a store record whose `repo` and `remote_url` name different repos.

### Phase 3 — Remediate actstat and bob-cli on athena

Operational scope (no code changes; depends on Phase 1):

- Using a sase build containing Phase 1, run `sase repo init` from each project's primary workspace
  (`~/projects/github/bbugyi200/actstat`, `~/projects/github/bobs-org/bob-cli`). Both research clones are clean, so the
  cutover replacement should re-point them to `git@github.com:sase-org/sase--research.git` (or the https equivalent) and
  rewrite consistent `.sase/sdd-store.json` records.
- Verify per project: research clone `origin` is the shared repo; the record's research half is consistent;
  `sase repo path research --ensure` prints the research clone path and `sase repo list` shows the shared slug and
  remote; `sase repo init --check` reports no sidecar work.
- Do NOT try to clean up the stray guide-refresh commits already pushed to the old `actstat--research` /
  `bob-cli--research` repos — those repos are slated for manual archiving by the user (flagged below), and the commits
  are harmless.

### Phase 4 — Land epic sase-60

Final landing steps (depends on Phase 3):

- `sase bead close sase-60`.
- After closing, run `just symvision` (epic-symbol whitelist entries for sase-60 expire at close) and remove the stale
  whitelist entries and any unused code it reports; run `just check` before finishing if this changes files.
- Set `status: done` in the frontmatter of the epic's plan file — the PLAN path printed by `sase bead show sase-60`
  (`202607/sdd_cli_retirement_and_sidecar_repos.md` in the plans sidecar; resolve the checkout with `/sase_repo` /
  `sase repo path plans`).
- Surface (do not perform) the outstanding user-permission / manual items listed below.

## Risks & Notes

- **Memory files need a user-approved refresh.** `sase init -c` reports memory drift on sase, actstat, and bob-cli
  (generated `memory/sase.md`, `memory/README.md`, and provider shims — mostly dropping the research entry from the
  generated linked-repo listings now that research is a config-declared sidecar). Memory files must not be modified
  without explicit user permission granted in-conversation; agents should surface this and let the user run `sase init`
  (or grant permission) themselves.
- **The globally installed `sase` predates the epic.** `~/.local/bin/sase` still has `sase sdd` and lacks
  `sase repo path`/`sase repo init`, while the deployed chezmoi xprompts and `sase_beads` skill already call
  `sase repo path`. Until the user updates the global install, `#research*` xprompts and the deployed skill will fail
  against it. Flag for the user; do not modify the global install unasked.
- **Manual follow-up (unchanged from the epic plan):** archive the now-orphaned `bbugyi200/actstat--research` and
  `bobs-org/bob-cli--research` GitHub repos after consolidating any content worth keeping into
  `sase-org/sase--research`.
- Repo/config logic here is Python-owned (`src/sase/repo_inventory.py` documents the future sase-core seam); nothing in
  this plan crosses the Rust core boundary.
- Phase 1 must keep the non-bypassable interactive default-No confirmation for provider repo creation intact — none of
  these changes should touch authorization semantics.
