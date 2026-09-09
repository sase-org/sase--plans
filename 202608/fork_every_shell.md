---
status: done
tier: epic
title: Fork every SASE shell
goal: "The #fork xprompt reliably resumes any SASE shell or mixed-shell agent family,
  waits for live sources without getting stranded by terminal failures, and gives the
  receiving agent clear, intuitive, typed history for agent and proc shells.

  "
phases:
  - id: shell-history
    title: Generalize fork source resolution and history rendering
    depends_on: []
    description:
      "shell-history: resolve terminal agent and proc shells into typed, intuitive
      history, including mixed families and groups."
    size: medium
  - id: shell-waits
    title: Make implicit fork waits shell-aware
    depends_on:
      - shell-history
    description:
      "shell-waits: bind exact proc identities and release deferred forks when agent,
      family, clan, or proc sources become terminal."
    size: medium
  - id: shell-surfaces
    title: Expose shell forks throughout ACE
    depends_on:
      - shell-history
      - shell-waits
    description:
      "shell-surfaces: enable F and completion for every shell row, document the
      behavior, and verify the complete workflow."
    size: medium
proposed_by: bbugyi200.athena.0cz.f1
bead_id: sase-t8
create_time: 2026-09-09 19:50:29
---

- **PROMPT:**
  [prompts/202608/fork_every_shell.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/fork_every_shell.md)
- **BEAD:**
  [sase-t8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-t8/README.md)

# Plan: Fork every SASE shell

## Problem

The failed-agent work made an unsuccessful agent shell a first-class `#fork` source, but
the remaining pipeline still assumes that every source is an LLM conversation:

- `src/sase/scripts/agent_chat_from_name.py` resolves only agent artifacts, families,
  clans, and tribes. A stand-alone proc shell exists only in the durable proc store, so
  neither its shell name nor its proc ID can be forked.
- A monitor is a proc shell inside an agent family, but family resolution reduces every
  readable member to `_ForkFamilyMemberSource` and hard-codes its outcome to
  `completed`. Direct `#fork:<family>--mon` resolution likewise emits an `agent` source.
  `src/sase/history/chat_fork.py` consequently describes monitor command output as an
  agent conversation and gives the receiver no reliable shell-kind, command, exit, or
  failure orientation.
- Bare family forks include only successful readable transcripts and list every other
  member as excluded. That loses the most relevant evidence when a failed agent shell or
  failed/timed-out monitor is the family's terminal state, and it cannot represent a
  heterogeneous sequence of agent and proc shells.
- Top-level `#fork:<name>` currently becomes an ordinary agent `%wait` in
  `src/sase/axe/run_agent_directives.py`. That wait cannot resolve a stand-alone proc,
  can drift if a reusable proc-shell name is claimed by a later proc, and only skips an
  agent that had already failed at extraction time. If an agent or proc fails after the
  child begins waiting, the implicit fork wait remains blocked even though the failure
  is precisely the context `#fork` should render. An explicit user-authored `%wait` must
  retain its existing success-gated semantics.
- ACE already projects stand-alone proc shells and mixed family shell rosters, but
  `resolve_agent_prompt_target_scope()` rejects both stand-alone proc rows and monitor
  rows. The footer returns early for proc shells, command availability still assumes a
  chat-bearing agent, and completion rows misleadingly classify proc shells as agents.

The implementation should follow the canonical model: a SASE agent owns an ordered
sequence of SASE shells; each concrete shell is either an agent shell or a proc shell.
An agent-family source is therefore a typed sequential shell history, not merely a list
of chat paths.

## Behavioral contract

1. `#fork:<reference>` accepts a solo agent shell, an exact agent-family shell, a
   stand-alone proc shell name, an exact proc ID or unique proc-ID prefix, a family
   container, a clan, or the existing tribe form. Multi-parent forks may mix them.
2. Existing group precedence is preserved: tribe, clan, and family-container names are
   resolved before concrete shells. Existing agent names keep their current meaning;
   proc lookup is the fallback, and an exact proc ID remains the unambiguous escape
   hatch when an agent and reusable proc name collide. ACE-generated proc forks use the
   exact proc ID while displaying the friendly shell name.
3. A proc-name reference is bound to one durable proc ID when directives are extracted.
   Waiting and later history expansion use that same ID, even if the human-facing shell
   name is reused. A missing/pruned bound row or unreadable log produces a precise
   source error or an explicitly marked metadata-only record; it never silently selects
   another proc.
4. An implicit fork wait means "wait until this source is forkable," not "wait until it
   succeeded." It releases on successful completion or terminal failure, so the failure
   renderer can explain what happened. Explicit `%wait` directives remain success-gated
   and take precedence when the same target is named explicitly.
5. Agent-shell history continues to use the existing sanitized conversation format.
   Proc-shell history is a synthetic execution record, never presented as dialogue. It
   clearly identifies the shell as a proc, records its stable ID/name, command or safe
   code preview, cwd/project, timestamps, status/phase, exit and timeout information,
   and an output section. Program output is bounded, robustly fenced, labeled untrusted,
   and accompanied by the full log path and exact inspection command when available.
