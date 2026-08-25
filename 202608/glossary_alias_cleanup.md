---
tier: tale
title: Narrow generic glossary aliases and add sase memory
goal:
  Keep SASE glossary lookup precise by retiring the generic project and repo aliases
  while making sase memory resolve to Xprompt Memory.
size: small
proposed_by: bbugyi200.athena.0dr.w0
---

- **AGENTS:**
  - [bbugyi200.athena.0dr.w0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0dr.w0.md)
- **COMMITS:**
  - [8776211](https://github.com/sase-org/sase/commit/877621113041f1ada918f8f9b0403f388ab2675f)
    — docs(memory): drop stale project/repo glossary aliases, add sase memory alias

# Plan

## Scope and current state

Update only the declarative glossary metadata for the three affected strands, then use
the required memory-initialization workflow to refresh every managed projection. The
current catalog reports `project` as the sole configured alias of `Sase Project`, `repo`
as the sole configured alias of `Sase Repo`, and `memory file` as the sole configured
alias of `Xprompt Memory`.

A read-only `sase memory init --check --diff` already reports pre-existing generated
drift: `AGENTS.md` and the four provider instruction shims are missing managed
`sase:strands` marker pairs around their Decisions, Glossary, and Task Type rosters.
Because memory initialization is mandatory after editing canonical memory, preserve
those generator-owned marker updates in the regenerated output and distinguish them from
the requested glossary roster changes during review.

## Implementation

1. Edit `sase/memory/glossary/sase-project.md` and `sase/memory/glossary/sase-repo.md`
   so their frontmatter no longer configures the overly broad `project` and `repo`
   aliases. Remove an empty `aliases` property rather than retaining an empty list, and
   leave each strand's keyword, body, and any other metadata unchanged.
2. Edit `sase/memory/glossary/xprompt-memory.md` so its aliases retain `memory file` and
   add `sase memory`. Preserve the existing alias first, followed by the new alias, and
   leave the strand definition and remaining metadata unchanged.
3. Run `sase memory init --no-commit` to synchronize the glossary descriptor's managed
   roster, `AGENTS.md`, the provider instruction shims, and any other generator-owned
   projections without bypassing the host-owned commit workflow. Confirm the rendered
   glossary roster shows `Sase Project` and `Sase Repo` without parenthesized aliases
   and shows `Xprompt Memory (memory file, sase memory)`. Review the full diff so the
   only source edits are the three intended strands and all other edits are explainable
   generated output, including the pre-existing marker drift noted above.

## Verification

1. Inspect `sase memory web show glossary -f json` and confirm the exact configured
   alias arrays are empty for `sase-project` and `sase-repo`, and are
   `["memory file", "sase memory"]` for `xprompt-memory`.
2. Exercise glossary resolution at depth zero: the canonical selectors
   `glossary:sase-project` and `glossary:sase-repo` must still resolve; the exact new
   selector `glossary:sase memory` must resolve to `Xprompt Memory`; and the retired
   generic selectors `glossary:project` and `glossary:repo` must fail to resolve.
3. Run `sase memory init --check --diff` and require a clean result, proving the
   canonical strands and all generated memory/instruction projections agree.
4. Run `just check` as the repository-wide lint and diff-scoped test gate. If the
   workspace dependencies are stale, run `just install` and repeat `just check`.

No new product-code test is expected for this declarative catalog-only change: the real
catalog, selector, regeneration, and repository checks above directly exercise the
changed behavior. Add or adjust a focused test only if implementation uncovers a gap or
regression in those existing mechanisms.
