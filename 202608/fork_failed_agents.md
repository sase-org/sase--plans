---
tier: tale
size: medium
title: Fork failed agents with the failure rendered in the injected transcript
goal:
  Pressing `F` on a FAILED row in the ACE Agents tab (and `#fork:<name>` on any
  terminally failed agent) produces a working fork whose injected history states
  unmistakably that the parent agent failed, renders the parent's failure message inside
  the conversation transcript, and never strands the new agent on an unreachable implied
  wait.
proposed_by: bbugyi200.athena.0cz
create_time: 2026-08-24 17:31:20
status: wip
---

# Plan: Fork failed agents

## Symptom

Forking a failed agent does not work today, at every layer:

1. **The TUI refuses.** `F` (`edit_hooks`) on a `FAILED` row notifies
   `"Agent not finished yet"` and opens nothing — `resolve_agent_prompt_target_scope()`
   in `src/sase/ace/tui/actions/agents/_wait_helpers.py:281`. `FAILED` is in
   `DISMISSABLE_STATUSES`, so it skips the running-agent branch
   (`_wait_helpers.py:250`), is not a plan handoff (`:260`), and is not in
   `RESUMABLE_DONE_STATUSES` (`src/sase/ace/tui/models/agent_status.py:27`). The footer
   never advertises the action either: the fork hint is explicitly suppressed for
   `FAILED` rows in `src/sase/ace/tui/widgets/_keybinding_bindings.py:338-345`
   (`if agent.status != "FAILED":`), and is additionally gated on `agent.response_path`,
   which failed rows never have.

2. **Name resolution fails.** `build_done_marker()` records `response_path` **only** for
   `outcome == "completed"` (`src/sase/axe/run_agent_markers.py:130`), even though the
   failure branch of `finalize_loop()` already saved a transcript and passes it in
   (`src/sase/axe/run_agent_exec_finalize.py:168` and `:263`). A plain failed agent also
   never writes `agent_meta.json["chat_path"]` — only the pipe, plan, monitor, and
   questions paths do that (`run_agent_exec_pipe.py:111`, `run_agent_exec_plan.py:170`,
   `run_agent_exec_monitor.py:80`, `run_agent_exec_questions.py:218`). So
   `_resolve_agent_chat_path()` (`src/sase/scripts/agent_chat_from_name.py:120`)
   exhausts both lookups and raises `No agent with chat history found for: <name>`,
   which surfaces as a workflow-step crash _in the freshly launched child_, not as a TUI
   warning.

3. **Nothing would say it failed.** If resolution did succeed, the injected block would
   read as an ordinary conversation. `load_chat_for_resume()`
   (`src/sase/history/chat_resume.py:210-268`) renders each turn as
   `**User:** … **Assistant:** …`; a failed run saves an _empty_ response
   (`run_agent_exec_finalize.py:160-176` passes `response_content = ""` for any
   non-`completed` outcome), so the child would see a blank assistant turn and no hint
   that its parent died — while `done.json` holds the exact `error` and `traceback`
   (`run_agent_markers.py:140-143`).

4. **The implied wait can never resolve.** A top-level `#fork:<name>` implies
   `%wait:<name>` (`src/sase/axe/run_agent_directives.py:209-217`). Wait resolution only
   accepts `WAIT_SUCCESS_OUTCOMES` (`src/sase/core/dismissed_agent_completion.py:26`),
   and the `wait_checks` chop merely _logs_ terminal blockers without ever writing
   `ready.json` (`src/sase/scripts/sase_chop_wait_checks.py:180-296`). So even after
   fixing 1–3, forking a terminally failed agent would launch a child that sits in
   `WAITING` forever.

## Goal

- `F` on a `FAILED` row opens the prompt bar prefilled with `#fork:<name> `, exactly
  like a `DONE` row.
- The launched child receives a `# Previous Conversation` block that says, in its
  heading and its first paragraph, that the parent **failed**, renders the parent's
  failure message (and a bounded traceback tail) as part of the transcript, and marks
  the point where the transcript stops.
- The child starts immediately instead of waiting on a dead parent.
- Multi-parent forks (`#fork:a,b`) keep working and mark exactly the failed parents.

## Design

