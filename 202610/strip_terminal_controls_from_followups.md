---
tier: tale
title:
  Strip terminal control bytes before they reach follow-up prompts and continuation
  capture
goal:
  A colored line in monitor or gate output can no longer break agent-delta capture and
  block an agent session's next monitor follow-up; any block that still happens names
  its cause and the recovery command.
size: medium
decisions:
  consolidate_strippers:
    ask:
      Should the two existing ANSI strippers also be switched over to the new shared
      helper?
    choices:
      leave: Only follow-up prompts and capture change, touching the fewest code paths
      consolidate:
        One escape grammar everywhere, but provider live_reply output changes slightly
    default: leave
    why:
      provider streaming is a hot, well-tested path that played no part in this failure
    answer: leave
proposed_by: bbugyi200.apollo.a8
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-24 12:08:38
status: wip
---

# Strip terminal control bytes before they reach follow-up prompts and continuation capture

## Why: the bob-cli-bc.4 failure

`bob-cli-bc.4` (project `gh_bobs-org__bob-cli`) stalled on 2026-10-24 with its last
member `bob-cli-bc.4--mon-4` showing as **Failed**. The chain of events:

1. Monitor `bob-cli-bc.4--mon-3` ran a script whose `cargo fmt --check` printed a
   colored diff. Its retained output contained
   `\x1b[31m-    assert!(empty_rows\n\x1b[m\x0f…`: ANSI SGR codes plus `\x0f` (SI),
   which is how `screen`/`tmux` terminfo writes `sgr0`.
2. `src/sase/monitor/followup_prompt.py` pasted the last 200 lines into the follow-up
   prompt under `## Last 200 lines of output` without changing those bytes.
   `untrusted_output_section` in `src/sase/turns/prompt.py` only neutralizes budget-span
   markers (`sanitize_span_text`). It does not remove control bytes.
3. The follow-up agent `bob-cli-bc.4--5` ran normally. Its turn-end agent-delta capture
   then failed. Its `continuation/capture_errors.jsonl` records
   `ValueError: validation: authored_local_request contains unsupported control characters`
   at stage `agent_delta:interrupted`. The error comes from sase-core
   `continuation::schema::validate_text`, which rejects every `char::is_control()`
   except `\n` and `\t`. `persist_agent_delta` passes the authored request through
   unchanged, so `bob-cli-bc.4--5` was left with no `continuation_node_id`.
4. Its handoff monitor `bob-cli-bc.4--mon-4` (`just check`, exit 101) published a result
   with `parent_node_ids: []`. Settlement blocked the follow-up:
   `monitor_followup_outcome: not-launchable`,
   `monitor result is missing its exact starter parent node; automatic dispatch is blocked pending recovery`.
   Nothing in `sase monitor show` says that the starter's capture failed, or why.

This has happened before. Since 2026-09-25 the same capture error appears 18 times, and
5 sessions dead-ended this way (`sase-198.2`, `sase-19f.4`, `50`, `sase-1m3.11`,
`bob-cli-bc.4`). Every failing prompt carried ESC bytes, mostly mypy's
`\x1b[1m\x1b[31merror:\x1b(B\x1b[m` from monitored `just check` runs.

The damage also spreads within a session. Session `50` shows it: only `50--8`'s raw
prompt contained ESC, yet `50--9` through `50--g` also failed capture. Each member's
captured authored request is the `#fork`-expanded prompt, which re-embeds the poisoned
ancestor. So one colored log line ends automatic continuation for the rest of the
session. The nesting itself is tracked separately as task bead `sase-1o1`, because it
also pushes the authored request toward the 256 KiB cap.

**Recovering bob-cli-bc.4 now (a manual step for the reviewer, not part of this plan).**
The `just check` run in `--mon-4` failed only the two Highlights continuation tests that
also fail on clean HEAD (1998 passed, 2 failed). Its frozen next action is therefore
"epic-symbols check, then close bob-cli-bc.4 with the PT3 p95 note". Resume it without
rerunning the check: `sase monitor resume h63b3e -m gpt-6-luna@xhigh`. Until this fix is
installed, the resumed member will hit the same capture failure. That only matters if it
hands off to yet another monitor.

## Goals

1. Untrusted output embedded in turn follow-up prompts and continuation replays contains
   no terminal escape sequences and no non-printing control characters.
2. Agent-delta capture never fails because the authored request contains control
   characters. The cleaned form is what gets replayed, so the failure cannot spread to
   later members.
3. When a monitor follow-up is still blocked for a missing starter parent, the recorded
   error names the starter's capture failure and the command that recovers it.

## Non-goals

