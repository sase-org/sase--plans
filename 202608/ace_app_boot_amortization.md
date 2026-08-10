---
tier: tale
title: Amortize ACE app startup across tests
goal:
  ACE TUI tests spend at least 50% fewer measured seconds entering apps while retaining
  every node, assertion, isolation guarantee, and contention check.
size: medium
proposed_by: bbugyi200.athena.sase-ib.3
bead: sase-ib.3
create_time: 2026-08-09 13:47:07
status: done
---

- **PROMPT:**
  [prompts/202608/ace_app_boot_amortization.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/ace_app_boot_amortization.md)
- **PARENT:**
  [202608/fast_test_suite_1.md](https://github.com/sase-org/sase--plans/blob/main/202608/fast_test_suite_1.md)
- **BEAD:**
  [sase-ib.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ib/sase-ib.3.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ib.3](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ib.3.md)
- **COMMITS:**
  - [44bf25f](https://github.com/sase-org/sase/commit/44bf25f84fecc2ee32c0c6fc8cf58a642f0f632b)
    — perf(ace): amortize ACE test app startup

# Plan: Amortize ACE app startup across tests

## Goal

Reduce the `tests/ace/tui` app-start seconds by at least 50% without deleting, skipping,
re-marking, or weakening any test. Make each fast-policy `AcePage` boot cheaper, add an
opt-in and auditable way for related pytest nodes to reuse one running app, migrate the
files with the largest measured full-app startup cost, and keep a CI lane that replays
every migrated node with one app per test.

The committed epic baseline recorded 2,148 Textual app boots taking 422s and 506
`AcePage` entries taking 390s, so full `AcePage` starts dominate the seconds even though
small modal hosts dominate the raw count. On the current tree, a serial cost run of
`tests/ace/tui/widgets/test_vim_normal_key_containment.py` records 45 nodes in 68.060s,
with 45 `AcePage` entries taking 32.027s and 45 `App.run_test` entries taking 28.967s. A
prototype that reuses a copied compiled stylesheet snapshot kept all 45 nodes green and
reduced those causes to 26.201s and 23.404s respectively. Use the cost harness, not
static call counts, to decide the final migration set.

## Implementation

1. Make fast-policy boots reuse compiled stylesheet work safely within a pytest worker.
   After the first successful `AcePage(startup_policy="fast")` boot, retain an immutable
   compiled stylesheet snapshot keyed by every input that can change its meaning: app
   class, Textual version, CSS variables/theme, app CSS text, and CSS-path identity and
   contents. Seed later apps with a fresh stylesheet container whose source/rule
   collections are copied so an app-level reparse or mutation cannot contaminate the
   snapshot or another app. Invalidate or bypass the cache when any key input differs,
   and keep `startup_policy="real"` completely uncached. Confirm the harness sets the
   running app's animation level to `none`; do so only for the fast policy if it does
   not already, leaving tests of real animation on the real policy. Add focused tests
   for cache hits, invalidation, mutable-state separation, fast/real policy behavior,
   and caller-supplied monkeypatch preservation.

2. Add a supported `AcePageGroup`-style async test helper under `sase.ace.testing`. The
   group owns one `AcePage`/`run_test` lifetime and exposes a per-test async checkout
   context. Require marked modules to use a module-scoped pytest-asyncio loop, reject
   overlapping checkouts, and make sharing explicit per file rather than a global
   `AcePage` behavior change. Between checkouts:
   - cancel and drain test-created workers/tasks that are owned by the app;
   - close every modal above the baseline screen;
   - remove the prompt bar without writing cancelled prompt history;
   - clear Textual notifications and toast state;
   - restore the baseline tab, subtab, query, selection, marked rows, and focus;
   - recompose the main app surface so lazily mounted pane/widget state is not reused;
   - settle through the event-driven ACE barrier; and
   - compare an explicit isolation snapshot with the baseline, failing with a field-by-
     field leak report instead of silently carrying state into the next test.

   Cover focus, modal stack, prompt state, selection/marks/query, notifications, widget
   recomposition, teardown after a failed checkout, and no cancelled-history side
   effect. Permit a file-specific async reset hook before the final audit when a group
   owns additional well-defined state; keep the hook visible beside that file's fixture.

3. Give the group an isolation mode, selected by one documented environment variable,
   that preserves the same checkout API but creates and closes a fresh `AcePage` for
   every checkout. Add a dedicated governed `tools/run_pytest`/Justfile lane for the
   migrated-file manifest. The lane must use the normal fast marker expression and
   suite-gate accounting, stay out of full-lane health/timing records, force isolation
   mode, and run exactly every node from every opted-in file. Wire the lane into CI and
   add runner/recipe contract tests so neither the environment flag nor the migrated
   selection can disappear unnoticed.

4. Migrate full-app files in descending measured `AcePage`-enter cost, starting with the
   two 19-case printable-key families in
   `tests/ace/tui/widgets/test_vim_normal_key_containment.py`, then the current heavy
   files headed by plugins-browser loading/install/update, AXE entry editing, statistics
   interactions/number selection, artifact scaffold/navigation, xprompt browser
   load/jump, config-pane/config-center interactions, help filtering, and the
   saved-query picker. Opt in only tests whose startup arguments and reset contract are
   uniform; leave incompatible tests in the same file on ordinary isolated `AcePage`
   contexts, or move to the next cost-ranked file. Preserve every parameter and
   assertion. Continue until a before/after `test-cost` comparison for `tests/ace/tui`
   reports at least a 50% reduction in `textual_app_run_test_enter` seconds, and record
   the exact migrated files, node count, boot count, and seconds in test/docs comments
   next to the isolation lane.

5. Add regression coverage for the helper and migration: the shared lane must retain the
   pre-change node IDs and count; a deliberate leak in each audited category must fail
   the next checkout; isolation mode must show one distinct app identity per node;
   normal mode must show reuse within a worker while never sharing across groups or
   workers. Do not change the default `AcePage` screen size merely for speed; the
   focused measurements did not support that candidate.

## Verification

1. Run the focused helper/unit tests plus both the normal shared execution and the
   forced-isolation lane for every migrated file. Compare collected node IDs before and
   after and account for any newly added test nodes explicitly.
2. Record before/after `just test-cost` data for `tests/ace/tui`, including
   `AcePage.__aenter__`, `Textual App.run_test enter`, total wall/CPU/idle seconds, and
   boot counts. Require at least the phase's 50% reduction in measured app-start
   seconds; do not claim cache or sharing wins from raw wall-clock noise.
3. Run `just check-full` because the change touches the shared ACE harness, pytest
   runner, Justfile, and CI selection. Run the dedicated isolated lane again after the
   full suite.
4. Run `just test-contention -- <all migrated files>` with its default three repeats and
   require no new per-node failures. Confirm the committed flake-baseline gate is clean
   through `just check-full`/`selection-health`.
5. Close only `sase-ib.3` with a note containing the verified node count, before/after
   boot seconds and counts, shared and isolated lane results, `just check-full`, and
   contention-repeat result. Record any out-of-scope discovery only as a
   `PROPOSED FOLLOW-UP:` note on `sase-ib.3`.
