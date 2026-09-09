---
status: done
tier: epic
title: Recover and land the sase-y5 usage-context surface
parent_bead: sase-y5
goal:
  Re-implement the lost sase-y5.10 capacity-hint wiring from the recovered artifact and
  clear the sase-y5 release residue so the subscription-capacity epic can close.
phases:
  - id: usage-context-rework
    title: Re-implement scoped capacity hints and usage attention
    size: medium
    depends_on: []
    description:
      "usage-context-rework: Restore the recovered usage-context modules, tests, and PNG
      goldens from the recovery artifact, adapt them to the post-flag-removal tree, and
      re-implement the lost picker, alias-detail, and indicator wiring they specify."
  - id: usage-release-residue
    title: Fix the verbose reset label, baseline the pager flake, and finish docs
    size: small
    depends_on: []
    description:
      "usage-release-residue: Fix the verbose reset label 0s-ago defect in shared usage
      presentation, add the sase-yq flake-baseline entry so the selection-health gate is
      green, and add the missing subscription-usage section to docs/agent_providers.md."
proposed_by: bbugyi200.athena.sase-y5.land
bead_id: sase-y5.12
create_time: 2026-09-09 19:52:54
---

- **PROMPT:**
  [prompts/202609/usage_context_recovery.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/usage_context_recovery.md)
