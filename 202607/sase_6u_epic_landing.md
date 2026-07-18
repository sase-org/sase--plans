---
tier: tale
title: Complete and land the sase-6u clan-summary folding epic
goal: 'The clan-summary folding feature is reconciled with changes that landed during
  the epic, its default and interactive behavior is verified against the approved
  contract, and epic sase-6u is closed with clean Symvision state and a completed
  durable plan.

  '
create_time: 2026-07-18 16:59:30
status: wip
prompt: 202607/prompts/sase_6u_epic_landing.md
---

# Plan: Complete and land the sase-6u clan-summary folding epic

## Context and verified baseline

Epic `sase-6u` has four closed children. The landed commits marked `sase-6u.1` through `sase-6u.4` implement the generic
panel fold state and keymaps, pure and cached clan aggregation, fold-aware rendering and snapshots, and user
documentation. A focused audit confirmed the keymap, command-palette, section-navigation, worker/cache, renderer, and
visual coverage is present. The current focused suite passes 221 tests, and the epic/zoom PNG suite passes all 8 tests.

Two issues remain before the epic is genuinely landable:

1. Clan-level tribe syntax changed after the epic began. Runtime bead-work now emits `%clan(<epic-id>, tribe=epic)` and
   rejects combining `%clan` with `%tribe`, but the epic documentation in `docs/agent_families.md` and the related
   bead-work documentation in `docs/beads.md` still teach the obsolete two-directive form.
2. The approved level-1 contract says every represented clan section appears as a collapsed heading without doing
   render-time I/O. The renderer currently proves that contract only when a disk snapshot has already been cached; the
   default uncached path does not expose reply, prompt, or slow-tool headings. Reconcile section-presence discovery with
   the existing pure first paint and off-thread, mtime-keyed enrichment rules, then pin the behavior with a regression
   test using representative member artifacts.

No later commit outside the epic otherwise duplicates the fold-state, aggregation, or renderer implementation. The
status-recency/status-priority, plan-filter, prompt-completion, negative-filter, and axe-facade changes are orthogonal.
The clan-tribe projection already reaches the clan header through the synthetic container's `clan_tags`; add focused
coverage while correcting the documentation so that integration remains explicit.

## Integration and contract completion

- Update every user-facing epic bead-work example to use `%clan(<epic-id>, tribe=epic)`. Keep prose aligned with the
  single generation-level clan tribe while retaining the documented legacy display fallback where it is still supported.
- Add a regression that constructs a new-style `clan_tribe="epic"` member, projects the synthetic clan row, and confirms
  the folded clan summary header renders `@epic`.
- Make the uncached level-1 clan summary converge on the complete set of represented section headings without filesystem
  access in the render path. Reuse the existing off-thread enrichment, identity revalidation, cache, and
  `DetailPanelDebouncer`; do not introduce synchronous stat/glob/read work or a second refresh path. Empty section kinds
  must still disappear after presence is known, and counts that require disk must remain absent rather than block.
- Cover the cold level-1 path, the enriched level-1 path, empty-section omission, and cache reuse. Preserve the
  level-2/3 bodies and current loading placeholder behavior.

## End-to-end and performance revalidation

- Exercise a clan-shaped Agents-tab fixture through `zz`, `zZ`, `za`, and `zA`, verify uppercase `Z` still opens zoom,
  and confirm ChangeSpec folding is unaffected. Prefer a mounted interaction test so the routing, current-section
  anchor, worker completion, and repaint path are covered together.
- Re-run the epic-focused unit/integration suites and the epic/swarm PNG snapshots at all three levels. Inspect any
  intentional snapshot changes before accepting them.
- Measure Agents-tab key-to-paint behavior with a clan selected at levels 1, 2, and 3 under `SASE_TUI_PERF=1`. Keep p95
  below 16 ms and confirm no new stall records appear during fold cycling. If the existing benchmark cannot express
  these scenarios, extend its synthetic fixture rather than relying on an unrecorded manual assertion.
- Run `just install` before validation commands, then run `just check` after all main-repository changes.

## Final landing

This is the final phase and must run only after integration, tests, visuals, and performance checks succeed:

1. Close the epic with `sase bead close sase-6u` and confirm all four children and the parent are closed.
2. After the close, run `just symvision`. Read the audited Symvision memory before fixing diagnostics, remove expired
   `sase-6u` whitelist entries and any newly exposed unused code, and rerun Symvision until clean. Rerun `just check` if
   cleanup changes main-repository files.
3. Open the plans sidecar through `sase repo open`, add `status: done` to the YAML frontmatter of
   `202607/agent_panel_folding_clan_summary.md`, and verify the final bead and plan states. Do not modify memory files
   or generated agent-instruction shims.

## Risks

The main risk is paying disk-discovery cost on the Textual event loop merely to populate collapsed headings. The first
paint must remain pure; discovery and cache invalidation belong in the existing worker path. A second risk is making all
possible headings permanently visible and violating empty-section omission, so tests must distinguish unknown/loading
presence from known-empty content. Finally, close the bead only after the feature checks pass because its Symvision
whitelist expires immediately at close.
