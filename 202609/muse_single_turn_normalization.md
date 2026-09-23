---
tier: epic
title: Make Muse agents single-turn with a synchronous shell, up-front monitor routing,
  and a stranded-wait guard
goal: 'Muse agents stop dying mid-task because they waited on Muse''s own post-turn
  background wake. Muse runs every command synchronously inside its turn, and anything
  that can outlast Muse''s 10-minute synchronous ceiling goes to a SASE monitor, chosen
  before the command starts. A reply that still ends by claiming to wait is caught
  and continued, or fails loudly. The skills and memory stop telling agents to declare
  and then wait, or to switch to a monitor mid-flight.

  '
phases:
- id: muse-shell
  title: Muse runs synchronously behind a sunset flag, with a single-turn directive
  depends_on: []
  size: medium
  description: 'muse-shell: add the muse_synchronous_shell sunset flag; when on, launch
    muse exec with --enable-shell-tool (never duplicated from extra-args env); prefix
    every Muse prompt with a mode-aware single-turn directive stating the 10-minute
    ceiling and up-front routing rules; test both flag states; document in docs/llms.md.'
- id: muse-wait-guard
  title: Muse stranded-wait guard
  depends_on:
  - muse-shell
  size: small
  description: 'muse-wait-guard: move Claude''s wait-signal regex into a shared module;
    after a clean Muse exit whose reply ends by claiming to wait, re-invoke with reconstructed
    context and a nudge up to a bounded budget, then raise LLMInvocationError; log
    each firing; tests and docs.'
- id: muse-shell-tool-calls
  title: Tool-call capture for Muse's legacy shell tool
  depends_on: []
  size: small
  description: 'muse-shell-tool-calls: capture a real muse exec --enable-shell-tool
    fixture, display `shell` calls as Bash with the best command target the stream
    allows, and record a timed-out tool result as a failure instead of a success;
    tests and docs.'
- id: wait-guidance
  title: Skill, memory, and decision text for the up-front routing rule
  depends_on: []
  size: medium
  description: 'wait-guidance: rewrite sase_monitor and sase_final skill sources,
    the core-memory SASE Final Declaration template (and its test), and lint_and_test.md
    so no agent is told to declare-then-wait or to cancel an in-flight command for
    a monitor; fold in sase-16q; add a companion decision record on adapter harness
    normalization; regenerate memory; record follow-ups.'
proposed_by: bbugyi200.athena.0qc--1
create_time: 2026-09-23 17:47:05
status: wip
bead_id: sase-177
---