- **PARENT:**
  [202609/subscription_capacity.md](https://github.com/sase-org/sase--plans/blob/main/202609/subscription_capacity.md)
- **BEAD:**
  [sase-y5.12](https://github.com/sase-org/sase--beads/blob/main/pages/sase-y5/sase-y5.12.md)

# Recover and land the sase-y5 usage-context surface

## Context

Epic sase-y5 (subscription capacity for Claude, Codex, and Grok) is fully landed except
its `usage-context` surface. Phase bead sase-y5.10 closed with a verification note, but
its commit never landed: the workflow's `sase stitch create` failed in the before-commit
hook (disk exhaustion, see task sase-yo and the land-agent note on bead sase-y5), and
the worker's edits to existing files were later lost to workspace resets. Only its new
files survived, recovered as artifact **`file:explicit:762a0fcfad720e2b70a9331b`** — a
`git apply` patch against commit `1cad7ed16` containing:

- `src/sase/llm_provider/usage/hints.py` (462 lines): pure capacity-hint helpers for
  picker rows, alias detail, and attention classification.
- `src/sase/llm_provider/usage/peek.py` (147 lines): cached usage peek + attention
  eligibility for the top-bar indicator (`usage_attention_enabled()`,
  `cached_usage_peek()`), modeled on the `provider_disable_peek.py` precedent.
- Six test files: `tests/llm_provider/test_usage_hints.py`,
  `tests/llm_provider/test_usage_peek.py`,
  `tests/llm_provider/test_usage_store_bindings.py`,
  `tests/test_model_picker_usage_hints.py`,
  `tests/test_models_panel_usage_alias_hints.py`, and two PNG suites
  `tests/ace/tui/visual/test_ace_png_snapshots_model_picker_usage.py` and
  `tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator.py`.
- Two PNG goldens: `model_picker_usage_hints_120x40.png` and
  `top_bar_usage_attention_80x24.png` under `tests/ace/tui/visual/snapshots/png/`.

The wiring the recovered tests exercise (edits to existing modules) is what was lost and
must be re-implemented. The authoritative behavior contract is the `usage-context` phase
of the epic plan (`sase artifact read plan:202609/subscription_capacity.md`), plus the
recovered tests themselves.

Two tree drifts happened after the recovered files were written; both phases must
reconcile against them:

1. Commit `1cad7ed16` removed the `provider_usage_metrics` beta flag (usage tracking is
   default-on; flag bead sase-yc closed). The recovered tests still reference
   `override_flags(provider_usage_metrics=...)` in 8 places and will not import.
2. The usage store was refactored (`d7e6ca1ff`); `src/sase/llm_provider/usage/store.py`
   does not currently re-export `provider_usage_window_applies`, which `usage/hints.py`
   imports. The binding exists in `sase_core_rs`.

The two phases touch disjoint files and can run in parallel.

Out of scope for this plan: closing epic sase-y5, its symvision/epic-symbol retirement,
and marking its plan file done — the sase-y5 landing resumes after this plan lands. Do
not edit SASE memory files. Phase workers must not create beads; record discovered work
as `PROPOSED FOLLOW-UP:` notes on the phase bead.

## Phase `usage-context-rework` (medium)

Read the `tui_perf.md` reference memory before touching TUI code, and read the epic plan
artifact above for the full usage-context invariants.

1. Resolve the recovery artifact to a path with
   `sase artifact path file:explicit:762a0fcfad720e2b70a9331b` (use
   `sase artifact read file:explicit:762a0fcfad720e2b70a9331b "<reason>"` for the
   audited read) and apply it with `git apply` onto the current tree. It applied cleanly
   to `1cad7ed16`; resolve any drift by hand.
2. Adapt the recovered files to the default-on tree: remove the 8
   `override_flags`/`provider_usage_metrics` references (the flag and its `FeatureFlag`
   member no longer exist). Where a recovered test exercised the flag-off path, convert
   it to the durable config opt-out (`llm_provider.usage_metrics.enabled: false`), which
   `peek.usage_attention_enabled()` already reads via `usage/config.py`.
3. Re-export `provider_usage_window_applies` (from `sase_core_rs`) in
   `src/sase/llm_provider/usage/store.py`; the recovered `test_usage_store_bindings.py`
   asserts the expected export surface.
4. Re-implement the lost wiring; the recovered tests are the spec:
   - `src/sase/ace/tui/modals/model_picker_rows.py` and `model_picker_options.py`
     (`rows_to_options`): scoped capacity hints on picker rows. Shared/account-wide
     rejection renders on the provider header, not on every model row; a low
     model-specific window renders only on that model; unknown scope stays on the
     header; hints coexist with existing model advisories; capacity never reorders,
     disables, or re-routes models; an alias row shows its member constraint without a
     combined percentage.
   - `src/sase/ace/tui/modals/models_panel_rendering_descriptions.py`: alias detail
     lists member capacity without a combined percentage, and alias capacity inspection
     must not advance round-robin selector state or change routing eligibility.
   - `src/sase/ace/tui/widgets/provider_disables_indicator.py`, consuming
     `usage/peek.py`: quiet usage attention merged into the existing disable/priority
     indicator. Ranking `rejected > very_low > low > collection_problem/unknown`
     (`hints.py` encodes it); show the highest-attention item plus a count with stable
     provider-ID tie-breaking; fresh healthy state does not alert; existing priority
     pills never suppress rejection/usage attention; narrow widths preserve routing text
     plus an attention count; a usage click opens the provider's Providers · Usage view
     via the existing `action_open_provider_usage`; all details reachable by keyboard.
     Peek/store reads happen off the event loop per `tui_perf.md`.
5. PNG goldens: the two recovered goldens are the intended rendering (opus picker hint
   at 120x40; top-bar usage attention count at 80x24). Run `just test-visual` for the
   two recovered suites; if the renderer drifted since they were captured, regenerate
   with `--sase-update-visual-snapshots` only after inspecting actual/expected/diff
   artifacts and confirming the differences are honest drift.
6. Symvision: new public helpers need real non-test consumers; keep single-file helpers
   private (the sase-y5.9 worker already kept `usage_meter`/`provider_attention_style`
   private for this reason). Do not add `--epic-symbol` whitelist entries keyed to
   closed beads.
7. Verify: `just install` (fresh workspace), targeted pytest on all six recovered test
   files, `just test-visual` for the two PNG suites, then `just check`.

## Phase `usage-release-residue` (small)

1. **Verbose reset label defect** (found by the sase-y5.9 worker, note #2 on bead
   sase-y5.9): `src/sase/llm_provider/usage/presentation.py` —
   `reset_label(..., verbose=True)` appends `timestamp_label(resets_at, now)`, and
   `timestamp_label` clamps relative age with `max(now - timestamp, 0.0)`, so any FUTURE
   reset renders a nonsensical `(0s ago)` suffix, e.g.
   `in 1h (2027-01-15 04:00:00 EST (0s ago))`. Change the rendering so a future
   timestamp shows the absolute time with no relative suffix (e.g.
   `in 1h (2027-01-15 04:00:00 EST)`) while past timestamps keep the `(<age> ago)`
   suffix. This presentation is shared by the CLI and ACE; add regression coverage in
   `tests/llm_provider/test_usage_presentation.py` for a future verbose reset and an
   unchanged past `observed_at` label.
2. **Flake-baseline entry**: `just selection-health --fail-on-new-flake` (the
   `just check-full` final gate) is currently red with exactly one baseline-exceeding
   node. Append to `tests/reproducible_flake_baseline.txt`, following the file's
   existing live-node convention (a `# <bead-id>` comment line, then the bare node ID):

   ```
   # sase-yq
   tests/pager/test_rendered_link_contract.py::test_kitchen_follow_copy_edit_and_media_for_each_supported_action
   ```

   Task bead sase-yq holds the evidence (full-lane failures on 2026-09-08 across
   multiple heads; passes serially at `1cad7ed16`). Verify the gate exits 0 afterwards.

3. **Docs**: `docs/agent_providers.md` never got the subscription-usage content the
   epic's doc-reconciliation required (llms.md, ace.md, configuration.md, and plugins.md
   were updated). Add a short section covering: included-allowance inspection via
   `sase usage list` / `sase usage refresh`, the ACE Providers · Usage view, that
   observations are dated best-effort readings from each provider's own CLI (not
   guarantees), and the `llm_provider.usage_metrics` opt-out. Keep terminology
   consistent with `docs/llms.md`.
4. Verify: `just check` (includes the selection-health gate via lint when applicable;
   run `just selection-health --fail-on-new-flake` explicitly to confirm the baseline
   entry works).
