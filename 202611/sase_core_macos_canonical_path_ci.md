---
tier: tale
title: Fix sase-core's red macOS CI leg (canonical vs. lexical temp paths)
goal:
  sase-core master CI is green on ubuntu and macOS, named bundle lookups report
  caller-rooted paths, and no failing test binary can hide another.
size: small
decisions:
  archive_paths:
    ask:
      Should named dismissed-bundle lookups report the bundle path under the caller's
      root instead of its canonical form?
    choices:
      lexical:
        Core returns caller-rooted paths from index and fallback hits; existing
        assertions stay as-is
      canonical:
        Keep canonical output; rewrite 8 Rust/Py assertions to expect canonicalized
        paths
    default: lexical
    why:
      Other store reads return the stored caller-rooted key, and the feature's own tests
      expect it
    answer: lexical
  ci_no_fail_fast:
    ask:
      Should the CI test step run with --no-fail-fast so one failing test binary cannot
      hide failures in later binaries?
    choices:
      ci:
        Add --no-fail-fast to the CI test step only; red runs list every binary's
        failures
      none:
        Leave CI as-is; a masked failure surfaces only after the one ahead of it is
        fixed
    default: ci
    why:
      cargo stopped at sase_core's lib tests and hid a 9th macOS failure in sase_core_py
    answer: ci
  agents_md_repro:
    ask:
      Should sase-core's AGENTS.md macOS bullet gain a Linux repro recipe that uses a
      symlinked TMPDIR?
    choices:
      add:
        Extend the existing canonicalization bullet with the TMPDIR-through-a-symlink
        repro command
      skip: Leave AGENTS.md unchanged
    default: add
    why:
      The repro turns a macOS-only CI round trip into a local test run of under a minute
    answer: add
proposed_by: bbugyi200.athena.1g8.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-25 17:22:29
status: wip
---

# Fix sase-core's red macOS CI leg: canonical vs. lexical temp paths

All work happens in the linked **sase-core** repo. Open it with
`sase repo open sase-core -r "<reason>"` and work only in the printed path; read its
`AGENTS.md` first. Paths below are relative to that checkout.

## Diagnosis (already done — do not re-investigate)

`actstat` shows every `CI` run on sase-core `master` has been red since 2026-11-12 (last
green: `fae59b46`). The early red runs were compile/clippy errors that later commits
fixed. Since `09391374` (2026-11-20), and through the newest commit `71944fb4`, the
**only** failing job is `cargo fmt + clippy + test (macos-latest)`, step
`cargo test --workspace`, with the same 8 `sase_core` lib tests:

- `agent_archive::store::tests::{named_bundle_lookup_selects_exact_name_or_workflow_across_visibility, named_bundle_lookup_skips_deleted_newest_index_row, missing_index_recovery_visits_nested_child_bundles_and_skips_malformed_json, named_bundle_lookup_returns_complete_fallback_after_explicit_maintenance, named_bundle_lookup_busy_timeout_is_bounded_and_busy_index_recovers, unavailable_named_bundle_indexes_keep_classification_when_recovery_exhausts, missing_name_columns_only_disable_named_lookup}`
  — e.g. `left: "/private/var/folders/.../2026111601.json"` vs
  `right: "/var/folders/.../2026111601.json"`.
- `agent_scan::index::tests::lookup::pointer_lookup_resolves_aliases_and_symlinks` —
  `path /private/var/.../ace-run/20261116120000`, `left: 0, right: 1` hits.

Root cause: on macOS the tempdir lives under `/var`, a symlink to `/private/var`, so a
temp path and its canonical form differ. Linux tempdirs have no symlinked ancestor, so
ubuntu CI and every local `just check` stay green.

**Hidden 9th failure.** `cargo test` stops at the first failing test binary, and
`sase_core --lib` runs first, so on macOS no other binary has run since then.
Reproducing on Linux with a symlinked `TMPDIR` and `--no-fail-fast` (see Verification)
reproduced exactly the 8 failures above **plus** `sase_core_py`
`agent_custody::tests::dismissed_bundle_index_bindings_are_exported_and_round_trip`
(`crates/sase_core_py/src/agent_custody/tests.rs:872`, `bundle_path` came back as the
`real/` path instead of the `link/` path). Every other binary (sase_core integration
tests, sase_gateway, sase_macro_lsp, doc-tests) passed under the symlinked `TMPDIR`.

### Cause 1 — `agent_archive` named lookup leaks canonical paths

In `crates/sase_core/src/agent_archive/store.rs` (feature added in `d2010865`):

