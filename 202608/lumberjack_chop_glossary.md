---
tier: tale
title: Add Lumberjack and Chop to the SASE glossary
goal:
  The SASE glossary teaches both AXE concepts precisely without wasting agent context.
size: small
proposed_by: bbugyi200.athena.0bb
create_time: 2026-08-22 18:41:31
status: wip
---

# Add Lumberjack and Chop to the SASE glossary

## Goal

Add concise, durable definitions for `Chop` and `Lumberjack` to this project's glossary
configuration, then regenerate and verify the derived agent instructions.

## Researched semantics

- Current AXE architecture gives each configured lumberjack its own supervised process,
  fixed tick interval, state, and metrics. Eligible script chops in one tick run
  concurrently, and the orchestrator restarts a lumberjack after a crash.
- A chop is now strictly a script-only unit of AXE automation. AXE supplies JSON context
  and applies its configured cadence, triggers, guards, timeout, and deduplication.
  Structured chop results may request follow-up launches, but the runner owns those side
  effects.
- The terms began as the scheduler lane and job unit in the lumberjack/orchestrator
  rewrite. Later changes moved per-chop cadence from cycle counts to durations, hardened
  per-lumberjack supervision, and removed the former agent-backed chop execution path.
  The definitions below intentionally describe the current contract, not those retired
  forms.
- Glossary plurals are derived automatically, so do not add redundant `chops` or
  `lumberjacks` aliases. Keep `Chop` dependency-light; let `Lumberjack` mention `chops`
  so reading the scheduler concept also resolves the scheduled unit without pulling
  unrelated glossary closures into agent context.

## Implementation

1. Run `just install` before repository checks, as required for an ephemeral workspace.
2. Use `sase glossary add` with `--no-init` to add these exact canonical entries to
   `memory.glossary` in `sase/sase.yml`; rely on the command's Rust validation and
   sorted insertion rather than editing YAML by hand:
   - `Chop`: "A chop is one short, script-only unit of AXE automation. AXE supplies its
     context and applies configured cadence, triggers, guards, timeouts, and
     deduplication; a structured result may request follow-up launches, which only the
     runner performs."
   - `Lumberjack`: "An AXE lumberjack is an independently supervised scheduler process
     that runs its configured chops concurrently on a fixed interval and records its own
     state and metrics. The AXE orchestrator starts it and restarts it after crashes."

3. Run `sase memory init` once after both additions. Inspect the diff to confirm the
   canonical entries are alphabetically placed and the generated glossary roster and
   provider instruction shims contain both terms, with no unrelated rewrites.
4. Query both entries together with
   `sase glossary read Chop Lumberjack -r "Verify the new AXE glossary definitions and dependency closure"`.
   Confirm both exact definitions render, plural matching needs no authored aliases, and
   `Lumberjack` resolves `Chop` through its `chops` mention without introducing an
   unintended broader closure.
5. Run `sase validate`, then run the required `just check`. If scoped verification
   escalates or reports unusual selection, use `/sase_monitor` to run `just check-full`
   with the required `TESTING` / `TESTED` statuses and a concrete follow-up action.

## Acceptance criteria

- `sase/sase.yml` contains sorted canonical `Chop` and `Lumberjack` entries with the
  exact concise definitions above and no redundant aliases.
- Generated memory and provider instructions expose both canonical terms and remain
  internally consistent.
- An audited batched glossary read returns both definitions and only the intended
  `Lumberjack` to `Chop` dependency.
- `sase validate` and the required repository check pass.
