---
tier: tale
size: medium
title:
  "E1 landing: fix scoreboard correctness bugs, re-capture evidence, close sase-1gu"
goal:
  "`sase instructions verify` and the `instructions` doctor checks report real delivery.
  Provider and agent filters select the right runs. Helper accepted and denied counts
  come only from real `sase final` attempts. The template marker is read from the helper
  prompt snapshot, the root guard-denial alarm works, Codex shows 2×, and `--since`
  units are sane. A corrected baseline JSON, after JSON, and acceptance record are
  attached to sase-1gu, and the epic is closed with its plan file marked done."
proposed_by: bbugyi200.athena.sase-1gu.land
bead: sase-1gu
create_time: 2026-10-06 11:16:20
status: wip
---

- **PARENT:**
  [202610/e1_instruction_scoreboard_and_stopgaps.md](https://github.com/sase-org/sase--plans/blob/main/202610/e1_instruction_scoreboard_and_stopgaps.md)
- **BEAD:** sase-1gu

# Plan: Finish landing epic sase-1gu (E1 instruction scoreboard and stopgaps)

## Context

Epic `sase-1gu` (plan `plan:202610/e1_instruction_scoreboard_and_stopgaps.md`) has all
six phases closed. Its stopgaps work and are deployed:

- The Grok `--rules` delivery works.
- The Claude helper template and PreToolUse guard work.
- Both sunset flags are on, and the doctor check `providers.claude_helper_channel` is
  OK.
- The decision record, the root-only sentence, and the docs are all in place.

The land agent's evidence is recorded in the last two notes on `sase-1gu`. Read them
with `sase bead read sase-1gu -r "<why>"`.

The land agent also found that the **`sase instructions verify` scoreboard and its
doctor checks report wrong numbers**. The scoreboard came from phase `sase-1gu.2` and
lives in `src/sase/instructions/` and `src/sase/doctor/checks_instructions.py`. These
bugs were caused by this epic, so they are remaining epic work. The attached baseline
JSON is also wrong (Codex `1×`, and a bogus Explore "1 accepted"). It fails the exit
criterion "baseline shows Claude 2×, Codex 2×, Muse home ✗, Grok 0", so it must be
re-captured after the fix.

This tale fixes the bugs, re-captures the evidence, and then performs the epic closeout.
**There is no other land agent: this tale's coder closes the epic.**

All of this work is Python-only (epic decision 9). Make no `sase-core` change and no
`sase-core-revision.txt` move. The artifact-index `candidate_filter` wire already
supports a `provider` field.

## Defects to fix (all reproduced 2026-10-06 at master `17c7f66bb3`)

### 1. `-p/--provider` and `-a/--agent` select from the wrong run window

- **Where:** `src/sase/instructions/_runs.py::enumerate_runs`.
- **What happens now:** it calls
  `listing_snapshot(project=…, requested_limit=max(n * len(wanted), 20))` with no
  provider filter, and only then filters by provider, agent, and the `--since`/`--until`
  window. So `-p grok` takes the newest 20 runs of all providers and usually finds 0
  Grok runs.
- **Repro:** `sase instructions verify -p grok --since 24h -j` prints `"providers": []`,
  while the unfiltered command shows a Grok row with 3 runs. `-p codex` shows 1 run
  instead of 7. `-a <older agent>` finds nothing once the agent falls outside the newest
  roughly 100 runs. The doctor checks under-sample rare providers the same way.
- **Fix:**
  - Add an optional keyword `candidate_filter: dict[str, object] | None = None` to
    `sase.agent.listing_snapshot.listing_snapshot`. When `project` is also set, AND it
    with the existing project filter as
    `{"kind": "all", "filters": [project_filter, candidate_filter]}`. Leave the
    source-scan fallback path unchanged; the caller still post-filters.
  - In `enumerate_runs`, run one bounded `listing_snapshot` query per wanted provider,
    with `candidate_filter={"kind": "equals", "field": "provider", "value": provider}`.
    Use `requested_limit=limit_per_provider`, raised to the 200 cap when `agent`,
    `since`, or `until` is set.
  - Keep the existing post-filters and the per-provider cap. Never use
    `include_full_history` or any O(history) walk.
  - The land agent confirmed that
    `{"kind": "equals", "field": "provider", "value": "grok"}` against the live index
    returns only Grok rows (it does the same for claude, codex, and muse).

### 2. Claude helper signals: false positives, a mis-sourced template marker, and a dead root alarm

All of these live in `src/sase/instructions/claude.py::helper_signals` and
`observe_claude_session`, plus `src/sase/instructions/verify.py`.

- **"accepted" and "denied" count marker text anywhere in any tool result.**
  - **Real false positive:** run `0x2--plan`, Explore helper
    `aee7524d-…/agent-a33476a734f33eecb`, ran
    `sed -n 1,100p src/sase/main/final_handler.py`. The source text contained
    `Accepted final declaration`, so the helper was scored as "1 accepted". That false
    positive appears in the attached baseline ("Explore: 0 attempts, 1 accepted"), and
    it makes `sase doctor -D -C instructions` WARN "1 accepted helper declaration(s)".
  - **Fix:** count `accepted` and `denied` only from `tool_result` blocks whose
    `tool_use_id` matches an **attempt** `tool_use`. An attempt is a Bash command
    matching `fp.FINAL_ATTEMPT_RE`, or a Skill call with `skill == "sase_final"`.
  - Additionally, count a guard denial of any Bash or Skill `tool_use` (as
    `guard_denials`) only when its result has `is_error: true` and contains
    `fp.GUARD_DENY_REASON_PREFIX`.
  - The real denial shape (from the sase-1gu.4 probe transcripts) is a `tool_result`
    with `"is_error": true` and content
    `"PreToolUse:Bash hook error: SASE helper guard: blocked 'sase final submit' - …"`.
    The record's `toolUseResult` is
    `"Error: PreToolUse:Bash hook error: SASE helper guard: …"`.
- **`has_template` never looks where the template actually lands.**
  - In real helper transcripts, the marker `# SASE Helper Instructions` is a string
    element of an `attachment` record with `attachment.type == "prompt_snapshot"`, in
    its `systemPrompt` list. One real helper had 5 parts, and part 4 starts with the
    marker.
  - Today the parser checks tool results, tool inputs, and assistant text instead. So it
    is false for normal helpers and true only when a prompt or reply quotes the marker.
  - **Fix:** set `has_template` from `system_prompt_text(records)` only.
- **`root_guard_denial` is never set.**
  - `SessionObservation.root_guard_denial` (in `models.py`) always defaults to `False`,
    so the "root guard denials" alarm column and the doctor check can never fire.
  - **Fix:** in `verify.py::_observe_claude`, set it for root (non-helper) sessions when
    the root transcript has an `is_error` tool result containing the guard prefix for a
    Bash or Skill tool use. Use the tool_use_id pairing from the fix above.
  - Root transcripts are read with the 512 KiB `ROOT_TRANSCRIPT_BYTE_CAP`, but a denial
    can come much later. So do a second, cheap streaming pass over the root transcript,
    bounded by `HELPER_TRANSCRIPT_BYTE_CAP`. That pass JSON-decodes only lines that
    contain the guard prefix or are needed to pair tool_use ids. Mark the observation
    `partial` if the pass hits its cap.
  - A root's Agent-tool hand-back that merely quotes the guard text is not a denial: it
    is not a Bash or Skill result.
- **The doctor sums root sessions into "accepted helper declarations".**
  - `src/sase/doctor/checks_instructions.py::check_instructions_helpers` sums
    `final_accepted` over all observations. A root's own accepted `sase final submit`
    would then WARN as a helper declaration.
  - **Fix:** sum only observations with `helper_type` set. Keep the root guard denial
    sum.

### 3. The Codex parser treats the combined home and project block as one source

- **Where:** `src/sase/instructions/codex.py`.
- **Real Codex 0.160.1 shape:** one user message whose text starts
  `# AGENTS.md instructions for <cwd>`, then `<INSTRUCTIONS>`, then the home `AGENTS.md`
  text, then `\n\n--- project-doc ---\n\n`, then the project `AGENTS.md` text, then
  `</INSTRUCTIONS>`.
- **What happens now:** the parser counts this as one source, so the scoreboard shows
  contract `1×` and project `✗` (the first H1 is the home H1). That is wrong; Codex
  really loads both.
- **Fix:** strip the header line and the `<INSTRUCTIONS>` wrapper, then split the body
  on the `--- project-doc ---` separator line. Treat each part as one loaded source for
  the contract count, `native_full_count`, home H1 matching, and project H1 matching.
  Keep supporting the older one-block-per-file shape.
- **Fixture:** rewrite `tests/instructions/fixtures/codex/rollout.jsonl` to the real
  combined shape, using synthetic text only. Keep an old-shape case in a test as well.
  After the fix, the live 24h Codex row should read contract `2×`, home `✓`, project
  `✓`. The sase-1gu.2 note "codex 1×" was this bug.

### 4. `--since`/`--until` parsing

- **Where:** `src/sase/instructions/_runs.py::parse_when`.
- **What happens now:** `sase instructions verify -s 24` crashes with a traceback
  (`ValueError: Invalid isoformat string: '24'`). `-s 30m` silently means 30 _months_.
- **Fix:**
  - Accept the units `m` (minutes), `h`, `d`, and `w`, plus ISO timestamps.
  - Turn anything else into a clean CLI error (message on stderr, exit 2) in
    `src/sase/main/instructions_handler.py`, not a traceback.
  - Update the help text and
    `tests/instructions/test_verify_cli.py::test_parse_when_accepts_durations_and_iso`.
  - Update `docs/cli.md` or `docs/configuration.md` only if they describe the units.

### 5. Stale test-module name from the cli-group phase

- `tests/main/test_memory_agent_docs_list.py` still has the docstring "Tests for
  `sase memory agent-docs list` …", but that command was deleted in sase-1gu.1. Its
  tests cover the inventory builder and renderer, which still exist.
- `git mv` it to `tests/main/test_instructions_list_inventory.py` and fix the docstring
  to name `sase instructions list`.

## Tests

- Real-shape fixtures, all synthetic text:
  - Update `tests/instructions/fixtures/claude/helper_after.jsonl`. Put the template
    marker in a `prompt_snapshot` attachment, and make the denial an `is_error`
    `PreToolUse:Bash hook error: SASE helper guard: …` result paired by `tool_use_id`.
  - Update `helper_gp.jsonl` so the accepted result stays paired to its attempt.
  - Add a helper fixture that reads source text containing `Accepted final declaration`
    and `SASE helper guard:` in a non-attempt tool result. It must score
    `accepted == 0`, `denied == 0`, and `has_template is False`.
  - Add a root fixture with an `is_error` guard denial on a Bash call, which must give
    `root_guard_denial is True`.
  - Add a root fixture whose Agent hand-back only quotes the guard text, which must give
    `False`.
- Codex: test the combined-block fixture (`2×`, home and project `✓`) and the legacy
  two-block case.
- `enumerate_runs`:
  - Monkeypatch `listing_snapshot` and assert it is called once per wanted provider with
    the provider `candidate_filter`.
  - Assert that `-p grok` returns Grok runs even when the newest overall runs are all
    other providers.
  - Assert the requested limit rules.
- `listing_snapshot`: a unit test that `candidate_filter` is ANDed with the project
  filter in the index query.
- Doctor: `check_instructions_helpers` ignores a root observation with
  `final_accepted=1` and WARNs on a helper accepted declaration or a root guard denial.
- `parse_when`: `30m` is 30 minutes, and a bare `24` gives a clean error with exit 2.
- Keep `tests/instructions/test_verify_baseline.py` reproducing the plan's baseline
  table and after-state rows, adjusted to the corrected fixtures.

## Re-capture the evidence (after the code and tests are green)

The installed `sase` runs from the host's primary checkout, which will not have these
fixes until this work lands. So run the scoreboard from this workspace's own build
(`.venv/bin/sase`).

1. Baseline:
   `.venv/bin/sase instructions verify -j -n 50 --until 2026-10-05T19:52:18Z > instructions_baseline_v2.json`
   (the epic's creation time). Also save the table form.
   - Expected: Claude `2×`, Codex `2×` (home and project `✓`), Muse `1×` with home `✗`,
     Grok `0`.
   - Explain any row that differs in the acceptance record.
2. After:
   - `.venv/bin/sase instructions verify -j --since 2026-10-05T20:34:00Z > e1_after_scoreboard_v2.json`
     (the Grok fix was deployed 2026-10-05 16:34 EDT). Expected Grok row: contract `1×`
     (spread allowed if pre-fix sessions fall in-window), project `✓`, directive `✓`,
     native-full `0`.
   - `.venv/bin/sase instructions verify -p grok --since 2026-10-05T20:34:00Z -j`. It
     must now return a Grok row.
   - `.venv/bin/sase instructions verify -a sase-1gu.land -H -j`. The land agent's run
     spawned one general-purpose and one Explore helper (2026-10-06 ~14:44Z). Expected:
     both helpers have `has_helper_template: true`, `final_attempts: 0`, and
     `final_accepted: 0`, and the root has `root_guard_denial: false`.
   - `.venv/bin/sase doctor -D -C instructions`. `instructions.helpers` must no longer
     WARN on the Explore false positive.
3. Write `e1_acceptance_record_v2.md`:
   - source revisions;
   - `claude --version`, `grok --version`, and `codex --version`;
   - the commands above;
   - the baseline and after JSON references;
   - expected versus observed results for every exit criterion in the epic plan.
4. For the Grok probe and Claude probe criteria, cite the evidence in the land notes on
   `sase-1gu` instead of the planned LaunchApproval probes. Those probes could not
   dispatch because of the pre-existing circular import, now task `sase-1h2`. The
   evidence is:
   - a real post-fix Grok root session (one `<human_rules>`, directive 1×, project H1
     1×, contract 1×, `agents_md_files: []`);
   - the land agent's own general-purpose and Explore helpers, which carry the template
     marker and refused `sase final submit` (0 attempts, 0 accepted);
   - the live `--settings` hook command, which denies helper `sase final submit`,
     `prepare`, and Skill `sase_final` and allows `sase final status` and root calls;
   - the sase-1gu.4 deny-under-bypass raw probe.

   Also state the declared gaps: forks, and the helpers of other providers.

5. Attach the three files with
   `sase bead attach sase-1gu <file> -N <name> -n "<summary>"`. Do not commit them to
   the repo.

## Verification

- Run the focused suites (`tests/instructions`, `tests/main/test_instructions_group.py`,
  the renamed inventory test, `tests/main/test_doctor_command.py`, and the
  listing-snapshot tests), then `sase tool run check`.
- Do not run `just check-full`. Treat any UNKNOWN triage item as yours, and explain any
  KNOWN or FLAKY item in the close note.
- Run `just symvision` if you add or remove public symbols. Prefer private helpers.

## Closeout (final step; do it in this same turn, after `sase tool run check` passes)

Do not wait for this work's own commit, push, or CI. The host commits after your turn
ends.

1. Run `sase bead epic-symbols sase-1gu`.
   - For every listed `--epic-symbol` entry, resolve it per the Symvision epic-whitelist
     policy: wire it up, privatize it, add a non-test pragma, or delete it.
   - Re-key the Justfile line only if a still-open later bead needs the exemption.
   - The land agent saw "No --epic-symbol entries for sase-1gu."; recheck anyway.
2. Close the epic:
   `sase bead close sase-1gu --note "<verification: the land notes' acceptance evidence, the scoreboard fixes made, the re-captured baseline/after JSON and acceptance record attached, sase tool run check result, follow-up triage outcomes (sase-1h2 filed; the macro-terminology, _lint-flags, and stale-core proposals were already resolved; codex 1× folded into this fix)>"`.
   - Never use `--force` merely to make the close succeed.
   - If the close is rejected for leftover epic symbols, clean them up and close again.
3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   `sase bead read sase-1gu -r "<why>"` shows
   (`plan:202610/e1_instruction_scoreboard_and_stopgaps.md` in the plans sidecar);
   change only `status: wip` to `status: done`.
5. `sase-1gu` has no `parent_bead`, so there is no parent to close.

## Out of scope

- Fixing the `sase.history` circular import (task `sase-1h2`).
- Any home-layer, chezmoi, or memory-file edit.
- Any provider argv change.
- E2–E7 work.
