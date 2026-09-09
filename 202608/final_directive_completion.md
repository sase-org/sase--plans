---
tier: epic
title: Beautiful and reliable final directive completion
goal: "The %final directive is easy to discover and safely completes configured
  finalizer selectors in ACE and external LSP editors, with shared semantics, responsive
  catalogs, clear policy metadata, and polished presentation.

  "
phases:
  - id: core_completion_contract
    title: Shared finalizer completion and LSP contract
    depends_on: []
    size: medium
    description:
      "core_completion_contract: add the typed host catalog, selector-aware core
      candidates, and LSP presentation while keeping the directive hidden."
  - id: host_prompt_experience
    title: Host catalog and ACE prompt experience
    depends_on:
      - core_completion_contract
    size: medium
    description:
      "host_prompt_experience: derive cached completion rows from effective finalizer
      config and integrate responsive, polished ACE completion."
  - id: surface_parity
    title: Public exposure, parity, and release verification
    depends_on:
      - core_completion_contract
      - host_prompt_experience
    size: small
    description:
      "surface_parity: reveal %final only after both clients are complete, then lock
      behavior with parity, visual, documentation, and full-repository checks."
proposed_by: bbugyi200.athena.sase-rr.land.w1
bead_id: sase-s0
create_time: 2026-09-09 19:50:19
status: wip
---

- **PROMPT:**
  [prompts/202608/final_directive_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/final_directive_completion.md)
