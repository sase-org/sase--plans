---
tier: tale
title: Stop approved epic archives from dropping their PROMPT link
goal:
  An approved epic plan always lands in the SDD archive with its reciprocal `- **PROMPT:**` link, because the host-owned
  epic launch tells `archive_plan_file` that a prompt snapshot is guaranteed instead of racing the planner agent's push
  with a local filesystem probe. Hand-written epic plans launched by a bare `sase bead work <plan.md>` keep today's
  probe behavior and never gain a dangling link.
proposed_by: bbugyi200.athena.qp
create_time: 2026-07-31 15:48:31
status: done
---

- **PROMPT:** [202607/prompts/epic_archive_prompt_link.md](prompts/epic_archive_prompt_link.md)

# Plan: Stop approved epic archives from dropping their PROMPT link

## Background

Bead `sase-cq` reports that `just check` fails at the "SASE validation" step for every agent in this repo:

```
error: 202607/prompts/sase_beads_memory.md: 202607/sase_beads_memory.md is missing a valid 'prompt' link (reverse-link)
error: 202607/sase_beads_memory.md: missing 'prompt' link to 202607/prompts/sase_beads_memory.md (missing-link)
```

The prompt snapshot `202607/prompts/sase_beads_memory.md` holds its `- **PLAN:**` link correctly. The archived plan
`202607/sase_beads_memory.md` has a `- **BEAD:**` section but no `- **PROMPT:**` section, so the pair is half-linked.

### How the link is supposed to be written

`archive_plan_file()` in `src/sase/sdd/plan_archive.py` projects the reciprocal `PROMPT` section through
`project_plan_header_sections(..., prompt_path=...)`. It resolves that path in `_resolved_prompt_path()`:

- with `expect_prompt_snapshot=False` (the default) it returns the canonical prompt path **only if
  `archived_prompt.is_file()`** — a local filesystem probe;
- with `expect_prompt_snapshot=True` it returns the canonical path unconditionally.

The `expect_prompt_snapshot` docstring already names this exact failure: "the probe otherwise races the snapshot's own
write/push and can lose, permanently archiving the plan without its link."

### Root cause

`expect_prompt_snapshot` was wired into the two approval-side archive call sites, both as
`expect_prompt_snapshot=(tier == "epic")`:

- `src/sase/plan_approval_actions.py:467` (`_archive_plan_for_approval`)
- `src/sase/ace/tui/actions/agents/_notification_plan_background.py:59` (`archive_plan_for_approval`)

Both call sites are gated behind `approval_choice_archives_plan(choice)`. In `src/sase/plan_approval_choices.py:254` the
`epic` choice is registered with `archive_side_effect=False`, because an epic approval deliberately defers archiving to
the host-owned `sase bead work` background task. So **neither of those two call sites ever runs with `tier == "epic"`**
— the condition is unreachable in production, and the existing tests (`tests/test_tui_plan_approval_archive.py:53`,
`tests/test_plan_approval_actions.py:246`) only pass because they call the helpers directly, bypassing that gate.

The one archive call site that actually handles approved epics is `src/sase/bead/cli_work_from_plan.py:235`, inside
`_work_from_plan_file_locked()`:

```python
archive_result = archive_plan_file(
    source_path,
    store,
    tier="epic",
    destination_name=destination_name,
    preserve_existing=True,
)
```

It never passes `expect_prompt_snapshot`, so it takes the racy probe.

### Why the race is lost

Epic approval splits the two writes across two processes and two clones of the plans sidecar:

1. The planner agent's runner writes the prompt snapshot via `write_sdd_spec()`
   (`src/sase/axe/run_agent_exec_plan_accept.py:335`) and commits/pushes it from **its own** workspace clone.
2. The host separately submits a detached `sase bead work <plan.md>` task (`prepare_epic_launch` →
   `submit_epic_launch_task`), which archives the plan from the **primary workspace's** clone.

The comment at `run_agent_exec_plan_accept.py:458` states outright that "the host-owned epic launch continues
independently". For `sase_beads_memory` both commits carry the same timestamp (15:00:50): the agent's
`Add SDD prompt for sase_beads_memory` push had not landed in the host's clone when the host probed for it, so the plan
was archived with no `PROMPT` section. Because the archive uses `preserve_existing=True`, no later run repairs it.

