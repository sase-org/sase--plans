---
tier: tale
title: Keep the first-paint reply placeholder until Muse text arrives
goal: "Selecting a live agent keeps that Reply card's first-paint empty copy until a
  timestamp chunk or reply text exists. A followed reply that is later cleared uses the
  same solo waiting sentence. Epic sase-1fu is then closed.

  "
size: small
proposed_by: bbugyi200.athena.sase-1fu.land
bead: sase-1fu
create_time: 2026-10-04 09:35:40
status: wip
---

- **PARENT:**
  [202610/muse_reply_streaming.md](https://github.com/sase-org/sase--plans/blob/main/202610/muse_reply_streaming.md)
- **BEAD:** sase-1fu

# Keep the first-paint reply placeholder until Muse text arrives

## Remaining work

Epic sase-1fu (plan `sase/repos/plans/202610/muse_reply_streaming.md`) is otherwise
complete. Its four phases are closed. This tale fixes the one defect the landing review
found in the epic's own code, then closes the epic. Do not reopen the phases. Do not
expand into transport, console framing, goldens, or a real Muse API capture.

## Cause

`LiveReplyFollowMixin` in `src/sase/ace/tui/widgets/prompt_panel/_live_reply_follow.py`
collects a snapshot as soon as a qualifying selected agent is shown.
`_snapshot_renderables` turns an empty snapshot (no timestamp chunks and no stripped
`live_text`) into `_WAITING_REPLY = "Waiting for agent response.\n"` and `collect_apply`
replaces the live-reply region.

That clobbers the first paint:

- A solo running agent first paints `Waiting for agent response...` in
  `src/sase/ace/tui/widgets/prompt_panel/_agent_display_render.py`.
- An empty in-flight session phase first paints `No response content yet.` in
  `src/sase/ace/tui/widgets/prompt_panel/_agent_display_agent_session_render.py`.

`docs/ace.md` (Agents tab, AGENT REPLY) currently documents the third, post-follow
sentence with a period. That sentence is the bug, not a second product string.
`docs/llms.md` does not mention these placeholders.

## Fix

Stay inside `_live_reply_follow.py`, `docs/ace.md`, and
`tests/ace/tui/widgets/test_live_reply_follow.py`.

1. Change `_WAITING_REPLY` to `"Waiting for agent response...\n"` so a cleared reply
   matches the solo first paint, including the ellipsis.
2. Remember whether this follow has already applied a snapshot that contained a
   timestamp chunk or stripped reply text. Clear that memory in
   `cancel_live_reply_follow`. Do not treat "signatures were recorded" as "body was
   shown".
3. In `collect_apply`, after the idle and generation recheck and before the
   equal-signature return: when the snapshot has no chunks and no stripped live text,
   and no reply body has been shown yet, record `snapshot.signatures` as applied, mark
   the collection accepted, and return without calling `update`. The first-paint
   placeholder stays. A later write that only changes the empty files' mtime or size
   must also leave that placeholder in place.
4. When an empty snapshot arrives after a reply body was shown, replace the region with
   the ellipsis waiting text and then treat the body as not shown, so further empty
   signature changes do not publish again. Timestamp chunks, including an empty chunk
   under a divider, are content and must still replace the region.
5. Non-empty snapshots keep the current replace path. Do not change throttle, poll,
   watcher routing, scroll, or which agents qualify.

## Docs

In `docs/ace.md`, replace the three-string empty-reply paragraph so it says:

- The first paint of an empty solo reply says `Waiting for agent response...`.
- An empty session phase first says `No response content yet.`.
- The follow leaves that first-paint text in place until a timestamp chunk or reply text
  exists.
- If a reply that was already followed is cleared, the body becomes
  `Waiting for agent response...`.
- Hint mode still keeps the first-paint text because the follow stays off.

Leave the rest of that AGENT REPLY section as it is.

## Tests

Extend `tests/ace/tui/widgets/test_live_reply_follow.py`. Use the existing
`_ReplyController` harness.

- An empty `live_reply.md` and empty timestamps file, with the region already showing
  `Waiting for agent response...`, must not publish a document. The visible text keeps
  the ellipsis. A second fire after touching the empty files must still not publish.
- The same controller, after a non-empty snapshot has been published, must publish the
  ellipsis waiting sentence when both files are truncated to empty, and must not publish
  again when the empty files are touched.
- The existing non-empty publish test must still publish `first words`.

Run:

```bash
.venv/bin/python -m pytest tests/ace/tui/widgets/test_live_reply_follow.py -q -o addopts=
```

Read `lint_and_test.md` with `sase memory read` before the guarded check. File-change
verification is `just check`. Do not run `just check-full`.

## Integration already reviewed

Do not redo this. Commits after `ca1ffac8e3` that are not sase-1fu commits do not need
code changes:

- `e7408ed219` only retargets the macro syntax import in `_agent_display_render.py`.
- `594766b750` and `727b64af7b` already document live Reply following; this tale only
  corrects the empty-copy sentences in `docs/ace.md`.
- `458dfe59dc` restarts the TUI process. `AgentPromptPanel.on_unmount` already cancels
  the follow.
- `2807692dc9` changes list-row finalizer labels and does not render the Reply card.
- JSONL providers still call `stream_json_lines`. Muse still calls
  `append_stream_delta`, which skips the Rich `FileProxy` flush.

## Closeout

Do this in the same turn as the code, after the focused tests pass. Do not wait for this
turn's commit SHA, push, or CI. `sase-1fu` has no `parent_id`. After the epic closes,
stop. Do not close another bead.

1. Run `sase bead epic-symbols sase-1fu`. Landing review found no `--epic-symbol`
   entries. If any are listed, resolve each one (wire it, privatize it, add a non-test
   pragma, or delete it) or re-key the Justfile line only to a still-open later bead
   that still needs the exemption. `sase bead close` refuses while any remain. Do not
   use `--force` to get past them.
2. Close the epic:

```bash
sase bead close sase-1fu --note "<the note below>"
```

Use a note that states all of the following:

- Verified phases sase-1fu.1 through sase-1fu.4 are closed and their code is on master:
  selected Reply follow in `_live_reply_follow.py`, bounded JSONL reads in
  `_subprocess_stream.py`, Rich `FileProxy` framing in `_subprocess_artifacts.py`, the
  mounted gated-Muse test, and the `docs/llms.md` streaming paragraph.
- This tale stops the empty snapshot from replacing first-paint reply copy, and
  `docs/ace.md` matches that behavior. Name the focused pytest result.
- Follow-ups, already triaged before this tale: sase-1fu.2's `tests/xprompt` terminology
  failure is the ignored-directory scan already recorded on epic sase-1eq (corroborated
  again at landing; no new task). sase-1fu.3's five `runner_kill_provenance.py`
  Symvision symbols are the pre-existing set already recorded on epic sase-1c1
  (corroborated again; no new task). sase-1fu.4's prompt-tab focus failures are flake
  task sase-1fy, linked to the proposing phase. The real Muse capture after the API 429
  was declined: installed Muse 1.4.2-R4684.1 was quota-blocked until
  2026-10-05T00:00:00Z, and the mounted production parser already proves
  transport-to-paint. The Muse comments that say a terminal event never returns deltas
  were declined: a textless terminal still returns deltas, which is the salvage
  contract, and `_resolve_muse_content` already says so.
- `sase bead epic-symbols sase-1fu` result at close.
- No parent bead.

If close is rejected because named phases were never completed, finish or reopen them.
Do not pass `--force` just to make close succeed.

3. Run `just symvision`.
4. Set `status: done` in the frontmatter of
   `sase/repos/plans/202610/muse_reply_streaming.md`. The current value is
   `status: wip`. Change only that field.
