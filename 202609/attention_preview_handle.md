---
tier: tale
title: Close the attention preview gap and verify sase-xe.14
goal:
  A pending remote question or gate carries a bounded, sanitized, digest-validated
  preview of its decision and command detail, served through the existing fleet content
  route, so the attention modal genuinely blocks approval until the reviewer can see
  what they are approving; both repositories verify clean and sase-xe.14 closes with a
  real verification note.
size: medium
proposed_by: bbugyi200.athena.sase-xe.14.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.sase-xy.5.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xy.5.2/README.md)
  - [bbugyi200.athena.sase-xy.5.3](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xy.5.3/README.md)
  - [bbugyi200.athena.sase-xy.5.4](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xy.5.4/README.md)
  - [bbugyi200.athena.sase-xy.5.5.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xy.5.5.1/README.md)
  - [bbugyi200.athena.sase-xy.5.5.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xy.5.5.2/README.md)
  - [bbugyi200.athena.sase-xy.5.5.3](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-xy.5.5.3.md)
  - [bbugyi200.athena.sase-xy.5.5.4.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xy.5.5.4.1/README.md)
  - [bbugyi200.athena.sase-xy.5.5.4.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xy.5.5.4.2/README.md)
  - [bbugyi200.athena.sase-xy.5.5.4.3](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-xy.5.5.4.3.md)
  - [bbugyi200.athena.sase-xy.5.5.4.land](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-xy.5.5.4.land.md)