The later `Link approved epic plan to its bead` step in `src/sase/bead/epic_from_plan.py:177` re-projects header
sections but uses the same probe (`archived_prompt if archived_prompt.is_file() else None`), so it missed the second
chance to heal the link too.

The race is intermittent, which matches the evidence: most recent epic plans in the archive do carry their
`- **PROMPT:**` link; `sase_beads_memory` is the pair that currently fails validation.

### Why `expect_prompt_snapshot=True` cannot simply be hardcoded

`cli_work_from_plan.py` also serves bare `sase bead work <plan.md>` runs on hand-written epic plans that never had an
approval-written prompt snapshot. 145 of the 472 `tier: epic` plans in the archive have no prompt snapshot at all,
including recent ones (`202607/rollout.md`, `202607/smoke_test_epic.md`, `202607/store_git_write_contention.md`,
`202607/sase_doctor_diagnostics.md`). Projecting the link unconditionally there would emit a `link-missing-target`
error, which `src/sase/sdd/_link_validation.py:176` reports at `severity="error"` — reintroducing exactly the
`just check` breakage `sase-cq` is about. The signal must therefore be threaded from the caller that knows a snapshot
was written, not inferred from the tier.

## Goal

Approved epic launches archive their plan with the reciprocal `PROMPT` link regardless of push timing, while bare
hand-written `sase bead work <plan.md>` launches keep the existing probe and never gain a dangling link. The one
currently broken pair in the plans sidecar is repaired so `just check` reaches its test step again.

## Implementation

### 1. Add the internal `--expect-prompt-snapshot` flag to `sase bead work`

In `src/sase/main/parser_bead_lifecycle.py`, inside `register_bead_work_parser()`, add:

```python
parser.add_argument(
    "--expect-prompt-snapshot",
    action="store_true",
    help=argparse.SUPPRESS,
)
```

Place it in the existing alphabetical run of long options, between `--dry-run` and `--json`. This is an internal
subprocess argument built only by `build_epic_launch_argv()`, so per `sase/memory/cli_rules.md` it takes no short alias
and is hidden with `argparse.SUPPRESS`, matching the existing `--launch-feedback` precedent at line 247.

### 2. Emit the flag from the host-owned epic launch

In `src/sase/bead/epic_launch.py`, extend `build_epic_launch_argv()` with a keyword-only
`expect_prompt_snapshot: bool = True` parameter and append `--expect-prompt-snapshot` when it is set.

Default it to `True`: every caller of `build_epic_launch_argv()` builds or resumes an **approval-driven** epic launch,
and `run_agent_exec_plan_accept.py` always writes the prompt snapshot on that path. Keeping the parameter (rather than
hardcoding the flag) leaves an explicit seam for a future caller that does not own a snapshot.

Note the resume commands built from `build_epic_launch_argv()` in `src/sase/_plan_approval_epic.py:59,81,113` inherit
the flag automatically, which is correct — they resume the same approval-driven launch.

`error_with_resume()` in `src/sase/bead/cli_work_from_plan_helpers.py:159` needs **no** change: its resume commands
point at the already-archived path, where `archive_plan_file()` returns `written=False` before the prompt path is ever
resolved.

### 3. Thread the flag to the archive and bead-link write points

- `src/sase/bead/cli_work_entry.py`: read
  `expect_prompt_snapshot = bool(getattr(args, "expect_prompt_snapshot", False))` alongside the other flags near line 23
  and pass it into the `work_from_plan_file(...)` call.
- `src/sase/bead/cli_work_from_plan.py`:
  - add a keyword-only `expect_prompt_snapshot: bool = False` parameter to `work_from_plan_file()` (line 53) and to
    `_work_from_plan_file_locked()` (line 205), forwarding it through the `_work_from_plan_file_locked(...)` call at
    line 187;
  - pass `expect_prompt_snapshot=expect_prompt_snapshot` to `archive_plan_file(...)` at line 235;
  - forward it into `create_and_launch_epic_from_plan(...)` at line 374.
