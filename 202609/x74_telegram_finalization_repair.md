---
tier: tale
title: Complete sase-x7.4 finalization and certify the manual landing
goal:
  The recovered sase-x7.4 Telegram adapter is landed in sase-telegram, the manually
  committed host/core state is certified green (including a full-suite run), and the
  reconciliation is recorded on the affected beads.
size: medium
proposed_by: bbugyi200.athena.05n
create_time: 2026-09-09 19:52:57
status: wip
---

# Complete sase-x7.4 Finalization: Land the Recovered Telegram Adapter and Certify the Manual Landing

## Background (read this before touching anything)

Phase bead `sase-x7.4` ("Move Telegram to the shared pending-action API") was closed
done at 2026-09-07T16:40:55Z after full verification, but its host-owned finalization
was interrupted mid-commit (see bead notes #15 and #16 on `sase-x7.4`). The user then
manually completed two of the three repository commits at ~16:50-16:54 EDT on
2026-09-07:

- **sase (host)**: commit `e0c5755033ff0e17c5b306a6c703901ada67b1f4` landed on master.
  Audit confirmed its four intended x7.4 files
  (`src/sase/notifications/pending_actions.py`, `tests/test_pending_actions.py`,
  `tests/ace/tui/test_config_center_resume.py`, `tests/reproducible_flake_baseline.txt`)
  are byte-identical to the finalizer's verified resolved diff. The commit ALSO swept in
  13 extra files (pager, artifact-ref, prompt-panel, and
  `src/sase/notifications/__init__.py` changes). Those extras are the completed,
  focused-verified payload of closed phase `sase-xy.5.1` (Rust document scanner's Python
  pager adapter), whose own finalizer commits never landed either. Audit found no
  clobbering: the merged versions retain the intermediate pager commits' changes (e.g.
  `a9f95ca5e`).
- **sase-core**: commit `f7852f54f2b2962b71b7692f71d3ef699ddafd1d` landed and shipped in
  release v0.32.38. It likewise contains both the x7.4 core payload and `sase-xy.5.1`'s
  Rust `artifact_ref` scanner work. Core CI passed and the release exists; treat core as
  done.
- **sase-telegram**: the adapter commit **never landed anywhere**. Its master HEAD is
  still `e7582519941e733397ee593753648effa0675e7f` — exactly the `telegram_base`
  recorded in the recovery bundle — with no branch carrying the work. This is the
  missing deliverable of this plan.

The verified Telegram adapter survives as `recovered-telegram.diff` inside the durable
recovery bundle artifact `file:explicit:761241478a4677382f030241` (a tar.gz that also
holds `recovery.json`, `resolved-main.diff`, `recovered-core.diff`, and the original
commit object). The diff touches four files (`src/sase_telegram/pending_actions.py`,
`tests/test_custom_gates.py`, `tests/test_integration.py`,
`tests/test_pending_actions.py`) and rewires the plugin to the five host transport
functions (`upsert_transport_action`, `get_transport_action`, `list_transport_actions`,
`remove_transport_action`, `cleanup_transport_actions`), which all exist on current host
master. It was verified as part of the phase's passing Telegram `just check` (587+
tests) and the published Telegram wheel artifact
`file:explicit:1e925693d40103585b71bdea` was built from it. A dry-run
`git apply --check` of the diff against `e7582519` succeeded during the audit.

Two loose ends motivate the certification steps:

1. Bead note #16 on `sase-x7.4` states "Main verification and resume are incomplete":
   the combined tree that actually landed on host master (x7.4 payload + xy.5.1 payload
   on top of the newer master) has never had a full-suite run.
2. `sase-xy.5.1` note #1 reports host `just check` failing on `pending_actions.py`
   Symvision URI pragmas because sase-telegram does not reference the five transport
   functions. The landed host version dropped those five URI pragmas in favor of
   re-exports in `src/sase/notifications/__init__.py` (only one URI pragma remains, on
   `merge_transport_record`, which sase-telegram master DOES reference in
   `src/sase_telegram/scripts/sase_tg_outbound.py`). Landing the Telegram adapter makes
   the external references real either way.

