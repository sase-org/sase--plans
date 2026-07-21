---
tier: tale
title: Stamp proposed plans from agent metadata, not popped env
goal: 'Plans proposed during epic bead work actually receive their managed `bead`,
  `parent`, and `parent_bead` frontmatter links, sourced from the proposing agent''s
  durable metadata, and the one plan file the gap already missed is backfilled.

  '
create_time: 2026-07-21 10:53:53
status: done
prompt: '[202607/prompts/bead_stamp_meta_fallback.md](prompts/bead_stamp_meta_fallback.md)'
---

# Plan: Stamp proposed plans from agent metadata, not popped env

## Context

Epic sase-88 (plan `202607/phase_plan_parent_links.md`, commits sase-core `298eb75` and sase `87e7a3a38`) taught
`sase plan propose` to stamp managed association frontmatter onto plans proposed during epic bead work: `bead` +
`parent` on tale proposals, `parent_bead` + `parent` on child-epic proposals. The stamping block in
`src/sase/main/plan_propose_handler.py` reads `SASE_PHASE_BEAD_ID`, `SASE_EPIC_BEAD_ID`, and `SASE_EPIC_PLAN_REF` from
`os.environ`.

**That stamping has never fired in production.** The agent runner consumes exactly those variables while writing the
child's `agent_meta.json` marker: `epic_work_metadata_from_env()` in `src/sase/axe/run_agent_directive_metadata.py` does
`os.environ.pop(...)` on all of them (deliberately, so an agent's own nested launches cannot re-attribute themselves to
the same epic role — see its docstring). The pop happens in the runner process before the agent CLI is spawned, so every
descendant process — including the agent's shell running `sase plan propose` — sees none of them. The env-pop predates
both stamping features (`bbb01e1fa`, 2026-07-16), so the earlier sase-7z.5 `parent_bead` stamping for child epics is
equally dead in the organic path. The sase-88 unit tests pass because they set `os.environ` directly around the handler,
which no real propose ever experiences.

Concrete evidence: tale plan `202607/reanchorable_date_bounds.md` was proposed by phase agent sase-8h.1 on 2026-07-21
(after sase-88 landed). Its agent's `agent_meta.json` carries `phase_bead_id: sase-8h.1`, `epic_bead_id: sase-8h`, and
`epic_plan_ref: sase/repos/plans/202607/commits_filter_correctness.md`, yet neither the archived nor the committed plan
copy has `bead` or `parent` frontmatter.

The good news: the runner already persists the exact values the handler needs into the proposing agent's
`agent_meta.json` (`phase_bead_id`, `epic_bead_id`, `epic_plan_ref`), the restart/exec-plan machinery preserves those
keys across agent-family members, and `handle_plan_propose_command` already hard-requires `SASE_ARTIFACTS_DIR` — the
directory that contains that marker. No sase-core change is needed: the Rust schema already accepts `bead` and `parent`
on both tiers.

## Design

- **Source of truth becomes the current agent's `agent_meta.json`.** In `handle_plan_propose_command`, resolve the three
  association values by reading `<SASE_ARTIFACTS_DIR>/agent_meta.json` keys `phase_bead_id`, `epic_bead_id`, and
  `epic_plan_ref`. Environment variables, when present, win over metadata (explicit override; also keeps the existing
  tests meaningful), but metadata is the path production takes.
- **Do NOT re-export the env vars to the agent CLI.** The pop's isolation rationale is sound: if the CLI inherited the
  epic role vars, every nested `sase run` launch from a phase agent would mis-attribute its children to the same epic
  phase. Reading the durable per-agent marker gives each agent exactly its own role and nothing to leak.
- **Stamp matrix is unchanged** from the sase-88 design: tale proposals get `bead` (phase-bead-over-epic-bead
  precedence, so land-agent proposals record the epic bead) and `parent`; epic proposals get `parent_bead` and `parent`;
  each field only when its value is non-empty; proposals outside bead work (no env, no metadata fields) stay unstamped.
  A missing or unparsable `agent_meta.json` must degrade to the unstamped path, never fail the propose.
- **Comment truth**: rewrite the stamping block comment to say the associations come from the agent metadata marker
  (with env override), and why the env alone is insufficient (runner pops it). Consider pointing at
  `epic_work_metadata_from_env` so the two sides reference each other.

## Implementation

1. `src/sase/main/plan_propose_handler.py`: add a small helper that reads the agent-meta association fields from
   `SASE_ARTIFACTS_DIR` (tolerating a missing file or malformed JSON by returning empties), merge with the env reads
   (env value wins per field when non-empty), and feed the existing stamp-matrix logic. Keep the existing
   `set_frontmatter_fields`-before-prettier ordering.
2. Keep `sase.bead.work` as the single home of the env names; import them as today. The keys read from `agent_meta.json`
   are the metadata names written by `epic_work_metadata_from_env` (`phase_bead_id`, `epic_bead_id`, `epic_plan_ref`).
3. **Backfill the missed plan file.** In the SDD plans store (resolve via `SASE_SDD_PLANS_DIR`), add to
   `202607/reanchorable_date_bounds.md` frontmatter: `bead: sase-8h.1` and
   `parent: sase/repos/plans/202607/commits_filter_correctness.md`. Before editing, check whether sibling phase plans
   for sase-8h.2 / sase-8h.3 have appeared in the meantime (epic sase-8h is still running); if they exist and are
   unstamped, backfill them the same way with their own phase bead ids. Commit the sidecar change following the
   plans-repo convention (a `chore`-style commit in that repo; it is a sidecar, not the primary checkout).
4. Optional bookkeeping cleanup while in the area: the sase-88 bead's notes record `COMMIT: 9bad3016`, which exists in
   neither the sase nor sase-core repo (the real commits are sase `87e7a3a38` and sase-core `298eb75`). Update the note
   via `sase bead update sase-88 --notes` to name the real commit(s).

## Testing

- Extend `tests/test_plan_command_handler.py` (the existing stamp parametrization):
  - metadata-only stamping (no env set): tale with `phase_bead_id` + `epic_plan_ref`; tale via land agent
    (`epic_bead_id` fallback, no `phase_bead_id`); epic gaining `parent_bead` + `parent`; metadata without
    `epic_plan_ref` (bead-only stamp).
  - precedence: env set AND differing metadata present → env values win.
  - robustness: no `agent_meta.json`, and malformed JSON → proposal succeeds unstamped.
  - keep the existing env-driven cases green (they now double as the override path).
- One regression-shaped test that mirrors production: build the stamp inputs after `epic_work_metadata_from_env()` has
  consumed the env (env cleared, marker written), then run the propose handler and assert stamps land — this is the
  exact interaction that silently broke sase-88.
- `just check` before finishing.

## Risks and notes

- The global `sase` used by launched agents is an editable install of the primary checkout, so the fix takes effect for
  all future proposals as soon as it merges to master — including the still-open sase-8h.2 / sase-8h.3 phases if they
  propose after the merge. If they propose before it, their plans join the backfill set (step 3 covers checking).
- `agent_meta.json` is rewritten during the run (plan submission, approval fields); the epic association keys are
  written at meta-build time and preserved by the restart machinery, so reading at propose time is stable. The reader
  must still tolerate absence, since `sase plan propose` can run in agents that are not part of epic bead work.
- No CLI surface or sase-core changes; the Rust schema and wire payload already expose `bead` and `parent`.
