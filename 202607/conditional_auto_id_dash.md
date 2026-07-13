---
create_time: 2026-07-13 07:36:05
status: wip
prompt: 202607/prompts/conditional_auto_id_dash.md
tier: tale
---
# Conditional Auto-ID Dash for Agent Names

## Goal

Move the readability rule for letter-leading auto IDs into the shared agent-name template contract. A template's `@`
marker should insert a dash immediately before the rendered token only when the token begins with a lowercase letter and
the marker is not already immediately preceded by a dash. Numeric-leading tokens should never gain an implicit dash.
Then simplify SASE's standard fork, wait, and retry templates from `.f-@`, `.w-@`, and `.r-@` to `.f@`, `.w@`, and `.r@`
so their numeric names are compact while their letter-leading names remain visually delimited.

This is a behavioral change to all agent-name templates, not a special case for the three derived-name helpers. The
auto-ID token sequence itself remains unchanged (`0` through `9`, `a` through `z`, then `00`, ...); only rendering and
matching change.

## Behavior Contract

Apply the rule literally and uniformly, including markers at the start of a template:

| Template           | Token | Concrete name       | Reason                                          |
| ------------------ | ----- | ------------------- | ----------------------------------------------- |
| `build@`           | `0`   | `build0`            | Numeric-leading token; no implicit dash         |
| `build@`           | `a`   | `build-a`           | Letter-leading token needs an implicit boundary |
| `build@`           | `0a`  | `build0a`           | The first token character controls the rule     |
| `build@`           | `a0`  | `build-a0`          | The first token character controls the rule     |
| `build-@`          | `0`   | `build-0`           | The explicit dash is preserved                  |
| `build-@`          | `a`   | `build-a`           | Do not duplicate an explicit dash               |
| `research.@.final` | `0`   | `research.0.final`  | Numeric-leading token; no implicit dash         |
| `research.@.final` | `a`   | `research.-a.final` | The marker is not immediately after a dash      |
| `@`                | `0`   | `0`                 | Bare numeric auto name                          |
| `@`                | `a`   | `-a`                | Bare marker still follows the generalized rule  |

Reverse matching must recognize exactly the canonical renderings. For example, `build@` matches `build0` as token `0`
and `build-a` as token `a`, but not `build-0` or `builda`. `build-@` continues to match `build-0` and `build-a`.
Re-rendering a recovered token should reproduce the concrete name, preventing permissive or ambiguous noncanonical
matches. Token comparison and sequence generation remain based on the undecorated token, so ordering is unchanged.

## Compatibility Boundaries

- Keep every existing one-marker template valid. In particular, explicit `-@` templates retain their current output; the
  implicit dash is idempotent with an explicit dash.
- Do not rewrite stored agent names. Historical names remain usable by exact/literal reference, while template lookup
  only treats names matching the new canonical rendering as members of that template.
- Changing the standard derived templates means new numeric names become `foo.f0`, `foo.w0`, and `foo.r0`, while the
  letter slots become `foo.f-a`, `foo.w-a`, and `foo.r-a`. Historical pre-template numeric names such as `foo.f1`
  naturally occupy the same exact namespace as the new numeric rendering. Names from the current dashed numeric form,
  such as `foo.f-0`, remain historical names and do not reserve `foo.f0`.
- Preserve runtime recognition of previously planned/current derived fork names during upgrades: accept old `.f-@`
  concrete shapes (including fan-out descendants) where SASE validates a parent-planned name, without making them the
  shape used for new allocation.
- Because `@` itself follows the generalized rule, bare auto names render as `0` ... `9`, `-a` ... `-z`, `00` ... .
  Update auto-root detection and registry compatibility helpers so dotted descendants of these rendered roots reserve
  the correct namespace. Existing bare letter names remain literal historical names; they are not canonical renderings
  of `@` after this change.

## Implementation Plan

