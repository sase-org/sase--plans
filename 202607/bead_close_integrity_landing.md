---
tier: epic
title: Land bead close integrity — unblock the core release and finish the repair
goal: 'The declared `sase-core-rs` floor names a published release that actually contains
  `note_appended` and `bead_doctor_report`, so an ordinary install can read the live
  bead store and the `published-core-minimum-smoke` CI lane is green; the beads still
  carrying dropped note revisions are either restored or explicitly declined by the
  user, with the decision recorded; and epic `sase-bd` is closed with its plan file
  marked done.

  '
phases:
- id: core-clippy
  title: Unblock sase-core master
  depends_on: []
  size: small
  description: 'core-clippy: fix the `clippy::question_mark` denial in the doctor
    reader that has kept sase-core master red since the projection-drift commit, sweep
    the neighbouring commits for the same class of denial, and confirm master CI goes
    green so release-plz can cut a release.'
- id: core-release
  title: Publish the release containing all four core commits
  depends_on:
  - core-clippy
  size: small
  description: 'core-release: merge the refreshed release PR once master is green,
    then confirm the published wheel is actually installable from the package index
    and carry the real published version forward.'
- id: floor-bump
  title: Make the declared core floor honest
  depends_on:
  - core-release
  size: small
  description: 'floor-bump: raise the `sase-core-rs` window to the newly published
    release, refresh the lockfile and the declared-minimum test, and prove in a clean
    venv that the minimum satisfies every required binding and can read a store containing
    note-append events.'
- id: lost-notes
  title: Put the lost-note restore to the user
  depends_on: []
  size: small
  description: 'lost-notes: re-measure the beads carrying dropped note revisions,
    present the restore to the user through an approval gate, run or decline it accordingly,
    and record the decision durably so it stops being absent.'
- id: land
  title: Close the epic
  depends_on:
  - floor-bump
  - lost-notes
  size: small
  description: 'land: close the epic bead with a note covering what was verified,
    run symvision and clear anything stale it reports, and mark the original plan
    file done.'
parent_bead: sase-bd
create_time: 2026-07-30 16:13:33
status: done
bead_id: sase-bd.9
---