- **BEAD:**
  [sase-s0](https://github.com/sase-org/sase--beads/blob/main/pages/sase-s0/README.md)

# Plan: Beautiful and reliable `%final` directive completion

## Outcome

Typing `%` in the ACE prompt input or requesting completion in an LSP-enabled external
editor will advertise `%final` alongside the other public directives. After `%final:` or
inside `%final(...)`, completion will describe and insert the effective configured
finalizer instances, support both add and remove selector forms, and offer `none` only
when clearing the selection is valid.

The implementation will use one Rust-owned completion contract for filtering,
replacement, ordering, and LSP behavior. Python will remain the host authority for
merged finalizer configuration and will expose a small read-only catalog used by ACE
directly and by the Rust LSP through the existing editor helper bridge. No prompt text
will be allowed to define providers or policy.

## User experience contract

| Prompt state            | Completion behavior                                                                                                                                                                                                                                                                |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `%f` / `%final`         | Offer the canonical `%final` directive, its argument hint, and LSP snippet forms.                                                                                                                                                                                                  |
| `%final:` / `%final(`   | Offer configured instance IDs in deterministic policy order, with required/default/optional state, provider reference, and concise policy documentation.                                                                                                                           |
| `%final:c`              | Prefix-filter add selectors and replace only the active selector fragment.                                                                                                                                                                                                         |
| `%final:!c`             | Offer `!instance` removal selectors, visibly label the operation as removal, and omit required instances because removing one can never validate.                                                                                                                                  |
| `%final:n`              | Offer `none` as an explicit clear operation only when the effective configuration has no required finalizers; rank it after configured instances rather than as the default-looking first choice.                                                                                  |
| `%final(commit, !l`     | Preserve the surrounding parenthesized selector list and replace only the active comma-delimited clause.                                                                                                                                                                           |
| Catalog loading/failure | Never block typing. ACE shows a non-selectable loading or unavailable row and refreshes an open menu when the worker finishes; LSP uses a bounded cached bridge request, falls back to a stale snapshot, and otherwise returns an empty list while manual entry remains available. |

ACE rows should form a compact, aligned grid: selector, policy state (`required`,
`default`, `optional`, or `clear`), and provider. Add/remove/clear must be communicated
in text or glyph plus text, not color alone. Required and default states should be
immediately scannable, `none` should read as a deliberate destructive clear operation,
long values should ellipsize cleanly, and the layout must remain legible in light, dark,
and narrow terminal presentations.

External editors should receive the same information through standard LSP fields: stable
item kinds and ordering, `labelDetails` for policy state/provider, Markdown
documentation for dependencies and retry policy, exact UTF-16 text edits, and snippet
forms for both the single-selector and parenthesized multi-selector syntax.

## Architecture and design decisions

- Keep `sase-core` authoritative for directive metadata, selector token classification,
  valid-operation filtering, shared ordering, replacement ranges, and
  completion-candidate semantics. This is behavior every frontend must share.
- Keep the `sase` host authoritative for `load_finalizer_config()` and provenance. Build
  catalog entries only from effective trusted configuration; completion never imports or
  executes a finalizer provider.
- Add a dedicated versioned `finalizer-catalog` editor-helper contract. Do not attach
  finalizers to `agent-catalog`: that path may scan agents and bead stores and would
  make the first `%final` completion unnecessarily slow. Do not materialize a one-time
  LSP launch file: it would become stale after watched config changes.
- Make catalog fields additive and mixed-version safe. At minimum each configured row
  carries its instance ID, provider reference, required/default flags, dependency IDs,
  attempt policy, and human-readable documentation. Invalid catalog envelopes or helper
  failures degrade without tracebacks or invented rows.
- Preserve the existing hidden-name guards through the first two phases. Remove them
  only in the final integration phase, so no partially wired user-facing behavior is
  exposed and no temporary feature flag is needed.
- Do not edit `sase/memory/*.md` or generated instruction shims as part of this plan.
  Update the ordinary xprompt documentation that currently describes `%final` as hidden;
  canonical memory changes require separate explicit user permission.

## Phase 1: Shared finalizer completion and LSP contract

Work in the linked `sase-core` repository and retain the `%final` directive-name
visibility guard until Phase 3.

1. Extend the editor wire contract with versioned finalizer-catalog request/response
   records and structured finalizer entry metadata. Add the method to
   `HelperHostBridge`, `DynHelperHostBridge`, command-backed and static bridges, and
   re-export/binding surfaces without breaking older helper responses.
2. Add an independently cached finalizer catalog to the xprompt LSP. Use the established
   blocking-worker timeout, short TTL, stale-on-error fallback, warn-once behavior, and
   watched-config invalidation. Fetch it only for a finalizer argument context.
3. Refine the Rust finalizer candidate builder so configured instances precede the clear
   action; `!` produces visibly prefixed removal labels/insertions; required instances
   are absent from remove completion; and `none` is absent when any required instance
   makes clear invalid. Preserve active-clause replacement ranges and case-insensitive
   prefix matching for colon, parenthesized, and repeated forms.
4. Give finalizer values a dedicated LSP conversion path rather than routing them
   through the agent renderer. Emit meaningful item kinds, stable `sortText`,
   operation-aware labels, policy/provider `labelDetails`, Markdown documentation, and
   UTF-16-safe edits. Add `%final:${1:instance}` and a parenthesized multi-selector
   snippet, but leave directive-name discovery hidden until Phase 3.
5. Cover the core candidate matrix, required/default/optional ordering, remove and clear
   legality, clause-local edits, catalog deserialization, helper timeout/stale fallback,
   config-watch invalidation, and complete LSP item metadata. Update every static bridge
   fixture required by the additive method and run the linked repository's `just check`.

## Phase 2: Host catalog and ACE prompt experience

Work in the main `sase` repository against the Phase 1 binding, while retaining the
Python directive-name visibility filters until Phase 3.

1. Add a side-effect-free finalizer completion catalog builder next to the finalizer
   configuration layer. Replay the effective config once per config token and emit
   deterministic rows in policy order: required entries, remaining defaults, then
   optional entries alphabetically. Build concise documentation from provider, `after`,
   retry policy, and provenance without loading provider code.
2. Expose that builder through `sase editor helper-bridge finalizer-catalog` with a
   strict schema-version check, compact JSON response, excellent internal CLI help, and
   fail-closed status/message envelopes for malformed configuration. Add parser,
   handler, and mixed-version contract tests.
3. Add a finalizer inventory state to the prompt text area using the established
   off-thread worker/result pattern. Warm it when prompt panes mount or restore, never
   perform config I/O in key handlers or render methods, coalesce duplicate loads, and
   refresh the currently open `%final` menu only if its cursor context is still current.
   Loading and unavailable states must use non-selectable placeholder rows, while
   catalog failure must never prevent manually typed selectors.
4. Pass structured finalizer inventory into the shared Rust candidate call and map the
   returned metadata into a dedicated ACE finalizer completion record. Keep name,
   argument, refresh, Tab, Enter, and `Ctrl+L` paths on the same builder so auto-open,
   narrowing, widening, unique acceptance, and mid-clause replacement cannot drift.
5. Add the aligned finalizer row renderer and its width calculation. Use accessible
   state labels, restrained theme-aware colors, a clear removal treatment for
   `!instance`, and an unmistakable but non-alarming clear row for `none`. Add unit
   tests for narrow widths and metadata fallback plus a deterministic PNG fixture
   containing required, default, optional, remove, and clear states.
6. Cover cold/warm/error catalog behavior, mount-time warming, stale-result guards,
   required-selector suppression, operation-aware insertions, colon and parenthesized
   interactions, and the absence of synchronous config/provider work on the typing path.
   Run `just install`, targeted ACE/directive tests, `just test-visual`, and
   `just check`.

## Phase 3: Public exposure, parity, and release verification

Integrate the completed core and host work, then expose the feature atomically.

1. Remove the temporary `%final` hidden-name sets from the Rust directive-name builder,
   the ACE adapter, and the generic directive candidate catalog. Replace the former
   negative tests with assertions that the canonical name, description, hint, and
   snippets are discoverable while retired directives remain absent.
2. Expand the ACE-versus-LSP parity harness and its fake helper to serve the finalizer
   catalog. Add parity cases for directive names, configured add rows, `!` removal rows,
   required/clear suppression, documentation/detail, helper failure, repeated
   directives, parenthesized clauses, and Unicode-adjacent replacement ranges.
3. Update `docs/xprompt.md` so the supported-directive table and completion matrix
   describe the configured instance inventory, add/remove/clear behavior, policy labels,
   and graceful degradation instead of saying `%final` is hidden. Cross-check
   `docs/configuration.md` examples without changing finalizer selector semantics.
4. Inspect and accept the intentional finalizer completion PNG golden only after
   verifying the actual/expected/diff artifacts at both normal and narrow widths. Keep
   the snapshot deterministic and independent of the user's installed providers or
   config by using fixed catalog fixtures.
5. Rebuild/install the local Rust binding and LSP, run `sase-core`'s `just check`, then
   run `just install` and `just check` in `sase`. Because this is an epic combined tree
   spanning the shared backend and user-facing TUI/LSP surfaces, finish with the main
   repository's `just check-full` through `/sase_monitor`, using `TESTING`/`TESTED` and
   a follow-up action that diagnoses any failure before landing.

## Acceptance criteria

- `%final` is discoverable in ACE, LSP clients with and without snippet support, and the
  generic directive catalog; all use the canonical shared description and syntax hint.
- ACE and LSP return the same configured selector values, ordering, documentation, and
  fragment replacements for equivalent contexts.
- Completion never suggests the provably invalid operations `!required` or `none` when
  required finalizers exist.
- Catalog loading does not block the Textual event loop or message pump, execute
  provider code, prompt interactively, or turn a helper/config failure into a typing
  failure.
- Required/default/optional and add/remove/clear meanings are understandable without
  relying on color, and the ACE menu has reviewed visual coverage.
- Parser and launch semantics remain unchanged: manual selectors still replay left to
  right, dependencies are still resolved by the finalizer planner, and the normal launch
  validator remains the final authority.
- Both repositories pass their required checks, the main repository passes visual
  snapshots and monitored `just check-full`, and no memory or generated instruction
  files are modified.
