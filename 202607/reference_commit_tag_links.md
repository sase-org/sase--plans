---
tier: tale
title: Reference-style links in SASE commit tags
goal: 'SASE commit metadata can carry Markdown reference links safely, and GitHub-backed
  SASE_PLAN tags link to the committed plan in the owning project''s plans sidecar.

  '
create_time: 2026-07-16 13:29:48
status: done
prompt: 202607/prompts/reference_commit_tag_links.md
---

# Plan: Reference-style Links in SASE Commit Tags

## Outcome

SASE-authored commit messages will support reference-style Markdown links as tag values while retaining the current
plain-value and legacy-tag behavior. A commit associated with a plan in a GitHub-backed plans sidecar will render the
portable plan path as the link label and place the reference definition after the tag block:

```text
SASE_MACHINE=athena
SASE_PLAN=[202607/amd_agents_template.md][1]

[1]: https://github.com/sase-org/sase--plans/blob/main/202607/amd_agents_template.md
```

The link must remain valid after SASE adds or replaces runtime, PR, type, bug, or other tags later in the commit
workflow. Readers must continue to see the logical value `202607/amd_agents_template.md`, and reference definitions must
not leak into commit bodies, ChangeSpec descriptions, or VCS-log body output.

## Current Context

- `src/sase/workflows/commit/runtime_tags.py` owns canonical tag names, parsing, and message updates. Its current parser
  assumes that the tag block is the final nonblank content, so a definition after the tags would make later runtime
  updates miss the block and create a second footer.
- `handle_sase_plan()` in `src/sase/workflows/commit/commit_hooks.py` resolves the plan's owning `SddStore`, computes
  the stable store-relative path through `plan_paths.py`, adds `PLAN`, marks the plan done, and commits the plan in its
  owning store. Runtime and PR tags are applied later, so their rewrites must preserve a plan link and its definition.
- A plan may belong to the primary code repository, a local/separate SDD store, the current project's flat plans
  sidecar, or a host project's plans sidecar while the code commit is made in a linked repository. The link target must
  come from the plan-owning repository, not merely from the commit's current working directory.
- `src/sase/vcs_provider/config.py` and `src/sase/vcs_log/tags.py` duplicate trailing-tag scans for PR inheritance and
  display. Attribution and revert discovery call the runtime parser. All of these paths need one footer grammar once
  definitions can follow tags.
- `src/sase/_git_remote.py` already parses SSH, scp-style, and URL-style hosted Git remotes. Resolved sidecar stores
  carry their provider and plans remote, but directly detected owning stores currently discard that remote metadata.
- The sibling Rust core does not yet model SASE commit footers. Because parsing and rewriting this metadata must agree
  across CLI, TUI, automation, and future frontends, the pure footer contract belongs in `sase-core`; Python should
  provide thin workflow and presentation adapters.

## Product Contract

### Footer shape

- A SASE footer consists of the existing ordered `KEY=value` tag block, optionally followed by one blank line and a
  contiguous block of Markdown reference definitions used by tag values.
- Plain tag values remain byte-compatible. Legacy unprefixed tags remain readable, while newly rendered keys continue to
  use `SASE_`.
- The first supported linked value is the explicit reference form `[label][id]`; SASE-generated references use stable,
  collision-free numeric identifiers. Parsing should preserve valid existing identifiers rather than renumbering a
  footer merely because another tag was updated.
- Footer recognition is conservative: reference definitions are footer metadata only when they follow a recognized tag
  block in the specified shape. Ordinary body links, tag-like text in the middle of a message, and trailing prose remain
  body content.
- Rewriting is idempotent. It preserves definitions used by retained tags, removes definitions owned only by a removed
  or replaced tag, and never leaves an unresolved reference or duplicates the tag block. Multiple linked tag values and
  shared targets must behave deterministically.
- Structured readers expose both the logical label and optional destination. Existing string-oriented consumers use the
  logical label, while writers and inheritance paths retain the destination so a later rewrite does not break the link.

### Plan links

- When the final plan lives in a GitHub-backed remote SDD/plans repository, render `SASE_PLAN` as a linked tag value.
  The label is exactly the existing portable/store-relative `plan_ref`, preserving flat-sidecar paths such as
  `202607/name.md` and legacy separate-store paths such as `plans/202607/name.md`.
- Build the browser URL from the plan-owning store's hosted remote identity and the branch that receives the plan
  commit. Resolve the checked-out/default branch locally (normally `main`) without a network lookup, normalize SSH and
  HTTPS clone transports through the existing remote parser, and URL-encode the branch and path safely.
- Support `github.com` and provider-declared GitHub Enterprise hosts where the repository identity is unambiguous. Never
  guess a GitHub blob URL for local, in-tree, unknown-provider, malformed, or non-GitHub remotes; those cases keep the
  current plain path tag.
- Preserve current plan copying, validation, status transition, targeted staging, sidecar commit, and push behavior.
  Link generation is metadata-only and must not introduce a second plan-store commit or require the asynchronous push to
  finish before the code commit is created.

## Implementation Sequence

### 1. Shared Commit-footer Link Contract

Implement a pure commit-footer model in the sibling Rust core and expose it through the existing PyO3 boundary.

- Model the message body, ordered raw/canonical tags, linked tag label/destination metadata, and attached reference
  definitions without coupling the core to GitHub or filesystem state.