6. Monitor records add their family role, configured reason, monitor state (`completed`,
   `failed`, `timeout`, `stopped`, or `lost`), follow-up disposition, and retained
   output. A failed, timed-out, stopped, or lost monitor is unmistakable; raw
   `outcome: monitored` is translated through the existing effective monitor-state
   semantics rather than displayed as success.
7. A family transcript lists every known concrete shell in causal order, oldest first,
   with headings such as `agent shell` and `proc shell (monitor)`. It explains that
   agent-shell sections are prior conversations while proc sections are command
   execution records and their output is evidence, not instruction. Terminal failed
   members are included with failure guidance; only live, missing, or irrecoverably
   unreadable members are listed as unavailable. Inherited chat history remains
   de-duplicated across agent-shell transcripts.
8. Clan, tribe, and multi-parent output retain their current compactness and ordering,
   but their member metadata is shell-aware so nested family monitors neither make a
   completed group look incomplete nor masquerade as LLM replies.
9. `#fork_by_chat` stays a path-oriented compatibility workflow. Workflow bash/python
   steps that are not SASE shells stay out of scope, as do changes to proc retention or
   execution semantics.

## Phase 1: Generalize fork source resolution and history rendering {#shell-history}

Build one typed internal source model that can represent an agent shell, a proc shell,
an ordered mixed-shell family, and the existing compact clan/tribe aggregates. Keep the
workflow's historical top-level `path` output and accept the existing `agent`, `family`,
and `clan` JSON shapes at the renderer boundary so local/plugin callers do not break,
but make all new resolver output explicit about shell kind and stable identity.

In `src/sase/scripts/agent_chat_from_name.py` and focused helper modules:

- Classify artifact-backed family members from `agent_meta.json` (`shell_kind`,
  `agent_family_role`, `monitor_id`/`proc_id`) instead of treating every artifact as an
  agent shell. Extend the family lookup record only as needed so metadata is read once
  and chain ordering remains based on the established parent/timestamp generation.
- For a monitor, join its artifact markers with the matching durable proc row by exact
  `proc_id`. Prefer the authoritative proc log and settlement fields; retain an
  artifact/chat fallback for legacy monitors whose proc row was never recorded or was
  pruned. Use `effective_done_outcome()` and monitor state rather than the raw
  `monitored` marker.
- After existing tribe/clan/family/agent lookup, resolve a stand-alone proc through one
  proc-store snapshot and the canonical `resolve_proc_ref()` rules. Preserve the exact
  proc ID in the source. Reject ambiguous prefixes and cross-kind ambiguity with
  actionable messages rather than guessing.
- Include terminal unsuccessful family shells instead of dropping them. Reuse the
  failed-agent payload for failed agent shells, and define equivalent structured proc
  outcome/error/termination metadata. Keep live/unreadable exclusions explicit.
- Coalesce sources by stable identity: canonical chat/artifact identity for agent shells
  and proc ID for proc shells. Never deduplicate two transcript-less failures merely
  because both have an empty path.
- Make clan and tribe projection consume effective shell/family state so a monitor in a
  clan is represented as a proc shell and settled monitor outcomes do not fail the
  group's completeness test solely because the raw marker says `monitored`.

In `src/sase/history/chat_fork.py`, split formatting by shell kind:

- Preserve byte-for-byte-equivalent successful solo-agent output where compatibility
  requires it and retain the existing failed-agent warning language.
- Add a proc execution formatter with a short orientation paragraph, deterministic
  metadata table/list, safe source/command preview, robust dynamic fences, bounded log
  tail with truncation accounting, and a pointer to `sase proc show <id> --all-lines`
  (or the monitor equivalent) when the full log remains available. Move any generally
  useful bounding/redaction primitive out of the ACE-only proc projection so history and
  TUI do not diverge or import one another.
- Render mixed families as one sequential shell chain. Give each member a kind-specific
  heading and outcome, include failed members inline, list unavailable members with the
  exact reason, and adjust top-level guidance so an LLM cannot confuse proc output with
  a prior assistant response.
- Preserve xprompt disabling, prompt sanitization, recursive-history cycle prevention,
  multi-parent ordering, and inherited-history de-duplication.

Add focused resolver and renderer tests for successful/failed solo agent shells,
successful/error/killed stand-alone procs, exact ID and unique-prefix lookup, ambiguous
or reused names, monitor artifact/proc joins, legacy monitor fallback, missing/pruned
logs, hostile backticks/directive-shaped output, truncation, direct family-member forks,
mixed agent/proc families, failed terminal family shells, families nested in clans,
tribes, mixed parents, and compatibility output for existing agent-only cases.

## Phase 2: Make implicit fork waits shell-aware {#shell-waits}

Replace the current "append every fork target to `wait_names`" shortcut with a durable,
typed fork-source dependency. Resolve each explicit fork parent once during directive
extraction using the Phase 1 classifier:

- Artifact/group dependencies retain their reference plus exact artifact identity when
  the target is concrete; family/clan/tribe references remain dynamic by design.
