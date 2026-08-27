---
tier: tale
title: Retire the obsolete question-asker family suffix
goal:
  Remove the live --q/root-question/phase-question naming taxonomy now that question
  gate shells own the durable decision, while historical --q agent artifacts remain
  readable and all family, status, context, and documentation behavior is covered by
  updated tests.
size: medium
proposed_by: bbugyi200.athena.sase-ud.12
bead: sase-ud.12
---

- **PARENT:** [202608/gate_shells.md](gate_shells.md)
- **BEAD:**
  [sase-ud.12](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ud/sase-ud.12.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ud.12](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ud.12.md)
- **COMMITS:**
  - [777e51e](https://github.com/sase-org/sase/commit/777e51e734a6770e232e039ecfa159a199247295)
    — feat(agents): retire q asker suffix

# Plan

Bead: **sase-ud.12** (`q-suffix-cleanup`) in the approved gate-shell epic.

The preceding question and plan migrations moved human-question ownership out of the
asking agent and into a durable gate shell. Live question continuations therefore use
the same ordinary family-member allocation as every other gate-shell follow-up; the
special `q` role and root/phase-question suffix grammar are no longer valid runtime
concepts. This phase removes that taxonomy without breaking read-side rendering of
historical artifact directories named with canonical `--q`.

## Implementation

### 1. Collapse plan-chain naming to ordinary agent-shell suffixes

- In `src/sase/plan_chain.py`, delete `PLAN_CHAIN_QUESTION_SUFFIX`, the `q` explicit
  family role, `.q`/`-q` legacy aliases, and the root-question/phase-question fields,
  parsing branches, regex/splitting helpers, predicates, and follow-up-template helper.
- Keep `--q` recognizable only as an ordinary canonical token so historical family
  artifact directories still load and render. Do not reintroduce `q` as a reserved or
  generated suffix, and do not preserve `.q` or `-q` as live aliases.
- Retain the existing plan, feedback, coder, epic, commit, monitor, gate, and arbitrary
  custom-token behavior. Add focused compatibility coverage proving historical `--q`
  remains readable while `.q` and `-q` are rejected.

### 2. Make question continuations ordinary family members

- Simplify `src/sase/axe/run_agent_exec_questions.py` in both the legacy flag branch and
  `%auto` short-circuit to allocate the next ordinary family suffix and inherit the
  interrupted agent shell's real family role. Remove all root-question and
  phase-question branching, `q` role writes, and suffix-template imports.
- Preserve the substantive behavior of both paths: the interrupted prompt/model, merged
  Q&A, saved chat, relationships, workspace handoff, and one-agent `%auto` continuation
  remain intact. Gate-shell settlement already follows this ordinary successor contract,
  so tests should assert parity rather than a special name.
- Remove `q` from `sase pipe --name`'s reserved suffixes; it becomes an ordinary custom
  token and must not imply question ownership.

### 3. Remove presentation and status dependence on asker classification

- Update family-context and prompt-panel labeling to derive ordinary agent labels from
  their suffix/actual role and to derive the question decision from the gate shell.
  Delete dedicated `PLAN_CHAIN_QUESTION_SUFFIX`/role-`q` maps and branches. A historical
  `--q` artifact may still render as the ordinary token `q`, but no live classification
  or behavior may depend on it.
- Remove `q`-role gates in family status policy. Where legacy flag-off behavior still
  needs to recognize question handoff state, use the durable question timestamps,
  response path, parent/child topology, and gate-shell rows rather than a suffix or
  family-role taxonomy. Leave the broader status-collapse and feature-flag removal to
  `sase-ud.13`.
- Update stale comments in the family-core policy so they describe ordinary agent shells
  and gate-shell-owned question state.

### 4. Rewrite affected tests, fixtures, goldens, and docs

- Replace live `--q`/`agent_family_role="q"` fixtures throughout plan-chain,
  question-flow, family-attachment, loader, context, status, prompt-panel, output-var,
  and visual tests with ordinary numeric/custom members carrying their underlying roles.
  Delete assertions for root/phase-question taxonomy and add assertions for ordinary
  continuation allocation and gate-shell-owned QUESTION/ANSWERED state.
- Retain only narrow historical-compatibility fixtures that intentionally exercise an
  existing canonical `--q` artifact directory; label those tests as historical so the
  dead suffix cannot accidentally become a live contract again.
- Update `docs/ace.md` and any other exact documentation hits so `--q` is no longer
  documented as a generated question agent or dedicated `AGENT (q)` phase.
- Run targeted tests for naming, question handoff (flag on/off and `%auto`), family
  attachment/status normalization, context/loaders, prompt-panel labels/output
  variables, and any changed visual fixtures before the repository gate.

## Verification and bead completion

1. Run `just install` if the numbered workspace environment is stale.
2. Run the targeted suites covering every changed module and fixture, then run
   `just check` as required for SASE repository changes. Use `/sase_monitor` if either
   command becomes long-running.
3. Re-run exact searches for `PLAN_CHAIN_QUESTION_SUFFIX`, root/phase-question helpers,
   live `agent_family_role="q"`, and documentation claiming `--q` is generated. Only an
   explicitly named historical `--q` compatibility case may remain.
4. Run `sase bead epic-symbols sase-ud.12`. Resolve every symbol before closing, or
   re-key its Justfile ownership annotation to the parent epic or a later open phase.
5. Record any genuinely out-of-scope discovery only as `PROPOSED FOLLOW-UP: ...` on
   `sase-ud.12`; do not create a task bead.
6. Close only `sase-ud.12` with a note naming the targeted verification, `just check`,
   the final symbol audit, and the historical-compatibility guarantee. Do not close
   `sase-ud` or another ancestor.

## Non-goals

- Do not remove the gate-shell feature flag or collapse the remaining status ladder;
  those belong to `sase-ud.13`.
- Do not remove read-side support for an already-existing canonical `--q` artifact
  directory.
- Do not preserve `.q`, `-q`, the `q` role, or question-specific numeric/phase suffix
  generation as compatibility surfaces.
- Do not edit SASE memory or generated provider instruction files.
