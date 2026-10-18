---
tier: epic
title: Finish core variadic portability and observe the compatible release
goal: Repair the introduced macOS openat regression, consume its committed core, and
  establish the required sase 0.18.0 publication proof.
parent_bead: sase-1lh.7.5.4.3.3.4
phases:
- id: core
  title: Promote variadic openat modes without changing permissions
  size: small
  description: 'core: repair the macOS E0617 regression in mobile pairing and verify
    the core.'
  depends_on: []
- id: pin
  title: Pin the committed portable core and verify the consumer
  size: small
  description: 'pin: ratchet the sase core pin past the completed portability fix
    and verify compatibility.'
  depends_on:
  - core
- id: watch
  title: Observe the containing core release and sase 0.18.0
  size: small
  description: 'watch: verify committed prerequisites and run one self-closing publication
    watch with no successor.'
  depends_on:
  - pin
proposed_by: bbugyi200.athena.sase-1lh.7.5.4.3.3.4.land
proposed_with: codex/gpt-6-astra
create_time: 2026-10-17 21:56:51
status: wip
bead_id: sase-1lh.7.5.4.3.3.4.4
---

- **PROMPT:** [prompts/202610/finish_portable_core_release.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/finish_portable_core_release.md)
- **PARENT:** [202610/finish_release_gate_fixtures.md](https://github.com/sase-org/sase--plans/blob/main/202610/finish_release_gate_fixtures.md)
- **BEAD:** [sase-1lh.7.5.4.3.3.4.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1lh/sase-1lh.7.5.4.3.3.4.4.md)

# Finish the portable core release

Remaining work from landing **sase-1lh.7.5.4.3.3.4**, linked plan
`plan:202610/finish_release_gate_fixtures.md`. That epic is incomplete: its watch phase
`.3` closed **canceled** with `MASTER_GATE_RED`, and the core macOS build still fails. A
closed watch is not publication evidence.

This requires an epic despite the small code repair. The core repair must receive its
host-created commit before sase can pin it; that pin must receive its own host-created
commit before a watcher can verify the released consumer. A single tale cannot finish
the required release proof and closeout without waiting for its own commit. These are
the three necessary phases; do not repeat the completed timestamp, rescue-audit,
fixture-sweep, or ten-run tutorial work.

Inherited decisions remain **dispatch=yes**, **visual_flakes=recapture**, and
**stale_epics=supersede**. No new decisions or memory edits. Only the existing release
watcher dispatches sase workflows; ci_watch owns the sase release merge. Core
release-plz owns versions and its daily release cut. Do not manually merge a release PR,
change package versions/changelogs, dispatch a core release, weaken gates, or force a
successful nested landing. Never run `just check-full`.

## Audited baseline

- Sase master is `933970ce72a5b5328499e80751a491c857bfc221`. Its explicit launch
  timestamp fixture handles unset and hostile ambient values, and the common test
  environment clears `SASE_LAUNCH_SUBMITTED_AT`. The strict directory audit names
  `_rescue_git.py:_rescue_git_repo_body`; concurrent `5cf773c154` updated that reviewed
  site during the rescue split. The deletion removes only its own unused rescue entry.
  Focused landing ToolRun `ea71a0a3feee0de027d98bfdd22a332a` passed all 10 cases
  (deferred workspace module, directory audit, and the UI node below).
- Phase `.2`'s full required `just check` record `775bb391c2dc9975321fcc238552c690` is
  succeeded/exit 0. Phase `.1`'s Linux core gate `5baf8fd82840a8a99ce136b0871bd3d4` is
  also succeeded/exit 0. Linux success does not establish macOS compilation. Tutorial
  monitor `s4rqbkt02qrj` confirms ten Python 3.12 attempts with fresh HOME/basetemp and
  `CI=1`, ending `TUTORIAL_ATTEMPTS=10 TUTORIAL_FAILURES=0`.
- Core master and the sase pin are `dd56b94a8ad9fdca541d16cb5b6f85a2b45e55af`. That fix
  changed all three mode constants in `crates/sase_core/src/mobile_pairing/fs.rs` to
  `libc::mode_t` and fixed `mkdirat`/`fchmod`, but its two `O_CREAT` calls to variadic
  `libc::openat` now pass a `u16` on macOS. Core PR #331 contains the fix at
  `0b23a73917eca7c140c3ffcb651c73d1daddf39f` and still proposes 0.42.0. CI
  **39071544870**, macOS job **117510033363**, fails E0617 at `fs.rs:191` and `:393`;
  the compiler recommends promotion to `c_uint`. Retained log:
  `file:explicit:ce30efab790c5cca3efd1466`. This regression is caused by phase `.1`, so
  `.3`'s sole `PROPOSED FOLLOW-UP` stays required epic work, not a task.
- Since the epic baseline, the runner-utils (`0f91d5eed0`), tutorial helpers
  (`cc64a8e8e6`), and rescue tests (`57e0e97874`) were split. Their preserved facades
  and tutorial STARTING/DONE detection, prior-name filtering, on-disk fallback, and
  diagnostics remain present. No additional source integration gap was found. Neither
  fetched master has commits after the two repair commits.
- Master Gate **39077497974**, attempts 1 and 2, fails
  `tests/ace/tui/modals/test_new_project_create_flow.py::test_name_collision_blocks_review_but_occupied_checkout_surfaces`
  at line 234: Review is enabled after the fixture enters `existing-project`. The exact
  unchanged-source node passes locally in the focused run above. This independent flake
  is **sase-1mi**, with adjacent reports **sase-1m9** and **sase-1l8** linked. Its root
  cause is unproven; CI uses Python 3.12 and the local pass used 3.14. Retained
  failed-job log: `file:explicit:0c00d9b69c28858fb5069bf4`. Do not silently absorb
  unrelated flake repair into this core portability work or claim that an isolated pass
  makes Master Gate green. The attempt-1 link-follow flake remains **sase-1lr**, already
  excluded from repair here; it passed attempt 2. Preserve other inherited follow-up
  outcomes.
- Sase PR #299 is OPEN at `d3f2ba36c7580e1da8ba9a9eaf90016e1da8b1fc` and targets 0.18.0.
  Its reconciler ratcheted the core window to `>=0.41.0,<0.42.0`. That published core
  lacks `mobile_pairing_issue`; a containing release is still needed. PyPI currently has
  sase 0.17.1 and core 0.41.0. No GitHub v0.18.0 release exists. Publish **39077813839**
  skipped actual build/publish. Full CI **39085508146** at the current master was still
  running during the audit. Refresh all remote facts before acting.

## Phase core

1. Use `/sase_repo` to open `sase-core` and read its `AGENTS.md`. Refresh master and PR
   #331 before editing; preserve any concurrent repair. Inspect the cited macOS job and
   actual source. Keep `libc::mode_t` at the fixed-argument `mkdirat` and `fchmod`
   boundaries.
2. At both `O_CREAT` `openat` calls, explicitly promote the mode to an admissible C
   variadic integer type. Keep exactly `0600`; use a checked/lossless widening
   conversion or justified narrow syscall-boundary cast compatible with Linux and macOS.
   Account for clippy treating a conversion as redundant on Linux; avoid broad lint
   suppression. Correct the existing comment that currently implies a native `mode_t`
   can be passed directly to every syscall. Audit every mode-bearing variadic call in
   this module. Preserve `0700` directories, no-follow/exclusive opens, ownership
   checks, umask repair, atomic writes, and fail-closed errors.
3. Use core's repository recipes, never bare cargo. Run formatting, focused
   mobile-pairing tests (including permissions, umask, and symlink coverage), and
   `sase tool run check` from the linked checkout. Seek supported macOS compile proof
   where available; a Linux pass or a cross-build blocked in SQLite is not macOS proof.
   The later observation phase must confirm real macOS CI.
4. Finalize the linked-core change normally. Do not wait for this turn's own commit,
   update sase's pin to an uncommitted tree, or edit release metadata. Record exact
   evidence and any unrelated findings on the phase.

## Phase pin

1. Refresh the clean sase checkout and open core via `/sase_repo`. Verify the previous
   phase's host-created core commit is on core master. Ratchet `sase-core-revision.txt`
   past it with the normal supported command.
2. Review intervening changes for required bindings and wire contracts. Run the
   binding-inventory and compatibility checks against the pinned cohort, using the
   established tools. Keep the release-branch reconciler authoritative for dependency
   metadata; do not hand-edit master's dependency window because an editable core
   checkout has a different version.
3. Read applicable verification memory, run `just fix` and `sase tool run check`. Record
   any independently tracked flake without weakening tests or running `just check-full`.
   Use `/sase_monitor` for long verification when necessary and let it finish. Finalize
   so the host commits the pin. No watch in the same turn depends on this turn's SHA or
   push.

## Phase watch

Make no source edits. Read `/sase_monitor`. This phase records a terminal verdict on
itself and always releases the lander; it cannot declare success on a deadline or red
gate.

1. If this phase is already closed, use `/sase_final` and finish. Otherwise fetch and
   fast-forward the clean sase checkout, verify the pin phase's commit is on master, and
   set **FIXED_SHA** to that tip. Open core via `/sase_repo` and verify the containing
   release PR includes the new core fix and the complete consumer binding cohort.
   Require the macOS job to pass before treating core portability as complete. If it is
   still running, the normal publication watch below may await the containing core
   release; do not start a separate wait monitor. If it has failed, capture its actual
   failure and cancel this phase explicitly. The child lander must verify the final
   macOS result.
2. Resolve **CORE** to the containing published or pending core release (currently
   0.42.0). Do not choose incompatible 0.41.0. Let release-plz's daily 07:41 UTC cut
   publish it normally. Once published, validate the downloaded wheel against the
   complete required-binding and contract inventory in isolation. The release-floor
   smoke remains authoritative after publication.
3. Run `sase bead epic-symbols <watch-bead>`. It must be empty. Any unexpected entry
   must be resolved or re-keyed only to this still-open child epic when it truly remains
   needed; if that edits the Justfile, cancel with an explanatory note and finalize
   before any watch so the host can commit it. Never key an exemption to a watch bead.
4. Refresh Master Gate, Full CI, the independent flake task states, and PR #299.
   Preserve the watcher's existing bounded rerun policy. An outstanding unrelated flake
   remains an external release blocker with a concrete task, not evidence that core code
   repair failed. Do not repeatedly create another repair epic for the same unrelated
   failure. If the release is already published, verify the complete release proof below
   and close this phase with it.
5. Otherwise preflight with concrete values:

   ```sh
   PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --once --dry-run --close-bead <watch-bead>
   ```

   Require exit 0 and a parsed state line. On failure, cancel this phase as
   `WATCHER_BROKEN (preflight)` with evidence and finalize. Record the chosen SHA, CORE,
   preflight, and full monitor command in the phase note.

6. Start exactly one no-successor monitor with placeholders replaced:

   ```sh
   sase monitor start -C "$PWD" -r 'Watch portable core and sase 0.18.0; watcher closes its phase' -t 14h -s 'WATCHING RELEASE' -S 'RELEASE WATCHED' -L 'sase 0.18.0 portable-core watch' -- 'PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 timeout -k 10m 750m tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --deadline 12h --close-bead <watch-bead>; rc=$?; echo "RELEASE_WATCH_EXIT=$rc"; sleep 120; sase bead close <watch-bead> -R canceled -r "release not confirmed: watcher exit $rc left the bead open" || true; exit 0'
   ```

   Do not pass `--next`, `-m`, `-p`, or `--idle-timeout`. Keep polling a yielded start
   session until the start command itself exits and the handoff is complete. If start
   fails twice, cancel this phase as `WATCHER_NOT_STARTED` with both failures and
   finalize. Never end while an unmonitored command is running.

The watcher closes its phase on every verdict; the fallback closes it on a crash and
exits 0 so the monitor settles. Host loss/SIGKILL still needs explicit recovery. A
canceled phase is not successful publication.

## Landing and continuation

The child lander must prove: core's containing macOS CI is green; ci_watch merged sase
PR #299; GitHub release v0.18.0 exists; actual Publish build and publish jobs passed;
PyPI lists 0.18.0; a fresh isolated **unpinned** install resolves sase 0.18.0 with the
compatible core version; and the Master Gate and recent Full CI used for release are
green. Retain job IDs, versions, the watcher verdict, and install output. Recheck any
post-pin binding additions. A skipped publish or old Full CI result is insufficient. Do
not run `just check-full`.

This child epic deliberately has no parent-close, Symvision, or parent-plan-status
phase. Its `parent_bead: sase-1lh.7.5.4.3.3.4` is the continuation back to the
interrupted landing. After this child's own verified normal closeout, resume that parent
using `plan:202610/finish_release_gate_fixtures.md`, rechecking its landing notes, every
descendant and note, linked-plan readiness, and drift. Preserve the recorded follow-up
dispositions. Never force a successful nested landing. Stop and record a concrete
blocker on the first incomplete ancestor. The direct plan ancestors after that parent
are sase-1lh.7.5.4.3.3, sase-1lh.7.5.4.3, sase-1lh.7.5.4, sase-1lh.7.5, sase-1lh.7, and
sase-1lh; none is complete merely because this plan's phases closed.
