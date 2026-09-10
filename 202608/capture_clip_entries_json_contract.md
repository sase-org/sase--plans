---
tier: tale
title:
  Fix the capture clipboard JSON contract that breaks every % capture in Bob Mac Capture
goal:
  Every `%` clipboard capture submitted from Bob Mac Capture decodes successfully and is
  reported as a success, because `bob capture --format json` always emits `clip.entries`
  and the app decodes omitted capture collections tolerantly.
size: medium
proposed_by: bbugyi200.athena.01c
create_time: 2026-09-09 19:59:48
status: wip
---

# Fix the capture clipboard JSON contract that breaks every % capture

## Symptom

Submitting any draft containing a `%` clipboard marker from the Bob Mac Capture panel
fails with:

```
bob command produced malformed JSON (exit 0): /Users/bbugyi/.cargo/bin/bob capture --format json -- <capture-text>. The data couldn’t be read because it is missing..
```

## Diagnosis (root cause confirmed, not hypothesized)

The message is `BobClientError.malformedJSON`, thrown from
`BobProcessClient.decodeCaptureResult` in bob-mac-capture's
`Sources/CaptureCore/BobProcessClient.swift`. Its `reason` is the
`error.localizedDescription` of a Swift `DecodingError`; on macOS a `keyNotFound` (or
`valueNotFound`) bridges to exactly "The data couldn't be read because it is missing."
So `bob` succeeded (exit 0) and printed well-formed JSON — the _app_ refused to decode
it.

The missing key is `clip.entries`.

**bob-cli side.** In `src/native/capture_clip.rs`, `ClipOutput.entries` carries
`#[serde(skip_serializing_if = "Vec::is_empty")]`:

```rust
#[serde(skip_serializing_if = "Vec::is_empty")]
pub(crate) entries: Vec<ClipOutput>,
```

`entries` is non-empty only for a multi-value history clip (`%<N>`). It is therefore
absent from the JSON for every single-value clip, and absent from every _nested_ entry
of a history clip.

**bob-mac-capture side.** In `Sources/CaptureCore/CaptureModels.swift`,
`CaptureClipOutput` declares:

```swift
public let entries: [CaptureClipOutput]
```

with no custom `init(from:)`. Swift synthesizes `decode(_:forKey:)` — not
`decodeIfPresent` — for a non-optional array, so the key is mandatory. Decoding the real
payload throws `DecodingError.keyNotFound("entries")`, which aborts the whole
`CaptureCommandResponse` decode.

**Verified against the real binary.** All four clip modes omit `entries` (`--dry-run`
output, elided for width):

```
%          -> "clip":{"header":null,"mode":"inline","lines":["\t- hello-clip"],"attachments":[]}
%mynote    -> "clip":{"header":"MYNOTE","mode":"inline","lines":["\t- **MYNOTE:** hello-clip"],"attachments":[]}
% (file)   -> "clip":{"header":null,"mode":"attachments","lines":["\t- ![[img/shot.png|400]]"],"attachments":[{"source":"/tmp/shot.png","saved":"img/shot.png","kind":"image","reused":false}]}
% (long)   -> "clip":{"header":null,"mode":"snippet","lines":["\t- [[file/clip-…]]"],"attachments":[],"snippet":"file/clip-….md"}
%2         -> "clip":{…,"mode":"history","entries":[{"header":null,"mode":"inline","lines":["\t- hello-clip"],"attachments":[]}, …]}
```

Note the `%2` case: the _top-level_ `entries` is present, but each nested entry still
omits its own `entries`, so history captures fail too. Decoding the first payload above
against the unmodified `CaptureModels.swift` reproduces the failure exactly:

```
DecodingError.keyNotFound: Key 'entries' not found in keyed decoding container.
```

**Blast radius.** Every clipboard-producing capture fails to decode: `%`, `%<header>`,
`%<N>`, `--clip`, and `--clip=HEADER`, on both the submit path
(`capture(_:dryRun:false)`) and the explicit clipboard preview path (`preview()`). The
_live_ preview path is unaffected because it forces `--no-clip`, so no `clip` object is
ever emitted — which is why a draft containing `%` looks perfectly healthy right up
until the user submits it.

`capture-parse`, `capture-complete`, and `capture-targets` are unaffected; `%` spans are
reported correctly by `bob capture-parse`, so syntax highlighting was never broken.

**Severity beyond the error message.** The failing invocation is not a dry run. bob
already wrote the note and any attachment or snippet files before printing the JSON the
app rejects. The app takes the `catch` branch, calls `failSubmit`, keeps the draft, and
shows an error — so the natural user response (retry) writes the capture a second time.
The user-visible bug is "`%` doesn't work"; the silent bug is duplicate captures.

