---
tier: tale
title: Finish shell completion measurement, inline references, and deployment
goal:
  Complete sase-rm.5 with deterministic completion tests, embedded prompt-reference
  completion, measured fish behavior, and verified managed deployment across SASE,
  sase-telegram, and chezmoi.
size: medium
proposed_by: bbugyi200.athena.sase-rm.5
bead: sase-rm.5
create_time: 2026-08-21 05:58:55
status: done
---

- **PARENT:** [202608/task_backlog_closeout.md](task_backlog_closeout.md)
- **BEAD:**
  [sase-rm.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rm/sase-rm.5.md)

# Finish shell completion measurement, inline references, and deployment

## Goal

Complete phase bead `sase-rm.5` by satisfying the six assigned task contracts across the
primary SASE checkout and the linked `sase-telegram` and `chezmoi` repositories.
Preserve the completion fast path, make managed and local script ownership explicit,
verify every repository changed, record close-ready evidence for the parent epic's land
agent, resolve this phase's epic symbols, and close only `sase-rm.5`.

## Current state and constraints

- `sase-ok`, `sase-ow`, `sase-ox`, `sase-oy`, `sase-p9`, and `sase-pg` are all still
  `ready`; none is owned by another worker. This phase implements and verifies their
  close conditions but does not close those task beads—the epic land agent owns that
  audit.
- The generated zsh, bash, and fish scripts already share the argparse-derived
  completion spec and pre-argparse candidate protocol. `sase run` currently offers
  whole-word files and xprompt names only; embedded `#`, `%`, and `@` completion is
  explicitly deferred in code and docs.
- Candidate latency currently uses `time.perf_counter()` around a subprocess, so the
  assertion measures host scheduling. The real-zsh install test delegates to the
  production five-second interactive probe, which can time out under parallel load.
- Local completion installs own their target through per-shell stamps, and the update
  hook refreshes every stamped target. Chezmoi-managed scripts therefore need a distinct
  ownership marker so update refresh, doctor, target-side zcompile, and explicit local
  installation cannot silently fight over the same files.
- Repository access and checks must follow each checkout's instructions. Run
  `just install` before primary checks; use native `just check` lanes in each linked
  repository changed. Do not create task beads; append any distinct discovery as a
  `PROPOSED FOLLOW-UP:` note on `sase-rm.5`.

## Implementation

1. Establish green baselines and deterministic test seams.
   - Install the primary workspace dependencies, run the focused completion contract,
     emitter, install, doctor, and smoke tests, and capture the existing timing and zsh
     probe behavior.
   - Replace the subprocess wall-clock budget with child process CPU accounting (and
     retain the forbidden-import contract) so every shipped candidate kind is measured
     without scheduler-luck failures. Include any newly added directive/reference kind
     in the same parametrized contract.
   - Keep the production zsh probe timeout unchanged. Give the real interactive-zsh
     registration test its own explicitly bounded probe callable, then exercise the
     exact node repeatedly and under contention so it still verifies a genuine
     interactive registration without consuming the production latency budget.

2. Add embedded prompt-reference completion without crossing the fast-path boundary.
   - Add a completion-safe directive-name catalog sourced from the shared Rust directive
     contract, with descriptions and hidden directives filtered consistently with ACE;
     do not import ACE, Textual, Rich, or the full argparse parser on the candidate
     path.
   - Define one small marker contract for `sase run` prompt fragments: `#` selects
     xprompts, `%` selects directives, and `@` selects canonical artifact references.
     Preserve marker prefixes in inserted values and retain whole-word file/xprompt
     fallback when the cursor is not inside a reference.
   - Extend the generated zsh, bash, and fish helpers using each shell's cursor-aware
     input surface (`PREFIX`/current compsys word, `COMP_LINE` plus `COMP_POINT`, and
     `commandline`) so quoted prompts and prompt text containing spaces complete the
     active embedded fragment rather than replacing the full prompt.
   - Add emitter assertions and skip-safe live shell tests for all three markers in all
     three shells, including quoted/spaced prompts, insertion prefixes, fallback file
     completion, cache behavior, and the no-heavy-import/latency contract. Add a real
     fish smoke test when fish is available and preserve clean skips elsewhere.

