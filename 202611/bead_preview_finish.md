---
title: Finish bead-preview cache freshness and async navigation, then land sase-1sy
tier: tale
goal:
  Bead previews and status mentions stay correct across store changes and async
  navigation, and sase-1sy closes with its original plan marked done.
size: medium
proposed_by: bbugyi200.athena.sase-1sy.land
proposed_with: codex/gpt-6-astra
bead: sase-1sy
create_time: 2026-11-15 13:50:17
status: wip
---

- **PARENT:**
  [202611/bead_k_preview.md](https://github.com/sase-org/sase--plans/blob/main/202611/bead_k_preview.md)
- **BEAD:**
  [sase-1sy](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1sy/README.md)

# Finish bead previews and land sase-1sy

## Outcome and scope

Complete the remaining correctness work from the landing audit of epic `sase-1sy`.
Resolved mentions refresh on discrete triggers, config changes cannot strand their
worker, mention colors use the correct owning store, and delayed requests never replace
the bead or prompt the user has moved on from. Then close `sase-1sy` and mark its
original plan done in this same implementation turn. This tale has no land agent; the
coder owns the entire closeout below.

Keep the accepted decisions: `mention_tint = status` and `jump_target = detail`. Keep
K's existing word-lookup fallback and the current card design. This is bounded work in
the existing prompt/card modules and their tests, suitable for one medium coding agent.
Do not redesign the feature or migrate the prompt recognizer as part of this repair.
Reuse the existing Rust-backed bead routing and read APIs; any new shared domain
behavior belongs in `sase-core`, not a duplicate Python resolver.

## Evidence already established

The lander audited the original plan with
`sase artifact read plan:202611/bead_k_preview.md`, read `sase-1sy`, and read every note
on its three children. The epic had no notes and no parent at that audit; all three
phases were closed done. Their commits are:

- `4006ec36ad`: recognition, warm directory, K fallback and core card.
- `baf23097d6`: relations, navigation, full detail, help/docs and card goldens.
- `0c929393a5`: status-tinted mentions, index, triggers and prompt goldens.

The source implements the selected decisions, the advertised card content and read-only
preview loading. Phase `.2`'s recovery note is addressed: `_build_bead_progress_meter`
is private and has its in-file consumer. `sase bead epic-symbols sase-1sy` returned no
entries.

Fetched `origin/master` matched `0c929393a5`. The only intervening non-epic commit since
`4006ec36ad` is `974c600c6a` (deferred prompt-archive grace); its production diff does
not consume or duplicate this feature. The wider interval since epic creation also
includes `c41c1e6337` (wait-marker scan optimization), likewise unrelated after source
review. No integration edit was warranted at that tip. Recheck later drift before
closeout.

Existing focused tests passed: 60 tests across recognition, preview model, mention index
and card rendering, then 34 across prompt preview, mentions, card modal and pager
resolution. The second run reported an unawaited `run_resolve` coroutine from the
mention-test fake worker; clean up that test worker lifecycle while extending the
coverage. These tests do not exercise the failures below.

Follow-up triage is already complete. The sole `PROPOSED FOLLOW-UP`, on `sase-1sy.3`
note #1, reports pre-existing `bench_tui_jk` failures on both clean base and feature
trees, specifically Axe samples from another tab. `/sase_new_task` found exact existing
task `sase-199` and recorded a +1 identifying the proposing bead (now three independent
reports). All-status searches, the last-week task sweep, and the active-epic sweep found
no additional distinct actionable report or credible causal epic owner. No duplicate
task was created. The generic remaining "slow benches" wording supplied no distinct
node/root-cause evidence and does not justify another task or block this epic. Do not
rerun or repair the broad slow benchmark suite as a substitute for feature verification.

## 1. Make discrete mention triggers revalidate cached results

Affected files:

- `src/sase/ace/tui/widgets/_prompt_bead_mentions.py`
- `src/sase/ace/tui/actions/_startup_prompt_bead_directory.py`
- `src/sase/ace/tui/models/bead_preview.py`

`_request_bead_mention_resolution_for_text` currently queues an ID only when
`index.peek(id) is None`. A cached status or cached miss therefore never reaches the
fingerprint check when the prompt contains only cached IDs. The lander's in-memory
reproduction seeded `sase-1ss` as in-progress with an old token, called the trigger, and
observed `requests == []`.

On mount, directory publication and NORMAL entry, schedule bounded coalesced
revalidation of the visible IDs even when cached; keep all fingerprint reads and status
queries off the UI thread. Reuse `resolve_bead_mention_updates` so an unchanged,
successfully obtained fingerprint avoids status re-query. Missing IDs must become
visible when created, and changed/deleted beads must update or lose their underline
after the next trigger. An unavailable fingerprint must not be treated as permanent
proof that a cached entry is current. Keep render/typing paths memory-only with the
existing text-size guards and bounded index.

Ensure status/token seeding from a card cannot certify an old status with a token read
after that status changed. Use a consistent read/token observation or seed as needing
revalidation. Preserve the chosen status palette and artifact-ref styling.

## 2. Repair generation changes and pending worker recovery

`_invalidate_bead_store_directory` increments the directory generation but leaves
`_bead_mention_resolution_in_flight` true. The old worker clears it only if its
generation is current. The lander paused `asyncio.to_thread`, invalidated and published
a new directory, queued an ID, then released the old worker. Result: `in_flight=True`,
zero replacement workers, and the ID stuck pending.

Give the active request explicit generation/ownership state, or an equivalent small
lifecycle fix. Invalidating a generation must allow the replacement to run; an old
worker must neither publish stale entries nor clear a newer worker's flag. Directory
invalidation must lead back to a warm directory for mounted prompts, without waiting for
the user to open a new pane or press K. Coalesce requests within a generation, drain
pending requests, and release ownership on cancellation, exception and failed
scheduling. Avoid tight automatic retries on a failing store.

Add deterministic tests with barriers/events for old-worker completion both before and
after the replacement starts. Verify the eventual visible state, not just a boolean
flag. Make fake workers in the existing tests finish or close their coroutines so test
teardown produces no unawaited-coroutine warning.

## 3. Align mention ownership with preview routing

`_plan_bead_mention_resolution` uses `owners[0]` for every ID. With two snapshots
sharing `sase` as a prefix, the lander's reproduction always planned only the first
store. A bead present only in the second store is therefore cached as missing; a
duplicate may take an arbitrary status. K correctly uses the existing
`route_full_id_with_snapshots`/`route_bead_mention` path instead.

Use the existing Rust-backed routing contract off-thread to determine the actual owner.
Do not implement a second ownership algorithm. An unqualified ambiguous reference must
remain unstyled; a unique resolved bead must use its actual store. Honor the prompt's
`+project` disambiguation consistently where it is applicable. Scope cache/request keys
to the resolved store and prompt context as necessary so seeding one qualified card
cannot color a different prompt's ambiguous or differently qualified occurrence.
Preserve batching/fingerprint reuse for the common unique owner case. Keep this as UI
cache/glue around the existing core router.

Cover two stores with one shared prefix: only the second contains the ID; both contain
it with different statuses; a `+project` tag selects one; and two mounted prompts have
different contexts. Include the unavailable-store outcome and ensure no new I/O runs
during a keystroke or highlight rebuild.

## 4. Reject stale prompt and modal results

Affected files:

- `src/sase/ace/tui/modals/bead_preview_modal.py`
- `src/sase/ace/tui/widgets/_prompt_bead_preview.py`

Backspace restores a cached parent without advancing `_load_request_id`. A child load
started before Backspace is still considered current and can replace the parent's
preview. The lander exercised `action_go_back` and confirmed the old child request still
passes `_load_is_current`. `_detail_request_id` also survives navigation, so a delayed
Enter result can open the previously selected bead.

Capture bead identity when scheduling loads/detail builds and invalidate both request
streams on every identity transition, including cached Back, parent, child, reload and
close. Check identity and generation after every await before publishing, seeding or
painting. Keep the immediate loading state and in-card error/retry behavior.

Prompt K/Ctrl+] guards check only request counter, mount state and cursor. Replacing the
text at the same cursor leaves an old request accepted; this was reproduced by calling
the guard with the old counter/cursor and a different bead in `text`. Capture the
relevant text/selection/context revision at scheduling and reject results when that
snapshot no longer applies. Keep the keypress snapshot for the word fallback. Convert
directory/routing exceptions into an appropriate current request warning/fallback rather
than an unhandled Textual worker error.

