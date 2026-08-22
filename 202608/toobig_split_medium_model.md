---
tier: tale
title: Route toobig split agents through the medium model pool
goal:
  Every agent proposed by the toobig_split chop launches with one explicit
  %model:@medium directive while preserving the existing sequential split workflow.
size: small
proposed_by: bbugyi200.athena.0b9
---

# Route `toobig_split` agents through `@medium`

## Context

The configured `toobig_split[sase]` chop is supplied by the external
`bbugyi200/bugyi-chops` repository. It currently emits one structured launch proposal
per oversized Python file with a prompt shaped like
`%auto %wait(priority=20) #split_file:<path>`, but it supplies no proposal model. Axe
therefore scaffolds each agent without a `%model` directive and routes it through the
ordinary no-directive launch default instead of the load-balanced `@medium` alias pool.

SASE's chop SDK already exposes a structured `model` field on
`ChopResultBuilder.propose`, and Axe renders that field as `%model:<value>` in the final
launch scaffold. Use that boundary rather than changing SASE core, changing the global
default model, or embedding a second source of model truth in the prompt body. The
resulting agents must each carry exactly one `%model:@medium` directive while retaining
the existing sequential clan, runner throttling, deduplication, and split-file workflow.

## Implementation

1. Open `gh:bbugyi200/bugyi-chops` through `sase repo open`. In
   `src/bugyi_chops/toobig_split.py`, set `model="@medium"` on every proposal created by
   the single `result.propose(...)` loop. Keep the existing prompt body, proposal IDs,
   `toobig-@` clan template and summary, content-sensitive dedupe keys, `wait_on` chain,
   and launch priority unchanged. Do not resolve `@medium` to a concrete provider/model
   in the plugin: Axe must preserve `%model:@medium` through launch so SASE consumes the
   pool once for each real agent invocation.
2. Extend `tests/test_toobig_split.py` at both sides of the chop/Axe boundary. Assert
   that every raw proposal contains `model == "@medium"`, including custom-tree and
   hard-limit findings, and that `plan_chop_proposals(...)` renders exactly one
   `%model:@medium` directive into every surviving scaffold (including a promoted clan
   tail after deduplication). Parse at least one planned prompt with SASE's directive
   parser and assert the normalized model and alias provenance are `medium`. Retain the
   existing assertions for prompt content, wait ordering, names, clan declaration,
   dedupe, reports, and error/no-op paths so the routing change cannot weaken those
   contracts.
3. Update `README.md` wherever it describes or exemplifies `toobig_split` launch
   proposals. Show that the chop selects `@medium` through the structured proposal model
   and that Axe emits `%model:@medium` alongside `%auto #split_file:<path>`. Explain
   that this deliberately chooses the configurable/load-balanced alias rather than
   pinning a concrete model. Keep scheduler/configuration examples focused on scan
   policy; no new chop variable or user configuration knob is required.

## Validation

1. Install the external repository's development environment as its `justfile` requires,
   then run the focused `tests/test_toobig_split.py` suite.
2. Run the repository's complete `just check` gate (format/lint, strict typing, full
   tests with coverage, and package build/twine validation).
3. Install the updated plugin into a compatible SASE development/runtime environment and
   force a dry run of the configured `toobig_split[sase]` chop with verbose output.
   Confirm every validated proposal reports model `@medium` and every proposal scaffold
   contains exactly one `%model:@medium` before the unchanged `%auto ... #split_file`
   prompt. Dry-run mode must launch no agents.
4. For the first eligible live chop run after deployment, inspect the launched agent
   metadata/ACE rows and confirm each member records `@medium` as direct alias
   provenance while concrete providers/models vary according to the configured pool;
   also confirm the clan still runs sequentially and advances the alias pool only when a
   member reaches its real invocation.

## Non-goals

- Changing SASE's `%model` parser, alias resolution, load-balancing cursor semantics,
  chop proposal scaffolding, or built-in `@medium` membership.
- Changing the global default model, split-file xprompt, scan thresholds/cadence,
  `wait_runners`, launch priority, clan identity/summary, or once-per deduplication.
- Adding a per-chop model configuration option or falling back to a concrete model;
  operators can retune the existing `@medium` pool centrally.
