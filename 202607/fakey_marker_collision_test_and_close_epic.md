---
create_time: 2026-07-10 18:55:21
status: done
prompt: 202607/prompts/fakey_marker_collision_test_and_close_epic.md
tier: tale
---
# Plan: Add the missing fakey marker-collision regression test and close the sase-5o epic

## Context

The `sase-5o` epic ("fakey — a first-class fake agent CLI provider for testing launches, failures, and retries") is
otherwise complete: all five phase beads are closed, their commits are on master, all planned deliverables (CLI,
scenario engine, provider integration, E2E retry harness, fixture-driven and E2E-driven PNG goldens, docs, config)
exist, and the fakey test suite (40 tests) plus the full visual snapshot suite (168 tests) pass.

One commitment from the epic plan's "Risks & mitigations" section was never delivered:

> **Marker collision with real provider errors.** The `FAKEY-` prefix is namespaced and covered by a regression test
> asserting no real provider's built-in patterns match fakey markers (and vice versa, except where a scenario opts in
> deliberately).

No such test exists anywhere in the repo. It matters because `find_retry_config_for_error()` in
`src/sase/llm_provider/retry_config.py` performs global, cross-provider, first-match substring matching: if any real
provider's built-in `error_patterns` ever matched a `FAKEY-RETRYABLE:`/`FAKEY-FAIL:` marker line (or fakey's
`FAKEY-RETRYABLE` pattern matched a real provider's error output), retry configs would be silently mis-attributed.

## Work items

### 1. Add the marker-collision regression test

Add a regression test to `tests/fakey/test_provider.py` (alongside the existing registry/retry-config tests, which
already demonstrate the required cache-clear idioms: `_build_llm_pm.cache_clear()` and
`_llm_metadata_payload.cache_clear()`).

The test should, using `_built_in_defaults()` and `is_retryable_error()` from `sase.llm_provider.retry_config`:

1. Build representative fakey marker outputs, e.g. `"FAKEY-RETRYABLE: simulated failure"` and
   `"FAKEY-FAIL: simulated failure"`.
2. Assert no **real** provider's built-in retry config matches either marker output (iterate every entry of
   `_built_in_defaults()` except `fakey`).
3. Assert fakey's own built-in config matches the retryable marker but NOT the non-retryable `FAKEY-FAIL` marker.
4. Assert the reverse direction: fakey's config does not match any real provider's built-in pattern text (treat each
   real provider's `error_patterns` entries as sample error outputs).
5. Pin the end-to-end attribution: `find_retry_config_for_error("FAKEY-RETRYABLE: simulated failure")` returns a config
   whose `error_patterns` contains `"FAKEY-RETRYABLE"` (i.e. fakey's), guarding against future global-matching
   regressions. Use a `load_merged_config` monkeypatch returning an empty user config so the assertion only exercises
   built-in defaults.

Note the deliberate opt-in exception from the epic plan: the bundled `@capacity` scenario emits a real codex error
message ("Selected model is at capacity") as scenario _output_. That is intentionally cross-matching and is NOT part of
the fakey marker namespace, so the test must only assert on the `FAKEY-` marker lines and fakey's built-in pattern.

### 2. Verify

- Run the fakey test suite (`tests/fakey/`) and confirm the new test passes.
- Run `just check` (per repo policy for any file changes).

### 3. Commit the change

Commit the new regression test with the epic bead ID (`sase-5o`) referenced in the commit message.

### 4. Close the epic bead

- `sase bead close sase-5o`
- Run `just pyvision` AFTER closing the epic (some symbols are ignored while an epic is open) to confirm no unused code
  is left behind.

### 5. Mark the epic plan file done

Update the frontmatter of the epic plan file `sdd/epics/202607/fakey_provider.md` (under the SASE project state
directory): set `status: done` (currently `status: proposed`).
