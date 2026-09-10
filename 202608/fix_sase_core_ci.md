---
tier: tale
title: Restore green CI on sase-core
goal:
  The CI workflow on sase-org/sase-core master passes fmt, clippy, and the full
  workspace test suite, and the artifact-reference context validator stops rejecting v1
  payloads that the LSP still receives from launcher-generated catalogs.
size: medium
proposed_by: bbugyi200.athena.y5
create_time: 2026-09-09 20:00:06
status: wip
---

# Plan: Restore green CI on sase-core

## Summary

`sase-org/sase-core` CI has been red on every run since `a71794c`. There are two
independent causes, one deterministic and one CI-only. Both trace back to `a509dcc`
("feat: resolve file artifact refs in core") and to a test that was already fragile
before it.

A third symptom in the history — `cargo fmt --all -- --check` failing on `c0f1ca4` and
`af2c643` — was already resolved by `a509dcc`. `cargo fmt --all -- --check` and
`cargo clippy --workspace --all-targets -- -D warnings` were both run against current
master (`a509dcc`) during this investigation and are clean. No action.

The `Release-plz` job failures ("Merge release PR" → step 4 "Wait for checks to pass")
are downstream of CI being red and should clear on their own once CI is green. No
action.

## Evidence

Current master is `a509dcc979db12126e26a55fc1b15fcb04785401` on both the remote and the
local checkout.

`cargo test --workspace --no-fail-fast` at that commit fails **2 targets / 10 tests**,
all deterministically:

| Target                   | Failing test                                                                              | Reported error                                                                                 |
| ------------------------ | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `sase_core_py --lib`     | `tests::artifact_ref_bindings_round_trip_json_shapes` (`lib.rs:10221`)                    | `unsupported artifact reference context schema_version 1; expected 2`                          |
| `sase_core_py --lib`     | `tests::artifact_ref_payload_inventory_binding_returns_plain_json_shape` (`lib.rs:11328`) | same                                                                                           |
| `sase_core_py --lib`     | `tests::prompt_artifact_bindings_round_trip_manifest_shapes` (`lib.rs:10505`)             | `assert_eq!` left `2`, right `1`                                                               |
| `sase_xprompt_lsp --lib` | `server::tests::appends_known_kind_artifact_diagnostics_from_active_catalog`              | diagnostic text contains `unsupported artifact reference context schema_version 1; expected 2` |
| `sase_xprompt_lsp --lib` | `server::tests::artifact_payload_inventory_cache_rebuilds_on_all_invalidation_paths`      | left `0`, right `1`                                                                            |
| `sase_xprompt_lsp --lib` | `server::tests::completes_artifact_kinds_and_local_payloads_per_active_project`           | left `0`, right `1`                                                                            |
| `sase_xprompt_lsp --lib` | `server::tests::completes_commit_payloads_from_a_real_git_checkout`                       | `expected ranked commit items`                                                                 |
| `sase_xprompt_lsp --lib` | `server::tests::completes_grouped_at_references_from_the_client_root`                     | left `0`, right `1`                                                                            |
| `sase_xprompt_lsp --lib` | `server::tests::artifact_completion_discloses_the_display_cap`                            | left `0`, right `200`                                                                          |
| `sase_xprompt_lsp --lib` | `server::tests::fuzzy_at_reference_payloads_survive_client_filtering`                     | left `[]`, right one `@designs:` payload                                                       |

These ten were invisible in CI until recently: `cargo test --workspace` stops at the
first failing target, and the `sase_core --lib` flake (cause 2) was aborting the run
before `sase_core_py` was reached. Runs 31540251689, 31559384410, and 31577086823 got
past the flake and exposed them.

## Cause 1 — `validate_artifact_ref_context` hard-rejects v1 (deterministic, 10 tests)

`a509dcc` bumped `ARTIFACT_REF_CONTEXT_WIRE_SCHEMA_VERSION` from 1 to 2
(`crates/sase_core/src/artifact_ref/wire.rs:9`) while adding only _additive,
`#[serde(default)]`_ fields to `ArtifactRefContextWire`: `file_roots`, `home_dir`,
`file_capture_max_bytes`. A v1 payload therefore still deserializes cleanly into the v2
struct — nothing about it is structurally invalid.

The same commit bumped `PROMPT_ARTIFACT_MANIFEST_SCHEMA_VERSION` from 1 to 2
(`crates/sase_core/src/prompt_artifact.rs:11`) and, correctly, relaxed that parser to a
range check so old rows keep reading (`crates/sase_core/src/prompt_artifact.rs:184`):