- **COMMITS:**
  - [d8a299c](https://github.com/sase-org/sase/commit/d8a299c2c3e401d9a08f5802849c15afb3eceb7b)
    — feat(pager-refs): add Python adapter for document-owned source-path resolution
  - [2b08ca3](https://github.com/sase-org/sase/commit/2b08ca3017aade6523ab8fcb16d36460f6ed8866)
    — feat(pager): carry semantic targets and owner provenance through every action
  - [2ba228d](https://github.com/sase-org/sase/commit/2ba228da326eedba37e66c5b8eb73ed4525b2967)
    — test(pager): enforce rendered-link contract through real navigation
  - [d101fbd](https://github.com/sase-org/sase/commit/d101fbd08657694233523cca298a526ada3b2609)
    — feat(artifact-ref): transport optional source path_globs on document owner
  - [e38f065](https://github.com/sase-org/sase/commit/e38f065b8ba25478f38a2edae7c8a94fa7c2abe7)
    — feat(pager): preserve semantic target identity
  - [b67c74c](https://github.com/sase-org/sase/commit/b67c74ce7ecf2268a3abba0d61e0fdbabf7f56e1)
    — feat(artifact-ref): ratchet sase-core-rs floor to 0.32.41 and extend contract
    validation
  - [eccc091](https://github.com/sase-org/sase/commit/eccc0916048d844dfeaa64ee13ae4a411df2cd35)
    — fix(artifact-ref): send selected_project and honor owner project in context
    assembly
  - [3763cce](https://github.com/sase-org/sase/commit/3763cce8fb584b3af323742a062606cbf7afc0e0)
    — fix(pager): honor owner-scoped copy and freeze configured kinds
  - [ce3d670](https://github.com/sase-org/sase/commit/ce3d6708714127486f069a04fbba065350e289a6)
    — chore(deps): ratchet sase-core-rs to 0.32.42
  - [6b7edb2](https://github.com/sase-org/sase/commit/6b7edb2dda9b54e0ce6d078ab77a231a895c0057)
    — fix(pager): freeze configured kinds on the ACE commit manifest

# Plan: Close the attention preview gap and verify sase-xe.14

## Objective

Finish the one unmet acceptance criterion of phase `sase-xe.14` — "Opening the attention
modal shows the request title, options, and a bounded, digest-validated preview of the
decision and command detail before approval is possible, and never exposes a remote
path" — then verify both repositories and close the phase bead with a real verification
note.

The phase's implementation landed in sase commit `287048d60` and sase-core commit
`b19c603`. The prior agent deliberately stopped short of the preview work and did not
close the bead; `sase stitch create` auto-closed it anyway, leaving a note that
explicitly disclaims verification and invites a reopen. This plan finishes the work the
prior agent left, not the parts it already completed.

## Background: what is already true

Verified against the committed trees before writing this plan. Do not redo this
discovery.

- `FleetAttentionEntryWire` already declares `pub preview: Option<ContentHandleWire>`
  and already validates it through `validate_content_handle`. The projection
  `project_fleet_attention` hard-codes `preview: None`, so no entry ever carries a
  handle and the modal's preview gate never engages. That single hard-coded `None` is
  the whole acceptance gap.
- `RemoteAttentionModal` already implements the gate correctly: with a handle present it
  keeps Submit disabled, fetches through `RemoteContentClient` on a thread worker, and
  enables Submit only after the chunk decodes. No modal rework is needed to make the
  gate function — only a populated handle.
- Content is served by the existing `POST /api/fleet/v1/content` route under the
  existing `fleet.content.read` scope, resolved through
  `CachedFleetSnapshot.content_by_handle: BTreeMap<String, FleetContentSource>`. **No
  new route, no new scope, and no new capability is required by this plan.**
- The gate's decision-and-command detail lives in the gate bundle `request.json`
  addressed by `notification.action_data["request_path"]`.
  `gate_branches_from_notification` in `crates/sase_core/src/notifications/mobile.rs`
  already reads exactly that path, so the file is established, readable owner-side
  input. Its per-option `command` field is present on disk (written by
  `src/sase/notification_gates/model_options.py`) but discarded by the mobile
  `GateEnvelopeWire`, which deserializes only `options`/`groups`/`branches`.
- For a question, the same `action_data["request_path"]` key addresses the session's
  `question_request.json`.
- `sase bead epic-symbols sase-xe.14` reports no `--epic-symbol` entries.
- Parent epic `sase-xe` is `IN_PROGRESS`, so reopening `sase-xe.14` disturbs no closed
  ancestor.

Two follow-ups the prior agent proposed are already resolved and must not be re-done:

- The 100-row bound on the CLI's non-ACE fallback is already present in
  `src/sase/ops/commands/machine_attention.py` (`{"schema_version": 1, "limit": 100}`).
- The `sase-core-revision.txt` pin bump is **not** an agent task.
  `.github/workflows/core-pin-ratchet.yml` runs on a 6-hour schedule and opens its own
  PR; the pin already trails two earlier phases, so its lag is normal steady state, not
  a `sase-xe.14` regression. Do not hand-edit that file.

## The two real defects to fix

### 1. The preview handle is never produced (the acceptance gap)

Nothing populates `FleetAttentionEntryWire.preview`, so "before approval is possible" is
vacuous today.

Serving the gate bundle's raw `request.json` bytes is **not** an acceptable fix: that
file carries `response_dir`, bundle paths, and other owner-local paths, which would
violate the same "never exposes a remote path" criterion the preview is meant to serve.
The preview must be a document the owning host _renders and sanitizes_, not a file it
streams.

### 2. `expected_digest` is dead code, so "digest-validated" is not true

In `src/sase/dispatch/content.py`:

```python
if expected_digest and expected_digest != digest and not reported:
    raise RemoteContentError("content digest does not match handle")
```

The host always sends `sha256`, so `reported` is always truthy and the `expected_digest`
branch is unreachable. The handle-digest binding is never enforced. Combined with
`digest: None` on every handle produced today, a host could return bytes that differ
from what the handle advertised and the client would accept them. The preview criterion
says _digest-validated_, so this must actually hold.

The fix must not break tailing: for a growing, chunked read the chunk digest
legitimately differs from the whole-document digest. Enforce `expected_digest` only when
the response covers the whole document (`offset == 0` and `eof` true), which is exactly
the bounded single-shot preview case.

## Implementation

Open the Rust core repository with the `/sase_repo` skill before touching any
`crates/...` path; every such path below is relative to that checkout. Read
`lint_and_test.md` and `tui_perf.md` with `/sase_memory_read` first. Do not edit
anything under `sase/memory/`.

### Step 0 — Rebuild the stale binding and establish a real baseline

The ephemeral workspace clone's virtualenv carries a **stale** `sase_core_rs` wheel: it
exposes the phase-`.13` mutation bindings but none of the five `.14` attention bindings,
so `tests/test_dispatch_attention_notices.py` fails there with
`AttributeError: ... does not expose binding 'fleet_decide_attention_notices'`. This is
environmental, not a code defect.

Run `just install` first. It runs `tools/validate_sase_core_rs` — which already requires
`fleet_project_attention`, `fleet_attention_payload_fingerprint`,
`fleet_validate_attention_request`, `fleet_evaluate_attention_precondition`, and
`fleet_decide_attention_notices` — and triggers `rust-install` to rebuild the binding
from the linked sase-core checkout. Then re-run the phase's test files and confirm they
pass before changing anything, so later failures are attributable to this plan's edits.

Use `/sase_monitor` for `just install` if it runs long.

### Step 1 — Render a bounded, sanitized preview document in the Rust core

In `crates/sase_core/src/fleet_attention.rs`:

- Add an optional owner-supplied detail input to `FleetAttentionNotificationRowWire`.
  That struct is `#[serde(deny_unknown_fields)]`, so the new field must carry
  `#[serde(default)]` to stay backward compatible with payloads that omit it. It holds
  the raw text the gateway read from the request bundle; the core never opens a file
  itself.
- Add a pure renderer that turns that raw detail into a deterministic preview document:
  - For a **gate**: one section per option, giving the option ID, its label, and its
    command detail, in the order the envelope declares. This is the "decision and
    command detail" the criterion names, and it is the reviewer's whole reason to
    approve or refuse.
  - For a **question**: the question text and the offered choices.
  - Bound the whole document with a new `MAX_ATTENTION_PREVIEW_BYTES` constant, on a
    character boundary, appending a visible truncation marker so a reviewer is never
    silently shown a partial command.
  - Drop the owner-local keys outright rather than echoing them: `response_dir`,
    `request_path`, `artifacts_dir`, and any bundle path. Run `reject_secretish` over
    the rendered document and refuse to publish a preview that trips it, rather than
    emitting a redacted one — a preview that cannot be rendered safely must be absent,
    and an absent preview is a state the modal already handles.
  - A command string may legitimately contain a path, and the reviewer must see it to
    judge the command. The criterion's "never exposes a remote path" governs the entry's
    _contract fields_ — already enforced by `reject_path_like` on titles, summaries, and
    labels — not the command text the reviewer is being asked to approve. Do not strip
    paths out of command bodies; that would defeat the preview's purpose.
- Compute the document's SHA-256 and build the `ContentHandleWire` with
  `digest: Some(<hex>)`, `byte_len: Some(<len>)`, `supports_range: true`,
  `supports_growth: false`. Derive `id` from the attention request key and revision
  (mirroring `content_handle_id`'s hashing style, with its own domain-separation prefix)
  so a superseded-and-re-asked request yields a new handle rather than a stale cache
  hit.
- Set `preview: Some(handle)` on the entry, keeping `None` when the row supplied no
  detail or the document failed the safety check. Keep the projection deterministic and
  its existing sort and dedupe behavior unchanged.
- Expose the rendered bytes to the caller alongside the snapshot so the gateway can
  register them; the snapshot wire type itself must keep carrying only the handle.

Unit-test in-module: a gate row renders options and commands; a question row renders the
question and choices; an oversized document truncates on a character boundary with the
marker; a `reject_secretish` hit yields `preview: None` rather than a redacted document;
`response_dir`/`request_path`/`artifacts_dir` never appear in output; the handle's
digest equals the SHA-256 of the rendered bytes; a changed revision changes the handle
ID; the projection stays deterministic across repeated calls.

### Step 2 — Serve the preview through the existing content route

In `crates/sase_gateway/src/`:

- The preview is generated, not a file on disk, so `FleetContentSource` must be able to
  hold in-memory bytes as well as a canonical path. Give it a source variant and teach
  `read_content_range` to serve the byte case: honor `offset`/`limit` the same way,
  return the same `sha256`-of-returned-chunk, `total_byte_len`, `next_offset`, and `eof`
  fields, and skip the on-disk canonicalize/containment re-check, which is meaningless
  for bytes that never came from the filesystem. Leave the path case, including its
  containment re-check, exactly as it is.
- Where attention is projected for a host, read
  `notification.action_data["request_path"]` for each pending row and pass its contents
  in as the new row detail field. Bound the read so a large or hostile bundle cannot
  blow up the snapshot, and treat an unreadable or oversized file as "no detail" — the
  entry is still returned with `preview: None`, matching the existing rule that an
  uncorrelated or detail-less request is never silently dropped.
- Register each rendered preview into `content_by_handle` keyed by the handle ID, with
  the attention entry's row revision as the source's `row_revision`, so the existing
  `/content` route's revision precondition applies to previews unchanged and a stale
  preview read is refused the same way a stale artifact read already is.
- Add gateway tests: a preview handle round-trips through `/content` and its bytes hash
  to the handle's advertised digest; a read with a mismatched `row_revision` is refused;
  an unknown handle ID is a not-found; an entry whose row supplied no detail carries no
  preview and serves nothing.
- Regenerate the committed contract snapshot with
  `UPDATE_FLEET_CONTRACT=1 cargo test -p sase_gateway committed_fleet_contract_snapshot_is_current`
  and commit the regenerated file. Confirm `/content` is unchanged in it — this plan
  adds no route.

Run `./scripts/check.sh all` in the sase-core checkout and fix everything it reports,
including `cargo fmt`.

### Step 3 — Make the client actually enforce the handle digest

In `src/sase/dispatch/content.py`, fix `_decode_and_validate` so `expected_digest` is
enforced when the payload covers the whole document (`offset == 0` and `eof` true), and
only chunk-digest-checked otherwise. Keep the existing chunk-`sha256` check for every
response. `_decode_and_validate` needs whatever offset/eof context it currently lacks;
thread it from the payload rather than widening the caller's contract.

Tests: a whole-document response whose bytes disagree with the handle digest raises
`RemoteContentError`; a whole-document response that agrees is accepted; a chunked or
growing read with a whole-file handle digest is still accepted and still validates its
chunk `sha256`; a corrupted chunk still raises.

### Step 4 — Confirm the modal gate now engages end to end

No `RemoteAttentionModal` rework is expected. Confirm and cover:

- Add or extend a test that with a populated preview handle Submit starts disabled and
  becomes enabled only after the preview resolves — the existing preview-gate test must
  exercise a real handle, not the handle-absent path, or it proves nothing.
- Add a test that a preview whose digest fails validation leaves Submit disabled and
  surfaces the failure, so a reviewer can never approve against unverified bytes.
- Keep the fetch off the event loop and off the message pump per `tui_perf.md`; the
  modal's existing thread worker already satisfies this, so do not introduce a new path.

### Step 5 — Verify both repositories, then reopen and close the bead

- `just install`, then `./scripts/check.sh all` in the sase-core checkout, then
  `just check` in the sase repo. This change touches the Rust wire contract and its
  Python consumers, so run `just check-full` through `/sase_monitor` and let it finish;
  a scoped lane can miss a consumer of a changed binding.
- `sase-core` is a repository opened through `/sase_repo`, so it carries its own commit
  obligation in this turn's final declaration alongside the sase repo. Both must be
  committed.
- Record anything genuinely out of scope only as
  `sase bead note sase-xe.14 'PROPOSED FOLLOW-UP: <summary — detail>'`. Do not create
  task beads.
- Re-run `sase bead epic-symbols sase-xe.14` and resolve or re-key any entry.
- The bead is currently closed by the stitch finalizer, and `close --note` on an
  already-closed bead is a no-op that records nothing. To land a real verification note,
  run `sase bead open sase-xe.14` and then
  `sase bead close sase-xe.14 --note "<what you actually verified>"`. The note must name
  the lanes that ran and their results. Close **only** `sase-xe.14`; never close
  `sase-xe` or any ancestor, and do not touch `sase-xe.15`.

If a verification lane fails and cannot be fixed within this plan, leave the bead open,
report the failure with its output, and do not close on a red tree.

## Acceptance criteria

- A pending remote gate's attention entry carries a preview handle whose document lists
  each option's ID, label, and command detail, and a pending remote question's entry
  carries one with the question text and choices.
- The preview document is bounded, truncates on a character boundary with a visible
  marker, and contains no `response_dir`, `request_path`, `artifacts_dir`, or bundle
  path. A document that trips `reject_secretish` yields no preview rather than a
  redacted one.
- The handle advertises `digest` equal to the SHA-256 of the rendered document, and the
  bytes served by the existing `/content` route hash to it.
- A preview read against a mismatched row revision is refused, and an unknown handle ID
  is not found.
- `src/sase/dispatch/content.py` refuses a whole-document response whose bytes disagree
  with the handle digest, while chunked and growing reads keep working.
- With a preview handle present, the modal's Submit is disabled until the preview
  resolves, and stays disabled when the digest fails.
- An attention entry whose row supplied no readable detail still appears, with no
  preview, and stays answerable — a missing preview never drops a request.
- No new fleet route, scope, or capability is introduced; the committed contract
  snapshot is regenerated and shows `/content` unchanged.
- `./scripts/check.sh all` passes in sase-core and `just check-full` passes in the sase
  repo, both after `just install` has rebuilt the binding.
- `sase-xe.14` is closed with a note naming the verification actually performed, and
  `sase-xe` and `sase-xe.15` are untouched.

## Out of scope

- The `sase-core-revision.txt` pin bump, which the scheduled `core-pin-ratchet` workflow
  owns.
- The CLI fallback row bound, already present.
- Any change to `sase-xe.15`'s flag-removal and fleet-wide acceptance work.
- Any new content-serving route, scope, or capability.
