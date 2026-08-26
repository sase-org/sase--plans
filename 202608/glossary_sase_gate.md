---
tier: tale
title: Add the "Sase Gate" glossary strand
goal:
  The glossary memory web defines the term "sase gate", and every generated memory
  surface lists it.
size: small
proposed_by: bbugyi200.athena.0ee
---

- **AGENTS:**
  - [bbugyi200.athena.0ee](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0ee.md)
- **COMMITS:**
  - [4a987d8](https://github.com/sase-org/sase/commit/4a987d8e8b2afeda8df636d04ceeee50b289a670)
    — docs(memory): add sase-gate glossary strand

# Plan: Add the "Sase Gate" glossary strand

## Context

`gate` is one of the most overloaded words in this repo, and the glossary web has no
strand for it. Today it can mean any of:

1. A **command-backed interaction gate** — the durable, user-answered decision bundle
   under `~/.sase/interaction_requests/<kind>/<request-id>/` created by
   `sase gate create`, `sase plan propose`, `sase questions`, `sase launch request`, and
   AXE triage chops. Source: `src/sase/notification_gates/`, `src/sase/plan_gate.py`,
   `src/sase/main/parser_gate.py`; docs: `docs/notifications.md` §"Command-backed
   interaction gates".
2. The pytest **suite gate** — a host-global worker-token lease for parallel test runs
   (`tests/_suite_gate*.py`, `tools/run_pytest`).
3. Informal uses: `just check`'s "lint gates", the coverage gate, `NavigationGate`.

Sense 1 is the SASE term the user asked to define; senses 2 and 3 are why the strand
must say what it is _not_.

The user explicitly requested this glossary addition, so the memory-file approval
required by the project's core memory instructions is already on record for this
specific change. Do not treat that as approval for any other memory edit.

## Research findings the definition encodes

These were verified against the source and docs; do not re-derive them.

- **Bundle, not a notification.** Each gate is written once to
  `~/.sase/interaction_requests/<kind>/<request-id>/` with a canonical `request.json`, a
  write-once `response.json` (or `cancellation.json`), reviewed previews and
  attachments, adapter-owned `command` resources, and hashes over the request and each
  owned resource. The notification row is a _typed transport projection_ of that bundle
  (`docs/notifications.md:10`), which is why ACE, Telegram, the mobile bridge, and the
  `sase gate` CLI all resolve and answer the same bundle.
- **Option query.** `query` is a boolean expression such as
  `(approve AND commit) OR reject`. Each `OR` branch is a mutually exclusive resolution
  path; an `AND` branch is a set of independently selectable options with one submit
  button. A submission's `selected_option_ids` must be a non-empty subset of exactly one
  branch, and the shared executor runs those options' commands in query order after
  re-verifying each command's hash.
- **Commands.** `argv` arrays executed without a shell; input arrives as JSON on stdin
  (never as arguments), and stdout must be exactly one JSON value satisfying the
  option's `result_schema`.
- **Actions vs. options.** `operations` declares repeatable actions (`edit_file`,
  `run_command`) that a reviewer may run any number of times and that never answer the
  gate; they refuse to run once the gate is terminal.
- **Terminal states.** A gate settles only as `answered`, `cancelled`, or `timeout` —
  the three statuses `sase gate wait` reports (exit codes 0, 3, 4). The 24-hour
  pending-action stale threshold is transport-only and never terminates a waiting
  producer.
- **Kinds and front doors.** `plan`, `epic_plan`, `task_triage`, `flag_triage`,
  `bead_stale_cleanup`, `plugins_required`, `question`, `launch`, `custom`, and legacy
  `hitl`. Typed kinds have their own producers (`sase plan propose`, `sase questions`,
  `sase launch request`, AXE chops) that call the in-process gate service directly;
  `sase gate create` is the front door for `custom`.
- **CLI surface.** `sase gate create | show | answer | act | wait`.

## Deliverable

Create `sase/memory/glossary/sase-gate.md` with exactly this content:

```markdown
---
keyword: Sase Gate
aliases:
  - gate
  - gates
---

A sase gate is a durable, command-backed request for one user decision, created when
work must pause for the user. Each gate is a bundle under
`~/.sase/interaction_requests/<kind>/<request-id>/` holding the hashed request, the
reviewed content, and the argv commands its options run; a notification row is only its
transport projection, so ACE, Telegram, mobile, and `sase gate` all answer the same
bundle. An option query such as `(approve AND commit) OR reject` names the mutually
exclusive branches; answering selects a non-empty subset of exactly one branch, runs
those options' commands, and writes `response.json` once, while a gate's declared
actions are repeatable and never answer it. Typed kinds (plan, epic plan, question,
launch, task triage) have their own front doors; `sase gate create` builds a `custom`
one. A gate settles only as answered, cancelled, or timed out — the statuses
`sase gate wait` reports. Unrelated to the pytest suite gate and `just check`'s lint
gates.
```

### Why this shape

- **Keyword `Sase Gate`, not `Gate`.** It follows the established convention for terms
  whose bare word is generic (`Sase Agent`, `Sase Monitor`, `Sase Repo`,
  `Sase Workspace`), and it is the term the user asked for.
- **Aliases limited to `gate` and `gates`.** The roster line renders into core memory on
  every turn, and aliases also drive the `closure: mentions` phrase matcher, so each one
  is a permanent cost. `interaction gate` and `notification gate` were considered and
  rejected: no existing strand body uses either phrase, so they would buy nothing today.
- **Length.** Roughly on par with the `sase-monitor` and `proc` strands. Strand bodies
  are never inlined into core memory, but they are still paid for on every batched
  `sase memory read glossary:...`, so the body states the mental model (bundle vs.
  projection), the answering contract, the terminal states, and the disambiguation, and
  nothing else. It deliberately omits schema details (input types, presentation fields,
  feedback modes, panels) that already live in `docs/notifications.md` and the
  `/sase_gate` skill.
- **The closing disambiguation sentence is load-bearing**, not filler: it is the only
  thing that stops an agent from reading "gate" in `sase/memory/build_and_run.md` and
  resolving it to this term.

## Steps

1. Write `sase/memory/glossary/sase-gate.md` exactly as above.
2. Run `just fmt`. Memory Markdown is prettier-formatted at `printWidth: 88` with
   `proseWrap: always`; let the formatter own the wrapping rather than hand-wrapping.
3. Run `sase memory init`. This regenerates the roster block between the
   `<!-- sase:strands -->` markers in `sase/memory/glossary.md`, the `AGENTS.md` /
   `CLAUDE.md` / `GEMINI.md` / `QWEN.md` / `OPENCODE.md` provider shims, and
   `sase/memory/README.md`. Do not hand-edit any of those generated surfaces.
4. Confirm the term resolves and reads correctly:
   ```bash
   sase memory web show glossary gate -b
   sase memory read glossary:gate -r "Verifying the new sase gate strand"
   ```
   `sase memory web show glossary gate -b` must find the strand by alias, and the read
   must print the body without a lookup error.
5. Run `just check` and fix anything it reports.

## Acceptance criteria

- `sase/memory/glossary/sase-gate.md` exists with the content above.
- `sase memory init --check` reports no drift after step 3.
- `Sase Gate (gate, gates)` appears in the `GLOSSARY TERMS` roster in
  `sase/memory/glossary.md` and in every regenerated provider instruction file.
- `sase memory read glossary:gate` resolves without a `GlossaryLookupError`.
- `just check` passes.

## Out of scope

- Any strand for the pytest suite gate, lint gates, or the coverage gate.
- Any change to `docs/notifications.md`, the `/sase_gate` skill, or
  `src/sase/notification_gates/`.
- Any other memory file edit. The user's approval covers this strand only.