```rust
Ok(record) if (1..=PROMPT_ARTIFACT_MANIFEST_SCHEMA_VERSION)
    .contains(&record.schema_version) =>
```

`validate_artifact_ref_context` did not get the same treatment. It is still an
exact-equality check (`crates/sase_core/src/artifact_ref/mod.rs:390`):

```rust
if context.schema_version != ARTIFACT_REF_CONTEXT_WIRE_SCHEMA_VERSION {
```

That inconsistency inside a single commit is the root cause. Every v1 context now fails
validation at the three entry points that call it (`artifact_ref/mod.rs:245`,
`artifact_ref/mod.rs:323`, `artifact_ref/list.rs:63`, `editor/completion.rs:445`).

**This is a production regression, not only a test-fixture problem.** The LSP does not
construct its context in code — it deserializes it from a launcher-generated catalog
file. `load_artifact_ref_catalog` (`crates/sase_xprompt_lsp/src/server.rs:2113`) reads
that JSON and each `projects[].context` becomes an `ArtifactRefContextWire`
(`crates/sase_xprompt_lsp/src/server.rs:171`). Any catalog still emitting a v1 context
makes _every_ `@ref` unresolvable at runtime. The
`appends_known_kind_artifact_diagnostics_from_active_catalog` failure is exactly that,
surfaced as a user-visible LSP diagnostic.

### Fix

1. In `crates/sase_core/src/artifact_ref/mod.rs`, change `validate_artifact_ref_context`
   to accept the supported range `1..=ARTIFACT_REF_CONTEXT_WIRE_SCHEMA_VERSION`,
   matching the prompt-artifact-manifest precedent set in the same commit. Update the
   error message so it names the supported range rather than a single expected value.

   Note that `schema_version` defaults to `0` when a payload omits the field
   (`#[serde(default = "old_artifact_ref_context_schema_version")]` at `wire.rs:221`,
   returning `0` at `wire.rs:266`). Starting the range at `1` preserves today's
   rejection of omitted and unknown versions — no validation coverage is lost.

2. In `crates/sase_core_py/src/lib.rs:10505`, update
   `assert_eq!(py_prompt_artifact_wire_schema_version(), 1)` to expect `2`. This is a
   genuinely stale fixture: the binding correctly returns the intentionally bumped
   manifest constant, so the assertion — not the code — is wrong.

3. No changes are needed to the 7 LSP fixtures or to the two `sase_core_py` context
   fixtures at `lib.rs:10221` and `lib.rs:11328`. They pass once step 1 lands and then
   serve as the regression coverage for v1 read-compat. Leave their
   `"schema_version": 1` values alone deliberately, and say so in a brief comment at
   each of the two `sase_core_py` sites so a future reader does not "helpfully" bump
   them to 2 and silently delete the compat coverage.

### New coverage

- `crates/sase_core/src/artifact_ref/mod.rs`: a unit test asserting
  `validate_artifact_ref_context` accepts `1` and `2` and rejects `0` and `3`. The `0`
  case is the important one — it pins the "omitted field" behaviour that the serde
  default produces.
- `crates/sase_core_py/src/lib.rs`: extend one of the two context tests to cover a v2
  context (with `file_roots` populated) alongside the existing v1 case, so both sides of
  the supported range are exercised through the Python boundary.

## Cause 2 — CI-only flake in the 1,000-commit inventory test

`editor::completion::tests::commit_inventory_skips_sidecars_before_reporting_the_row_cap`
(`crates/sase_core/src/editor/completion.rs:3540`) builds
`ARTIFACT_REF_COMMIT_MAX_ROWS / ARTIFACT_REF_COMMIT_SCAN_LIMIT` = 5 repositories of 200
commits each, plus a sidecar repository: **1,001 real commits in 6 real git
repositories.**

Every commit goes through `commit_at` (`completion.rs:3191`), which spawns _two_ git
processes: `git commit --quiet --allow-empty`, and then `git rev-parse HEAD` for a
return value that both scale loops discard. That is roughly **2,000 subprocesses inside
one test**, and `assert!(output.status.success(), "git commit failed: {}", ...)`
(`completion.rs:3213`) converts any single transient subprocess failure into a red
build.

### What was observed and what was ruled out

- Failed on runs 31524425835 (`a71794c`) and 31539276159 (`a509dcc`) with
  `git commit failed: fatal: could not parse HEAD`. Passed on 31540251689, 31559384410,
  and 31577086823 — roughly 2 in 5 on an unchanged tree.