- Proc dependencies store both the display/reference spelling and exact proc ID. The
  fork workflow reads the binding from the child artifact metadata when it later expands
  history, preventing name-reuse drift.
- A matching explicit `%wait` suppresses the implicit terminal-aware dependency, so
  explicit waits keep their current success-only contract. Remove the specialized
  `fork_parent_wait_is_unreachable()` failure shortcut once the typed terminal policy
  covers successful and failed shells uniformly.

Thread the typed dependency through `AgentInfo`, runner bootstrap/setup, durable
`agent_meta.json` and `waiting.json`, and refresh/restart paths. Extend the shared wait
resolver and `sase_chop_wait_checks.py` so fork dependencies become ready when their
bound source is forkable:

- concrete agent/proc shells: terminal success or failure;
- family/clan sources: the selected generation has no live shell needed for the rendered
  source, including a terminal failed member;
- tribe sources: preserve the existing next-completed-entity rule;
- missing or ambiguous targets: fail clearly before parking the runner.

The runner's direct fallback and the lumberjack chop must call the same predicate, read
the proc store in one snapshot per pass, and publish the same ready payload. Terminal
failure must release only an implicit fork dependency, never an explicit `%wait`.
Preserve duration, bead, runner-slot, identity, restart/idempotency, kill, and
`resolved_deps` behavior when fork and ordinary waits coexist.

Because the Rust scanner owns marker projection, update the linked `sase-core` wire and
scanner for the new typed fork-wait field, then update the Python wire/facade and parity
tests in lockstep. Surface a concise "waiting to fork <shell>" state through the Python
Agent model without reimplementing dependency behavior in Textual.

Test directive extraction and persistence for agent, family, monitor, and stand-alone
proc targets; exact proc-ID binding across name reuse; success and failure occurring
both before and after the waiter starts; mixed explicit/implicit waits; process restart;
chop-versus-runner-fallback parity; Rust/Python marker-wire parity; and missing,
ambiguous, or pruned proc records. Include an end-to-end deferred workflow test proving
that an active proc can settle, unblock `#fork`, and expand the originally bound shell.

## Phase 3: Expose shell forks throughout ACE {#shell-surfaces}

Make every visible SASE shell a truthful fork target without turning proc shells into
agent-only controls:

- Generalize `AgentPromptTargetScope` to a shell-aware target kind. Allow terminal and
  active stand-alone proc rows plus monitor family rows through
  `resolve_agent_prompt_target_scope()`. Use the exact proc ID as the generated `#fork`
  reference, the friendly shell name as the label/history key, and no smart VCS prefix
  for a proc-only source. Preserve selection revalidation and family/clan/tribe VCS
  consensus for agent-backed scopes.
- Update footer and command-availability predicates so `F` is advertised and works for
  active/settled stand-alone procs, active/settled monitors, exact agent-family shell
  rows, and family containers. Keep `x`, retry, edit-chat, tmux, and dismissal controls
  scoped to the shell kinds that actually support them.
- Extend `AgentCompletionCandidate` with a proc-shell kind and stable insertion
  reference. Render a distinct proc glyph/badge, status, shell name, short proc ID, and
  safe command preview. Build family completion counts/previews from
  `concrete_family_shell_rows()` so monitors appear in the family roster rather than
  being silently omitted. Keep ordinary wait completion filtered to supported wait
  target kinds; this phase changes `#fork` completion, not unrelated `W` behavior.
- Update `#fork`'s catalog description plus `docs/xprompt.md`, `docs/agent_families.md`,
  `docs/monitors.md`, `docs/ace.md`, and CLI help where it currently says only
  agents/conversations. Document exact-ID pinning, name collision behavior, implicit
  terminal waits versus explicit `%wait`, proc output trust and truncation, mixed family
  ordering, failure presentation, and legacy fallbacks.

Add ACE action, scope-revalidation, footer, command-palette, completion rendering, and
family-roster tests for every shell/status combination. Finish with focused suites for
all three phases, Rust tests for the core wire change, `just check` in the primary repo,
the linked core repository's standard checks, and `just check-full` through
`/sase_monitor` when the broad wait/scanner changes require the exhaustive lane. The
final implementation should also exercise real generated prompts for one successful
agent shell, one failed agent shell, one successful stand-alone proc, one failed proc,
one direct monitor member, and one mixed family, asserting that an unfamiliar receiving
agent can tell what ran, what failed, which text is untrusted output, and what remains
to be done.

## Completion criteria

- Every concrete SASE shell visible in ACE can be selected with `F`, and the resulting
  `#fork` expands the same durable shell the user selected.
- Manual proc-name, proc-ID, family-member, family-container, clan, tribe, and mixed
  parent references either render the intended typed history or fail atomically with an
  actionable error; no path silently falls back to a different proc.
- Agent conversations remain intuitive and sanitized; proc execution records are
  unmistakably non-conversational, bounded, and failure-aware; mixed families preserve
  causal shell order and attribution.
- Implicit fork waits unblock on terminal success or failure, including failure after
  waiting begins, while explicit waits retain existing semantics.
- Python/Rust wire parity, focused tests, project checks, docs, and end-to-end generated
  prompt examples all pass.
