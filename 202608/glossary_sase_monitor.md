---
tier: tale
title: Add a Sase Monitor glossary entry
goal:
  '"monitor" resolves to one canonical SASE meaning in the glossary, generated into the
  memory note, the Tier-1 term list, and the ACE/LSP glossary catalog.'
size: small
proposed_by: bbugyi200.athena.047
create_time: 2026-08-16 15:55:20
status: wip
---

# Plan: Add a `Sase Monitor` glossary entry

## Goal

Define the SASE monitor concept as a first-class glossary term so that "monitor" in a
prompt, bead, plan, or code comment resolves to one canonical meaning, and so ACE's
glossary highlighting and the xprompt LSP surface it like every other SASE term.

## Critical Context

**The glossary memory note is generated. Do not edit it.**

- Source of truth: `sase/sase.yml` -> `memory.glossary` (term -> `definition`, optional
  `aliases`).
- `sase memory init` renders that mapping into `sase/memory/glossary.md`, into the
  Tier-1 term list embedded in `AGENTS.md` and every provider shim (`CLAUDE.md`,
  `GEMINI.md`, `OPENCODE.md`, `QWEN.md`), and into the memory README.
- Generator: `src/sase/main/init_memory/glossary.py`. Entry order in the rendered note
  follows config insertion order, and the existing block is alphabetical, so the new
  term must be inserted in its alphabetical slot.
- Plural forms are derived by the Rust core matcher, so `aliases` must list only the
  singular; `monitors` matches for free.

**Authorization.** The user asked for this glossary entry in the current conversation.
Per `CLAUDE.md`, that request carries approval both for the `sase/memory` change and for
the mandatory `sase memory init` regeneration. Do not stop to re-ask for either.

## Naming Decision

Term: **`Sase Monitor`**, with a single alias `monitor`.

Rationale: the `Sase ` prefix is the established convention for SASE terms whose bare
word has a strong competing meaning elsewhere in software (`Sase Agent`, `Sase Node`,
`Sase Project`, `Sase Repo`, `Sase Shell`, `Sase Workspace`). "Monitor" qualifies - this
repo also has an ACE stall watchdog and a workflows-runner monitor that are internal
machinery, not sase monitors. The `monitor` alias keeps the common spelling matchable.

Alphabetical placement: between `Sase Agent` and `Sase Node`.

## Token Budget Rationale

Adding a term costs one line in the always-loaded Tier-1 list of every agent instruction
file (`- Sase Monitor (aka monitor)`); the definition itself lives in Tier-2 and loads
only on `sase memory read glossary.md`. That is worth it here: `monitor` appears
constantly in this project's beads and prompts, and its SASE meaning (a command that
outlives the agent that started it) is not guessable from the English word.

The definition must therefore stay tight. Two rules for the author:

1. **Define the noun; do not document the workflow.** The `/sase_monitor` skill already
   covers invocation, flags, and hazards. Restating any of that here is pure cost.
2. **Reuse already-defined terms instead of restating them.** The existing `Proc Shell`
   entry already says a family-attached proc shell is a monitor and may carry timeout,
   workspace-claim, and follow-up policy. The new entry should lean on that rather than
   re-explain timeouts or claims.

## The Edit

Insert into `sase/sase.yml` under `memory.glossary`, between `Sase Agent:` and
`Sase Node:`:

```yaml
Sase Monitor:
  aliases:
    - monitor
  definition: >-
    A sase monitor is a family-attached proc shell that runs one long command under a
    detached supervisor, so the command outlives the agent that started it. Starting one
    from inside an agent hands off and kills that agent's turn, and an agent has at most
    one active monitor at a time; monitor members are named `<family>--mon`, then
    `--mon-0`, `--mon-1`. A monitor settles as `completed`, `failed`, `timeout`,
    `stopped`, or `lost`, and only the first three launch the follow-up agent recorded
    by `--next`. Inspect monitors with `sase monitor`.
```

Every claim above was read out of the implementation and should be treated as verified:
detached supervisor owning the process group and the agent handoff
(`src/sase/monitor/ start.py`, `handoff.py`), suffix allocation
(`src/sase/monitor/naming.py`), the state set (`src/sase/monitor/models.py`), and the
rule that stopped and lost monitors do not launch their recorded follow-up
(`src/sase/monitor/followup.py`, and the Follow-Up Context section of
`src/sase/xprompts/skills/sase_monitor.md`).

The wording is a strong default, not a mandate - tighten it if you can do so without
dropping a fact. Preserve the existing two-space YAML indentation style and the `>-`
folded scalar form used by neighboring entries.

## One Thing To Confirm Before Writing

The opening clause leans on the existing `Proc Shell` entry ("A family-attached proc
shell is a monitor..."), which the new entry mirrors in reverse. This is internally
consistent with `Sase Shell` (a sase agent is a sequence of agent shells and proc
shells), which is what makes a monitor a family member at all.

Note that `src/sase/monitor/proc_adapter.py` projects monitors onto the shared proc
service while `proc_shell_owns()` explicitly excludes rows whose lifecycle is
`proc-shell` from the monitor facade - i.e. the code appears to be mid-migration toward
that model. Skim `proc_adapter.py` and `src/sase/procs/models.py` and confirm the
"family-attached proc shell" framing is not actively contradicted. If it is, keep the
rest of the definition and open with a standalone framing instead ("A sase monitor is a
supervised long-running command attached to a sase agent as a family member...") and say
so in the final reply so the user can decide whether `Proc Shell` also needs a fix.

## Steps

1. Edit `sase/sase.yml` as above. Do not touch `sase/memory/glossary.md` by hand.
2. Run `sase memory init`. This is required, not optional, and regenerates the memory
   note, the agent instruction files, and the memory README.
3. Confirm the rendered result with
   `sase memory read glossary.md --reason "Verify the new Sase Monitor entry rendered"`:
   the entry appears in its alphabetical slot with an `ALIASES: monitor` line.
4. Confirm the Tier-1 list picked it up: `AGENTS.md` and each provider shim should now
   list `- Sase Monitor (aka monitor)`.
5. Run `just install`, then `just check`.

## Verification

- `sase memory init` exits clean. It runs `validate_glossary_entries`, so an alias
  collision with an existing term or alias surfaces here as an error rather than
  silently.
- `just check` passes. The two gates most likely to catch a problem are `fmt (markdown)`
  (prettier must be happy with the generated 88-column wrap of the new definition) and
  `SASE validation`.
- `git status` / `git diff --stat` show changes confined to `sase/sase.yml`, the
  generated memory files, and the generated instruction files. Any other path in the
  diff means something was hand-edited that should not have been.

## Non-Goals

- **Do not trim the `monitor` clause out of the `Proc Shell` entry.** It is the bridge
  in the other direction and costs about one line. This was considered and rejected.
- Do not add glossary entries for adjacent terms (follow-up agent, supervisor, monitor
  state) in this change.
- Do not edit `sase/memory/build_and_run.md` or the `/sase_monitor` skill.
- No CHANGELOG entry: this is a memory/config change with no user-facing code behavior.

## Out-Of-Scope Observation

While reading the CLI, `sase monitor start --help` documents the command as a positional
`-- COMMAND`, while `sase/memory/build_and_run.md` and the `/sase_monitor` skill both
use the `--command` flag (still accepted, `src/sase/main/parser_monitor.py:221`). That
is a docs/CLI drift worth a separate task bead via `/sase_new_task`; it is explicitly
not part of this change.