### Change 1 — persist the transcript path on failed done markers

File: `src/sase/axe/run_agent_markers.py`, `build_done_marker()`.

A failed run _does_ have a transcript: `finalize_loop()` saves it and already passes
`response_path=saved_path` into the failure branch. Record it:

```python
if outcome == "completed":
    ...existing result fields...
elif response_path:
    # A failed/terminal run still saved its transcript before the marker was
    # written; it is the primary evidence of the failure, so record the path
    # without the completed-only result fields (step_output, diff, artifacts).
    marker["response_path"] = response_path
```

Only `response_path` is added — deliberately not `step_output` / `diff_path` /
`plan_path` / the artifact lists, so "completed outcome always includes result fields"
keeps its meaning. Downstream, `done.response_path` is already typed on the wire
(`src/sase/core/agent_scan_wire_markers.py:46`), so no Rust
(`../sase-core/crates/sase_core`) change is needed; the TUI's `Agent.response_path`
(`_done_loaders.py:330`, `:540`) simply becomes populated for failed rows, which also
makes the existing "open the chat" surfaces work on them. This additionally makes nested
expansion work: `resolve_resume_to_chat_path()` (`chat_resume.py:94-127`) reads
`done.json["response_path"]` with no outcome check, so a transcript that itself contains
`#fork:<failed-parent>` now expands.

### Change 2 — failure-aware fork source resolution

File: `src/sase/scripts/agent_chat_from_name.py`.

Add a `_ForkFailure` frozen dataclass and an optional `failure` field on the
`kind: "agent"` wire shape emitted by `_ForkSource.to_json_data()` (`:81`):

```json
{
  "kind": "agent",
  "name": "alpha",
  "path": "~/.sase/chats/202608/main-ace-run-alpha-20260824150102.md",
  "failure": {
    "outcome": "failed",
    "error": "RuntimeError: boom",
    "traceback": "Traceback (most recent call last):\n…",
    "ended_at": "2026-08-24 15:04:05 EDT",
    "transcript_available": true,
    "launch_prompt": null
  }
}
```

`failure` is absent for successful parents, so every existing wire consumer and test is
unchanged.

Resolution, in a new `_resolve_agent_fork_source(name)` that `_resolve_fork_source()`
(`:280`) uses for the `kind == "agent"` branch. Keep `_resolve_agent_chat_path()` as-is
(tests import it, and it stays the successful path) and layer the failure handling
around a shared artifact lookup:

1. Resolve the named agent as today. If the resolved artifact has no `done.json` failure
   outcome, behave exactly as today.
2. When `done.json["outcome"]` is in
   `sase.core.dismissed_agent_completion.FAILURE_OUTCOMES` (`failed`, `killed`,
   `stopped`, `epic_launch_failed`), build the `failure` payload from `outcome`,
   `error`, `traceback`, and `finished_at` (formatted with
   `sase.core.time.get_timezone()` into the same `%Y-%m-%d %H:%M:%S %Z` shape
   `write_chat_history()` uses, so the resolver — not the renderer — owns formatting).
3. Resolve the transcript with fallbacks, first hit wins: `done.json["response_path"]` →
   `agent_meta.json["chat_path"]` → `find_chat_by_timestamp(<artifact dir name>)`
   (`src/sase/history/chat_storage.py:298`). The third tier is what makes
   _already-existing_ failed agents forkable, since they predate Change 1. Each
   candidate goes through `_validate_readable_transcript()` (`:509`) and an unreadable
   candidate falls through to the next tier rather than aborting.
4. If no transcript resolves (a runner error before `finalize_loop` ever saved one —
   `src/sase/axe/run_agent_runner_errors.py:29`), do **not** raise. Emit
   `transcript_available: false`, `path: ""`, and `launch_prompt` read from the artifact
   dir's `raw_xprompt.md`, passed through
   `sase.history.chat_resume.sanitize_resume_prompt()` and truncated to
   `_MAX_LAUNCH_PROMPT_CHARS = 2000`. A failed launch is exactly when the user most
   wants to fork, and the parent's prompt plus its error is genuinely useful context.
