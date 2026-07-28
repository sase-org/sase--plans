---
tier: epic
title: 'Make bead plan linkage durable with logical plans: references'
goal: 'A bead''s link to its plan survives machines, workspaces, and SDD layout changes:
  beads store a logical `plans:<YYYYMM>/<name>.md` reference resolved against the
  active plan roots at read time, one shared API owns canonicalization and resolution
  for every caller, `sase bead doctor` reports broken and ambiguous links instead
  of silently passing, and `--fix-design-refs` repairs the 227 stored links that no
  longer resolve.

  '
phases:
- id: refs
  title: Canonical plans reference scheme in the Rust core
  depends_on: []
  size: medium
  description: 'refs: add the parse/render/resolve API for `plans:` references to
    the Rust core, expose it through the PyO3 binding, and give the Python side one
    root-resolution facade that also accepts every legacy path form.

    '
- id: resolve
  title: Route every plan-reference resolver through the shared API
  depends_on:
  - refs
  size: medium
  description: 'resolve: replace the five independent path-guessing implementations
    that turn a stored reference into a file with calls into the shared resolver,
    so every surface agrees on what a reference means.

    '
- id: write
  title: Persist plans references on new beads
  depends_on:
  - refs
  - resolve
  size: small
  description: 'write: make every writer of a bead''s plan link emit the canonical
    `plans:` form, in both the Python epic-creation path and the Rust `bead create`
    path.

    '
- id: display
  title: Show the logical reference and its resolved path
  depends_on:
  - refs
  - resolve
  size: small
  description: 'display: render the stable reference together with where it currently
    resolves, and say so plainly when it resolves nowhere, in `sase bead show` and
    the ACE Plans tab.

    '
- id: doctor
  title: Validate and repair stored plan links
  depends_on:
  - refs
  - resolve
  size: large
  description: 'doctor: make the health check actually check plan links, add an opt-in
    repair that rewrites legacy paths to canonical references using the plan `bead_id`
    reverse index, and migrate the live store.

    '
create_time: 2026-07-27 08:39:03
status: done
bead_id: sase-9z
---

