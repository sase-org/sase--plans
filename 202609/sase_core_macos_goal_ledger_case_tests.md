---
tier: tale
title: Fix sase-core macOS CI by making goal-ledger id-normalization tests case-portable
goal:
  sase-core CI (cargo test on macos-latest) passes again on master and on the
  release-plz v0.36.1 PR, and the goal-ledger normalization tests prove the same
  property on case-insensitive and case-sensitive filesystems without weakening any
  assertion.
size: small
proposed_by: bbugyi200.athena.0tw
create_time: 2026-09-29 08:30:45
status: wip
---

# Plan: Fix sase-core macOS CI by making goal-ledger id-normalization tests case-portable

## Goal

Turn sase-core GitHub Actions CI green again on `master` and on the open release-plz PR
(`chore: release v0.36.1`). Make two goal-ledger tests portable to the macOS runner's
case-insensitive filesystem without weakening what they prove.

## Diagnosis (already done; do not re-derive)

- `actstat` flags `sase-org/sase-core` as red. Its sase-core row is stale: it shows
  `9956773` from 2026-09-24. That commit's `Release-plz` failure is historical, and
  every Release-plz run since the v0.36.0 release on 2026-09-28 has succeeded.
  `gh run list -R sase-org/sase-core` shows the live problem. The `CI` workflow fails on
  every master push since `32d80d6` (`32d80d6`, `6557e01`, `df23cce`). It also fails on
  every run of the release-plz PR branch `release-plz-2026-09-28T16-32-41Z`, and that
  blocks the v0.36.1 release.
- The only failing job is `cargo fmt + clippy + test (macos-latest)`, at step
  `cargo test --workspace`. The ubuntu, maturin and release-script jobs pass. The same
  two tests fail in 5 of 5 runs, so this is deterministic, not a load flake. No
  `sase-core flake:` bead exists for them.
  - `goal::ledger::tests::normalized_edit_writes_only_under_the_canonical_id`, which
    panics at `crates/sase_core/src/goal/ledger/tests.rs:606`:
    `assertion failed: !root.join("items").join("7K2MQ").exists()`
  - `goal::ledger::tests::merge_writes_normalized_ids_for_into_and_from`, which panics
    at `tests.rs:626` with the same assertion.
- Root cause: commit `32d80d6` (`fix(goals): land G1 ledger correctness fixes`) added
  these tests. They check that an upper-case goal id is written only under its lowercase
  canonical form by asserting that the upper-case path does _not_ exist. macOS runners
  use case-insensitive (but case-preserving) APFS. There, `items/7K2MQ` resolves to the
  real `items/7k2mq` directory, so `exists()` returns true.
- The production code is correct. `parse_goal_id` (`crates/sase_core/src/goal/ids.rs`)
  folds ASCII upper-case to lowercase, and `normalize_action_ids`
  (`crates/sase_core/src/goal/ledger/append.rs`) applies it before any path is built.
  Only the tests are wrong. Lines 607 (`goal_marker_path(&root, "7K2MQ")`) and 627
  (`items/3FQ9T`) have the same defect and would fail next.

## Change

Work only in the linked `sase-core` repo. Open it with
`sase repo open sase-core -r "Fix macOS-only goal ledger test failures"` and use the
printed path. Read that repo's `AGENTS.md` first.

Edit only `crates/sase_core/src/goal/ledger/tests.rs`:

1. Add a helper next to `assert_no_trace`. It reads a directory's real on-disk entry
   names. A listing reports the stored, case-preserved names, so the check means the
   same thing on case-insensitive macOS APFS as on case-sensitive Linux:

   ```rust
   /// Sorted on-disk entry names in `dir`. Unlike probing a case-variant
   /// path with `exists()`, a listing reports the stored (case-preserved)
   /// names, so it means the same thing on case-insensitive macOS APFS as
   /// on case-sensitive Linux.
   fn dir_entry_names(dir: &Path) -> Vec<String> {
       let mut names: Vec<String> = std::fs::read_dir(dir)
           .expect("list dir")
           .map(|entry| {
               entry
                   .expect("dir entry")
                   .file_name()
                   .to_string_lossy()
                   .into_owned()
           })
           .collect();
       names.sort();
       names
   }
   ```

2. In `normalized_edit_writes_only_under_the_canonical_id`, replace the two case-variant
   `exists()` assertions (the `items/7K2MQ` one and the
   `goal_marker_path(&root, "7K2MQ")` one) with exact listings:

   ```rust
   assert_eq!(dir_entry_names(&root.join("items")), vec!["7k2mq"]);
   assert_eq!(dir_entry_names(&root.join("live")), vec!["7k2mq"]);
   ```

3. In `merge_writes_normalized_ids_for_into_and_from`, replace the `items/7K2MQ` and
   `items/3FQ9T` `exists()` assertions with:

   ```rust
   assert_eq!(dir_entry_names(&root.join("items")), vec!["3fq9t", "7k2mq"]);
   assert_eq!(dir_entry_names(&root.join("live")), vec!["3fq9t"]);
   ```

   After the merge, the source goal is `Merged`, which is settled. The append post-step
   therefore removes its live marker, while the target stays `Active` and keeps its
   marker. If the `live` listing differs on Linux, investigate before you change the
   expectation. Never loosen it to make the test pass.

4. Keep every other assertion in both tests unchanged, especially the
   `events[..].goal_id` checks and the `goal_ledger_show` checks. They carry the
   normalization proof on macOS. Leave `assert_no_trace` as it is, because it only
   probes the all-lowercase `zzzzz`.

These changes do not weaken the tests, and AGENTS.md requires that. On Linux, an exact
listing is strictly stronger than ruling out one upper-case name, because any stray
entry fails it. On macOS, the tests now check something meaningful instead of failing
unconditionally.

Also re-run the scan below to confirm that no other sase-core test probes a case-variant
path with `exists()`. The diagnosis found only the four lines above. If the scan finds
another hit, fix it the same way:

```bash
rg -n 'join\("[0-9A-Z]*[A-Z][0-9A-Z]*"\)\.exists|marker_path\([^)]*"[^"]*[A-Z]' crates/
```

Do not touch production code, `version` fields, path-dependency pins, `CHANGELOG.md`, or
the release-plz PR branch. release-plz owns those and refreshes its PR after the fix
lands on master.

## Verification

1. Targeted run while you iterate: `just test -p sase_core goal::ledger`.
2. Full gate: `sase tool run check` in the sase-core checkout. It is the guarded wrapper
   for `just check` and takes about 5 minutes, so give it a tool timeout of 10 minutes
   or more. It must pass.
3. The Linux host cannot mount a case-insensitive filesystem, so it cannot reproduce the
   macOS failure. The final proof is the `cargo fmt + clippy + test (macos-latest)` job
   going green after the host-owned finalizer lands the change on sase-core `master`.
   Check it afterwards with `gh run list -R sase-org/sase-core -w CI -L 3`.

## Completion

Completion is host-owned: do not create commits, branches, or PRs yourself. Declare the
sase-core repo change in the final declaration with a Conventional Commit subject, for
example:
`test(goals): assert canonical goal ids via dir listings so ledger tests pass on macOS`.

## Out of scope (note only; do not fix here)

- actstat's stale sase-core row. It reports a four-day-old commit although the GitHub
  API returns `df23cce` as the newest master push. That tool lives in a separate repo
  (`bbugyi200/actstat`).
- `sase-org/sase-research-artifacts` CI, which fails at `Install dependencies`. It is a
  different repo from the one requested.
- Node.js 20 deprecation annotations on `actions/checkout@v4` and
  `actions/setup-python@v5`. These are warnings, not failures.