Use delayed loaders/resolvers to test Back during child loading, navigating while Enter
is pending, rapid child/parent/reload changes, close during load, same-cursor text
replacement and directory invalidation. Assert that no stale card, pager, status seed,
toast or fallback appears.

## 5. Verify and finish the epic in this turn

1. Read the required TUI/performance and lint/test memory. Add focused regressions for
   the above failures and run them with the existing feature tests listed below.
   Preserve non-bead word/jump behavior and audit-free preview reads.
2. Run `just fix` as appropriate and `sase tool run check` (the declared `just check`
   recipe). Do not run `just check-full`. Use `/sase_monitor` if needed for a long gate;
   its follow-up must explicitly finish this same closeout. There is no separate lander
   waiting to do it. Do not require the final code's commit SHA, push or CI result
   before closing.
3. If rendered output changes, run targeted `just fix-tui-screenshots` for the bead
   card/mention fixtures and inspect every generated image per TUI snapshot memory.
   Preserve current goldens when only async lifecycle changes. Recheck help/docs if the
   repair changes any documented behavior.
4. Fetch and review changes since audit tip `0c929393a5`, reread `sase-1sy` and all its
   descendants/notes and the original linked plan through audited commands, and finish
   any feature integration required by that drift. The earlier notes, commits, decisions
   and follow-up disposition are summarized above; include this tale's results in the
   final close note. If this tale has its own descendant bead, close that completed bead
   normally before closing its parent epic.