1. **Make rendering and matching canonical in `sase-core`.** Update the pure-Rust `AgentNameTemplate` implementation to
   derive the conditional separator from the parsed prefix and the first character of the validated token. Make `render`
   insert at most one dash, and make `match_token` recover the undecorated token only from canonical numeric or
   implicit-letter forms. Keep the public Rust and PyO3 signatures and parsed template wire shape unchanged. Continue
   deriving namespace templates structurally, but render their concrete namespaces through the same conditional rule so
   exact-name and dotted-descendant reservation cannot disagree with allocation.

2. **Lock down the core contract and binding freshness.** Expand the Rust unit matrix across marker positions,
   explicit-dash forms, numeric- and letter-leading multi-character tokens, canonical rejection cases, matching, and
   namespace rendering. Mirror the important cases through the Python adapter tests. Add a semantic probe to SASE's
   `sase_core_rs` validator so a stale installed extension that exposes the right function names but implements the old
   rendering contract is rejected and rebuilt.

3. **Adopt delimiter-free standard derived templates in SASE.** Change the fork/resume, wait, and retry template
   factories and their public descriptions to return `.f@`, `.w@`, and `.r@`. Keep all allocation paths routed through
   the generic template allocator, including single allocations, repeat batches, multi-prompt planning, model/alt
   fan-out, and retry relaunches. Refactor the planned fork-name validator to recognize canonical numeric and
   letter-leading descendants through template matching while retaining a narrow fallback for historical dashed and
   legacy fork shapes.

4. **Audit auto-name and namespace consumers for rendered roots.** Update assumptions that a bare auto name is always an
   unpunctuated lowercase alphanumeric string, especially prefix extraction used while rebuilding/reading registry
   state. Keep token validation itself restricted to `0-9` and `a-z`; the dash belongs to rendering, never to the token.
   Verify that allocation gaps, latest-template lookup, grouped-template allocation, dotted descendants, and exact-name
   collisions all operate on canonical rendered names and still compare undecorated tokens.

5. **Update end-to-end expectations and user documentation.** Change fork/wait/retry expectations throughout launch
   preparation, directive extraction, repeat chains, multi-prompt reference rewriting, model/alt naming, tags, retry
   editing, mobile lifecycle flows, and environment propagation from the dashed numeric shape to the new numeric shape.
   Add rollover tests that reserve tokens `0` through `9` and prove the next derived name uses the implicit letter dash;
   add equivalent coverage for bare `@`, explicit `-@`, descendants, historical-name fallbacks, and grouped templates
   whose renderings can converge for letter tokens. Update the agent-name-template and derived-name sections of the
   xprompt documentation with the conditional rule and examples from the behavior table.

6. **Deliver the cross-repository change in dependency order.** Land and release the behavior from `sase-core` first,
   using release-plz rather than manually editing Cargo versions. Then require the first compatible `sase-core-rs`
   release from SASE and refresh the Python lockfile, so a packaged SASE installation cannot silently use the old
   renderer with the new `.f@`/`.w@`/`.r@` callers. Treat this as a behavioral compatibility change when choosing the
   core release/commit metadata and adjust SASE's compatible version window to the version release-plz produces.

## Verification

- In `sase-core`, run formatting, Clippy with warnings denied, and the complete Rust workspace test suite.
- In `sase`, run `just install` first so the editable environment uses the linked core, then run focused agent-template,
  auto-name, derived-name, launch-planning, retry/mobile, and directive tests while iterating.
- Run `just check` in SASE as the final repository-wide gate, including the Rust checks against the linked core.
- Confirm both repositories are otherwise clean and that no historical artifact migration or manual release-version edit
  was introduced.

## Acceptance Criteria

- Every `@` template follows the same first-character/preceding-dash rule for rendering, matching, namespace
  reservation, allocation, and latest-reference resolution.
- Numeric-leading IDs never receive an implicit dash; letter-leading IDs receive exactly one unless the template already
  supplies it.
- New fork, wait, and retry names use `f0`/`w0`/`r0` and roll over to `f-a`/`w-a`/`r-a`.
- Existing explicit `-@` templates and literal historical agent references continue to work, and upgrade-time planned
  fork names are recognized without restoring the old allocation shape.
- Source checkouts, editable installs, and released SASE packages cannot pair the new Python callers with an old core
  rendering implementation.