- Add parse and update/render operations implementing the footer shape and compatibility rules above. The updater must
  accept plain values and typed linked values, canonicalize owned keys, preserve stable ordering and later-duplicate
  semantics, allocate identifiers without collisions, and serialize definitions after all tags with exactly one blank
  separator.
- Expose stable binding payloads suitable for Python adapters. Update binding availability/version health checks and
  compatibility metadata as needed so a SASE runtime cannot silently use a core build that lacks the new contract.
- Cover the Rust API and binding conversion with focused tests for plain and prefixed tags, legacy reads, one and many
  links, retained/removal behavior, identifier collisions, duplicate tags, repeated updates, trailing whitespace, and
  body content that resembles tags or Markdown definitions.

This workstream owns no Git remote or SDD policy. Its output is a transport-neutral footer primitive that other SASE
surfaces can reuse.

### 2. GitHub Plan Links and Python Integration

Adopt the core contract throughout the Python commit and display paths, then use it for plan attribution.

- Add a thin typed Python facade over the new core bindings. Refactor `runtime_tags.py` to keep runtime identity and
  sanitization policy locally while delegating footer parsing/updating. Preserve its public canonical-key behavior for
  existing attribution, attachment discovery, and revert callers.
- Route PR tag extraction/stripping and VCS-log tag/body views through the same footer representation. Parent-tag
  inheritance must carry linked values with their destinations; stripping removes both the tags and their attached
  definitions; tag displays and JSON use the readable link label rather than unresolved `[label][id]` markup.
- Extend hosted-remote/plan-path helpers to derive an optional GitHub blob target from a resolved plan-owning store.
  Retain remote metadata when `_store_owning_plan_path()` detects an external plans clone so commits made in linked
  repositories still point at the host project's plan. Prefer the authoritative resolved `SddStore` metadata and use the
  clone's matching origin only when direct ownership detection is required.
- Have `handle_sase_plan()` pass a typed linked `PLAN` value only after it has the final destination path and a valid
  GitHub target. Subsequent PR/runtime updates must preserve the definition and keep it below the complete tag block.
  All unsupported storage/provider cases continue to pass the existing plain `plan_ref`.
- Make PR-body composition footer-aware: insert the agent/model information separator into the body before the
  structured SASE footer, then serialize the tags and definitions as the terminal block. Ensure ChangeSpec description
  cleanup and full VCS-log rendering do not expose the reference definitions as prose.

## Tests and Acceptance Criteria

Add cross-layer regression coverage rather than updating only exact strings in plan-hook tests:

- Footer/facade tests prove that adding runtime tags to an already linked `PLAN` produces one ordered tag block plus one
  definition block; updating/removing tags is idempotent and cleans only owned definitions; legacy and plain messages
  retain their behavior.
- Plan-path and commit-hook tests cover a flat GitHub plans sidecar, the example SSH remote and `main` URL, a legacy
  separate GitHub store, and a plan-owning sidecar used while committing in a linked code repository.
- Negative plan tests cover in-tree/local storage, a non-GitHub hosted remote, and missing or malformed remote metadata;
  each must degrade to the existing plain `SASE_PLAN` value rather than fail the commit. Separate branch/path tests
  cover a detached clone that can resolve its origin default, an unresolvable branch that falls back safely, and paths
  requiring URL encoding.
- Workflow tests exercise the real ordering `handle_sase_plan` -> PR/config tags -> runtime tags -> PR body and assert
  that the final commit/PR message matches the requested terminal-footer shape with no duplicate tags or dangling
  reference.
- PR tag extraction/inheritance and `strip_pr_tags()` tests cover linked values and attached definitions.
- VCS-log helper/render tests verify that tags still opt in as before, the displayed plan value is the path label, the
  definition is excluded from the rendered body, and non-footer Markdown references remain untouched.
- Existing agent/machine attribution, image-attachment discovery, revert discovery, commit tracking, and plain plan path
  suites remain green.

Verification after the integrated implementation:

```bash
just install
just rust-check
pytest tests/test_commit_runtime_tags.py tests/test_commit_hooks_artifacts.py tests/workflows/test_plan_paths.py
pytest tests/test_pr_tags.py tests/test_strip_pr_tags.py tests/test_vcs_log_tags.py tests/test_vcs_log_render.py
just check
```

The implementation is complete when a real SASE commit for this project produces a clickable `SASE_PLAN` reference to
`sase-org/sase--plans`, later tag writers preserve it, plain/non-GitHub configurations remain compatible, both core and
Python validation pass, and neither repository has unintended changes.

## Risks and Guardrails

- Footer parsing is intentionally structural. Do not treat every trailing Markdown definition as SASE metadata or strip
  user-authored body content.
- A reference identifier is message-local. Never persist a global counter or assume `[1]` is unused without parsing the
  existing footer.
- Remote URLs may contain credentials, ports, or unsupported transports. Generate only sanitized HTTPS browser URLs from
  parsed host/repository components; never echo credentials into commit messages.
- Keep Git and provider I/O in Python. The Rust workstream owns only deterministic parse/update behavior, consistent
  with the backend boundary and reusable from non-Python frontends.
- Do not broaden this change into clickable Rich widgets or a VCS-log wire-schema migration. The structured footer
  retains destinations for future presentation work, but this rollout only requires correct Markdown commit/PR rendering
  and clean existing displays.