5. Run `sase bead epic-symbols sase-1sy`. Resolve each listed exemption by wiring,
   privatizing, adding a valid non-test consumer pragma, or deleting a dead symbol;
   re-key only to a still-open later bead that actually needs the exemption. Then run
   `sase bead close sase-1sy --note "<concrete verification, drift review and follow-up outcomes>"`.
   If named phases or symbol entries block closing, finish them and retry. Never use
   `--force` to make a successful nested landing advance. A deliberate
   canceled/superseded outcome needs its real reason, not convenience.
6. After the close, run `just symvision` to confirm the whitelist is clean. Open the
   plans repo with `/sase_repo`
   (`sase repo open plans -r "Mark completed epic plan done"`), use its printed path,
   and set `status: done` in `202611/bead_k_preview.md` (original artifact
   `plan:202611/bead_k_preview.md`). Use `sase artifact read` for content reads. Include
   that repository's edits in `/sase_final`.
7. Run `sase bead read sase-1sy -r "Need the parent link"`. No parent existed in the
   lander's audit. If still absent, finish normally. If a new parent is present, inspect
   it: for a phase, verify this work fulfills it and close only that phase normally;
   leave its containing epic for its waiting lander. For a plan, reread its previous
   landing note, all descendants/notes and linked plan, inspect drift, and rerun
   readiness checks. Close only if complete, retire its epic symbols, run
   `just symvision`, and mark its linked plan done; repeat through directly parented
   complete plans. At the first incomplete/ambiguous parent, leave it open, record the
   blocker there and report it. Never force a successful chain.
8. End through `/sase_final` with all changed repos committed declaratively. The epic
   closes in the same turn as the final verified code, before host commits.

Focused existing tests:

```text
tests/test_bead_mentions.py
tests/ace/tui/models/test_bead_preview.py
tests/ace/tui/models/test_bead_mention_index.py
tests/ace/tui/modals/test_bead_preview_render.py
tests/ace/tui/modals/test_bead_preview_modal.py
tests/ace/tui/widgets/test_prompt_bead_preview.py
tests/ace/tui/widgets/test_prompt_bead_mentions.py
tests/pager/test_resolve_beads.py
```