5. `main()` (`:236`) keeps printing the legacy `"path"` compatibility field; it becomes
   `""` for a transcript-less parent. Nothing reads it — `src/sase/xprompts/fork.yml`
   only consumes `sources_json` — so document that in the existing comment.

`_coalesce_fork_sources()` (`:186`) de-dupes on transcript path; a transcript-less
failed source has no path, so give it a sentinel that cannot collide (skip
de-duplication when `path` is empty) instead of collapsing two such parents into one.

Non-goal inside this change: family and clan sources keep their current rules — a failed
family member stays listed under **Not shown** (`agent_chat_from_name.py:337`) and an
incomplete clan still refuses to fork (`:288`).

### Change 3 — render the failure in the injected history

File: `src/sase/history/chat_fork.py`.

Add module constants `_MAX_FAILURE_MESSAGE_CHARS = 4000`, `_MAX_TRACEBACK_LINES = 20`,
and two helpers:

- `_fork_source_failure(source)` → the validated `failure` mapping or `None`.
- `_format_failure_block(name, failure)` → the shared block below.

**Single failed agent parent** (`build_fork_injected_history()`, the `len == 1` branch
at `:37`) produces:

````markdown
%xprompts_enabled:false

# Previous Conversation — PARENT AGENT FAILED

**The parent agent `alpha` did not finish: it ended with outcome `failed`.** Everything
below is the transcript of that failed run, so it is incomplete — the last reply may be
missing, truncated, or describe work that was never finished. Do not assume any of it
succeeded: verify the repository, artifacts, and any claimed results yourself, and treat
diagnosing the failure as part of the New Query unless told otherwise.

## Parent Failure — agent `alpha`

- **Outcome:** `failed`
- **Ended:** `2026-08-24 15:04:05 EDT`

**Failure message:**

```text
RuntimeError: boom
```

**Traceback (last 20 lines):**

```text
  File "…/run_agent_exec.py", line 210, in _run
    …
RuntimeError: boom
```

## Transcript — agent `alpha`

**User:**

…

**Assistant:**

…

**End of transcript — agent `alpha` failed here: `RuntimeError: boom`.**

---

%xprompts_enabled:true

# New Query
````

Rules that make this hold together:

- The heading suffix comes from `_wrap_fork_history()` (`:95`) taking the heading it is
  already given; the failed single-parent branch passes
  `"# Previous Conversation — PARENT AGENT FAILED"`.
- The trailing **End of transcript** line repeats the failure's first non-empty line so
  the signal sits adjacent to the empty `**Assistant:**` turn the child would otherwise
  read as a normal reply. Repeating it is intentional: the top block is for orientation,
  the bottom line is for the truncation point.
- Fenced blocks use ` ```text ` so an error containing markdown or backticks cannot leak
  into the surrounding structure. Truncated content is marked with a final
  `… (truncated)` line.
- Missing fields degrade: no `error` renders `**Failure message:** _(none recorded)_`;
  no `traceback` omits that block; no `ended_at` omits that row.
- `transcript_available: false` replaces the `## Transcript` section with:

  ```markdown
  ## Transcript — agent `alpha`

  _No transcript was saved: the agent failed before it recorded one._

  **Its launch prompt was:**

  > …sanitized launch prompt…
  ```

  and omits the launch-prompt part entirely when `launch_prompt` is absent.

**Multiple parents.** Both multi-source branches (`:41` all-agent, `:61` mixed) reuse
`_format_failure_block()`:

- The section heading of a failed parent gains a marker:
  ``## Conversation 2 of 3 — agent `alpha` (FAILED)`` and
  ``## Source 2 of 3 — agent `alpha` (FAILED)``.
- The failure block and the **End of transcript** line are emitted inside that parent's
  section only.
- The shared guidance paragraph gains one sentence when any parent failed:
  `"One or more parent sections are marked FAILED: those transcripts are incomplete and their work is unverified — check the marked sections before relying on anything they claim."`

### Change 4 — do not imply a wait that can never resolve

Files: `src/sase/agent/names/_lookup_resolution.py` (+ export in
`src/sase/agent/names/__init__.py`), `src/sase/axe/run_agent_directives.py`.

Add:

```python
def fork_parent_wait_is_unreachable(name: str) -> bool:
    """Return whether a `#fork` parent is terminally failed, so waiting is futile."""
