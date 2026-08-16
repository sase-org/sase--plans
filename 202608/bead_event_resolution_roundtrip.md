---
tier: tale
size: medium
title: Fix non-round-trip-stable bead event resolution field wedging the bead store
goal:
  Make bead event stream serialization byte-stable across a read/write round trip so the
  append-only integrity guard stops rejecting every bead store commit, repair the
  committed store junk, and relaunch the epic that the wedge blocked.
proposed_by: bbugyi200.athena.038
create_time: 2026-08-15 22:21:37
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.038](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.038.md)
  - [bbugyi200.athena.038.f1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.038.f1/README.md)
- **COMMITS:**
  - [f935aca](https://github.com/sase-org/sase/commit/f935acacee35d7261aa3b4dbe0bd57342e09d43d)
    — fix(bead): name concrete field diff in append-only rewrite guard message
  - [5184f5a](https://github.com/sase-org/sase/commit/5184f5ab0ad992c56f967c39a352b4d51edc5a7c)
    — fix(plan-gate): accept every EpicLaunchMode in the epic approve schema

# Fix non-round-trip-stable bead event `resolution` field wedging the bead store

## Problem

Every bead store mutation currently fails to commit, push, or roll back with:

```
cannot publish non-append-only bead event stream sase-mk: worktree rewrote ancestor event 5
```

This wedged `sase bead work` for the `primary_workspace_ownership.md` epic: the epic
graph commit failed, the rollback failed with the same error, and even the "restore
launch state" commit failed. It also stranded an unrelated agent's bead note as an
uncommitted change in the store (`events/streams/sase-m9.3.1.jsonl`, `issues.jsonl`).

The store is _not_ actually corrupt. The guard is correct and is reporting a real
non-append-only rewrite; the rewrite is produced deterministically by sase-core on every
single write.

## Root cause (confirmed by reproduction)

`crates/sase_core/src/bead/events.rs:354-355` in the sibling `sase-core` repo declares
the bead `issue_updated` payload's `resolution` field as an asymmetric double option:

```rust
#[serde(default, skip_serializing_if = "Option::is_none")]
pub resolution: Option<Option<BeadResolutionWire>>,
```

The outer `Option` encodes "is this field being updated at all"; the inner `Option`
encodes the new value. Serde's _derived_ deserializer maps JSON `null` onto the
**outer** `None`, not `Some(None)`. So the two halves disagree:

- Serializing `Some(None)` (an explicit "clear the resolution") emits
  `"resolution": null`.
- Deserializing `"resolution": null` yields `None` ("field absent").
- Re-serializing that `None` **omits the key** because of `skip_serializing_if`.

The event is therefore not round-trip stable. `event_id` is carried through verbatim, so
a re-serialized event keeps its identity while changing its bytes — which is exactly the
signature the append-only guard names as a rewrite.

`resolution` is the only field in the struct carrying that asymmetric attribute pair.
Its double-option siblings `closed_at` and `close_reason` (`events.rs:350-353`) have no
`skip_serializing_if`, so they always emit `null` and round-trip cleanly.

### Why one bead wedges the whole store

`write_event_store` (`crates/sase_core/src/bead/jsonl.rs:288-306`) re-serializes and
rewrites **every** stream file on every mutation, not just the streams that changed. So
any bead write anywhere re-emits `sase-mk.jsonl`, re-drops the key, and trips the guard.
That is also why the failed launch reported `slow_launch_stage ... elapsed_ms=200901.7`:
each stage rewrote all 851 stream files.

### Reproduction (already performed)

Copy the committed `sase-mk.jsonl` into a scratch event store and apply any mutation
through the live binding:

```bash
mkdir -p /tmp/beadprobe/events/streams
cp <beads>/events/streams/sase-mk.jsonl /tmp/beadprobe/events/streams/
printf '{"schema_version":1,"stream_count":1,"generated_from":"issues.jsonl","migration_tool":"sase-core bead events"}' \
  > /tmp/beadprobe/events/manifest.json
touch /tmp/beadprobe/issues.jsonl
python -c "import sase_core_rs as m; m.bead_append_note('/tmp/beadprobe','sase-mk','probe')"
```

Result: event 5's `event_id` is unchanged, its content is not, and the only difference
is the dropped `resolution` key. This is the exact failure the guard reports.

### Blast radius of the data itself is nil

A survey of all 851 committed streams (4439 `issue_updated` events) found exactly
**one** event carrying the `resolution` key at all — `sase-mk` event 5, written when the
user launched work on that task, which clears the resolution as part of the transition.
Every other `issue_updated` event omits the key entirely and round-trips
byte-identically today. So the fix changes the on-disk bytes of zero existing events.

## Approach

### 1. Fix the wire type in `sase-core`

Open the repo with `sase repo open sase-core -r "<reason>"` and work only in the path it
prints.

Use the double-option deserializer the same repo already defines for exactly this
purpose — `deserialize_present_option` at `crates/sase_core/src/procs/wire.rs:442-450`:

```rust
fn deserialize_present_option<'de, D, T>(deserializer: D) -> Result<Option<Option<T>>, D::Error>
where D: Deserializer<'de>, T: Deserialize<'de>
{
    Option::<T>::deserialize(deserializer).map(Some)
}
```

Promote it to a shared location both modules can use (a small `serde` helper module in
`crates/sase_core/src`) rather than copying it, and point `procs/wire.rs` at the shared
copy so there is one implementation. Then apply it to the bead field:

```rust
#[serde(
    default,
    deserialize_with = "deserialize_present_option",
    skip_serializing_if = "Option::is_none"
)]
pub resolution: Option<Option<BeadResolutionWire>>,
```

With that pairing the encoding becomes total and symmetric:

| on disk                | deserializes to    | re-serializes to       |
| ---------------------- | ------------------ | ---------------------- |
| key absent             | `None`             | key absent             |
| `"resolution": null`   | `Some(None)`       | `"resolution": null`   |
| `"resolution": "done"` | `Some(Some(Done))` | `"resolution": "done"` |

Do **not** "fix" this by dropping `skip_serializing_if`. That would start emitting
`"resolution": null` on all 4439 previously-clean events, rewriting every one of them
and turning a one-bead wedge into a whole-store wedge.

Confirm the replay semantics change is inert: the reducer at `events.rs:1435-1436` is
`if let Some(value) = &fields.resolution { issue.resolution = value.clone(); }`, so the
newly-decoded `Some(None)` clears the resolution. `sase-mk` is an open task whose
resolution is already `None`, so the projection is unchanged. Verify this rather than
assume it (see step 4).

### 2. Audit the same bug class

Grep the crate for every `Option<Option<...>>` field and check each one pairs a
presence-preserving deserializer with any `skip_serializing_if`. Known sites:

- `crates/sase_core/src/procs/wire.rs:214-397` — already uses
  `deserialize_present_option` throughout; confirm none were missed.
- `crates/sase_core/src/bead/events.rs:350-355` — `closed_at`, `close_reason` are safe
  (no `skip_serializing_if`); `resolution` is the bug.
- `crates/sase_core/src/bead/mutation.rs:100-105` — `BeadUpdateFieldsWire.resolution` is
  a plain derived double option, so a caller sending `{"resolution": null}` to mean
  "clear" is silently read as "leave unchanged". Record the finding; only change it here
  if the audit shows a live caller depends on clearing through that wire, otherwise file
  it as follow-up work.

### 3. Regression tests in `sase-core`

The gap that let this ship is that no test asserted round-trip byte stability of an
event stream. Add tests, not just a fixture tweak:

- A unit test in `crates/sase_core/src/bead/events.rs`'s test module asserting each of
  the three `resolution` encodings above survives `serde_json` decode → encode
  unchanged.
- A property-style or table-driven test over `BeadIssueUpdateWire` asserting that for
  every field, decode(encode(x)) == x and encode(decode(json)) == json.
- An event-store level test (`crates/sase_core/tests/bead_event_parity.rs`) that writes
  a stream containing `"resolution": null`, runs a mutation through `read_event_store` /
  `write_event_store`, and asserts the untouched stream's bytes are identical. This is
  the test that would have caught the wedge.

Add a fixture stream carrying `"resolution": null` so the golden files cover the shape.

Update `CHANGELOG.md` per the repo's release-please conventions.

Run the crate's own gates (`cargo test`, `cargo clippy`, `cargo fmt --check`, or the
repo's `just` equivalents) before committing.

### 4. Prove the store unwedges before touching it

With the rebuilt binding, re-run the reproduction harness from the Problem section
against a scratch copy and assert `sase-mk.jsonl` is byte-identical after the mutation.

Then verify the projection is unchanged across the real store, read-only: capture
`sase bead show sase-mk` (and the full `sase bead list` projection) before and after the
rebuild and diff them. They must match.

### 5. Repair committed store junk

Five `write_file_atomic` temp files were committed into the bead store and are still
tracked:

```
events/streams/.sase-1d.jsonl.tmp.3329931.395
events/streams/.sase-8z.jsonl.tmp.377002.50081
events/streams/.sase-9b.jsonl.tmp.1313192.39690
events/streams/.sase-dk.jsonl.tmp.927461.1017
events/streams/.sase-e6.jsonl.tmp.2850048.501
```

They are dead bytes: `is_event_stream_relpath` requires a `.jsonl` suffix, so the
integrity guard ignores them, but they are stale partial writes from July/August that
pollute the store. Remove them in their own commit, and add a `.gitignore` rule for
`events/streams/.*.tmp.*` so the next interrupted atomic write cannot be committed
again.

Do this **after** the store commits cleanly again, and confirm no live process is
mid-write first.

Also let the stranded uncommitted mutation in the store
(`events/streams/sase-m9.3.1.jsonl`, `issues.jsonl`) commit normally once the wedge
clears — it is a real agent's bead note, so preserve it rather than discarding it.

### 6. Make the guard's error actionable (this repo)

The guard message names an event number but not what changed, which is what made this
take a full investigation to identify. In `src/sase/bead/_stream_integrity.py`, extend
`_rewrite_message` (and the analysis that feeds it) so the error also names the concrete
difference — the keys added and removed, and whether the mismatch is a value change —
for the first rewritten event. Keep it a single compact line; the message is printed on
a failure path. Add tests alongside the existing ones in
`tests/test_bead/test_stream_integrity.py`.

This is scoped deliberately: presentation of an already-computed diagnosis is
presentation-layer work and belongs here, not behind the Rust boundary.

### 7. Deliver, then relaunch

Ordering matters, because `sase update -y` in dev mode fast-forwards clean sase-core
checkouts to their configured upstream before reinstalling editables. The user's global
`sase` is an editable install whose `sase_core_rs.pth` points at the primary sase-core
checkout, so the running `.so` is built from whatever that checkout resolves to.

1. Land the sase-core fix upstream (commit via the `/sase_git_commit` skill and get the
   patch merged) so the fast-forward cannot revert it. The primary sase-core checkout is
   currently one commit behind `origin/master`, so it will fast-forward.
2. Run `sase update -y`.
3. Re-run the reproduction harness once more against the _updated_ binary and confirm
   the round trip is byte-stable. Do not skip this: it is the only check that the fix is
   actually in the binary the user's shell runs.
4. Confirm the bead store commits cleanly (the previously stranded mutation lands).
5. Relaunch the epic:
   `sase bead work /home/bryan/.sase/plans/202608/primary_workspace_ownership.md -Y`

The failed launch left no residue to clean up first: `sase-mn` does not exist in
`issues.jsonl`, no `sase-mn*` stream file exists, and the archived plan carries no stale
`bead_id` link. The rollback's "could not remove epic sase-mn" was itself just the
commit failing. Re-verify this immediately before relaunching, in case another agent has
mutated the store in the meantime.

## Verification

- The reproduction harness produces a byte-identical `sase-mk.jsonl` after a mutation.
- New sase-core round-trip tests fail on the current code and pass on the fixed code.
- sase-core's own lint and test gates pass.
- `just check` passes in this repo for the `_stream_integrity.py` change.
- `sase bead show sase-mk` projection is unchanged before vs. after.
- A bead mutation commits and pushes cleanly.
- The epic launches and reports its 7 phase agents across 5 waves plus the land agent.

## Follow-up work to file (via `/sase_new_task`)

Do not fold these into this tale; file them and let the owner triage:

- `write_event_store` rewrites all 851 stream files on every mutation. This is why a
  single bad event wedges the entire store and why phase creation took 200s. It should
  write only the streams whose events changed.
- The `events/manifest.json` `stream_count` flip-flopped 851 → 852 → 851 across
  consecutive commits (`d1ee870f`, `420fd244`, both "repair event manifest"), which
  suggests two clones disagree about the stream set and keep repairing each other.
- `BeadUpdateFieldsWire.resolution` in `mutation.rs` cannot express "clear the
  resolution" over the wire (from step 2), if the audit shows it should be able to.

## Out of scope

- Changing the append-only guard's _rules_. The guard did its job — it refused to
  publish a genuine rewrite and restored the ancestor bytes. Only its error message
  improves here.
- Any history rewrite of the bead store. The committed `"resolution": null` event is
  legitimate and becomes stable once the decoder is fixed; no surgery is needed.
- The `primary_workspace_ownership` epic's own subject matter. That the user's
  production `sase` binary is bound to a live dev checkout is a real fragility, and it
  is precisely what that epic exists to address — this tale only unblocks its launch.