**Why the existing tests missed it.** bob-cli's
`capture_percent_one_is_an_exact_single_clip_alias` in `tests/cli.rs` actively pins the
omission (`assert!(json["clip"].get("entries").is_none())`), and bob-mac-capture's
`testCaptureCommandResponseDecodesRealSuccessShapeWithNoSchemaVersion` uses a payload
with no `clip` at all (`XCTAssertNil(success.clip)`). Neither side has ever decoded a
real clip payload.

## Fix strategy

Fix both sides. They are not redundant:

- **bob-cli** stops omitting `entries`, which removes the mismatch at its source and
  makes `lines`, `attachments`, and `entries` uniformly present on every `ClipOutput`.
  This alone restores `%` for the currently installed app after a plain
  `cargo install --path .`, with no Xcode rebuild.
- **bob-mac-capture** decodes omitted collections tolerantly, so the app also works
  against any already-installed `bob` that predates the change, and so this failure
  class cannot recur.

Do not "fix" this by making `entries` optional in Swift and leaving the contract
ambiguous, and do not add a `schema_version` to `bob capture` — the intentional absence
of one is documented in `CaptureModels.swift` and is out of scope here.

## Part A — bob-cli (this repository)

1. In `src/native/capture_clip.rs`, remove the
   `#[serde(skip_serializing_if = "Vec::is_empty")]` attribute from `ClipOutput.entries`
   so a leaf clip serializes `"entries":[]`. Leave `snippet`'s
   `skip_serializing_if = "Option::is_none"` alone: it maps to a Swift `String?`, which
   already decodes an absent key as `nil`.
2. Add a short comment on the field recording _why_ it is always emitted:
   bob-mac-capture decodes `clip` into a struct whose collections are required, and an
   omitted `entries` is what broke every `%` capture.
3. Update `tests/cli.rs`:
   - In `capture_percent_one_is_an_exact_single_clip_alias`, replace
     `assert!(json["clip"].get("entries").is_none(), "{json}")` with an assertion that
     `json["clip"]["entries"]` equals `serde_json::json!([])`.
   - In `capture_history_is_headerless_structured_and_composes_with_routes`, add an
     assertion that each nested entry also carries `entries: []`, so the nested-leaf
     shape is pinned too.
   - Add a regression test asserting that `lines`, `attachments`, and `entries` are all
     present on the `clip` object of a plain `%` capture, named so its purpose is
     obvious (for example `capture_clip_json_always_emits_collection_fields`). Assert
     presence of the keys, not just their values, since presence is the contract that
     broke.
4. Grep for any other consumer expectations before finishing: `clip.entries` appears
   nowhere in `docs/` or `scripts/`, and bob-mac-capture is the only consumer of
   `bob capture --format json`, so this is a safe additive shape change.

Validate with the repository's own gate:

```sh
just all
```

Then confirm the shape by hand against a scratch vault:

```sh
BOB_DIR=<scratch-vault> BOB_CLIPBOARD_CMD="printf hello-clip" \
  cargo run --bin bob -- capture --dry-run --format json -- "test %"
```

The `clip` object must now contain `"entries":[]`.

## Part B — bob-mac-capture (linked repository)

Open the repo with `sase repo open bob-mac-capture -r "<reason>"` and work only under
the path it prints. Never edit an installed app bundle or a hand-cloned copy.

1. In `Sources/CaptureCore/CaptureModels.swift`, give `CaptureClipOutput` a hand-written
   `init(from decoder: Decoder) throws` that decodes `lines`, `attachments`, and
   `entries` with `decodeIfPresent(...) ?? []`, keeps `header` and `snippet` as
   `decodeIfPresent`, and keeps `mode` required. Follow the existing precedent in this
   same file: `CaptureTargetsResponse.init(from:)` already does exactly this for
   `targets`. Add a `private enum CodingKeys` only if one is needed; leave `Encodable`
   synthesis untouched.
2. Apply the same tolerance, as hardening against an identical future outage, to the
   other required collections decoded from bob output: `CaptureParseResponse`'s `needs`,
   `spans`, and `diagnostics`, and `CaptureCompletionResponse`'s `candidates`. These
   keys are always emitted by bob today, so this must be a pure no-op for present keys —
   do not change any other field's optionality or any `CodingKeys` mapping.
3. Replace the stale comment on `CaptureClipOutput` — and the one above
   `CaptureCommandResponse` claiming the models mirror bob's actual JSON — with an
   accurate statement of the convention: bob omits empty collections and `None` scalars,
   so every collection decoded from bob must use `decodeIfPresent ?? []`.