```

Implementation, deliberately cheap (no wait index build — the directive path runs on
every launch):

- Tribe references (`parse_tribe_reference`) → `False`; they bind dynamically to a
  _future_ entity and must keep waiting.
- A name that resolves to a clan (`find_agent_clan`) or family (`find_agent_family`) →
  `False`; group completeness rules are unchanged by this plan.
- Otherwise `agent = find_named_agent(name)`; return
  `agent is not None and agent.is_done and agent.outcome in FAILURE_OUTCOMES`.
  `find_named_agent()` prefers a _live_ agent of that name
  (`src/sase/agent/names/_lookup_named.py:138-150`), so a newer running namesake
  correctly keeps the wait.

In `run_agent_directives.py:209-217`, skip the implied append when the helper returns
`True` (explicit `%wait:` entries from `directives.wait` are never removed — a user who
typed the wait keeps it):

```python
if fork_wait_target in wait_names:
    continue
if fork_parent_wait_is_unreachable(fork_wait_target):
    continue
wait_names.append(fork_wait_target)
```

### Change 5 — let the Agents tab fork failed rows

Files: `src/sase/ace/tui/models/agent_status.py`,
`src/sase/ace/tui/actions/agents/_wait_helpers.py`,
`src/sase/ace/tui/widgets/_keybinding_bindings.py`.

- Add `is_failed_agent_status(status: str) -> bool` next to
  `is_revertable_agent_status()` (`agent_status.py:45`), returning
  `status.startswith("FAILED")` so `FAILED` and `FAILED (RETRIED)` are one concept, and
  reuse it inside `is_revertable_agent_status()`, which already open-codes that test.
- In `resolve_agent_prompt_target_scope()`, insert a branch after the plan-handoff block
  and before the `is_resumable_done_status()` rejection (`_wait_helpers.py:279`):

  ```python
  if is_failed_agent_status(agent.status):
      if not prompt_name:
          return None, "No agent name found"
      return (
          agent_prompt_target_scope(agent, prompt_name, history_fallback="fork"),
          None,
      )
  ```

  This only affects `action == "fork"` (the `wait` action returns earlier at `:236`).
  `FAILED (RETRIED)` already reached the fork path incidentally, by not being in
  `DISMISSABLE_STATUSES`; routing it through the explicit branch makes that deliberate
  rather than accidental.

- In `_keybinding_bindings.py:338-345`, advertise the action on failed rows:

  ```python
  if agent.status == "FAILED" or is_resumable_done_status(agent.status):
      if marked_count == 0:
          bindings.append((x, "dismiss"))
      if agent.status == "FAILED":
          bindings.append((self._kd("edit_hooks"), "fork"))
      else:
          ...existing edit-chat / response_path-gated fork...
  ```

  The failed hint is **not** gated on `agent.response_path`: Change 2's timestamp-scan
  fallback makes pre-existing failed agents forkable even though their markers have no
  `response_path`. The enclosing `agent.status == "FAILED"` branch condition is left
  alone, so `FAILED (RETRIED)` footer rows keep rendering through the active-status
  branch exactly as they do today.

### Documentation

- `docs/xprompt.md` (`#fork` prose, ~`:1368-1386`): a paragraph stating that a
  terminally failed parent is a valid fork source, that its injected block is marked
  `PARENT AGENT FAILED` and carries the recorded failure message, and that the normally
  implied `%wait` is skipped for an already-failed parent.
- `docs/ace.md` ("Forking Agents and Groups", ~`:1024`): failed rows are fork targets.

## Tests

1. `tests/test_fork_history.py` — renderer, driven purely off wire dicts:
   - single failed agent source → heading contains `PARENT AGENT FAILED`, body contains
     the outcome, the exact error text, the `Traceback (last 20 lines)` block, and the
     `End of transcript` line; the transcript text is still present.
   - traceback longer than the cap → only the tail is present, and the truncation marker
     is rendered.
   - `error` absent → `_(none recorded)_`, no crash; `traceback` absent → no traceback
     block.
   - `transcript_available: false` with a `launch_prompt` → no `**Assistant:**` turns,
     the "no transcript" sentence, and the quoted launch prompt.
   - two agent sources, second failed → only section 2 is marked `(FAILED)` and carries
     the failure block; the shared guidance gains the failed-parent sentence; a
     successful-only multi-source fork renders byte-identically to today (regression
     guard).