- `src/sase/bead/epic_from_plan.py`: add a keyword-only `expect_prompt_snapshot: bool = False` parameter to
  `create_and_launch_epic_from_plan()` and use it at line 177 so the bead-link step becomes a genuine second chance:

  ```python
  prompt_path=(
      archived_prompt
      if expect_prompt_snapshot or archived_prompt.is_file()
      else None
  ),
  ```

  This matters for the `preserve_existing=True` case, where a retry finds the plan already archived (possibly without
  its link) and the bead-link commit is the only remaining write point.

Keep the default `False` on every new parameter so nothing but an explicitly flagged approval launch changes behavior.

### 4. Repair the live broken pair

The code fix does not repair plans already archived. In the plans sidecar repo (resolve it with `sase repo path plans`;
**never** hand-edit or `git pull` the sidecar with raw git), run:

```bash
sase plan links repair --write
```

The dry run (`sase plan links repair`) currently reports exactly one action, and no other files:

```json
{
  "path": "202607/sase_beads_memory.md",
  "field": "prompt",
  "old": null,
  "new": "- **PROMPT:** [202607/prompts/sase_beads_memory.md](prompts/sase_beads_memory.md)"
}
```

Confirm the repair with `sase plan links validate`, which must report 0 errors afterward. Commit the sidecar change
through the normal sase commit path — do not craft a raw `git commit` in the sidecar.

## Testing

- `tests/sdd/test_plan_archive.py` already covers both `_resolved_prompt_path()` branches; no change needed there.
- Add to `tests/test_bead/test_cli_work_from_plan_store.py` (reusing the existing `_BeadsLocation` / monkeypatched
  `_resolve_context` scaffolding around lines 40-80): a test asserting that
  `work_from_plan_file(..., expect_prompt_snapshot=True)` reaches `archive_plan_file` with
  `expect_prompt_snapshot=True`, and a companion asserting the default call passes `False`. Capture the kwargs by
  monkeypatching `sase.sdd.plan_archive.archive_plan_file`, mirroring the existing patch at line 167.
- Add an end-to-end regression test (same module) proving the actual defect: with **no** prompt snapshot present on
  disk, an `expect_prompt_snapshot=True` plan-file launch produces an archived plan containing
  `- **PROMPT:** [...](prompts/<stem>.md)`, and the default launch produces one without it. Assert on the archived
  file's parsed link via `parse_sdd_artifact_link`, as `tests/sdd/test_plan_archive.py` does.
- Add a CLI wiring test asserting `build_epic_launch_argv(...)` includes `--expect-prompt-snapshot` by default and omits
  it when `expect_prompt_snapshot=False`, next to `test_build_epic_launch_argv_carries_approval_linking_options` in
  `tests/test_bead/test_epic_launch.py:23`. That existing test asserts an exact argv list, so it needs updating for the
  new default flag.
- Add a test that `sase bead work` parses `--expect-prompt-snapshot` into `args.expect_prompt_snapshot`.
- Add a test for the `epic_from_plan` second chance: with no prompt file on disk and `expect_prompt_snapshot=True`, the
  content handed to `commit_plan_update` contains the `- **PROMPT:**` section.
- Run `just install` first (ephemeral workspace), then `just check`. The SASE validation step must pass, which is the
  user-visible acceptance criterion from `sase-cq`.

## Notes

- Leave the now-provably-unreachable `expect_prompt_snapshot=(tier == "epic")` conditions in
  `plan_approval_actions.py:467` and `_notification_plan_background.py:59` in place. They are correct as written and
  cost nothing; removing them would only enlarge the diff and would silently break the epic path if
  `archive_side_effect` for the `epic` choice is ever flipped back to `True`.
- Do not "fix" this by making `archive_plan_file()` default `expect_prompt_snapshot` from `tier == "epic"`. That
  reintroduces `link-missing-target` errors for the 145 archived epic plans' worth of hand-written-plan workflow
  described in Background.
- Close bead `sase-cq` with a note recording the root cause and the verification performed once `just check` passes and
  the sidecar repair is committed.