- **No sase-core change.** The validator's rule that records are free of control
  characters stays. It is the producer's job to emit valid text. Both producers here are
  Python host-runtime code: prompt composition in `sase/turns/prompt.py` and capture in
  `sase/continuation_capture/`. No frontend has to reproduce this behavior, so it passes
  the Rust-boundary litmus test. A conformance test (below) pins the Python predicate to
  the Rust validator.
- Retained logs, `live_reply.md`, and `sase monitor show` output stay raw. The TUI and
  CLI already render them as ANSI-aware logs.
- The fail-closed "exact starter parent" dispatch rule is unchanged.
- The nested-ancestry growth of `authored_local_request` belongs to `sase-1o1`.

## Design

### 1. New shared helper: `src/sase/terminal_text.py`

Two public functions that share one compiled escape-sequence pattern:

- `strip_terminal_controls(text: str) -> str` cleans command output for display to a
  model:
  1. Remove escape sequences, in this order (a single alternation is fine):
     - OSC: `ESC ] … (BEL | ESC \)`
     - DCS/SOS/PM/APC strings: `ESC [PX^_] … ESC \`
     - CSI: `ESC \[ [0-?]* [ -/]* [@-~]`
     - nF sequences, including charset designation such as mypy's `ESC ( B`:
       `ESC [ -/]+ [0-~]`
     - two-character Fp/Fe/Fs: `ESC [0-~]`

     Removing whole sequences before dropping lone controls matters. Otherwise `ESC ( B`
     leaves a stray `(B` behind.

  2. Convert `\r\n` to `\n`. Then, within each line, keep only the text after the last
     bare `\r`. This is terminal overwrite semantics: progress bars collapse to their
     final frame, the same as sase-core's triage `normalize_line`.
  3. Drop every remaining Unicode general-category `Cc` character except `\n` and `\t`.
     That covers SI/SO (`\x0e`/`\x0f`), NUL, BEL, DEL, and C1 controls such as a stray
     `\x9b`.

- `continuation_wire_text(text: str) -> str` is the content-preserving variant used by
  capture. It applies the same sequence removal and turns `\r\n` and bare `\r` into `\n`
  (it drops no text). It then drops the remaining `Cc` characters other than `\n` and
  `\t`. The docstring should state that it mirrors sase-core
  `continuation::schema::validate_text`. Rust's `char::is_control` is exactly Unicode
  `Cc`, so `unicodedata.category(ch) == "Cc"` is the matching predicate.

Both functions return the input unchanged on a fast path when it contains no `Cc`
character other than `\n` and `\t`. Authored requests are already over 100 KB in long
sessions, so this keeps the common case cheap.

> [!decision] consolidate_strippers = leave
> `src/sase/llm_provider/_subprocess_artifacts.py::strip_ansi` and
> `src/sase/dev_update/stream_command.py::_sanitize_line` stay as they are. Neither
> handles SI or nF sequences, but neither feeds follow-up prompts or capture.

> [!decision] consolidate_strippers = consolidate Also make `strip_ansi` and
> `_sanitize_line` delegate to `terminal_text`, keeping their existing call signatures.
> Re-run the provider streaming and dev-update tests and update any expectations about
> leftover `\x0f` or `(B`.

### 2. Clean text where it enters prompts: `src/sase/turns/prompt.py`

- `untrusted_output_section`: call `strip_terminal_controls(text)` first, before
  `sanitize_span_text` and before `tail_text_by_lines_and_chars`. The 200-line and
  12,000-character tail budgets then count visible text.
- `fenced_block`: apply the same call before `sanitize_span_text`.

These two helpers cover every place untrusted text enters a turn prompt or a replay:

- monitor follow-up prompts (`src/sase/monitor/followup_prompt.py`), including the
  Command and Directory blocks, the "Selected diagnostics" block, the raw tail, and the
  checkpoint and host-completion blocks;
- gate-turn follow-up prompts (`src/sase/gate_turn/followup_prompt.py`), including the
  reviewer note and the output tail;
- continuation replay of monitor results
  (`src/sase/monitor/result_projection.py::render_monitor_evidence_section` and the
  command block).

### 3. Normalize the authored request at capture

In `src/sase/continuation_capture/agent_delta.py::persist_agent_delta`, right after
`authored_local_request = required_text(...)`, add
`authored_local_request = continuation_wire_text(authored_local_request)`. That cleaned
value is the one used for the node-id seed (`_agent_delta_node_id`) and the
`AgentDeltaWire` payload.

`prepared_prompt.json` and the `local_authored` text blob keep the raw bytes. They are
the debug archive, stored by ref and never validated inline, so no evidence is lost.
Nothing cross-checks the authored request's hash against the segment blob (verified in
sase-core and in Python).

The replay (`history/chat_fork/continuation/_render.py::_render_agent_delta`) renders
the cleaned delta, so later `#fork` members inherit clean text. That stops the spread.
It also covers sources other than monitor output: user-pasted CR/LF text, a `#fork` of
an older chat that already contains ANSI, and gate output.

Leave the other inline-validated wire text fields alone (monitor reason and next action,
checkpoint bodies). No failures have been seen there.

### 4. Say why dispatch is blocked and how to recover

- In `src/sase/continuation_capture/_monitor_parents.py`, add
  `starter_capture_error_summary(starter_artifacts_dir: str | None) -> str | None`. It
  reads `<starter>/continuation/capture_errors.jsonl` (`CAPTURE_ERRORS_FILENAME`) and
  returns `"<stage>: <message>"` for the last well-formed line. It returns `None` when
  the file is missing or unreadable, so it never adds a new way to fail. Export it the
  same way the module's other helpers are exported.
- In `src/sase/monitor/settlement.py`, when `blocked_reason` is set, build the
  `monitor_followup_error` value (and the returned error) as:
  - the existing reason;
  - if the reason is the missing-starter-parent error and a summary exists:
    `; starter <monitor_starter_agent> continuation capture failed (<summary>)`;
  - always: ``; recover with `sase monitor resume <short id> -k FILE` or `-m MODEL` ``.
    Use the existing `short_monitor_id` helper that `sase/monitor/resume.py` uses.
- Leave `continuation_capture_error` byte-for-byte equal to
  `_MISSING_STARTER_PARENT_ERROR`. `repair_missing_starter_parent_disposition` compares
  it exactly. Only the user-facing `monitor_followup_error` changes. `sase monitor show`
  renders that field as `Next: Needs attention - …`.

### 5. Docs

In `docs/reference/automation/monitor-operations.md`, in the `sase monitor resume`
paragraph that already explains the missing-parent repair, add one sentence. It should
say that when the starter settled but its continuation capture failed, the
`not-launchable` error names that capture failure and the `-k FILE` / `-m MODEL` resume
command. Read the `docs_writing` reference memory before editing `docs/`.

## Tests

- **New `tests/test_terminal_text.py`:**
  - the exact bob-cli bytes `"\x1b[31m-    assert!(empty_rows\n\x1b[m\x0f"` become
    `"-    assert!(empty_rows\n"`;
  - the mypy bytes `"x.py:1: \x1b[1m\x1b[31merror:\x1b(B\x1b[m Argument"` become
    `"x.py:1: error: Argument"`;
  - an OSC-8 hyperlink and a DCS string are removed;
  - CRLF input;
  - `"a\rb\n"` gives `"b\n"` from `strip_terminal_controls` and `"a\nb\n"` from
    `continuation_wire_text`;
  - tabs, newlines, and non-ASCII text (`é`, `✅`, `─`, emoji) are preserved;
  - fast-path identity for clean input.
- **Conformance with the Rust validator** (same file or `tests/continuation/`): for
  every `Cc` code point (`0x00`–`0x1f`, `0x7f`–`0x9f`), `validate_agent_delta` from
  `sase.core.continuation_facade` accepts a minimal delta whose `authored_local_request`
  is `continuation_wire_text(f"x{chr(cp)}y")`. The raw form must be rejected for every
  code point except `\n` and `\t`. Copy the minimal valid delta shape from the existing
  continuation tests.
- **`tests/test_turn_prompt_output_tail.py`:** ANSI-laden output renders with no `Cc`
  character other than `\n` and `\t`, and the tail budget counts the stripped text.
- **Monitor follow-up prompt** (next to
  `tests/monitor/test_monitor_followup_prompt_diagnostics.py`):
  `compose_followup_prompt` with ANSI in `output_text` and `selected_diagnostics_text`
  produces a prompt with no forbidden `Cc` characters. Do the same for a gate-turn
  follow-up in `tests/gate_turn/test_followup_prompt.py`.
- **Capture regression** (alongside `tests/continuation/test_capture_prompt.py` or the
  agent-delta capture tests): the authored request contains the bob-cli bytes. Expected:
  `persist_agent_delta` publishes a node, no `capture_errors.jsonl` line is written, the
  delta's `authored_local_request` is clean, and `prepared_prompt.json` still holds the
  raw text.
- **Settlement** (next to `tests/monitor/test_monitor_settlement_starter_race.py`): the
  starter artifacts dir has a `capture_errors.jsonl` entry and no node. Expected:
  `monitor_followup_outcome == "not-launchable"`, `monitor_followup_error` contains the
  capture message and `sase monitor resume`, and `continuation_capture_error` is
  unchanged, so the repair path still matches. Update any existing assertions that
  compare `monitor_followup_error` exactly.
- Existing follow-up prompt goldens contain no control bytes, so they should not change.
  If one does, investigate before regenerating it.

## Verification

Read the `lint_and_test` reference memory and follow it. That means formatting and lint,
then the canonical `just check` through the guarded tool path. Do not run
`just check-full` unless told to.