**Do not** revert, "clean up", or re-attribute the swept-in pager/artifact-ref content
on host master: it is verified `sase-xy.5.1` work, epic `sase-xy.5`'s later phases are
actively building on top of it, and rewriting pushed history is off the table. **Do
not** hand-edit any bead status: `sase-x7.4` and `sase-xy.5.1` stay closed; downstream
phases `sase-x7.5` / `sase-x7.7` stay untouched (the x7 supervisor reconciles them).

## Steps

### 1. Land the recovered Telegram adapter in sase-telegram

1. Open the repo the sanctioned way: `/sase_repo` skill, then
   `sase repo open sase-telegram -r "Land the recovered sase-x7.4 adapter commit"`. Use
   only the printed path.
2. Confirm HEAD. If it is still `e7582519941e733397ee593753648effa0675e7f`, the diff
   applies cleanly. If master has moved, apply with 3-way merge and resolve
   conservatively (the diff is small and self-contained).
3. Read the recovery bundle through
   `sase artifact read file:explicit:761241478a4677382f030241 "<reason>"` (the read must
   be audited; do not open the artifact file path directly without it). Extract the
   tarball it names to a scratch directory and apply `recovered-telegram.diff` to the
   sase-telegram checkout with `git apply`.
4. Verify in the sase-telegram checkout per its own Justfile: `just install` (if
   present/needed) then `just check`. Expect the previous baseline (~587 tests) plus the
   recovered pending-action/integration tests to pass. Use the `/sase_monitor` skill if
   the run outlasts the turn.

### 2. Certify the manually landed host tree

5. In the primary sase checkout: `just install`, then `just check` (inline is fine on a
   clean tree; hand to `/sase_monitor` if slow). This answers whether master's lint
   gates — Symvision in particular — are green after the manual landing.
6. Reproduce the CI-strict Symvision external check against the now-updated Telegram
   checkout (see `sase memory read symvision.md`):
   `SYMVISION_EXTERNAL_REPO_PATHS=<printed sase-telegram path> just _lint-symvision`.
   This confirms the `merge_transport_record` URI pragma and the transport-function
   consumers resolve against the real, updated plugin source.
7. Launch `just check-full` through the `/sase_monitor` skill (never inline; use the
   TESTING/TESTED status pair). This is the certification bead note #16 says is missing.
   Reuse is acceptable only if a green `just check-full` receipt already exists for
   exactly the current master SHA.
8. Triage any failures:
   - Caused by the mixed manual landing (pending-action <-> pager/scanner interaction):
     fix forward in the primary checkout; those fixes become part of this tale's commit.
     Read `sase memory read lint_and_test.md` and `symvision.md` before fixing lint
     failures.
   - Unrelated to the landing (pre-existing flake or independent regression): do NOT
     scope-creep; file it through the `/sase_new_task` skill instead.

### 3. Record the reconciliation

9. Add one note to bead `sase-x7.4` (`sase bead note ...`) recording: the Telegram
   adapter commit now landed (include the new SHA), host/core landed earlier via the
   user's manual commits `e0c575503` / `f7852f5` (which also carried sase-xy.5.1's
   verified payload), and the check-full certification result with its monitor id.
10. Add one note to bead `sase-xy.5.1` stating its implementation landed inside host
    `e0c575503` and core `f7852f5` (mis-attributed to sase-x7.4 in the trailers —
    accepted, history is pushed), and that its PROPOSED FOLLOW-UP about
    `pending_actions.py` Symvision URI pragmas is resolved by the landed `__init__.py`
    re-exports plus the Telegram adapter landing (cite the step 6 result).

### 4. Finalize

11. The modified sase-telegram checkout is a repository obligation of this turn: submit
    it through the normal `/sase_final` declaration with a commit decision. Subject
    convention should mirror the core commit:
    `feat: move Telegram to the shared pending-action API (sase-x7.4)`. Any host-repo
    fixes from step 8 ride in the primary checkout's own commit decision. Do not commit
    by hand.

## Acceptance

- sase-telegram master contains the recovered adapter; its `just check` passes.
- Host `just check` passes and one green `just check-full` exists for the landed master
  tree (monitor receipt cited in the sase-x7.4 bead note).
- The CI-strict Symvision external check passes against the updated Telegram checkout.
- Reconciliation notes exist on `sase-x7.4` and `sase-xy.5.1`.
- No bead statuses were hand-edited, no pushed history rewritten, and no changes made to
  sase-core.
