---
tier: tale
goal:
  Replace the project-specific toobig xprompt workflow with a reusable, concurrency-safe script chop and verify that it
  launches one split-file SASE agent for every oversized Python file.
---

# Plan: Generalize the toobig split chop

The current `xprompts/toobig_split.yml` workflow mixes scanning and fan-out logic into an inline Python xprompt, assumes
the `sase` GitHub project, and requires an otherwise unnecessary wrapper agent. The athena axe configuration launches
that workflow as an agent chop even though axe can discover and execute standalone chop scripts directly. The live
scanner currently reports 18 qualifying files (7 below `src` and 11 below `tests`); this is the verification baseline at
planning time, while the implementation will always derive the expected launch count from the scan itself so normal
repository evolution cannot make the check stale.

## Implementation

1. Add a chezmoi-managed executable chop script with a generic `toobig_split` identity. It will accept the chop runner's
   `--context` argument, resolve a configured SASE project through the supported machine-readable project CLI, and also
   support direct execution/configuration for another repository without embedding `#gh:sase` or a fixed filesystem
   path. Repository roots, launch refs, scan trees, and thresholds will have explicit CLI/environment boundaries with
   sensible `src`, `tests`, and `1000 850 700` defaults.
2. Keep the launcher independent of SASE's private Python modules. Locate the repository's installed `toobig`
   executable, collect and deduplicate qualifying paths, build one `%group:chop %auto #split_file:<path>` prompt per
   file with collision-safe indexed names, preserve the existing runner-capacity and serial wait behavior, and submit
   the complete multi-prompt through the public `sase run` CLI. Emit concise scan, launch, and failure summaries so axe
   run history is diagnosable.
3. Acquire a nonblocking advisory lock before scanning and hold it through prompt submission. Scope the lock by the
   resolved repository so duplicate scheduled/manual invocations for one checkout abort cleanly while independent
   repositories can use the generic chop concurrently; use an overrideable state directory to keep the behavior
   deterministic in tests.
4. Change `sase_athena.yml` from the `sase_toobig_split` agent workflow to the discovered `toobig_split` script chop,
   configure its target project explicitly as `sase`, and update the chezmoi chop assertions so this entry is no longer
   classified as a standalone-workflow agent chop.
5. Remove the now-unused `xprompts/toobig_split.yml` from the SASE repository. Search both repositories for stale
   references and retain only the reusable script/config/test surface.

## Validation

- Add focused chezmoi bashunit coverage with fake `sase` and `toobig` commands for project resolution, src/test path
  deduplication, generated prompt contents and wait chaining, no-files behavior, scanner/launcher failures, and the
  overlapping-process lock guard. Assert that config selects a script chop and supplies the intended target project.
- Run the relevant chezmoi checks, then its full check target if practical. In the SASE checkout, run `just install`
  followed by the mandatory `just check` because an xprompt source file is removed.
- Manually execute the new source script against the `sase` project. Capture the live agent inventory immediately before
  and after, compare the number and file identities of newly created split-file agents to the scanner's reported
  qualifying paths, and confirm all expected agents were registered (18 with the planning-time tree, unless the tree
  changes before execution). Re-run the lock regression during final verification to prove a concurrent invocation
  cannot submit a duplicate multi-prompt.

## Risks and safeguards

- Axe script chops run from state directories, so repository inference from process CWD is unsafe; explicit project
  resolution is the scheduled path, with direct-repository mode reserved for manual/reuse scenarios.
- A partial SASE fan-out failure must produce a nonzero chop result and visible diagnostics. The script will make one
  public multi-prompt submission under the lock, minimizing the window for partial duplicate launch sets.
- Agent names are permanent reservations. Indexed name templates preserve readable file-based names while allowing
  repeated scheduled runs and same-stem paths without importing the private name registry.