- **PROMPT:** [202607/prompts/durable_plan_refs.md](prompts/durable_plan_refs.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-9z.4](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-9z.4/README.md)
  - [bbugyi200.athena.sase-9z.5](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-9z.5/README.md)
  - [bbugyi200.athena.sase-9z.5--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-9z.5.md#member-code)
- **COMMITS:**
  - [7ac5b91](https://github.com/sase-org/sase/commit/7ac5b917c08ebe10f847caacdcabf2a2fcc401a6) — feat(beads): repair legacy design references (sase-9z.5)

# Plan: Make bead plan linkage durable with logical plans: references

## Context

A bead's `design` field is the only pointer from a bead back to the plan that produced it. It is stored as a filesystem
path, so it encodes a machine, a numbered workspace, and an SDD layout — none of which are stable. The pointer is the
foundation of every "beads as durable memory" behavior: `sase bead show`, the ACE Plans tab, the epic plan snapshot
handed to launching agents, the mobile bridge, and clan summaries all start from it.

### Evidence from the live store

Measured against the `sase` bead store (`sase/repos/plans/beads/issues.jsonl`, 2,109 beads) from a numbered workspace:

- 343 beads carry a `design` value. **227 of them do not resolve** as stored; 116 do.
- Broken values by form: 137 absolute paths into a checkout that no longer exists
  (`/home/bryan/projects/github/sase-org/sase/plans/202604/...`), 83 workspace-relative `sdd/plans/...` paths from the
  retired in-tree layout, and 7 one-off shapes including a bare `plan.md`, a `sdd/legends/...` path, and a leaked pytest
  tmpdir path.
- Everything that resolves today uses the current workspace-relative `sase/repos/plans/...` form (113) — which is only
  correct while a plans-sidecar clone happens to sit at that relative path. One of this host's 21 workspaces has no
  sidecar clone, and nothing outside a workspace can resolve it at all.
- One stored value is already `202607/epic_clan_summary_rich.md` — the logical form, minus a scheme to make it legible.

The damage is fully repairable: 212 of the 227 broken links have a basename match under the sidecar's month directories,
and only one of those matches is ambiguous. 440 sidecar plans carry `bead_id:` frontmatter, giving an authoritative
reverse index for the rest. Only 15 links have no recoverable target anywhere.

`sase bead doctor` reports none of this. `doctor` in `crates/sase_core/src/bead/read.rs` checks store files, orphan
phases, and legacy-projection drift; it never looks at `design`. A store where two thirds of the plan links are dead
currently reports clean.

### Why a path cannot work and a logical reference can

The same plan file legitimately lives at more than one absolute path over its life, and at more than one at a time:

1. `sase plan propose` moves the authored plan into the machine-local archive at `~/.sase/plans/<YYYYMM>/<name>.md`
   (`sase.main.plan_propose_handler` → `sase.llm_provider._plan_utils.move_plan_to_sase`).
2. Approval copies it into the SDD store's plans root — for this project the `sase--plans` sidecar — via
   `sase.sdd.plan_archive.archive_plan_file`.
3. That sidecar is cloned **once per workspace**, so the same plan exists at ~21 distinct absolute paths concurrently,
   plus the primary checkout.

A reference rooted at "the plans root" rather than at a directory is stable across all of these. It is also stable
across the SDD storage modes SASE supports (`in_tree`, `local`, `separate_repo`, `sidecar_repos`), because each mode
already knows its own plans root through `SddStore.kind_root("plans")`.

One wrinkle to design for rather than ignore: `archive_plan_file` shards an incoming plan by the _current_ month, so a
plan proposed in one month and approved in the next changes month directory between the local archive and the store. The
resolver therefore treats the month segment as a hint, not a hard key.

### Reference syntax

```
plans:<relative-posix-path>
```

The payload is relative to the resolved plans root, in practice `<YYYYMM>/<name>.md`. A reference is rejected if it is
empty, absolute, contains a `..` or empty segment, or contains a backslash — the same containment guarantee
`sase.bead.cli_work_plan_snapshot.epic_plan_source_path` enforces today with its "escapes its plans store" check.

Parse the scheme as `<kind>:<path>` with `plans` the only registered kind, so `research:` can join later without a
format change. Do not add other kinds now.

Any value with no registered scheme prefix is a **legacy path** and keeps resolving through the existing heuristics.
Legacy values are never rejected and never silently rewritten; only `doctor --fix-design-refs` rewrites them.

### Resolution

Resolution takes an ordered list of plan roots. The order is store-first, matching how `sase.plan_search.facade` already
ranks the two corpora:

1. the active SDD store's plans root (`SddStore.kind_root("plans")`),
2. the machine-local archive `~/.sase/plans/`.

The algorithm, in order:

1. **Exact.** First root where `<root>/<payload>` is a file wins.
2. **Month drift.** If no exact hit and the payload is `<month>/<name>`, glob `<root>/*/<name>` across the roots in
   order. Exactly one match resolves; more than one is _ambiguous_ and resolves to nothing.
3. **Missing.** No match.

Resolution returns a structured outcome — resolved path, one of `exact` / `drifted` / `ambiguous` / `missing`, and the
candidates considered — so `show` and `doctor` can report precisely instead of each re-deriving a verdict.

### The five resolvers this replaces

Every one of these independently re-implements "strip a known prefix, then join against a guessed root", and they do not
agree:

- `src/sase/bead/cli_work_plan_snapshot.py::epic_plan_source_path` — markers `sdd/plans`, `plans`, `sase/repos/plans`,
  `.sase/sdd/plans`, branching on the beads-dir layout.
- `src/sase/ace/tui/models/_agent_associated_plan_paths.py::resolve_plan_reference` — a different candidate list built
  from the agent's workspace, its primary workspace, and the SDD plans root.
- `src/sase/ace/tui/widgets/artifacts/plans_data.py` linked-document loading, against a passed-in `plans_root`.
- `src/sase/plan_documents.py::_plan_root_relative_path` and `_effective_plans_root`, for the `SASE_PLAN` commit tag.
- `src/sase/scripts/sase_clan_summary_plan.py::_resolve_plan_reference`.

Display is duplicated too: `sase.bead.cli_query._display_design_path` (Python, the live `sase bead show`) and
`display_design_path` in `crates/sase_core/src/bead/cli.rs` (the Rust parity renderer —
`src/sase/main/bead_fast_path.py` excludes `show` and `list` from the fast path, so today only Rust tests exercise it).
Both must stay in step.

### Boundary and constraints

Reference syntax, canonicalization, and resolution are core backend behavior under the Rust core boundary: the CLI, the
TUI, the mobile bridge, and any future frontend must agree on what `plans:202607/foo.md` means. That logic belongs in
the `sase-core` linked repo, with plan roots passed in as parameters — exactly the split
`crates/sase_core/src/plan/search.rs` already uses, where Python resolves `repo_sdd_root` and `local_plans_dir` and Rust
owns the algorithm. Root _discovery_ stays in Python, because `sase.sdd.store` owns storage-policy resolution.

Open the `sase-core` repo with the `/sase_repo` skill; do not clone or web-fetch it. Any wire or binding change lands
there first, together with the `sase-core-rs` version requirement bump in this repo.

Non-goals, to keep this epic bounded:

- No change to the `parent:` plan frontmatter field. It is stamped from `SASE_EPIC_PLAN_REF` in `plan_propose_handler`,
  so it inherits the new form for free, and it currently has no reader.
- No `research:` references, no new `sase plan` subcommand for resolving a reference, and no rewrite of the generated
  `sase_beads` skill beyond documenting the new `doctor` flag.
- No change to how plans are archived, sharded, or committed.

New and changed CLI options follow `sase/memory/cli_rules.md`: alphabetically sorted, a short alias for every public
long option, and help output that scans cleanly.

## Canonical plans reference scheme in the Rust core

Add the reference API to `sase-core` as a new module under `crates/sase_core/src/plan/` (`refs.rs`, re-exported from
`plan/mod.rs`), with no PyO3 types, so bindings and future frontends share it.

The module owns four operations:

- **Parse** a stored value into either a typed reference (registered scheme + validated payload) or a legacy path.
  Enforce the containment rules from the Context section; a malformed `plans:` value is a parse error, not a silent
  fallback to legacy.
- **Render** a typed reference back to its canonical string.
- **Canonicalize** an absolute plan path against the ordered roots into a typed reference, returning nothing when the
  path lies under no root so callers can keep their present fallback.
- **Resolve** a stored value against the ordered roots, implementing the exact/drift/ambiguous/missing algorithm and
  returning the structured outcome. Legacy values resolve through the consolidated prefix-stripping markers
  (`sase/repos/plans`, `.sase/sdd/plans`, `sdd/plans`, `plans`, bare, absolute) gathered from the five existing
  implementations, so no currently-working link regresses.

Add a wire record for the resolution outcome with its own schema version constant, following the
`PLAN_*_WIRE_SCHEMA_VERSION` convention in `crates/sase_core/src/plan/wire.rs`. Cover the module with Rust unit tests:
round-trip render/parse, every rejection case, first-root-wins ordering, month drift resolving to a single hit, drift
with two hits reporting ambiguity, and each legacy form resolving to the same file as before.

Expose parse/render/canonicalize/resolve through `crates/sase_core_py/src/lib.rs` following the `py_plan_search`
pattern, and register the functions in the module init.

In this repo, rewrite `src/sase/sdd/plan_refs.py` into the single Python entry point. It keeps `plan_ref_for_store` as
the canonicalization façade and gains the root-resolution and resolution helpers, resolving the ordered roots from
`sase.sdd.store.resolve_sdd_store(...).kind_root("plans")` and `sase.core.paths.sase_subdir("plans")`, with the Rust
calls behind `sase.core.rust.require_rust_binding`. Callers pass a workspace directory and number; nothing outside this
module hardcodes a plans root again. Behavior of existing callers is unchanged in this phase — `plan_ref_for_store`
still emits its current output — so this phase ships an API plus tests, not a data-format change.

## Route every plan-reference resolver through the shared API

Replace each of the five resolvers listed in the Context section with a call into `sase.sdd.plan_refs`, and delete the
prefix-marker tables they carry. `plan_ref_after_marker` in `src/sase/bead/cli_work_plan_snapshot.py` exists only to
serve those tables; remove it once its last caller is gone.

Two call sites need care rather than a mechanical swap:

- `epic_plan_source_path` currently derives its root from the _beads_ directory layout and raises on an unsupported
  layout. It must keep raising a clear error when the reference cannot be resolved, because its caller copies the result
  into the launching agent's plan snapshot, and it must keep the containment guarantee that a reference cannot escape
  the plans root — now enforced by the parser rather than by a post-hoc `relative_to` check.
- `_agent_associated_plan_paths.resolve_plan_reference` returns a path even when nothing exists, and ACE relies on that
  for its "plan file missing" rendering. Preserve that contract: return the best candidate on a miss, and surface the
  resolution status separately for the `display` phase to use.

`src/sase/integrations/_mobile_helper_beads.py` returns the raw `design` string to the mobile client; have it return the
resolved path when resolution succeeds and the stored reference otherwise, so the bridge does not hand a client a string
it cannot open.

Add tests that pin the cross-surface agreement this phase buys: one fixture plan reachable through the store root and
one only through the local archive, asserting the snapshot path, the ACE association path, and the clan-summary path all
resolve identically. Include a regression case for each legacy form found in the live store (absolute, `sdd/plans/...`,
`sase/repos/plans/...`, bare month-relative).

## Persist plans references on new beads

Switch the writers to the canonical form once every reader understands it.

`plan_ref_for_store` in `src/sase/sdd/plan_refs.py` returns `plans:<payload>` when the plan lies under any resolved
root, and keeps its current workspace-relative or absolute fallback otherwise. This covers the epic-creation path:
`src/sase/bead/cli_work_from_plan.py` computes the ref and `src/sase/bead/epic_from_plan.py` stores it as `design` on
the epic.

`epic_from_plan` also feeds the same string to `generated_phase_description` in `src/sase/bead/phase_description.py`,
which embeds it in phase prose as ``Phase `<id>` in approved epic plan `<ref>`.`` — that text now carries the canonical
reference too, which is the desired outcome and needs its test expectations updated.

On the Rust side, `storage_design_path` in `crates/sase_core/src/bead/cli.rs` canonicalizes the `--type plan(<path>)`
argument by stripping a root derived from the beads directory. Reimplement it on the new canonicalize API, keeping
`design_storage_root`'s layout detection as the source of the store root, and keeping the existing "plan file not found"
error. Update the golden CLI contract tests in that file.

Two consequences to verify rather than assume: `design` is indexed by `sase bead search`, so a query matching a path
substring now matches a shorter string; and `SASE_EPIC_PLAN_REF` (`src/sase/bead/work.py`) carries the new form into
launched agents and into the `parent:` frontmatter stamped by `plan_propose_handler`. Confirm both behave sensibly and
note anything surprising in the phase's completion message.

## Show the logical reference and its resolved path

`sase bead show` prints one line under `PLAN` today. Print the stable reference and where it currently resolves, and
mark the failure states explicitly — a bead whose plan cannot be found must say so rather than print a path that does
not exist. Keep the existing behavior of relativizing a displayed path against the cwd for in-tree stores.

Update both renderers together: `_display_design_path` and its call sites in `src/sase/bead/cli_query.py` (the live
path, including the `PARENT PLAN` / `EPIC PLAN` fallbacks that render an ancestor's link), and `display_design_path` in
`crates/sase_core/src/bead/cli.rs` with its golden tests, so the parity implementation does not drift.

In the ACE Plans tab, `src/sase/ace/tui/widgets/artifacts/plans_detail.py` renders a `Design path` row and a
`**Design:**` markdown line; show the logical reference as the primary value with the resolved path secondary, and
render an unresolved link as such. `plans_filtering.py` includes `issue.design` in its filter corpus — decide and test
whether a query should match the reference, the resolved path, or both, and state the choice in the phase's completion
message. `plans_navigation.py` passes `row.issue.design` as `source_path` to an external tool; that must become the
resolved path, since a reference is not openable.

`src/sase/scripts/sase_clan_summary_epic.py` prints `epic.design` verbatim in a clan summary panel; give it the same
treatment.

Cover the rendering with the existing TUI test patterns, including a bead whose link is missing and one that resolves
only through month drift.

## Validate and repair stored plan links

Two capabilities: honest reporting by default, and opt-in repair.

**Reporting.** Extend `doctor` in `crates/sase_core/src/bead/read.rs` to check every bead with a non-empty `design` and
report, per bead: unresolvable references, ambiguous references, and references whose target plan's `bead_id:` or
`bead:` frontmatter names a different bead. Summarize counts rather than printing 227 lines. `doctor` currently takes
only `beads_dir`, so this needs plan roots threaded through the core signature, the `py_bead_doctor` binding, and
`sase.core.bead_read_facade.doctor`, with `src/sase/bead/project.py` and `src/sase/bead/cli_admin.py` supplying roots
from `sase.sdd.plan_refs`. Reporting stays strictly read-only and must not fail the health check when the SDD store is
unavailable — degrade to skipping the check with a clear note.

**Repair.** Add `-F, --fix-design-refs` to `sase bead doctor` in `src/sase/main/parser_bead.py` and
`src/sase/bead/cli_admin.py`. For each bead whose stored value is a legacy path or an unresolvable reference, pick a
target in this order, and never guess past the first ambiguity:

1. the plan whose `bead_id:` frontmatter names this bead (authoritative; 440 sidecar plans carry it);
2. a unique basename match under the store's month directories;
3. a unique basename match under `~/.sase/plans/`.

Rewrite the value to the canonical `plans:` form through a normal bead update so the change is an ordinary event that
syncs like any other. Report every bead left unrepaired with the reason (no candidate, ambiguous, or frontmatter naming
a different bead) — expect roughly 15 with no candidate and 1 ambiguous. Print a plan of action and require confirmation
before mutating, consistent with how the repo gates destructive-ish operations; support a report-only invocation for
inspection.

Decide deliberately whether repair also rewrites already-resolving `sase/repos/plans/...` values to the canonical form.
Doing so is the point of the epic — those 113 links are correct only by workspace coincidence — but it turns a targeted
repair into a 343-record migration, so make the choice explicit, test both paths, and say which one ran.

**Migration.** Run the repair against the live `sase` store as part of this phase, commit the resulting bead events
through the normal bead sync path, and report before/after counts. Re-run plain `doctor` afterwards and include its
output in the completion message; the store should report clean plan links apart from the handful with no recoverable
target.

Finally, document the new flag in the generated `sase_beads` skill source under `src/sase/xprompts/skills/` per
`sase/memory/generated_skills.md` — edit the template, not a deployed copy — and in the `sase bead onboard` help text in
`src/sase/bead/cli_admin.py`.