- **PROMPT:** [prompts/202609/muse_single_turn_normalization.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/muse_single_turn_normalization.md)
- **BEAD:** [sase-177](https://github.com/sase-org/sase--beads/blob/main/pages/sase-177/README.md)

# Plan: Make Muse agents single-turn

## Problem

Muse agents often end without closing their bead. The cause is Muse Code's own
background mechanism, which SASE cannot see or control.

- **What Muse does.** Muse Code 1.3.0's managed `bash` tool moves any command longer
  than about 10 s into the background, and never waits longer than 300 s. After the
  model ends its turn, `muse exec` stays alive and wakes the model when the command
  finishes. It also sends an overdue reminder every 30 minutes. That wake is a
  keep-alive, not a suspend and resume:
  - it never appears in the `--json` stream SASE reads;
  - it dies with the provider process;
  - it arrived silently with a CLI update on Sep 17.
- **How SASE breaks it.** Measured on athena, Sep 20–23:
  - **A. Watchdog kill (17 kills).** `/sase_final` and core memory tell agents to
    declare before an "I will wait" reply. The teardown watchdog
    (`src/sase/llm_provider/_subprocess_plain.py:start_completion_watchdog`) kills the
    provider 120 s after any declaration. Every kill was a Muse run, and 16 of the 17
    were still waiting on the command. Beads stayed `in_progress`, and work landed
    unverified.
  - **B. Cancel and restart (46+ calls).** A 30-minute reminder wakes the model. It
    reads `/sase_monitor`, which says to hand off "whenever it is taking a long time",
    or `lint_and_test.md`. It then cancels the in-flight check and starts a monitor that
    reruns it from zero.
  - **C. Stranded wait (4+ runs).** The command finishes during the turn. The model ends
    the turn anyway, claiming to wait, and Muse exits cleanly. Nothing wakes it, and
    `muse.py` has no guard.
- **Why the fix is synchronous execution.** Muse cannot run a command synchronously for
  more than 10 minutes in any mode:
  - Managed `bash` backgrounds after at most 300 s.
  - The legacy `shell` tool (`muse exec --enable-shell-tool`, "Use the legacy shell tool
    instead of the managed platform shell") is truly synchronous: no background, no
    wake. But it kills the command at 600.1 s and returns only `tool timed out`
    (`correlation_facts.outcome: timeout`), discarding all output.
  - No setting raises either limit.

  So anything that can run past 10 minutes must go to a SASE monitor, chosen before the
  command starts. The final verification gate should use **prepared monitor
  completion**: `sase final prepare`, then
  `sase monitor start -p verify -f <ref> -- just check`. On green, the host commits and
  applies `bead_action` without another model turn. On red, it launches one recovery
  successor. This feature exists but has been used 0 times in 1,344 monitors.

Research (read with `sase artifact read`):

- `research:202609/muse_harness_waits_vs_sase_monitors.md` — the design; its §7 is the
  recommended solution.
- `research:202609/muse_harness_waits_vs_sase_monitors__critique.md` — adjustments
  adopted here: mode-conditional rules, scoping of `timeout` wrapping, and duration
  classes instead of prediction for the later routing guard.
- `research:202609/provider_wait_contract_early_exits/provider_wait_contract_early_exits.md`
  — the evidence for modes A–C. Its "embrace the wait, build a pending-work ledger"
  recommendation is **rejected**.

**Principle.** There is one wait contract for every provider:

- Commands run synchronously within the turn, up to the provider's stated ceiling.
- Waiting across a turn happens only through a host-owned handoff, chosen before the
  command starts.
- Each provider adapter makes its harness conform mechanically. The host never models a
  harness's private wait semantics.

The Claude adapter already works this way (`claude.py`:
`CLAUDE_CODE_DISABLE_BACKGROUND_TASKS`, the 4 h `BASH_MAX_TIMEOUT_MS`,
`--disallowedTools ScheduleWakeup`, `_SINGLE_TURN_DIRECTIVE`, and a nudge-then-fail wait
guard), and it has had 0 watchdog kills. `muse.py` has none of it.

## Phase `muse-shell`: synchronous Muse behind a sunset flag, plus a directive

Files: `src/sase/llm_provider/muse.py`, the flag registry and schema entries that
`sase flag new` prints, `tests/llm_provider/test_muse_provider_invocation.py` (and
`_muse_provider_helpers.py` as needed), `docs/llms.md`.

1. **Flag.** Read `sase_flags.md` with `/sase_memory_read`, then create the flag only
   through `sase flag new muse_synchronous_shell -k sunset …`. Paste the registry entry
   it prints, and follow its both-states test checklist. Suggested prose:
   - `--when-enabled`: SASE launches `muse exec` with `--enable-shell-tool`. Muse then
     runs every command synchronously in its legacy `shell` tool, which has a hard
     10-minute kill and no post-turn background wake. The Muse directive states that
     ceiling and the up-front monitor routing rules.
   - `--when-disabled`: SASE launches `muse exec` with Muse's managed `bash` tool, which
     backgrounds long commands and can wake the model after its turn ends. The Muse
     directive instead forbids ending the turn or declaring while a backgrounded command
     is still running.
   - `--remove-when`: Muse runs on the legacy shell tool show no regression in task
     completion or model quality for several weeks, and no Muse release has dropped or
     changed `--enable-shell-tool`.

   Kind is `sunset`, default on: the new behavior ships as the default, and
   `sase flag disable muse_synchronous_shell` is the rollback. Read it with
   `current_flags().enabled(...)`. On `FeatureFlagError`, fall back to the registry
   default (on), following the existing callers such as
   `ace/tui/modals/models_panel_provider_modal_workers.py`.

2. **Argv.** When the flag is on, add `--enable-shell-tool` to `muse exec`'s safety and
   approval flag block in `MuseProvider.invoke`.
   - Skip it if the resolved extra-args string (`SASE_LLM_{LARGE,SMALL}_ARGS` /
     `SASE_MUSE_{LARGE,SMALL}_ARGS`) already contains it. A duplicate boolean flag is a
     `muse exec` usage error (exit 2).
   - Do not change `llm_interactive_cli`; human interactive sessions keep Muse's
     defaults.
   - If a future Muse release drops the flag, every run fails with the existing
     exit-code-2 "CLI usage error" note. That fails closed and visibly; the remedy is
     the flag rollback.
3. **Directive.** Add a Muse single-turn directive, delivered as a prompt prefix the way
   `agy.py`'s `_wrap_agy_print_prompt` does:
   `<directive>\n\n--- User Prompt ---\n<prompt>`. Muse has no append-system-prompt
   flag.
   - Wrap once, at the top of `invoke`, so the interrupt path's reconstructed context
     (`f"{prompt}\n\n--- Work So Far ---…"`) and the next phase's guard also carry it.
   - Render the ceiling from one constant (`600` s → "10 minutes", and `timeout 540`).
   - Keep it compact; the detailed recipes live in the skills (phase `wait-guidance`).

   Proposed text for **flag on** (tune the wording, keep the rules):

   > SASE single-turn instructions for Muse Code: this session is exactly one turn, and
   > nothing can wake you after you end it. Your `shell` tool runs each command
   > synchronously but kills any command still running after 10 minutes and discards all
   > of its output, so blocking for up to about 9 minutes is expected and correct.
   > Decide where a command runs before you start it: (1) Final verification: prefer
   > prepared monitor completion (`/sase_final`, "Prepared Monitor Completion"). Run
   > `sase final prepare` with your finished manifest (`bead_action: close` when the
   > bead is done), then
   > `sase monitor start -p verify -f <ref> -- <verification command>` (`just check` in
   > SASE repos). Passing work lands with no further turn. (2) Commands that can take
   > longer than 10 minutes go to `/sase_monitor` with `--next` before you start them.
   > Examples: full builds and installs, full test or visual suites, `just check-full`,
   > CI, deploy, release, or rate-limit waits. The project's memory names its known-long
   > commands. Combine dependent steps into one monitored command. (3) Run everything
   > else inline. Wrap an unrecorded command of uncertain length as
   > `timeout 540 <cmd> > <log> 2>&1; echo "exit=$?"; tail -n 80 <log>` so a slow run
   > still leaves evidence. `sase tool run` output is retained; replay it with
   > `sase tool show RUN -l`. Never background or detach a command (`&`, `nohup`,
   > `setsid`). Never use cron, workflow, subagent, or snooze tools to wait. Never
   > cancel or rerun an in-flight command to move it to a monitor. Never end your turn
   > to wait.

   For **flag off**, keep the shared sentences and replace the `shell` sentence and rule
   (3):
   - Your `bash` tool may move a long command to the background.
   - Never submit your final declaration or end your turn while a command you started is
     still running. SASE stops the Muse process about two minutes after your final
     declaration, killing anything still running.
   - Read each backgrounded command's result before finishing.

4. **Tests** (both flag states):
   - `--enable-shell-tool` is present if and only if the flag is on.
   - It is never duplicated when extra args already carry it.
   - The prompt file starts with the matching directive.
   - The interrupt reconstruction keeps the directive.
5. **Docs.** In `docs/llms.md` → "Muse Code Integration", update the argv example and
   add a short "Single-turn normalization" subsection. It covers:
   - the flag and its rollback;
   - the legacy shell tool and its 10-minute kill with output discarded;
   - the directive;
   - why managed `bash` is not used: its post-turn wake is invisible to SASE.

## Phase `muse-wait-guard`: Muse stranded-wait guard

Files: `src/sase/llm_provider/_wait_signals.py` (new), `claude.py`, `muse.py`,
`tests/llm_provider/test_muse_wait_guard.py` (new), `docs/llms.md`.

1. **Shared module.** Move Claude's `_WAIT_SIGNAL_RE` into a new `_wait_signals.py` as a
   public `WAIT_SIGNAL_RE`, plus a helper such as
   `ends_with_wait_claim(text, tail_chars=700)`. Make `claude.py` import it with no
   behavior change: `tests/llm_provider/test_claude_wait_guard.py` must still pass.
   Don't import another module's private name; symvision flags that.
2. **Guard.** In `MuseProvider.invoke`, after a cycle that is not an interrupt and exits
   0, check whether the cycle's reply ends with a wait claim. There is no structural
   signal to combine it with: with the flag on nothing can be pending, so a wait claim
   at a clean exit is always stranded (mode C). With the flag off, the same check
   catches replies left behind by a watchdog teardown (mode A), which the stream reports
   as a clean exit.
   - **Within the budget** (`SASE_MUSE_MAX_WAIT_CONTINUATIONS`, default 2, parsed like
     `_claude_max_wait_continuations`), re-invoke with reconstructed context: the
     wrapped original prompt, `--- Work So Far ---` holding the accumulated reply, then
     `--- Required Continuation ---` holding the nudge. This mirrors the existing
     interrupt path and `agy.py:_build_no_progress_continuation_prompt`. Muse has no
     headless resume.
   - **Budget exhausted:** raise `LLMInvocationError` naming the reason and
     `SASE_ARTIFACTS_DIR`, as `claude.py` does. A stranded wait must never be recorded
     as success.
   - **Nudge text (suggested):** "Your previous reply ended the turn claiming to wait
     for a command, notification, or wake-up. This SASE session is single-turn: nothing
     will wake you, and nothing you started is still running. Finish now. Rerun anything
     unfinished in the foreground, or hand a genuinely long command to `/sase_monitor`
     with `--next`. Then submit your final declaration and give your final answer. If
     your work was already complete, restate your final answer without claiming to
     wait."
   - **Log each firing** as one JSONL entry (`reason`, `cycle`, `timestamp`) in
     `SASE_ARTIFACTS_DIR/wait_guard_log.jsonl`, like `_log_interrupt`. "Runs ending with
     a wait claim" is the primary success metric, and this log is how it gets counted.
3. **Tests:**
   - a wait-claim reply triggers exactly one re-invoke, with the reconstructed prompt
     and nudge, and the second reply is returned;
   - an exhausted budget raises;
   - a normal final answer does not re-invoke;
   - the env budget parses correctly: default, `0`, and garbage input;
   - interrupt handling still takes precedence;
   - the log entry is written.
4. **Docs.** Add one paragraph to the Muse section of `docs/llms.md`.

## Phase `muse-shell-tool-calls`: tool-call capture for the legacy `shell` tool

Files: `src/sase/llm_provider/_tool_call_muse.py`, a new fixture under
`tests/llm_provider/fixtures/`, the Muse stream and artifact tests, and `docs/llms.md`
("Muse Tool-Call Capture").

1. **Capture a real fixture.**
   - Work in a scratch temp directory, not a SASE workspace.
   - Run
     `muse exec --json --enable-shell-tool --disable-approval --disable-sandbox --trust-workspace --no-foreign-personal-context --model muse-spark-1.3 --reasoning-effort low --session-id <uuid>`.
     Use a prompt that runs `echo sase-shell-fixture` and one command that exits
     non-zero.
   - Save stdout as `muse_exec_shell_tool_R3401.1.jsonl`, redacting host paths the same
     way the existing `R708.1` fixtures do. Use the non-Contributor model.
   - Do not capture a real 10-minute timeout. Build a `tool.result` with
     `correlation_facts.outcome: "timeout"` and text `tool timed out` in a unit test
     instead.
2. **Parser.**
   - Map `"shell"` to display name `Bash` in `_MUSE_DISPLAY_TOOL_NAMES`.
   - Extend the `bash`-only `command`/`description` target derivation to `shell`, but
     only as far as the fixture shows the result body carries those fields. If it does
     not, keep the honest result-preview fallback and say so in the module docstring.
     Never invent arguments.
3. **Timeout status.** `_result_status` currently maps an unknown outcome such as
   `timeout` to `success`, which is wrong. Map timeout outcomes (`timeout`, `timed_out`,
   and whatever the fixture or binary strings show) to `failure`. The status vocabulary
   is fixed (`src/sase/ace/tui/llm_calls/_constants.py`), so keep the `tool timed out`
   text visible through the response summary.
4. **Tests and docs.** Test the fixture parsing, the `shell` display name, timeout →
   failure, and that `bash` behavior is unchanged. Update the capture docs.

## Phase `wait-guidance`: skills, memory, and a decision record

Items 1–2 edit skill sources, not memory. The user approved each of the three memory
edits in items 3–5 when asked during planning: the core-memory "SASE Final Declaration"
template, `lint_and_test.md`, and the new decision record. The approval covers only
these three edits. For them, use `/sase_memory_write` (edit and republish), and never
hand-edit generated shims. Any other memory change you find necessary goes in a
`PROPOSED FOLLOW-UP:` note on this phase's bead, not an edit.

1. **`src/sase/xprompts/skills/sase_monitor.md`.** Add a "Decide Before You Start"
   section right after "Core Rule":
   - Choose a command's place before starting it. Run it inline when it fits within your
     provider's synchronous limit (as your SASE provider instructions state it) and
     waits on nothing external.
   - Start it with a monitor when it can outlast that limit, or waits on CI, a deploy, a
     release, or a rate limit.
   - For final verification, prefer prepared monitor completion; point to `/sase_final`.
   - **Never cancel, kill, or rerun an in-flight command to move it to a monitor.** Let
     it finish.
   - Keep the description's "provider-native background … do not work in SASE" claim;
     after this epic it is true for every provider.
   - Do not render per-provider numbers into the skill. Skills are deployed once per
     provider, but the Muse ceiling depends on a machine-local flag. The runtime
     directive carries the number.
2. **`src/sase/xprompts/skills/sase_final.md`.**
   - Rewrite line 11 so the declaration is mandatory for final answers and
     incomplete-status responses, and never for waiting.
   - Add two rules:
     - Never end a turn to wait for a command or promise to resume later. Nothing can
       wake you; hand long commands to `/sase_monitor` before starting them.
     - Never submit a declaration while a command you started is still running.
   - Expand "Prepared Monitor Completion" into a complete worked recipe:
     1. Run `just fix` first.
     2. Build the wrapper from `sase final context -f json`, with
        `verification.command: ["just", "check"]` and `bead_action: "close"` when the
        bead is done.
     3. Run `sase final prepare <wrapper> -j`.
     4. Run `sase monitor start -p verify -f <ref> -r '…' -- just check`.
   - Then state the outcomes: green means the host commits, closes the bead, and runs no
     successor; red means one recovery successor. Check the wrapper shape against
     `docs/monitors.md` ("Prepared host completion") and
     `src/sase/finalizers/prepare.py`, where the verification command may be an argv
     list or a shell string.
3. **Core memory template.** In
   `src/sase/main/init_memory/templates/memory-sase.template.md`, rewrite "SASE Final
   Declaration" to roughly this:

   > Before any normal response that ends this SASE provider turn, use your
   > `/sase_final` skill as the last action. This includes a final answer and an
   > incomplete-status response; an unfinished turn still declares so its work is
   > committed. Never end a turn to wait for a command or to resume later: nothing can
   > wake you, so hand long commands to `/sase_monitor` before starting them. Only a
   > successfully executed plan, monitor, pipe, or questions handoff is exempt, because
   > those commands terminate the runner mechanically.

   Update `_FINAL_DECLARATION_MARKERS` in
   `tests/main/test_init_memory_handler_outputs.py` to match: drop `"I will wait"` and
   the old resume sentence, and add the new never-wait sentence. `sase/memory/sase.md`
   is generated and refuses direct edits.

4. **`sase/memory/lint_and_test.md`.**
   - Replace "`just check` may be run inline, but hand it to a monitor the same way
     whenever it is taking a long time" with the up-front rule:
     - run it inline when it fits your provider's synchronous limit;
     - for final verification, prefer a prepared-completion monitor (`/sase_final`);
     - never cancel or rerun an in-flight check to move it to a monitor.
   - Name this repo's known-long commands, which the provider-generic Muse directive
     defers to project memory. They often exceed 10 minutes on the shared host:
     `just install`, `just rust-install`, `just test-scoped`, full `just test-visual` or
     `just fix-tui-screenshots` runs, and `just check-full`. `sase tool run check`
     itself exceeded 10 minutes in about 19% of Muse runs.
   - Apply `sase-16q`'s replacement for the stale "Prepared-completion `-f` monitors
     still use the raw `just check`…" sentence (read it with `sase bead read sase-16q`).
   - Keep `[[...]]` links valid.
5. **New decision record** `sase/memory/decisions/adapters-normalize-harnesses.md`.
   Match the frontmatter shape of `single-turn-agents.md`: `status: accepted`, decided
   on the landing date.
   - **Claim:** each provider adapter makes its CLI harness conform to the single-turn
     contract. Where the CLI allows, it disables native background and wake primitives
     mechanically. It tells the model the provider's real synchronous ceiling. Otherwise
     it detects violations with a bounded guard and fails loudly. The host never models
     a harness's private wait semantics.
   - **Why:** Muse's wake is an invisible, unbounded keep-alive, not a suspend. Rejected
     alternatives: embracing the wait with a host pending-work ledger (it would parse
     Muse's private session log and need re-checking on every CLI release), and a
     prompt-only "use monitors" rule (Muse's harness, not the model, picks what goes to
     the background).
   - **Cost:** Muse loses in-turn waits past 10 minutes, which means more monitor hops,
     and it depends on a "legacy" upstream tool.
   - **Reopens when:** a harness wait meets [[decisions/single-turn-agents]]'s reopen
     condition, or a provider offers no way to disable background execution.
   - Link `[[decisions/single-turn-agents]]`. Do **not** edit or supersede that record.
6. **Regenerate.** Run `sase memory init` to regenerate `sase/memory/sase.md`,
   `AGENTS.md`, and the provider shims (`CLAUDE.md`, `GEMINI.md`, `QWEN.md`,
   `OPENCODE.md`) plus the memory README. Then run `just fmt`.
7. **Follow-ups.** Record every item in "Follow-ups" below as a `PROPOSED FOLLOW-UP:`
   note on this phase's bead. Phase workers do not create beads.

## Landing

- Close `sase-16q` with `-R done`: the `wait-guidance` phase applied its memory edit.
- Close `sase-16i` with `-R superseded` and a note. Its pending-work-ledger direction is
  replaced: with `muse_synchronous_shell` on, Muse has no post-turn wake, so "still
  alive 120 s after the declaration" means a hang again and the watchdog is correct as
  is. The stranded-wait guard covers the rollback path.
- After landing, from a clean landed tree, deploy the changed skills
  (`sase skill init --force`, then `chezmoi apply` if it was skipped). Rerun
  `sase memory init` in the home project so its generated "SASE Final Declaration" text
  picks up the approved template. That regenerates output; no note is hand-edited.
- File each item in "Follow-ups" below through `/sase_new_task`. The lander does this,
  not the phase workers. File follow-up 1 as a `large` task bead: the user chose a
  post-landing task for it rather than a phase in this epic.

## Follow-ups (not in this epic; file through `/sase_new_task` after landing)

1. **Mechanical routing in `sase tool run` (large; the user deferred it out of this epic
   during planning).** Declare a duration class per tool catalog entry (`short`, `long`,
   or `unbounded`), and have the adapters export `SASE_PROVIDER_SYNC_CEILING_SECONDS`
   (600 for Muse's shell mode).
   - `sase tool run` refuses, before starting, a `long` or `unbounded` tool whose class
     floor exceeds the caller's ceiling, and prints the exact prepared-completion or
     `--next` monitor command to use instead. This follows `decisions:guarded-recipes`.
   - Use the duration corpus for calibration, not as the gate.
   - This crosses the sase-core boundary: binding, tests, and the revision pin.
2. **Cheaper monitor hops (large).**
   - The successor prompt gets the starter's last reply and a tool-call digest.
   - A bounded, visible slot-reservation lease covers the hop.
   - Dispatch stops waiting on the starter runner's post-kill bookkeeping (median 77 s).
3. **SASE-owned "inline, then escalate" for ToolRuns (xlarge; the `sase tool` roadmap's
   E2).** Add `sase tool run --detach`, a ceiling-bounded `sase tool wait <id>`, and
   `sase monitor start --join <id>`. This removes Muse's up-front guess and makes
   cancel-and-restart impossible.
4. **Muse CLI-update smoke test (medium).** In `sase agent-cli update muse`, assert that
   `--enable-shell-tool` is still accepted and still yields a synchronous `shell` tool
   with no managed `bash`. Fail loudly otherwise.
5. **Muse's remaining async tools (`workflow`, `cron_*`, `subagent_*`,
   `snooze_reminder`).** Add a per-invocation `run.toolset` allowlist only if telemetry
   shows these tools being used to wait.

Related, unchanged: `sase-12s` (provider-neutral incomplete classification) and
`sase-163` (Claude's leftover async tools, same principle).

## Rollout, rollback, and how to tell it worked

- **Rollout.** The flag ships default-on, so there is no live canary without the
  directive: the directive and the synchronous shell land together.
- **Rollback.** Run `sase flag disable muse_synchronous_shell`. The directive switches
  to its managed-`bash` variant, and the guard stays active.
- **Measure** from the same artifact sources as the Sep 20–23 baseline:

| Metric                                                                                         | Baseline |                Target |
| ---------------------------------------------------------------------------------------------- | -------: | --------------------: |
| Muse `provider_teardown_stall.json` files                                                      |       17 |                     0 |
| Muse runs whose final reply claims a wait (`wait_guard_log.jsonl` firings plus guard failures) |      ≥12 |    0 reaching success |
| Muse in-flight cancel/terminate of a running command                                           |      46+ |                     0 |
| Muse `shell` 600 s kills (routing misses)                                                      |      n/a |               < 1/day |
| Final gates landed through prepared completion                                                 |   0 ever | most Muse final gates |
| Beads left `in_progress` or hand-closed after a lost wait                                      |       9+ |                     0 |

## Verification (every phase)

- Read `lint_and_test.md` with `/sase_memory_read` before finishing.
- Run `sase tool run check`. Muse-provider tests live in `tests/llm_provider/`, and
  memory-template tests in `tests/main/test_init_memory_handler_outputs.py`.
- Only the `muse-shell-tool-calls` phase makes a real Muse call (a tiny fixture
  capture). No phase needs a 10-minute run.