4. Add regression tests to `Tests/CaptureCoreTests/CaptureModelTests.swift` that decode
   **verbatim** bob output (the payloads reproduced in the Diagnosis section above,
   which were captured from a real binary — do not hand-tidy them into a shape bob never
   emits). Cover, at minimum:
   - a headerless single clip (`%`) — asserts `clip?.entries` is empty and does not
     throw;
   - a headered clip (`%mynote`) — asserts `header == "MYNOTE"`;
   - an attachments clip — asserts the decoded `CaptureAttachmentOutput`;
   - a snippet clip — asserts `snippet` decodes and `entries` is empty;
   - a history clip (`%2`) — asserts the nested entries decode even though each nested
     object omits its own `entries`.
5. Add a `BobProcessClientTests` case in the style of the existing stub-driven tests
   that drives `capture(_:)` against a stub emitting the single-clip payload and asserts
   a `.success` response rather than a thrown `malformedJSON`, so the end-to-end submit
   path is covered and not just the model.

## Verification

The bob-mac-capture package declares `platforms: [.macOS("26.0")]`, so `just build`,
`just test`, and `just format-lint` cannot run on a Linux agent host. Do not report the
Swift change as verified on the strength of a code read.

A Linux implementing agent **can and must** prove the decode fix locally, because
`Sources/CaptureCore/CaptureModels.swift` is Foundation-only and typechecks standalone:

```sh
# Type-checks the edited model file on Linux.
swiftc -typecheck Sources/CaptureCore/CaptureModels.swift

# Proves the real payload now decodes. Write a small main that decodes the verbatim
# single-clip JSON, concatenate it onto the real model source, and interpret the result.
cat > /tmp/harness_main.swift <<'SWIFT'
let clipJSON = #"{"header":null,"mode":"inline","lines":["\t- hello-clip"],"attachments":[]}"#
do {
    let clip = try JSONDecoder().decode(CaptureClipOutput.self, from: Data(clipJSON.utf8))
    print("CLIP DECODED OK entries=\(clip.entries.count) lines=\(clip.lines)")
} catch {
    print("CLIP FAILED: \(error)")
}
SWIFT
cat Sources/CaptureCore/CaptureModels.swift /tmp/harness_main.swift > /tmp/harness.swift
swift /tmp/harness.swift
```

The harness must print a success rather than
`DecodingError.keyNotFound: Key 'entries' not found`. Both commands were confirmed to
work on this project's Linux host against the current, unfixed source, so a failure to
run them is a problem to solve, not a reason to skip verification.

On top of that:

- Run `just all` in bob-cli; it must pass.
- Push the bob-mac-capture branch so the `macOS 26 SwiftPM` CI workflow runs
  `format-lint`, `build`, `test`, `bundle`, and the launch smoke test. Treat green CI as
  the authoritative Swift verification.
- If no macOS 26 machine or CI run is available, say so explicitly and record the
  SwiftPM suite as outstanding — do not claim the app is fixed.

Final manual smoke test on the Mac, after `cargo install --path .` for bob-cli and
reinstalling the app bundle:

1. Copy a short string, open the capture panel, type a draft ending in `%`, and submit.
   The panel must report success, clear the draft, and the note must contain the
   clipboard child bullet exactly once.
2. Repeat with `%mynote` and confirm the bolded header child bullet.
3. Repeat with `%3` against clipboard history and confirm three children and no error.
4. Copy a file path, submit with `%`, and confirm the attachment is saved and embedded.
5. Use the explicit clipboard preview action on a `%` draft and confirm it renders
   instead of erroring.

## Acceptance criteria

- `bob capture --format json` emits `clip.entries` on every clip object, including every
  nested history entry.
- bob-mac-capture decodes every clip payload bob can emit — inline, headered,
  attachments, snippet, and history — with no thrown `malformedJSON`.
- bob-mac-capture still decodes a clip payload whose `entries` key is absent, so an
  older installed `bob` keeps working.
- Both repositories carry regression tests that fail if either half is reverted.
- `just all` passes in bob-cli; bob-mac-capture CI is green on macOS 26.
- No behavior change for captures with no clipboard marker, and no change to
  `capture-parse`, `capture-complete`, or `capture-targets` output.

## Out of scope (record, do not implement)

A decode failure on the submit path currently surfaces as a plain capture failure even
though bob already wrote the note, which invites a duplicate-writing retry. Making the
app distinguish "bob succeeded but its output could not be read" from "the capture
failed" is a real robustness gap, but it is a separate design question about the submit
path's error taxonomy. Propose it as a follow-up task bead via `/sase_new_task` rather
than folding it into this fix.