2. `tests/test_agent_chat_from_name.py` (helper `write_agent()` in
   `tests/_agent_chat_from_name_helpers.py` needs to accept a
   `chat file with no done.response_path` case):
   - failed agent whose `done.json` has `response_path` → source carries that path plus
     a `failure` payload with outcome/error/traceback.
   - failed agent with **no** `response_path` and no `chat_path`, but a chat file whose
     basename ends in `-<artifact suffix>.md` → resolved by the timestamp-scan tier.
   - failed agent with no transcript at all, with a `raw_xprompt.md` → `path == ""`,
     `transcript_available` false, sanitized `launch_prompt` present.
   - successful agent → **no** `failure` key (wire-compat guard).
   - mixed `#fork:good,bad` → both sources present, in order, only the second marked.
3. `tests/test_axe_run_agent_exec_finalize_metadata.py` (or the closest existing
   done-marker test): a failed `finalize_loop` writes `response_path` and still omits
   `step_output` / `diff_path` / artifact lists; a completed one is unchanged.
4. Directive/wait coverage, alongside the existing implied-wait tests
   (`tests/test_directives_has_helpers.py`, `tests/test_run_agent_directive_metadata.py`
   — pick whichever already exercises `#fork` implied waits):
   - `#fork:<terminally failed>` adds no implied wait;
   - `#fork:<running>` still adds it;
   - an explicit `%wait:<failed>` typed by the user is preserved;
   - a failed agent shadowed by a newer live namesake still waits.
5. TUI:
   - `tests/ace/tui/test_agent_marking_wait_fork.py`: `F` on a `FAILED` named row
     prefills `#fork:<name> ` with `display_name == "fork(<name>)"`; an unnamed `FAILED`
     row still warns `"No agent name found"`; a `STOPPED` row still warns (scope guard —
     this plan does not widen to `STOPPED` / `PLAN REJECTED`).
   - `tests/test_keybinding_footer_agent.py`: a `FAILED` row advertises
     `(<edit_hooks key>, "fork")` even with `response_path=None`; the existing
     `DONE`-without-chat and proc-shell assertions still hold.

## Verification

- `just install` first (workspace venvs go stale), then `just check`.
- The diff touches the agent runner, the fork resolver, the history renderer, and the
  TUI, which is broad enough to warrant a final `just check-full` through
  `/sase_monitor`
  (`sase monitor start --command 'just check-full' --start-status TESTING --stop-status TESTED --next '…'`)
  before landing.
- `just test-visual` is not expected to be affected (no renderer/layout changes).
- Manual smoke: find or create a `FAILED` agent, press `F` on its row in `sase ace`,
  submit a trivial prompt, and confirm (a) the child starts immediately instead of
  sitting in `WAITING`, and (b) `sase chat show` on the child's transcript shows the
  `PARENT AGENT FAILED` block with the parent's error inside it.

## Non-goals and follow-ups

- **A parent that fails _after_ the fork launches.** `#fork:<running>` records its
  implied wait in `waiting.json` indistinguishably from an explicit `%wait`, so if that
  parent later fails, the child still waits forever. Fixing it means giving fork-implied
  waits their own channel in `waiting.json` that resolves on any terminal outcome. Out
  of scope here; worth its own task bead.
- **Failed family members and incomplete clans.** Family sources keep excluding failed
  members (listing them under **Not shown**) and clan forks keep requiring a complete
  generation. Now that a failure can be rendered, including them is a plausible
  follow-up, but it changes group semantics that other features depend on.
- **`STOPPED` and `PLAN REJECTED` rows.** Still not fork targets; they are terminal but
  not failures, and neither is what was asked for.
- **`docs/ace.md` says `f` while the default keymap binds `F`** (`edit_hooks` in
  `src/sase/default_config.yml:526`). Pre-existing doc drift, noticed while writing this
  plan; fix it opportunistically only if the docs paragraph is being rewritten anyway.
