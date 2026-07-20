---
tier: tale
title: Percent-id bead runner lifecycle
goal: 'A bead association declared by %id(bead=...) survives every launch rewrite,
  is published consistently as runner metadata and environment, and atomically claims
  the bead only after all waits and workspace preparation succeed but before model
  execution begins.

  '
create_time: 2026-07-20 16:27:54
status: done
prompt: 202607/prompts/id_bead_runner_lifecycle.md
---

# Plan: Percent-id bead runner lifecycle

## Context and outcome

The preceding `sase-8f.1` phase exposed the Rust-backed `claim_for_agent_launch` bead mutation through the Python facade
and `BeadProject`. This phase connects that operation to the generic agent launch lifecycle. It does not change
`sase bead work` rendering or remove its eager preclaim path; those producer-side changes belong to the later
`sase-8f.3` phase.

The new `bead=` keyword belongs to `%id`, but is orthogonal to the existing `clan=`, `family=`, and `tribe=` membership
axis. A prompt may therefore bind one non-empty, whitespace-free bead ID to a plain, auto-named, clan, family, or tribe
agent. Legacy prompts without the keyword keep their current behavior. The association may be parsed and displayed
before waits complete, but no bead mutation may occur until the runner has passed dependency waits, code refresh,
repeat-stop detection, runner-slot admission, deferred-workspace allocation, primary workspace preparation, and
linked/SDD repository preparation.

## Directive model and preservation

Extend the `%id` argument model, raw collection, expanded value resolution, `PromptDirectives`, and runner `AgentInfo`
with an optional bead ID. Accept and expand `bead=` using the same xprompt-reference path as the other `%id` keywords,
retain the existing targeted duplicate/unsupported-key diagnostics, and add a targeted error for empty or
whitespace-bearing values. Keep `clan=`, `family=`, and `tribe=` mutually exclusive with one another while allowing
`bead=` beside any one of them or by itself. In particular, `%id(bead=...)` must still allocate an automatic agent name.

Audit every transformation that consumes and re-emits `%id`: static name and family inspection, alternative/model
fan-out naming, repeat fan-out, forced reuse, retry/relaunch naming, clan declaration demotion, and clan member
rewriting. Preserve `bead=` whenever the transformed prompt still represents the same logical worker. A family retry
must remove only the obsolete `family=` attachment while retaining `bead=` and any other orthogonal identity metadata.
Keep literal/fenced and disabled-region behavior unchanged.

The Rust launch fan-out planner currently strips the whole `%id` directive for repeat slots and returns only its
positional name. Extend that shared backend contract narrowly so repeat-slot prompts can re-emit the orthogonal bead
association without duplicating membership semantics. Keep the Python launch wire adapter and repeat launcher in parity,
and add Rust/Python round-trip coverage. Update both Rust editor metadata and Python TUI completion/help text so `bead=`
is discoverable alongside the existing `%id` keyword candidates.

## Metadata and environment contract

Publish the parsed association as `bead_id` in `agent_meta.json` and return it through `AgentInfo`. Preserve it through
bootstrap/re-exec metadata refresh and follow-up metadata rewrites without replacing the richer existing `epic_bead_id`
and `phase_bead_id` role fields.

Treat the directive as the source of the generic launch association. When it is present, establish `SASE_BEAD_ID` if the
environment does not already provide it. If the environment already provides a non-empty value, require an exact match
and fail before model execution on a mismatch. A legacy launch that supplies `SASE_BEAD_ID` only for commit attribution
but has no `%id(bead=...)` remains non-claiming until the producer migration in the next epic phase.

## Post-preparation launch claim

Add a focused runner lifecycle helper that receives the resolved agent name, bead ID, final workspace directory/number,
and artifacts directory. It must:

1. Resolve the effective SDD/bead store from the final workspace after all deferred and linked repository preparation
   has completed.
2. Open the supported bead-store layout and call the existing atomic `BeadProject.claim_for_agent_launch` operation.
3. Persist the resulting bead event/projection using the managed SDD commit and push policy, treating store resolution,
   mutation, local staging/commit, and durable persistence failures as fatal while retaining a successful local commit
   when a configured remote push only emits a warning.
4. Refresh any local projection/cache required by subsequent Python readers and return the claimed issue for diagnostics
   or tests.

Wrap failures with an actionable message containing both the resolved agent name and bead ID while preserving the
underlying cause. Invoke the helper once per runner process after wait/repeat-stop handling and workspace/linked-repo
preparation, but before the post-claim SDD base SHA is captured and before `run_execution_loop` can invoke a model. Do
not roll a successful claim back if later model execution fails or the agent is killed. Code-refresh re-exec occurs
before this point, in-process model retries reuse the one claim, and an explicit spawn retry may idempotently reassign
the already-in-progress bead to its new agent name.

## Verification

Add focused parser tests for plain, auto-named, clan, family, and tribe forms; xprompt expansion; duplicate, empty,
whitespace, and unknown-key diagnostics; and prompts without `bead=`. Add preservation tests for static inspection,
alternative/model and repeat fan-out, retry/family retry, forced reuse, clan demotion, and clan member rewrites.
Exercise Python and Rust completion/help parity for the new keyword.

Add metadata/environment tests covering initial publication, matching and mismatching `SASE_BEAD_ID`,
bootstrap/code-refresh preservation, and coexistence with epic/phase role metadata. Add helper tests for store
resolution, successful atomic claim and persistence, reassignment, and useful missing/closed/store/ commit errors.

Extend runner-order tests to prove that the bead remains untouched during dependency and runner-slot waits, repeat-stop
exits without a claim, deferred workspace plus linked/SDD preparation precedes the claim, a successful claim precedes
SDD-base capture and model execution, and every mismatch, resolution/mutation/persistence failure suppresses model
execution and reaches the normal runner error/done artifacts. Confirm a blocking code refresh claims only on the
refreshed post-wait pass and that legacy launches remain non-claiming.

In `sase-core`, run the focused launch fan-out/editor tests, then `cargo fmt --all -- --check`,
`cargo test --workspace`, and `cargo clippy --workspace --all-targets -- -D warnings`. In `sase`, run `just install`
before focused directive, completion, metadata, repeat, and runner lifecycle tests, then run the required `just check`.
Do not modify SASE memory files or close the parent epic.