- `validate_named_bundle` (index source) correctly canonicalizes **both** root and
  candidate for its `starts_with` containment check, then **returns**
  `bundle_path: canonical_path` and falls back to `canonical_path.parent()` for
  `artifacts_dir`.
- `bounded_named_bundle_fallback` (fallback source) seeds its directory walk with
  `canonical_root`, so every reported `bundle_path` / `artifacts_dir` is canonical.

Every other read in the same store (`query_summaries`, `query_bundle_paths_by_suffixes`,
…) returns `bundle_path` exactly as stored, and that stored value is the caller-rooted
path and the table's primary key. The named lookup is the one inconsistent reader. Both
the Rust store tests and the `sase_core_py` round-trip test written with the feature
expect the caller-rooted form (the py test also asserts `artifacts_dir == temp.path()`).
The only production consumer, sase's `_find_named_dismissed_bundle`, uses
`winner.artifacts_dir`.

### Cause 2 — the pointer-lookup test assumes a canonical projects root

`agent_scan` stores `artifact_dir` in the projects root's _lexical_ form, and its layout
parser (`agent_scan/layout.rs::parse_agent_artifact_path`) uses a lexical
`strip_prefix(projects_root)`. The test builds its fixture under the lexical
`tmp.path()` and then queries with `dir.canonicalize()`, which matches the indexed path
only when the tempdir has no symlinked ancestor. The test's real subject is the alias
and the **inner** `symlink-pointer`, not macOS's `/var` link. This is the same pattern
as the earlier test-side fix `d4377825` ("canonicalize both sides so the assertion
checks the inner symlink").

## Changes

### 1. `crates/sase_core/src/agent_archive/store.rs` — report caller-rooted paths

> [!decision] archive_paths = lexical

- `validate_named_bundle`: keep the canonical containment check and keep reading and
  validating via `canonical_path`, so filename, suffix, and size semantics are
  unchanged. In the returned `NamedDismissedBundleHitWire`, set `bundle_path` from the
  lexical `path` (the indexed value, or `root.join(..)` when relative) and take the
  `artifacts_dir` fallback from `path.parent()` instead of `canonical_path.parent()`.
- `bounded_named_bundle_fallback`: keep the existing `root.canonicalize()` match for the
  missing-root (`source = "none"`) and error classification, but seed `pending` with
  `root.to_path_buf()` instead of the canonical root. The walk already refuses to follow
  symlinked entries (`entry.file_type()` is lstat-based), so walking from the lexical
  root visits the same files and only changes how the reported paths are spelled.
- Add a one-line doc comment on `NamedDismissedBundleHitWire::bundle_path` in
  `agent_archive/wire.rs`: paths are reported under the caller's `root`, matching the
  stored `bundle_path` key.
- Add a Linux-visible regression test to `agent_archive/store/tests.rs`, for example
  `named_bundle_lookup_reports_paths_under_the_callers_symlinked_root`. Create
  `tmp/real` and a symlink `tmp/link -> tmp/real`, using `std::os::unix::fs::symlink`
  the way `agent_scan/index/tests/lookup.rs` already does, and pass `tmp/link` as
  `root`. Write a bundle without `artifacts_dir`, then assert:
  - With no index (`source == "fallback"`), `winner.bundle_path == link/<file>` and
    `winner.artifacts_dir == link`.
  - After `upsert_bundle_summary(link, link/<file>, ..)` (`source == "index"`), the same
    two assertions hold.

  This keeps the contract covered on ubuntu CI, not only on macOS. Keep the file under
  1,500 lines; it is about 1,150 now.

- Do **not** edit the 7 existing store-test assertions or the `sase_core_py` test; they
  should pass unchanged.

> [!decision] archive_paths = canonical Leave `store.rs` unchanged. Add a small test
> helper that returns `fs::canonicalize(p).unwrap().to_string_lossy().into_owned()`.
> Compare each of the 7 store tests' `winner.bundle_path` against the canonicalized
> expected file. In `crates/sase_core_py/src/agent_custody/tests.rs` (~lines 872–873),
> compare `bundle_path` and `artifacts_dir` against the canonicalized bundle path and
> tempdir. Keep the lexical `root` input so the symlinked-ancestor containment check
> stays exercised on macOS. Document on `NamedDismissedBundleHitWire::bundle_path` that
> the path is canonical. Skip the new symlinked-root regression test, or make it assert
> the canonical form instead.

### 2. `crates/sase_core/src/agent_scan/index/tests/lookup.rs` — canonical fixture root

In `pointer_lookup_resolves_aliases_and_symlinks`, build everything under
`let root = tmp.path().canonicalize().unwrap();`. That covers `projects`,
`rebuild_index(&root, &projects)`, `alias-pointer`, and `symlink-pointer`. Add a
two-line comment: the index stores artifact dirs in the projects root's lexical form,
and macOS's `/var` → `/private/var` link would otherwise make `canonical` differ from
the indexed path. The assertions stay unchanged; the inner symlink and alias remain the
subject under test.

### 3. `.github/workflows/ci.yml` — stop one binary from masking others

> [!decision] ci_no_fail_fast = ci

Change the `cargo test --workspace` step's command to
`./scripts/check.sh test --no-fail-fast`. `default_scope` still adds `--workspace`
because the flag is neither `-p` nor `--package`. Keep the step name. Add a short
comment that the macOS leg previously hid a `sase_core_py` failure behind the first
failing binary. Leave `scripts/check.sh` and the `just` recipes unchanged. Under `none`,
skip this step.

### 4. sase-core `AGENTS.md` — Linux repro tip

> [!decision] agents_md_repro = add

This is sase-core's hand-written repo doc, not a SASE memory note; `CLAUDE.md` there is
just `@AGENTS.md`. Extend the existing "Canonicalize both sides of a path comparison…"
bullet with one sentence, for example: "Reproduce these failures on Linux by running
tests with `TMPDIR` pointing through a symlink, e.g.
`ln -s "$PWD/real" link && TMPDIR=$PWD/link just test --no-fail-fast`." Adjust it so
`real` is created first and the command is copy-pasteable. Under `skip`, leave the file
alone.

### 5. File the latent production gap as a bug bead

Use `/sase_new_task` (task type `bug`, size `medium`). The latent gap: sase's
`_lookup_groups._query_pointer` sends a `resolve()`d (canonical) pointer, while
sase-core indexes and parses artifact dirs lexically under `sase_projects_dir()`. When
`SASE_HOME` / `~/.sase` has a symlinked ancestor, a pointer lookup therefore misses in
both the index query (`agent_scan/index/lookup.rs::query_pointer_hits` only tries
`[raw, canonical(raw)]`) and recovery (`scan_agent_artifact_dir` →
`parse_agent_artifact_path` lexical `strip_prefix`). The likely fix is to rebase a
canonical pointer that sits under `canonical(projects_root)` onto the lexical
`projects_root` before querying and recovering. This host is unaffected today (`~/.sase`
is not symlinked), so it is out of scope for this CI fix.

## Verification

Run from the sase-core checkout (the `test` recipe is not guarded; `check` is).

1. **Reproduce first, before editing**, to prove the harness:

   ```bash
   d=$(mktemp -d) && mkdir "$d/real" && ln -s "$d/real" "$d/link"
   TMPDIR="$d/link" just test --no-fail-fast 2>&1 | tee /tmp/symtmp_before.log
   ```

   Expect exactly the 9 failures listed above (8 `sase_core --lib`, 1
   `sase_core_py --lib`). Use a tool timeout of 20 minutes or more.

2. After the changes, rerun the same command: expect **0 failures** in every binary.
3. `just test -p sase_core named_bundle_lookup_reports_paths` (or your test's name)
   passes with the default `TMPDIR`. Temporarily reverting the `store.rs` hunk should
   make it fail, proving it guards the contract.
4. `sase tool run check` must pass (fmt-check, features, clippy `-D warnings`, all
   tests, script tests). Give it a 15-minute or longer tool timeout.
5. Suggested commit subject (Conventional Commits, not breaking — no binding or wire
   shape change):
   `fix(agent-archive): report named bundle lookup paths under the caller's root`. Do
   not touch versions or `CHANGELOG.md`; release-plz owns them.

## After landing

The next `master` CI run should be green on both legs. Check with
`actstat --only-failures`. Residual risk: the macOS leg has not executed any binary past
`sase_core --lib` since Nov 17, and the symlinked-`TMPDIR` repro covers only the
`/private/var` class of macOS differences. With `--no-fail-fast` in CI, any other
macOS-only failure will show up all at once on the first run; fix it as a follow-up. No
`sase-core-revision.txt` pin move is needed in sase, because no binding changes.

## Out of scope

- The product-side pointer-lookup fix (step 5 files it).
- Any change to `scripts/check.sh` / local `just` behavior.
- The `release-plz-*` branch CI run currently in progress; it inherits the fix once
  `master` is green.