- **PROMPT:** [prompts/202607/bead_close_integrity_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/bead_close_integrity_landing.md)
- **PARENT:** [202607/bead_close_integrity.md](https://github.com/sase-org/sase--plans/blob/main/202607/bead_close_integrity.md)
- **BEAD:** [sase-bd.9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-bd/sase-bd.9.md)

# Land bead close integrity: unblock the core release and finish the repair

Completes epic `sase-bd` (`@plan:202607/bead_close_integrity.md`). That epic's eight phases all closed, and its
load-bearing behavior is real and verified: the reducer is idempotent under duplicate closes, the live projection is
clean, and `sase-b8.8` — the epic's own reproduction case — now projects `closed_at: 2026-07-30T16:10:17Z`, the first
close, instead of the 16:22:45Z rewrite.

Two things the epic promised did not actually land.

## Problem

### 1. The declared core floor names a release that cannot read the live store

Phase `floor-docs` was required to "raise the `sase-core-rs` window ... to the release-plz-published version containing
all three core phases", and said plainly: "If that version is not published yet, this phase waits on it." It did not
wait. It set the window to `>=0.14.2,<0.15.0`.

The three core phases did not all make v0.14.2:

| Core commit | Phase                               | In v0.14.2? |
| ----------- | ----------------------------------- | ----------- |
| `160ff9e`   | `core-close-interval` (`sase-bd.1`) | yes         |
| `293ccb2`   | `core-close-verified` (`sase-bd.2`) | yes         |
| `81a82d5`   | `core-note-append` (`sase-bd.3`)    | **no**      |
| `6468cb9`   | `doctor-projection` (`sase-bd.5`)   | **no**      |

`81a82d5` and `6468cb9` sit after the `v0.14.2` tag, unreleased. So the window is wrong in both directions: its floor is
below the behavior the code requires, and its `<0.15.0` cap actively **excludes** the release that would fix it.

This is not theoretical. The live beads sidecar already contains 5 `note_appended` events, and installing the exact
declared minimum from the package index reproduces the failure the phase existed to prevent:

```text
$ uv pip install "sase-core-rs==0.14.2"
$ python -c "import sase_core_rs; sase_core_rs.bead_show('<beads sidecar>', 'sase-bd.3')"
ValueError: validation: invalid bead event stream .../events/streams/sase-bc.jsonl line 23:
  unknown variant `note_appended`, expected one of `issue_created`, `issue_updated`, ...
```

It is also a raw serde message rather than the actionable "run `just install`" error, because that error shipped in
`81a82d5` too.

The repo already has a guard for exactly this, and it is failing:

```text
$ /tmp/published-core-smoke/bin/python tools/check_sase_core_rs_bindings
sase_core_rs 0.14.2 is missing 1 of 230 required binding(s):
  bead_doctor_report
# exit 1
```

`bead_doctor_report` is the binding `src/sase/core/bead_read_facade.py:131` requires. CI's
`published-core-minimum-smoke` lane runs that command, so that lane is red on master.

**This bug is invisible to a local `just check`, by design.** `just install` builds `sase_core_rs` from the linked
`sase-core` checkout and passes `--overrides` so the `pyproject.toml` constraint is bypassed; the Justfile says so
outright — the override file exists to keep "the pyproject constraint authoritative for normal published-wheel
resolution" only when no source checkout is present. A local `just check` therefore runs against core master and passes,
which is exactly what happened for phase `floor-docs` and why the gap survived. Confirmed again while writing this plan:
`just check` currently exits 0 with every gate green. Do not treat a green local `just check` as evidence that this
phase's problem is fixed — the acceptance test is the clean-venv published-minimum path in `floor-bump`.

**Root cause of the blockage.** v0.15.0 cannot be published because sase-core master is red. Core CI run `30572163624`
fails on `6468cb9` — `doctor-projection`'s own commit — with a clippy denial, and release PR #63
(`chore: release v0.15.0`) carries the same failure:

```text
error: this `match` expression can be replaced with `?`
   --> crates/sase_core/src/bead/read.rs:232:9
    |
232 | /         match read_legacy_jsonl_issues(beads_dir) {
233 | |             Ok(issues) => (issues, None),
234 | |             Err(err) => return Err(err),
235 | |         }
    | |_________^
    = note: `-D clippy::question-mark` implied by `-D warnings`
error: could not compile `sase_core` (lib) due to 1 previous error
```

Reproduced locally at clippy 0.1.97. `sase-bd.5`'s completion note claimed `cargo test --workspace` passed, which is
true; clippy was evidently not rerun against the final state.

So the chain is: clippy failure → core master red → release-plz cannot cut v0.15.0 → the sase floor cannot name a
release containing `note_appended`/`bead_doctor_report` → published installs cannot read the store and one CI lane stays
red.

### 2. The lost-note restore was never put to the user

Phase `repair` step 5 required presenting the lost-note count to the user through `/sase_gate` for explicit approval,
"because this appends text to hundreds of beads and is the user's call, not the agent's", and step 6 required saying
plainly if the restore was deferred.

`sase-bd.8` closed with **no note at all** and 0 commits. `sase bead history --lost-notes` still reports 301 beads
carrying dropped revisions — the same order the research measured. Nothing was restored and nothing was recorded, so the
outcome is currently undocumented rather than deliberately deferred.

## Non-goals

- **Re-running `sase bead doctor --fix-projection` against the live store.** The projection is already clean; the
  corrected reducer regenerated `issues.jsonl` through ordinary mutation traffic, since `export_jsonl` is
  load-then-write. `sase bead doctor` reports no drift and the expected census (450 redundant close events across 312
  beads, 2 in the last 7 days). Do not manufacture a repair commit for a store that has nothing to repair.
- **Reopening any closed `sase-bd` phase.** The behavior each phase promised is present and covered; only the two items
  above are outstanding.
- **Widening the epic's scope.** Research items 4, 7, 9 and 11 remain out of scope, as the original plan set out.

## Phases

Rust work lands in the linked core repo; open it with `sase repo open sase-core -r "<reason>"` and use the printed path
as the only path for reads and writes. Commit there with Conventional Commits so release-plz computes the version, and
never hand-edit `[workspace.package].version`. Remember `just install` before `just check` in an ephemeral workspace.

### Phase `core-clippy` — unblock sase-core master

**Depends on:** nothing.

In `crates/sase_core/src/bead/read.rs`, replace the `match` at the `doctor_impl` legacy branch with the `?` form clippy
asks for:

```rust
} else {
    (read_legacy_jsonl_issues(beads_dir)?, None)
};
```

Then sweep the rest of `6468cb9` and `81a82d5` for the same class of denial rather than fixing only the first reported
one — clippy stops at the first error in a crate, so a second violation may be hiding behind it.

Verify with the exact commands CI runs, not a subset:

- `cargo fmt --all -- --check`
- `cargo clippy --workspace --all-targets -- -D warnings` (set `PYO3_PYTHON` to a Python 3.12 interpreter if the
  `pyo3-build-config` build script cannot find one)
- `cargo test --workspace`

Commit and push to the core repo's canonical branch, then confirm the master CI run for that commit goes green before
reporting the phase done.

**Done when:** `cargo clippy --workspace --all-targets -- -D warnings` is clean locally and the core master CI run for
the pushed commit has concluded successfully.

### Phase `core-release` — publish the release containing all four core commits

**Depends on:** `core-clippy`.

release-plz maintains the release PR from master, so the existing PR #63 (`chore: release v0.15.0`) will refresh once
master is green. Confirm it contains `160ff9e`, `293ccb2`, `81a82d5` and `6468cb9`, that its checks pass, and that the
version it proposes is a breaking-change bump (`81a82d5` is `feat(bead)!`), then merge it.

Wait for the release workflow to tag and publish the wheel. Confirm publication from the index rather than from the repo
— the release is only real when it is installable:

```bash
uv venv --python 3.12 /tmp/core-release-check
uv pip install --python /tmp/core-release-check/bin/python "sase-core-rs==<published version>"
```

If the version release-plz computes differs from `0.15.0`, use whatever it actually publishes and carry that number into
the next phase rather than forcing the number in this plan.

**Done when:** the release PR is merged and the published version is installable from the package index.

### Phase `floor-bump` — make the declared floor honest

**Depends on:** `core-release`.

- `pyproject.toml`: raise the `sase-core-rs` window to the version published by `core-release`
  (`>=<published>,<<next-minor>`). The upper cap must not exclude the published release — that is the specific mistake
  being corrected here.
- Refresh `uv.lock` so the locked `sase-core-rs` matches, using the project's normal lock workflow rather than a
  hand-edit.
- `tests/test_sase_core_rs_telemetry_smoke_tool.py`: `test_declared_minimum_tracks_pyproject_dependency` asserts the
  literal `"0.14.2"`. Update it to the new minimum. Search for any other test pinning the old floor and update those
  too.

Then prove the new minimum is sufficient, in a clean venv with no source checkout and no dependency override, exactly as
CI's `published-core-minimum-smoke` lane does:

```bash
core_minimum="$(python3 tools/smoke_sase_core_rs_telemetry --print-minimum pyproject.toml)"
uv venv --python 3.12 /tmp/published-core-smoke
uv pip install --python /tmp/published-core-smoke/bin/python "sase-core-rs==${core_minimum}"
/tmp/published-core-smoke/bin/python tools/check_sase_core_rs_bindings   # must exit 0
/tmp/published-core-smoke/bin/python tools/smoke_sase_core_rs_telemetry
```

`check_sase_core_rs_bindings` reporting zero missing bindings is the phase's real acceptance test; it is what is failing
today. Additionally confirm the published minimum can read a store containing `note_appended`, since that is the failure
users would actually hit — point `bead_show` at the beads sidecar (open it with `sase repo open beads`) and confirm it
returns instead of raising `unknown variant note_appended`.

Finally run `just install`, then `just check`.

**Done when:** `check_sase_core_rs_bindings` passes against the declared minimum, that minimum reads a `note_appended`
store, and `just check` passes.

### Phase `lost-notes` — put the restore to the user

**Depends on:** nothing.

Run `sase bead history --lost-notes` and report the current bead and revision counts (301 beads at the time of writing;
re-measure rather than quoting this number).

Present that count to the user through the `/sase_gate` skill for explicit approval, as phase `repair` required. The
gate must make clear that approving appends restored text to hundreds of beads.

- **On approval:** run `sase bead history --lost-notes --restore --yes`, then re-run `sase bead history --lost-notes`
  and confirm the set is empty.
- **On decline:** change nothing.

Either way, record the outcome as a note on `sase-bd.8` (`sase bead note sase-bd.8 "..."`) so the decision is durable
instead of absent — that is the step whose omission created this phase. Note that `sase bead note` is safe on a closed
bead by this epic's own contract.

**Done when:** the restore has either run to an empty lost-note set or been explicitly declined by the user, and the
outcome is recorded on `sase-bd.8`.

### Phase `land` — close the epic

**Depends on:** `floor-bump`, `lost-notes`.

1. `sase bead close sase-bd --note "<what was verified>"`. The note should state that the reducer, mutation, CLI, doctor
   and history behavior were verified against the source and the live store; that the projection is clean and
   `sase-b8.8` projects its first close; that the core floor now names a release containing `note_appended` and
   `bead_doctor_report`; and what the user decided about the lost-note restore.
2. Run `just symvision` and remove any stale `sase-bd` epic-symbol whitelist entries and unused code it reports. A
   pre-check found no `sase-bd` whitelist entries, so this is expected to be a no-op — confirm rather than assume.
3. Set `status: done` in the frontmatter of `@plan:202607/bead_close_integrity.md`.

If the close is rejected, the named phases were never completed: finish or reopen them rather than forcing. Never force
merely to make the command succeed.

**Done when:** `sase-bd` is closed, `just symvision` is clean, and the epic's plan file reads `status: done`.

## Verification

1. `python3 tools/smoke_sase_core_rs_telemetry --print-minimum pyproject.toml` prints the published version containing
   `81a82d5` and `6468cb9`, and `check_sase_core_rs_bindings` against that exact version from the index exits 0 with no
   missing bindings.
2. A clean venv holding only that published minimum reads the live beads sidecar without raising
   `unknown variant note_appended`.
3. sase-core master CI is green, and `cargo clippy --workspace --all-targets -- -D warnings` is clean.
4. `just check` passes in this repo.
5. `sase bead history --lost-notes` is either empty or the deferral is recorded as a note on `sase-bd.8`.
6. `sase bead show sase-bd` reports the epic closed, and `sase bead doctor` still reports no projection drift with a
   recent-window redundant-close count that has not grown.
