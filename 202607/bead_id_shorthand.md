---
tier: tale
title: Accept dash-free shorthand in sase bead ID arguments
goal:
  Every sase bead CLI argument that names an existing bead accepts the ID suffix without its project prefix while
  preserving canonical full IDs in storage, output, and automation.
proposed_by: bbugyi200.athena.qi
create_time: 2026-07-31 13:13:53
status: done
---

- **PROMPT:** [prompts/202607/bead_id_shorthand.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/bead_id_shorthand.md)
- **AGENTS:**
  - [bbugyi200.athena.qi](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.qi.md)
- **COMMITS:**
  - [791e751](https://github.com/sase-org/sase-core/commit/791e751fe58f99f4b632ebb4d00d125f5bb4946f) — feat(beads): resolve shorthand bead ids in core

# Plan: Accept dash-free shorthand in `sase bead` ID arguments

## Outcome

Users can pass `a1` anywhere a `sase bead` subcommand currently expects the full ID `sase-a1`. Hierarchical IDs use the
entire dash-free suffix, so `a1.2` resolves to `sase-a1.2`. Existing full IDs continue to work unchanged, including IDs
whose project prefix itself contains dashes. Resolution happens against the active bead store, but canonical full IDs
remain the only form written to bead records, dependency edges, events, mutation summaries, agent metadata, page paths,
and command output.

This is a `tale` because the core resolver, shell integration, documentation, and tests form one bounded change that a
single implementation agent can complete and verify coherently. Splitting them would create a period where the Rust fast
path and Python slow path disagree about accepted CLI syntax.

## Resolution contract

Add one shared resolver in the linked `sase-core` bead backend and expose it through the PyO3 binding for the Python
shell:

- Treat any non-empty token containing `-` as an already-qualified ID and return it unchanged. The command that consumes
  it retains responsibility for the existing not-found behavior, so this feature does not reinterpret or silently repair
  misspelled full IDs.
- For a token with no dash, compare it with the suffix of every current bead ID after that ID's final `-`. This handles
  hyphenated prefixes and forward-only prefix repairs, where old and new full-ID prefixes can coexist in one store.
- Return the unique matching full ID. If there is no match, report the existing not-found failure using the shorthand
  the user supplied. If malformed/imported state produces more than one match, fail with a deterministic ambiguity error
  that names the shorthand and sorted candidate full IDs.
- Never choose an arbitrary candidate and never mutate while resolution is incomplete. Commands accepting multiple bead
  IDs must resolve the entire input set before starting the domain mutation, preserving their current atomicity.
- Keep the resolver at the user-input boundary. Internal stored relationships and backend-to-backend calls continue to
  use canonical IDs, preventing shorthand from leaking into events or wire data.

Cover the pure resolver in `sase-core` with cases for top-level and dotted descendant suffixes, hyphenated and mixed
prefixes, exact full-ID pass-through, unknown shorthand, and ambiguous shorthand. Add the binding registration and a
thin Python facade/`BeadProject` adapter so every Python-owned command uses exactly the same semantics and errors as the
Rust bead CLI fast path.

## Wire every bead CLI ID position

Canonicalize IDs only after the command has resolved the active store (and, for mutations, while it holds the normal
store mutation lock). Replace the parsed value with the canonical full ID before downstream validation, rendering,
commit-message generation, timers, or launch naming. Apply this consistently to:

1. Single- and multi-bead lifecycle/query commands: `show`, `update`, `note`, `open`, `close`, `rm`, and `history`,
   including scoped `history --lost-notes`. For `close --phases`, resolve the one epic ID before deriving its canonical
   child phase IDs. Resolve every explicit `close`/`rm` ID before making an all-or-nothing mutation.
2. Dependency commands: both endpoints of `dep add`, the source and every target of `dep rm`, and the optional scope of
   `dep list`/`dep tree`. Graph lookup, readiness reporting, mutation summaries, and output must use canonical IDs.
3. Artifact-reference commands: the optional scope of `ref list` and the bead target of `ref add`/`ref rm`. Reference
   payloads such as `bead:sase-a1` are durable artifact-reference values, not bead-ID arguments, and remain outside this
   shorthand feature.
4. Work and creation commands: bead targets passed to `bead work`; `--parent` on plan-file work except for the
   `top-level` sentinel; and parent IDs embedded in `bead create --type 'phase(...)'` or `--type 'plan(...,<parent>)'`.
   Detect a real plan-file target before attempting bead shorthand resolution, then use the canonical target for type
   checks, dry-run/JSON output, launch checkpoints, deterministic agent names, and all resumed-work comparisons.
   Canonicalize a create/work parent before previewing or creating hierarchical child IDs.
5. Generated-page commands: `pages refresh --bead` and `pages url`. Resolve the scope before page selection or URL/path
   construction so pages remain addressed by their canonical full ID.

The Rust fast-path handlers and Python fallback handlers must both call the shared resolver; do not implement separate
prefix-concatenation rules. Preserve the current fast-path deferral boundaries (`show`, `close`, `create`, full-format
search, and host-coupled commands) and existing full-ID behavior.

## User-facing contract and documentation

Update parser help/metavars and representative examples to say that an ID may be full or shorthand without changing
option names or command structure. Add a concise shared explanation to `docs/beads.md`: the shorthand is the complete
suffix after the final dash, dotted descendants are supported, output always shows the canonical full ID, and an
ambiguous suffix is rejected.

Update the canonical generated-skill source at `src/sase/xprompts/skills/sase_beads.md` with the same rule and one or
two shorthand examples. Do not edit installed/chezmoi `SKILL.md` files. Preview the generated provider copies with
`sase skill init --diff`; deployment remains the normal post-merge `sase skill init --force` workflow from a clean
canonical tree.

## Tests and acceptance criteria

Add focused Rust CLI/core tests plus Python CLI tests that exercise both execution paths and the host-coupled commands:

- A store containing `sase-a1`, `sase-a1.2`, and a hyphenated-prefix bead accepts `a1`, `a1.2`, and that bead's
  dash-free suffix. The equivalent full IDs still produce the same results.
- Read commands return the canonical issue; mutating commands update only the canonical issue and persist canonical IDs
  in events, dependencies, reference ownership, commit summaries, and output.
- Parameterized coverage spans every ID-bearing command family listed above, including both dependency endpoints,
  multiple close/remove targets, `close --phases`, work dry runs/task launch setup, plan-parent creation, and page
  scoping. Use mocks or dry-run boundaries where needed so tests do not launch real agents or publish stores.
- Unknown shorthand exits nonzero with a useful not-found message. Ambiguous shorthand lists the full candidates and
  exits before any event, projection, commit, page write, or launch side effect. A multi-ID request containing one bad
  shorthand remains entirely unchanged.
- Plan-file detection and the `top-level` sentinel retain their current behavior, and arbitrary artifact-reference
  payloads are not rewritten.
- CLI help and the canonical `sase_beads` skill source document the shorthand, with source-contract tests updated where
  the repository already validates command examples.

After implementation, run the checks in dependency order:

1. `cargo fmt --all -- --check`, focused bead resolver/CLI tests, then
   `cargo clippy --workspace --all-targets -- -D warnings` and `cargo test --workspace` in the opened `sase-core`
   repository (or the equivalent `just rust-check` from the SASE checkout).
2. `just install` in the SASE checkout so the editable environment uses the changed local PyO3 binding.
3. Focused Python bead CLI tests for resolver, lifecycle, dependency, reference, work/create-parent, and pages coverage.
4. `sase skill init --diff` to verify generated skill output without deploying from a dirty tree.
5. `just check` for the required full SASE formatting, lint, validation, and test gate.

The feature is complete when every user-facing bead-ID argument accepts the unique dash-free suffix, every observable or
persisted ID remains canonical, full IDs remain backward compatible, and invalid/ambiguous shorthand cannot cause a
partial mutation or launch.