- Not reproducible locally: 27 full `sase_core --lib` runs, 15 unconstrained
  (`--test-threads 16`) and 12 pinned to four cores (`taskset -c 0-3 --test-threads 4`,
  emulating the runner's core count), produced zero failures.
- Local git is 2.47.3; the runners report 2.54.0. The trigger is environment-specific.
- `fatal: could not parse HEAD` is emitted by `lookup_commit_or_die()` in git's
  `builtin/commit.c`, meaning HEAD resolved to an object id whose object git then could
  not read. The runner-side reason for that is **unconfirmed** — do not write the fix as
  though a specific git bug were established.

The controllable root cause is the test's design: suite reliability is currently
proportional to ~2,000 independent subprocess successes on a shared CI runner, and the
behaviour under test does not need that. The row-cap merge logic itself is already
covered without git at all by `commit_inventory_reports_the_merged_row_cap`
(`completion.rs:3673`), which drives `append_ranked_commit_candidates` with in-memory
`CommitCandidate` values.

### Fix

4. Add a batch history helper to the `completion.rs` test module that builds one
   repository's history with a **single `git fast-import`** invocation, taking a list of
   `(timestamp, subject, body)` tuples.

   This approach was verified locally before writing this plan: 200 empty commits from
   one `git fast-import` process, `git fsck` clean, and
   `git log --no-color -n 200 -z --format=%H%x1f%h%x1f%at%x1f%s%x1f%b HEAD` — the exact
   invocation the production reader uses at `completion.rs:744-755` — reads them back
   with the expected `%at` and `%s`. `fast-import` defaults the author identity to the
   committer identity, so `%at` and `%an` come out correct without extra fields.

5. Convert the two scale tests to the helper:
   - `commit_inventory_skips_sidecars_before_reporting_the_row_cap`
     (`completion.rs:3540`): ~2,000 processes → 6.
   - `commit_inventory_enforces_the_per_repository_scan_limit` (`completion.rs:3607`,
     201 commits): ~402 processes → 1.

   Keep the existing assertions byte-for-byte. The point is to change how the fixture
   history is built, not what is asserted about it.

6. Leave `commit_at` in place for the tests that deliberately exercise one awkward
   commit — `commit_inventory_preserves_subject_and_multiline_body`
   (`completion.rs:3586`, quoted/tab/CJK subject and a multiline body) and the sidecar's
   single commit. Those are cheap and they are testing subject/body handling, which is
   worth keeping on the real `git commit` path.

7. Include the repository path and the commit index in the `commit_at` and helper
   failure messages. The current message identifies neither, which is why the two CI
   failures could not be narrowed to a specific repository or iteration.

Do **not** paper over this with a retry loop, a `#[ignore]`, or a CI-side git pin. A
retry would hide a genuine future regression in the commit-log reader, and the
process-count reduction addresses the fragility directly.

## Verification

Run from the sase-core checkout. `sase_core_py` needs a Python ≥ 3.12 because the crate
builds against `abi3-py312`; the system interpreter on this host is 3.11, so both
variables below are required locally (CI uses its own toolchain and needs neither):

```bash
PY=/home/bryan/.local/share/uv/python/cpython-3.12-linux-x86_64-gnu
export PYO3_PYTHON=$PY/bin/python3.12 LD_LIBRARY_PATH=$PY/lib

cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

Expected end state: `cargo test --workspace` green, with `sase_core_py --lib` at 64
passing and `sase_xprompt_lsp --lib` at 93 passing (61+3 and 86+7 respectively), plus
the new tests from steps 3 and 4.

Then confirm the flake fix and the speedup:

```bash
# Should be much faster than the ~6s the 8 completion tests take today.
cargo test -p sase_core --lib editor::completion::tests::commit_inventory

# No failures across repeated full-suite runs.
BIN=$(ls -t target/debug/deps/sase_core-* | grep -v '\.d$' | head -1)
for i in $(seq 1 20); do "$BIN" --test-threads 4 >/dev/null || echo "FAILED $i"; done
```

Finally, after the change lands on master, confirm with
`actstat --repo sase-org/sase-core --only-failures -n 3` that the CI run is green and
that the `Release-plz` "Wait for checks to pass" step clears.

## Out of scope

- The launcher that generates the LSP's artifact-reference catalog lives outside this
  repository. Step 1 makes core accept the v1 contexts it emits today, so no coordinated
  change is required. Moving that producer to v2 so it can populate `file_roots` is
  separate, non-urgent work.
- The `wheel-smoke` CI job is currently passing and is not touched here.
- CI workflow configuration (`.github/workflows/ci.yml`) is not modified.