3. Measure fish and update the user-facing contract.
   - Install or otherwise run the Debian fish 4.0.2 binary on `athena`, generate and
     source the live fish script, and measure script load, warm cached TAB, and cold
     first-fetch TAB with a documented repeatable method. Record host, shell version,
     sample count/statistic, and cache state.
   - Replace all three fish placeholders and the estimate prose in `docs/completion.md`;
     also remove the embedded-reference deferral and document the `#`, `%`, and `@`
     behavior and managed-script ownership semantics.

4. Implement explicit chezmoi-managed completion deployment.
   - Add a completion deployment surface, following the CLI sorting/help/short-option
     rules, that renders all three scripts plus version/digest ownership metadata into
     the configured chezmoi source tree and delegates commit/push/apply behavior through
     `src/sase/main/_init_chezmoi_deploy.py`. Planning/dry-run modes must be read-only,
     and the implementation must remain testable against a temporary chezmoi root.
   - Extend completion stamp/status handling so `local` and `chezmoi` ownership are
     distinguishable and backward compatible. The update refresh hook must skip
     chezmoi-owned targets, doctor/list must validate their metadata and scripts, and an
     explicit local install must have a deliberate transition or refusal instead of
     silently taking ownership.
   - In the linked `chezmoi` checkout, add the generated `_sase`, `sase`, and
     `sase.fish` sources at their conventional target paths, add the managed metadata,
     put `~/.zfunc` on `fpath` before `compinit`, and add an idempotent target-side
     onchange hook that zcompiles `_sase` only when needed. Add linked-repository tests
     that render/apply into a temporary home and verify paths, metadata, zcompile
     freshness, and repeat application.

5. Stabilize the linked Telegram CI bootstrap.
   - In `sase-telegram`, replace the unauthenticated latest-version lookup with a pinned
     `just` release and an authenticated action input where supported. Keep permissions
     least-privileged and add a workflow contract test or equivalent static assertion so
     the pin/token cannot regress unnoticed.
   - Run `just install` and `just check` in `sase-telegram`, and validate the affected
     workflow syntax/contract repeatedly. Record the linked-repository path and exact
     verification in phase evidence.

6. Verify, record evidence, and close the assigned phase only.
   - Run focused tests while iterating, regenerate and inspect the completion structural
     snapshot if candidate kinds or parser surfaces change, then run primary
     `just check`. If selection broadens or reports an unusual escalation, use the SASE
     monitor workflow for `just check-full` as required by project instructions.
   - Run the full native checks required by the linked `chezmoi` and `sase-telegram`
     repositories, inspect all three worktrees for only intentional changes, and rerun
     the live zsh/fish probes and performance measurements on the final tree.
   - Append one evidence-rich, close-ready block per assigned task (`sase-ok`,
     `sase-ow`, `sase-ox`, `sase-oy`, `sase-p9`, `sase-pg`) to `sase-rm.5`, naming the
     cause, files/repository, and exact verification. Leave the task beads open for the
     epic land agent.
   - Run `sase bead epic-symbols sase-rm.5`. Resolve every remaining symbol or re-key
     its Justfile ownership to the parent epic or a later open phase, rerun required
     checks after any re-key, and finally close only `sase-rm.5` with a note summarizing
     the verified primary and linked-repository results.

## Acceptance criteria

- Zsh, bash, and fish complete embedded `#xprompt`, `%directive`, and canonical
  `@artifact-reference` fragments inside quoted or spaced `sase run` prompts, while
  whole-word file/xprompt completion and completion caches continue to work.
- Every shipped candidate kind satisfies a deterministic CPU/import contract; the flaky
  wall-clock assertion is gone.
- The real interactive-zsh registration test passes repeated contention runs through a
  test-owned bound, while the production five-second probe contract remains unchanged.
- `docs/completion.md` contains real fish 4.0.2 measurements for load, warm TAB, and
  cold TAB, including host and method, and a skip-safe live fish smoke test passes when
  fish is present.
- `sase-telegram` CI uses a pinned, authenticated/reliable `just` setup and its full
  check lane passes.
- Chezmoi deploys current scripts, metadata, zsh `fpath` ordering, and an idempotent
  `.zwc` refresh hook; managed stamps are accepted by doctor/list but skipped by local
  update refresh, and the linked checkout's full check lane passes.
- Primary focused tests and `just check` pass; all phase epic symbols are resolved or
  validly re-keyed; close-ready evidence is recorded; only `sase-rm.5` is closed.
