---
tier: tale
title: Finish and land epic sase-en
goal:
  Published SASE installs require the core release that contains the single-pass bead detail API, the integrated epic is
  fully revalidated and closed, post-close Symvision is clean, and the durable epic plan is marked done.
proposed_by: bbugyi200.athena.sase-en.land
bead: sase-en
create_time: 2026-08-03 10:53:44
status: done
---

- **PARENT:** [202608/bead_show_speed.md](https://github.com/sase-org/sase--plans/blob/main/202608/bead_show_speed.md)
- **BEAD:** [sase-en](https://github.com/sase-org/sase--beads/blob/main/pages/sase-en/README.md)

# Finish and land epic `sase-en`

## Objective

Close the one packaging gap found by the `sase-en` land audit, revalidate the complete performance and output contract
on the integrated `master` tree, and only then close the epic, run the required post-close Symvision cleanup, and mark
its durable plan done.

The live epic is `sase-en`; all four children (`sase-en.1` through `sase-en.4`) are already closed. Its durable plan is
`plans:202608/bead_show_speed.md`, resolved through the linked `plans` repository. Do not force-close the epic.

## Audit evidence to preserve

The primary-repo epic commits are:

- `25e706f76` (`sase-en.1`): memoized origin/config/sidecar derivations plus explicit ACE and store-mutation cache
  invalidation.
- `e5208ec97` (`sase-en.2`): lazy one-command parser registration with full-parser fallbacks for help, option-first, and
  unknown-command paths.
- `7a66461b9` (`sase-en.3`): Python facade and detail resolver for the Rust single-snapshot API.
- `18d438bc0` (`sase-en.4`): composed one-read/probe-budget/style guards and `docs/beads.md` documentation.

The linked `sase-core` commit is `5f39c3dc2` (`sase-en.3`), released in `sase-core-rs 0.17.15` by `628dcacd1`.

The land audit read each implementation and test, reviewed commits from the plan's creation time onward in both repos,
and found no output or semantic conflict. In particular:

- Later Agent CLI history work completed the visual fixtures that had caused the two proposed PNG mismatches; both exact
  nodes now pass.
- Later timezone commit `c449ce27c` removed the four stale `sase-ej` Symvision entries; their proposing phase was also
  corroborated on active epic `sase-ej` for ownership/history.
- Sidecar publication, timestamp, bead-reference migration, agent-name, and completion changes either touched disjoint
  paths or are already exercised through the current lazy parser/detail/inventory tests. The later `docs/beads.md`
  rewrite retained the epic's single-read/narrow-parser/memoized-inventory text.
- The real integration defect is packaging: `pyproject.toml` and `uv.lock` still accept/lock `sase-core-rs 0.17.14`,
  while `bead_show_issue_detail` first exists in the published `0.17.15` wheel.
  `uv lock --dry-run --upgrade-package sase-core-rs` resolves `0.17.15`, and an isolated
  `uv run --with sase-core-rs==0.17.15` import proved that wheel exposes `bead_show_issue_detail`.

Current verification evidence before the packaging edit:

- 143 focused Python tests pass across repo identity cache invalidation, bead show/detail/style, structural budgets,
  Rust-facade conversion, and parser narrowing.
- All 13 linked-core `bead_read_parity` tests and `cargo fmt --all -- --check` pass.
- The two previously failing Agent CLI PNG nodes pass against committed goldens.
- Benchmarks reproduce the feature: `sase bead show sase-bv --style plain` averages 804 ms versus the 1.841 s baseline
  (2.29x faster), and ref-bearing `sase-cl` averages 1.602 s versus 3.184 s (1.99x faster).

Follow-up triage is already recorded and must be summarized in the close note:

- `sase-e2` received a `+1` from the `sase-en.3` contention-flake proposal; its exact node passes alone in 3.62 s.
- `sase-eq` is ready for the five tests that accidentally discover an ambient `/tmp/sdd/beads` above `tmp_path`.
- The duplicated PNG proposals from `sase-en.1`, `.2`, and `.3` were declined because later `sase-el` work resolved them
  and both nodes now pass.
- The stale `sase-ej` whitelist proposal is owned by that active epic and is already resolved on current `master`.
- The epic plan's five deliberately deferred optimizations are ready as `sase-er` (read fast-path mutation imports),
  `sase-es` (full-help parser imports), `sase-eu` (LibYAML C loader), `sase-ev` (dormant Rust show renderer), and
  `sase-ew` (projection-backed reads).

## Phase 1: Require the core release that contains the feature

Update the host dependency contract so a normal published install cannot select a core without `bead_show_issue_detail`:

1. Change the `sase-core-rs` floor in `pyproject.toml` from `0.17.14` to `0.17.15`, retaining the `<0.18.0` ceiling.
2. Refresh only that package in `uv.lock` so it resolves the published `0.17.15` artifacts and hashes.
3. Update `tests/test_sase_core_rs_telemetry_smoke_tool.py` so its declared-minimum assertion expects `0.17.15`.
4. Keep the change minimal; do not edit the linked `sase-core` repository, whose feature and release commits are already
   complete.

Then run:

```bash
just install
.venv/bin/python -m pytest -q -n 0 \
  tests/test_sase_core_rs_telemetry_smoke_tool.py \
  tests/test_core_facade/test_bead_read.py \
  tests/test_bead/test_cli_show_budget.py
uv run --isolated --no-project --with sase-core-rs==0.17.15 \
  python -c "import sase_core_rs; assert hasattr(sase_core_rs, 'bead_show_issue_detail')"
git diff --check
```

Confirm the lock contains `sase-core-rs 0.17.15`, the editable development binding still reports `0.17.15`, and a live
`sase bead show` succeeds after reinstalling.

## Phase 2: Revalidate the integrated epic

Run the repository-required full gate:

```bash
just check
```

Also rerun the focused Rust parity test in the already-opened linked `sase-core` checkout if the full gate does not
exercise it directly:

```bash
cargo fmt --all -- --check
cargo test -p sase_core --test bead_read_parity
```

Re-run short `hyperfine` samples for `sase-bv` and ref-bearing `sase-cl`; record the means against the 1.841 s and 3.184
s baselines. Do not widen this tale for independent failures: use the already-created follow-up tasks where they match,
and apply the `/sase_new_task` procedure only for a genuinely distinct independent issue. Any regression caused by the
core-floor edit or the epic itself remains work here and must be fixed before closing.

## Phase 3: Land and close the epic

This is the final phase. Re-read `sase bead show sase-en` and its four children to ensure they are still closed and no
new notes appeared during implementation. Then close normally:

```bash
sase bead close sase-en --note "<complete audit, integration, verification, benchmark, and follow-up outcomes>"
```

The close note must name the four primary commits and linked-core commit, state that source and later commits were
reviewed, report the `0.17.15` floor/lock integration, report focused/Rust/full-gate results and final benchmarks, and
list every follow-up outcome (`sase-e2`, `sase-eq`, resolved PNG and `sase-ej` proposals, plus `sase-er`, `sase-es`,
`sase-eu`, `sase-ev`, and `sase-ew`). Do not use `--force`; if close is rejected, resolve the named descendant state
deliberately.

Only after the close succeeds, run:

```bash
just symvision
```

Remove any now-stale `sase-en` whitelist entries or unused code it reports. There are no current `sase-en`
`--epic-symbol` entries, so a clean run is expected; nevertheless the post-close run is mandatory. If it causes source
or configuration edits, run `just install` as needed and repeat `just check` before finishing.

Finally, open the linked `plans` repo through `sase repo open plans`, change only the durable plan's frontmatter from
`status: wip` to `status: done` in `202608/bead_show_speed.md`, and verify `sase bead show sase-en` reports `CLOSED` and
the plan file reports `status: done`.
