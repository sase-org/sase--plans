---
tier: tale
title: Finish and land bead merge replay stability
goal: Published SASE installs use the replay-stable bead merge implementation, and
  epic sase-9x is fully validated and landed.
create_time: 2026-07-27 09:20:58
status: done
---

- **PROMPT:** [202607/prompts/finish_bead_merge_replay_stability.md](prompts/finish_bead_merge_replay_stability.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-9x.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-9x.land/README.md)
  - [bbugyi200.athena.sase-9x.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-9x.land.md#member-code)
- **COMMITS:**
  - [fa07151](https://github.com/sase-org/sase/commit/fa07151cf1414f652b836ac1511f302e8dafac2d) — test: track sase-core-rs 0.11.2 minimum (sase-9x)

# Finish and land bead merge replay stability

## Goal

Finish epic bead `sase-9x` after a land-agent audit found that its Rust implementation was released after the epic
started but the Python repository still accepts and locks the preceding release. Integrate that published release,
revalidate the completed epic across both repositories, and perform the required closeout in the correct order.

## Verified starting point

- `sase bead show sase-9x` reports six closed child phases, `sase-9x.1` through `sase-9x.6`, and links
  `$SASE_SDD_PLANS_DIR/202607/bead_merge_replay_stability.md`.
- None of the child beads has a separate `NOTES` field. Their descriptions are the acceptance claims.
- Rust commit `4376ec2d0adc16f5d6883010991d43b6fc05c372` implements `sase-9x.1`: content-derived disambiguation for
  newly minted event IDs, preservation of recorded IDs during merge, order-preserving base containment, deterministic
  union, corruption rejection, algebraic merge tests, sequential replay coverage, and Python binding coverage.
- Main-repository commits `59930584`, `19bb1adc`, `0b51af99`, `87dd076f`, and `7538b939` implement phases `sase-9x.4`,
  `.2`, `.3`, `.5`, and `.6`, respectively. The current source contains the claimed UTF-8 encoding, untouched-stream
  preservation, recovery-ref safety, bounded integration retry, store-lock serialization, deep replay, rollback,
  convergence, and bounded recurring-failure diagnostic coverage.
- The commits made in the main repository since the epic began were reviewed. Most are unrelated UI, xprompt, agent
  status, and test-isolation changes. Commit `672ecbb4` adds bead list output formats and shares query rendering, but
  does not replace or duplicate the existing post-command sync diagnostics path.
- The linked Rust repository released the phase-1 implementation as `sase-core-rs` `0.11.2` in commit `906d33ec`. The
  main repository still declares `sase-core-rs>=0.11.1,<0.12.0`, and `uv.lock` still resolves `0.11.1`. This violates
  the epic plan's explicit requirement to land the corresponding version requirement and would leave published
  installations without replay-stable merge behavior even though editable local tests use the linked checkout.

## Implementation

1. Update the main repository's minimum `sase-core-rs` requirement from `0.11.1` to `0.11.2` while preserving the
   existing `<0.12.0` compatibility cap.
2. Regenerate `uv.lock` using the repository's normal uv/Justfile workflow so the package requirement and resolved
   `sase-core-rs` artifact are both `0.11.2`. Do not hand-edit generated artifact hashes.
3. Inspect the resulting diff and confirm it contains only the intended dependency and lockfile integration. Preserve
   unrelated user changes if any appear.

## Validation

1. Run `just install` before repository checks, as required by the project instructions.
2. Run focused replay/encoding/safety/retry/health tests covering the six epic phases, including the Rust
   `bead_event_parity` coverage in the linked `sase-core` checkout and the Python managed-sync conflict regressions.
3. Run `just check` in the main repository. If a failure is pre-existing or environmental, isolate it with a focused
   rerun and report concrete evidence; otherwise fix regressions introduced by this integration.
4. Recheck the main repository history since the first epic implementation commit and confirm no later commit now needs
   another integration with the replay-stability feature.

## Final landing phase

Perform this phase only after the implementation and validation above succeed:

1. Close the epic with `sase bead close sase-9x`.
2. After the close succeeds, run `just symvision` if that recipe is available.
3. If Symvision reports expired `sase-9x` epic-symbol whitelist entries or code made unused by their removal, first read
   the required Symvision long-term memory through the `sase_memory_read` skill, then remove the stale entries and
   unused code, rerun Symvision, and rerun `just check` for any main-repository file changes.
4. Set `status: done` in the YAML frontmatter of `$SASE_SDD_PLANS_DIR/202607/bead_merge_replay_stability.md`. Use the
   audited plans-sidecar checkout opened through the `sase_repo` skill and preserve all other plan content.
5. Confirm `sase bead show sase-9x` reports the epic closed, the plan frontmatter reports `status: done`, both
   repositories are clean except for the intended uncommitted landing edits, and all required validation commands
   passed.
