---
tier: tale
title: Finish and land epic sase-90
goal: Artifacts Chats integrates publication quarantine correctly, renders every provenance
  state distinctly, and lands with the epic closed, Symvision clean, and its durable
  plan marked done.
bead: sase-90
create_time: 2026-07-24 21:52:38
status: done
---

- **PROMPT:** [202607/prompts/finish_sase_90.md](prompts/finish_sase_90.md)
- **PARENT:** [202607/artifacts_chats_subtab.md](https://github.com/sase-org/sase--plans/blob/main/202607/artifacts_chats_subtab.md)
- **BEAD:** [sase-90](https://github.com/sase-org/sase--beads/blob/main/pages/sase-90/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-90.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-90.land/README.md)
  - [bbugyi200.athena.sase-90.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-90.land.md#member-code)
- **COMMITS:**
  - [e5d953e](https://github.com/sase-org/sase/commit/e5d953eadd0b66ce4c9d8806d045048943107825) — feat(chats): expose publication quarantine provenance (sase-90)

# Plan: Finish and land epic sase-90

## Goal

Complete the final integration work for the Artifacts → Chats epic, prove the feature against its linked plan and the
changes that landed while it was in flight, then close `sase-90`, clean up its expired Symvision allowances, and mark
the durable epic plan done.

## Verified baseline

- `sase bead show sase-90` reports eight closed children, `sase-90.1` through `sase-90.8`, linked to
  `${SASE_SDD_PLANS_DIR}/202607/artifacts_chats_subtab.md`.
- The landed epic commits are `e7da5cd18`, `7bb87b1f5`, `5fa150739`, `c1c0e1557`, `cc85ef89b`, `99bcd567f`, `8a0ae2730`,
  and `58765147a`. Child-bead notes retain the agents' pre-rebase commit IDs; the surviving pre-rebase objects for
  phases 1 and 8 have the same stable patch IDs as the landed commits.
- The focused history, CLI, TUI, help, keymap, and Chats tests pass (127 passed), and the two dedicated PNG snapshot
  tests pass against the committed goldens.
- The checkout is directly on `master`, synchronized with `origin/master`, so there is no separate PR base branch to
  merge.
- Eight non-epic commits landed after the first epic commit. Test-only splits and plan-gate shortcut changes do not
  overlap Chats. The agent naming, inventory-diagnostic, and publication changes were reviewed against the catalog and
  agent-jump code.

## Remaining work

### 1. Integrate publication quarantine state

Commit `7bb485d33` added schema-v2 publication outbox items that remain in the outbox with `quarantined: true` after
exhausting retries. The Chats catalog currently treats every outbox item as `publication_pending=True`, so its detail
panel incorrectly says "Queued to publish" for work that will not retry until the user explicitly releases quarantine.

Update the headless catalog and its consumers:

- In `src/sase/history/chat_catalog_provenance/sidecars.py`, preserve the read-only raw outbox scan (do not create lock
  files or mutate the outbox), but decode `quarantined` as a strict boolean. Schema-v1/missing values must default to
  false.
- Give the publication-backlog value a clear typed representation rather than extending an opaque positional tuple if
  that keeps the code simpler.
- In `models.py` and `catalog.py`, carry an explicit `publication_quarantined` field through `ChatCatalogEntry`. Active
  items are pending; quarantined items are not pending, but retain attempts and last error for diagnostics.
- In `src/sase/ace/tui/widgets/artifacts/chats_detail.py`, keep the existing "Queued to publish" copy for active items.
  Render quarantined items as "Publication quarantined", include attempts/last error when present, and tell the user to
  run `sase agent sync --retry-quarantined` to retry.
- Extend the stable `sase chat list -j` provenance fields and the generated `sase_chats` skill source documentation with
  `publication_quarantined`, keeping the existing key order stable and appending the new field after the current
  publication fields.
- Add headless tests for both schema-v1 active items and schema-v2 quarantined items, plus TUI detail-copy and CLI
  JSON-schema coverage. Assert that a quarantined item is never described as queued.

### 2. Restore the promised three-channel provenance badges

The committed populated PNG renders `⇣` (remote) and `◌` (unknown) as the same missing-glyph box. The visual test
renderer intentionally uses only bundled Fira Code 6.2; its charset excludes U+21E3 and U+25CC but includes U+2193 and
U+25CB. This violates the plan's requirement that all four states remain distinct by glyph shape as well as label and
color.

- Change remote to the supported down arrow `↓` (U+2193) and unknown to the supported open circle `○` (U+25CB). Keep
  local `◇` and shared `◆`.
- Make `CHAT_PROVENANCE_BADGES` in `src/sase/history/chat_catalog_provenance/badges.py` the single source of truth.
  Derive the TUI glyph/color maps in `chats_rendering.py` from those badge objects instead of duplicating the values.
- Update CLI, rendering, filtering, and visual assertions for the supported glyphs.
- Regenerate only the intentionally changed Chats PNG goldens. Inspect the populated PNG and confirm
  local/shared/remote/unknown now have four visible, distinct shapes and retain their distinct labels and colors.

## Validation

1. Run `just install` before repository checks.
2. Run the focused history, CLI, Artifacts Chats, scaffold, help, onboarding, quickstart, and keymap tests touched by
   this epic.
3. Run the dedicated Chats visual tests with `just test-visual`; use `--sase-update-visual-snapshots` only to accept the
   intentional badge/detail changes, then rerun without the update flag.
4. Run `just check`.
5. Re-read the diff and verify no memory files, generated provider instruction shims, unrelated snapshots, or unrelated
   user changes were modified.

## Final landing phase

Do this only after the integration work and validation above pass:

1. Close the epic with `sase bead close sase-90`.
2. Because closing expires epic-symbol whitelist entries, use the `sase_memory_read` skill to read
   `sase/memory/symvision.md`, then run `just symvision` (the recipe exists). Remove stale `sase-90` whitelist entries
   and any now-unused code it reports. If this changes source, rerun the relevant tests and `just check`.
3. Use the `sase_repo` skill to open the `plans` sidecar and edit
   `${SASE_SDD_PLANS_DIR}/202607/artifacts_chats_subtab.md`, changing only the frontmatter value from `status: wip` to
   `status: done`.
4. Confirm `sase bead show sase-90` reports `CLOSED`, the plan reports `status: done`, Symvision passes, and both the
   primary and plans checkouts contain only the intended final changes.

Do not create a git commit unless the user separately requests one.
