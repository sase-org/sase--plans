---
tier: tale
title: Make SDD prompt and plan frontmatter links clickable
goal: 'Prompt and plan artifacts link to each other from GitHub while SASE continues
  to validate, search, repair, and migrate both new Markdown links and historical
  plain-path frontmatter safely.

  '
create_time: 2026-07-21 08:53:32
status: done
prompt: 202607/prompts/clickable_sdd_frontmatter_links.md
---

# Plan: Make SDD prompt and plan frontmatter links clickable

## Context and outcome

SASE currently stores `prompt` and `plan` frontmatter values as plain strings rooted according to the active SDD layout.
GitHub presents those fields in its rendered frontmatter table, but plain strings are not navigable. The same values
also serve as machine references in SDD validation, repair, prompt search, plan search, and the commit-time path that
attaches a prompt to a copied plan, so wrapping them in Markdown without teaching those consumers about the new shape
would turn a valid pair into a literal nonexistent filename.

Make Markdown links the canonical write format while retaining historical plain paths as a read-compatible format. For
the flat plans-sidecar example in the request, the stored values should be equivalent to:

| Source artifact             | Field    | Markdown label shown by GitHub | Link target resolved from the source file |
| --------------------------- | -------- | ------------------------------ | ----------------------------------------- |
| `202607/example.md`         | `prompt` | `202607/prompts/example.md`    | `prompts/example.md`                      |
| `202607/prompts/example.md` | `plan`   | `../202607/example.md`         | `../example.md`                           |

The label and target are intentionally different. GitHub resolves relative Markdown targets from the current file, so
the plan-to-prompt href must descend into `prompts/`, while the prompt-to-plan href must ascend one directory. The
reverse label includes the requested `../` prefix without duplicating the month in the actual href. Preserve the
equivalent stable labels for in-tree and local SDD layouts, and always derive hrefs from the physical source and target
files rather than from the process working directory.

## Shared link contract

Add a small SDD frontmatter-link value contract to the Rust core and expose it through the Python binding. It should:

- render one controlled inline Markdown link from a non-empty label and relative target;
- parse that canonical form into label and target components;
- classify a plain non-empty string as a legacy path instead of rejecting it; and
- reject or safely leave malformed/ambiguous Markdown-like strings for the caller to diagnose rather than guessing a
  filesystem target.

Use the core contract in Rust-backed plan reads so `PlanWire.prompt_link` remains a useful display/reference value
instead of leaking raw Markdown syntax. Add a thin Python adapter for SDD writers, validators, repair, prompt search,
and prompt-kind plan search, keeping raw frontmatter available for round trips while projecting the parsed label where
existing CLI/TUI models expect a link string.

## Canonical writes and path resolution

Refactor the paired SDD writer to construct each link from both the source artifact and its counterpart. Continue to use
the current storage-layout-aware stable reference as the visible label, add the requested `../` prefix to the prompt
file's plan label, and compute the href with a POSIX relative path from the source file's parent. Apply the same helper
to the commit hook's late prompt-link inference so copied/archive plans do not retain the old plain format.

Update link validation and collection to distinguish formats:

- canonical Markdown links resolve their parsed href relative to the file containing the frontmatter;
- legacy plain strings retain the current SDD-root/project-root fallback resolution so existing repositories continue to
  validate; and
- reverse-link checks compare resolved filesystem targets, allowing a canonical link to pair with a legacy link during a
  gradual migration.

Make `sase plan links repair` the explicit backfill path. Its dry run should report plain or stale links as repairs, and
`--write` should replace each unambiguous pair with the canonical Markdown representation while preserving unrelated
frontmatter and body content. Reads and validation must not silently rewrite historical repositories.

## Compatibility, documentation, and rollout

Keep old plain-path prompt/plan metadata searchable and valid. Ensure list/search JSON and text output remain meaningful
by exposing a parsed label or resolved reference rather than the raw Markdown wrapper where those APIs have historically
returned a path. Document the canonical Markdown examples, GitHub-relative href behavior, legacy-read compatibility, and
the one-time `sase plan links repair --write` migration workflow in the SDD guide and generated in-tree/sidecar README
templates. Update the Rust plan-frontmatter schema example for the system-managed `prompt` field so authoring guidance
matches the new representation.

The existing `sase--plans` pair named in the request should be migratable through this repair workflow without manual
editing. Do not make ordinary search, validation, initialization, or package upgrade commands bulk-edit existing plan
stores.

## Verification

Add focused Rust tests for canonical render/parse behavior, legacy values, malformed Markdown-like input, Python-binding
parity, and normalized `PlanWire.prompt_link` output. In the SASE repository, cover exact serialized YAML for flat
sidecar, in-tree `sdd/`, and local `.sase/sdd` layouts; both writer entry points; commit-time inferred links; mixed
legacy/canonical validation; broken canonical href diagnostics; list/search projections; repair dry-run and write
migration; frontmatter/body preservation; and idempotence after repair.

Use a temporary repository fixture to assert that the concrete forward target resolves to `202607/prompts/example.md`
and the reverse target resolves to `202607/example.md`. Run the relevant Rust tests, then run `just install` followed by
the required `just check` in SASE, including generated documentation/SDD validation checks affected by the README
template changes.
