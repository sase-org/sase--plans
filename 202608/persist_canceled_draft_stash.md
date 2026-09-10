---
tier: tale
title: Persist the bob-mac-capture canceled-draft stash across app restarts
goal:
  Canceled drafts retained with Control-C survive Quit, Restart, crash, and logout, and
  are restorable from the stash picker after the next launch.
size: medium
proposed_by: bbugyi200.athena.07p
create_time: 2026-09-09 20:00:17
status: wip
---

# Plan: Persist the canceled-draft stash across restarts

## Repository

All work happens in the **`bob-mac-capture`** linked repo, not in the `bob-cli`
workspace checkout. Open it first and use the printed path as the only path for reads
and writes:

```bash
sase repo open bob-mac-capture -r "Persist the canceled-draft stash across restarts"
```

No `bob-cli` (Rust) change is required. `bob-cli` has no stash concept — its capture
surface is only `capture-parse`, `capture-complete`, `capture-targets`,
`capture-sections`, `capture-tasks`, and `capture-task-id`. The stash is app-owned
presentation state, so persisting it app-side does not violate the "`bob-cli` owns
grammar/preview/completion/vault mutation" contract in the app README.

## Problem

The user's report is correct, and the cause is not a bug in the sense of a broken code
path — the stash was **designed** to be app-lifetime-only and simply has no persistence
at all:

- `Sources/BobMacCapture/CanceledDraftStash.swift:11-28` — `CanceledDraftStash` holds
  `@Published private(set) var entries: [CanceledDraftEntry] = []` and its `init` never
  reads anything. There is no encode, decode, or file I/O anywhere in the type.
- `Sources/BobMacCapture/AppDelegate.swift:9` —
  `lazy var canceledDraftStash = CanceledDraftStash(capacity: settings.canceledDraftStashCapacity)`
  constructs a fresh empty stash on every launch.
- `Sources/BobMacCapture/AppDelegate.swift:82-86` — `applicationWillTerminate` tears
  down the hotkey, watcher, and process client. It never saves the stash.
- Only `AppSettings` persists anything, and only the _capacity_ integer, to
  `UserDefaults` (`Sources/BobMacCapture/AppSettings.swift:18-26,94-99`).

Quit and Restart both route through `NSApp.terminate` (`AppRelauncher.restart()` at
`Sources/BobMacCapture/AppRelauncher.swift:42`), so every exit path drops the stash.

**This is therefore a deliberate behavior change, not a silent repair.** `README.md`
currently documents the current behavior as a guarantee in four places, including the
Privacy section: "it never writes captured text to disk itself" (`README.md:432-433`)
and "Canceled drafts retained with Control-C live only in the app-lifetime in-memory
stash … never written to `UserDefaults`, notifications, signposts, logs, or Diagnostics"
(`README.md:442-445`). Implementing this plan means captured text now lives on disk, and
the README must be rewritten to state the new, narrower guarantee honestly. Do not skip
the documentation work — in this repo the README is the spec.

## Environment constraints (read before starting)

The implementing agent runs on **athena (Linux)**, where the app target cannot be built:

- Swift 6.3.3 is available via swiftly. In any non-bash shell, set
  `export PATH="$HOME/.local/share/swiftly/bin:$PATH"` rather than sourcing `env.sh` (it
  uses bash-only `[[ ]]`).
- `swift build --target CaptureCore` — **works** (verified).
- `swift build --target CaptureCoreTests` — **works** and really does type-check the
  test sources (verified with a deliberately broken file).
- `swift build --target BobMacCapture`, `swift test`, `swift test --filter …`,
  `just build`, `just test` — **all fail on Linux** with `no such module 'AppKit'`,
  because `swift test` always builds every target including `BobMacCaptureTests`. Do not
  waste time trying to make these work locally.
- `swift format lint` from the Linux toolchain disagrees with this repo's style (it
  flags the entire existing tree, and exits 0 anyway). Do **not** use it as a gate;
  instead match the surrounding 4-space, trailing-comma Swift style by hand.

The real gate is GitHub Actions (`.github/workflows/ci.yml`, `runs-on: macos-26`), which
runs format lint, build, the full test suite, bundling, signature verification, and a
launch smoke test. **Push the branch and confirm CI is green before reporting done.**

This constraint drives the central design decision below: put everything that can be
unit-tested into `CaptureCore`, which is Foundation-only and Linux-type-checkable.

## Design

### Storage location and format

One JSON file, created lazily:

```
~/Library/Application Support/org.bobs.bob-mac-capture/canceled-draft-stash.json
```

```json
{
  "schemaVersion": 1,
  "drafts": [{ "id": "F0E1…", "text": "café\n- 子 task\n", "createdAt": 1755561234.5 }]
}
```

- `schemaVersion` mirrors the naming already used by `CaptureCore/CaptureModels.swift`.
- `createdAt` is encoded with `JSONEncoder.DateEncodingStrategy.secondsSince1970` (and
  the matching decoding strategy) — lossless and formatter/locale-free. It only feeds
  the "3m ago" row metadata; row order is the array order, newest first, exactly as in
  memory.
- The app is not sandboxed (it spawns `bob` via `Process` and owns a Carbon hotkey), so
  the real `~/Library/Application Support` is correct. Use the literal bundle identifier
  `org.bobs.bob-mac-capture` rather than `Bundle.main.bundleIdentifier`, which is `nil`
  under `swift test`.

### Write-through, not save-on-quit

Persist synchronously after **every** mutation (`push`, `remove`, `clear`,
`updateCapacity`), not in `applicationWillTerminate`.

Rationale: a terminate-only save reintroduces the exact bug class being fixed — force
quit, a crash, a logout `SIGKILL`, or a `just install` that replaces the bundle would
all still lose the stash. Mutations are user-driven and rare (one per Control-C,
restore, or clear), and the payload is at most 36 short drafts, so a synchronous atomic
write costs about a millisecond and needs no debounce, no flush hook, and no ordering
machinery. An async/coalesced writer was considered and rejected: it would need a
generation counter to keep out-of-order `Task.detached` writes from resurrecting stale
snapshots, for no measurable benefit at this size.

Consequence: `applicationWillTerminate` needs **no** change. Say so in a code comment so
a future reader does not "fix" the missing flush.

### Atomicity, permissions, and the empty-stash invariant

- Create the parent directory lazily on first save with
  `createDirectory(at:withIntermediateDirectories:attributes: [.posixPermissions: 0o700])`,
  and `setAttributes` 0700 on an already-existing directory. **`load()` must never
  create anything** — otherwise `AppDelegate()` construction inside `swift test` would
  litter the developer's real Application Support directory.
- Write with `Data.write(to:options: [.atomic])`, then `setAttributes`
  `[.posixPermissions: 0o600]` on the final path. The 0700 directory is the real access
  control; 0600 on the file is defense in depth, which is why the brief post-rename
  window at 0644 is acceptable.
- **Saving an empty array deletes the file** (and the quarantine file described below)
  instead of writing `{"schemaVersion":1,"drafts":[]}`. This gives the invariant worth
  documenting: an empty stash means no captured text on disk at all, so Settings'
  **Clear Stash...** and capacity 0 are genuine off switches.

### Failure handling

- Missing file → empty stash, no error, no file created.
- Malformed JSON, or a `schemaVersion` this build does not understand (i.e. a downgrade
  after a future format bump) → rename the file to `canceled-draft-stash.corrupt.json`
  (single slot, overwritten each time), start empty, and report one metadata-only
  message. Renaming rather than deleting means a downgrade never destroys the user's
  drafts, and the quarantine slot is bounded to one file.
- Save failure → report a metadata-only message; never throw into a UI call site. A
  failed save must not take the entry out of the in-memory stash.
- Every reported message is metadata only and must never interpolate draft text. Use
  exactly these:
  - `"Canceled draft stash unreadable; quarantined and started empty"`
  - `"Canceled draft stash unreadable; started empty"` (quarantine rename itself failed)
  - `"Canceled draft stash not saved: <error.localizedDescription>"`

  `localizedDescription` for `CocoaError`/POSIX file errors carries paths and codes, not
  file contents, so it is safe here — but do not build messages from anything derived
  from `text`.

### Capacity interactions

- Capacity 0 at launch: skip the load entirely and ask the store to delete the file.
- Capacity lowered below the persisted count: load, trim newest-first as today, and
  persist the trimmed list so disk and memory agree immediately.
- Restoring an entry already routes through `remove(id:)`, so the pop is persisted by
  the same write-through path — no extra work in `CapturePanelModel`.

## Implementation steps

### 1. New `Sources/CaptureCore/CanceledDraftStashStore.swift`

Foundation-only, so it builds and is exercised on Linux and in CI:

```swift
public struct PersistedCanceledDraft: Codable, Equatable, Sendable {
    public let id: UUID
    public let text: String
    public let createdAt: Date
    public init(id: UUID, text: String, createdAt: Date)
}

public struct CanceledDraftStashFile: Codable, Equatable, Sendable {
    public static let currentSchemaVersion = 1
    public let schemaVersion: Int
    public let drafts: [PersistedCanceledDraft]
}

public protocol CanceledDraftStashStoring: Sendable {
    func load() -> [PersistedCanceledDraft]
    func save(_ drafts: [PersistedCanceledDraft])
}

public struct FileCanceledDraftStashStore: CanceledDraftStashStoring {
    public static let bundleIdentifier = "org.bobs.bob-mac-capture"
    public static let fileName = "canceled-draft-stash.json"
    public static let quarantineFileName = "canceled-draft-stash.corrupt.json"

    public init(
        fileURL: URL,
        fileManager: FileManager = .default,
        onError: @escaping @Sendable (String) -> Void = { _ in }
    )

    public static func defaultFileURL(fileManager: FileManager = .default) throws -> URL
}
```

`defaultFileURL` resolves `.applicationSupportDirectory` in `.userDomainMask` with
`create: false` and appends `bundleIdentifier/fileName`. Injecting `fileURL` is what
makes the store testable against a temp directory.

### 2. Teach `CanceledDraftStash` to load and write through

In `Sources/BobMacCapture/CanceledDraftStash.swift`:

- Add `import CaptureCore` and a `store: CanceledDraftStashStoring? = nil` parameter
  **between** `capacity` and `now` in `init`. Every existing call site uses labeled
  arguments, so a defaulted parameter keeps all current tests compiling and keeps the
  default construction (`CapturePanelModel`'s fallback `CanceledDraftStash()`) purely
  in-memory.
- In `init`: when `capacity > 0`, map `store?.load()` into `[CanceledDraftEntry]`,
  assign, `trimToCapacity()`, and persist if the trim dropped anything; when
  `capacity == 0`, call `store?.save([])` to remove the file.
- Add `private func persist()` mapping `entries` to `[PersistedCanceledDraft]` and
  calling `store?.save(_:)`. Call it **after** the mutation in `push`, `remove`,
  `clear`, and `updateCapacity`, so what is published always matches disk. `push`'s
  existing `guard capacity > 0` early return needs no persist.
- Keep `CanceledDraftEntry` where it is; map to and from `PersistedCanceledDraft` at
  this boundary rather than making the domain type itself `Codable`, matching how
  `CaptureCore` keeps wire types explicit.

### 3. Wire the real store in `AppDelegate`

In `Sources/BobMacCapture/AppDelegate.swift:9`, build the store in the lazy initializer:

```swift
lazy var canceledDraftStash = CanceledDraftStash(
    capacity: settings.canceledDraftStashCapacity,
    store: Self.makeCanceledDraftStashStore(settings: settings)
)
```

The factory returns `nil` (in-memory only) if `defaultFileURL()` throws, after recording
a diagnostic. Capture `settings` — never `self` — in the `onError` closure to avoid a
retain cycle, and hop to the main actor inside it:

```swift
onError: { message in Task { @MainActor in settings.diagnosticStatus = message } }
```

`diagnosticStatus` already feeds Settings' Recent Activity through its `didSet`
(`AppSettings.swift:30-37`), so failures become visible without new UI.

### 4. Update the user-facing strings that now assert a falsehood

No test asserts these (verified), but all five are wrong after this change:

- `SettingsView.swift:40` — "Canceled draft text stays only in memory for this app
  session. Quitting or restarting clears it." → state that drafts are stored on disk and
  survive quit and restart, and that capacity 0 or **Clear Stash...** removes the file.
- `SettingsView.swift:48` and `SettingsView.swift:124` — drop "from this app session".
- `CapturePanelView.swift:906` and `CapturePanelView.swift:941` — drop "from this app
  session" / "from the current app session".

### 5. Rewrite the README contract

Line anchors as of `af02659`:

- `README.md:76-78` — Restart no longer clears the stash. It still discards an unsent
  draft; say that retained canceled drafts now survive it.
- `README.md:174-179` — replace "live in a bounded in-memory stash for the lifetime of
  the running app" with the persisted description: the file path, `schemaVersion`,
  newest-first order, 0700 directory / 0600 file, capacity still `UserDefaults`-only and
  clamped to `0...36`, capacity 0 turning the feature off **and deleting the file**.
- `README.md:197-198` — keyboard table: drop "for this session" from the Control-S and
  Control-C rows.
- `README.md:220-223` — Shift-D "clears every retained canceled draft from the current
  app session" → clears them permanently, including on disk.
- `README.md:424-433` — Uninstalling: the "it never writes captured text to disk itself"
  claim must go, and the `rm -rf` block must gain
  `rm -rf ~/Library/Application\ Support/org.bobs.bob-mac-capture`.
- `README.md:442-445` — Privacy: replace the "app-lifetime in-memory stash" bullet with
  exactly what is now on disk, where, at what permissions, that it is still never
  written to `UserDefaults`, notifications, signposts, logs, or Diagnostics, that the
  quarantine file can also hold captured text, and that Clear Stash / capacity 0 removes
  both. Leave the other Privacy bullets (live draft, completion metadata, signposts)
  intact and true.

## Tests

### New `Tests/CaptureCoreTests/CanceledDraftStashStoreTests.swift`

Temp-directory based, so it runs anywhere the package's tests run. These sources
type-check on Linux via `swift build --target CaptureCoreTests`.

1. Round-trips exact multiline Unicode text, ids, and dates, preserving newest-first
   order.
2. Duplicate texts round-trip as separate entries with distinct ids.
3. `load()` on a missing file returns `[]`, reports no error, and creates nothing on
   disk (assert the parent directory still does not exist).
4. `save([])` deletes an existing stash file and an existing quarantine file.
5. Malformed JSON → `load()` returns `[]`, the file is renamed to
   `canceled-draft-stash.corrupt.json`, and exactly one error message is reported.
6. A file whose `schemaVersion` is greater than `currentSchemaVersion` is quarantined
   the same way rather than crashing or being decoded partially.
7. First `save` creates the parent directory with `0o700` and the file with `0o600`
   (assert via `attributesOfItem(atPath:)[.posixPermissions]`).
8. Saving over an existing file replaces its contents and leaves it at `0o600`.
9. An unwritable location (parent directory `chmod 0o500`) reports an error and does not
   trap — then restore permissions in the test teardown.

### Extend `Tests/BobMacCaptureTests/CanceledDraftStashTests.swift`

Use an in-memory fake `CanceledDraftStashStoring` that records every saved snapshot — no
file I/O in these tests.

10. `init` loads persisted entries newest-first.
11. `init` with a capacity lower than the persisted count keeps the newest entries and
    immediately persists the trimmed list.
12. `init` with capacity 0 does not load and asks the store to save `[]`.
13. `push`, `remove(id:)`, and `clear()` each write through, and the last recorded
    snapshot equals `entries` after each one.
14. `updateCapacity(0)` empties memory and records an empty save.
15. A store whose `load()` reports an error leaves the stash empty and forwards the
    message unchanged.
16. Constructing `CanceledDraftStash` without a store still behaves exactly as today
    (the existing tests already cover this; confirm they are untouched).

Also add one `CapturePanelModelTests` case asserting that restoring an entry through the
picker leaves the store holding the remaining entries in order — that is the path a user
exercises most and the one where an in-memory/disk divergence would be least visible.

## Validation

On athena, in the `bob-mac-capture` checkout, with
`export PATH="$HOME/.local/share/swiftly/bin:$PATH"`:

```bash
swift build --target CaptureCore
swift build --target CaptureCoreTests
```

Both must succeed. Then commit (via the `/sase_git_commit` skill), push the branch, and
**wait for the macOS CI run to pass** — that is the only place format lint, the app
target, and the full test suite actually run.

### Manual verification (macOS, owner)

Worth listing in the handoff message; the agent cannot run these from Linux:

1. Control-C a draft → Quit from the menu → relaunch → the draft is in the Control-S
   picker with its text and order intact.
2. Same, via `Bob → Restart Bob Mac Capture`.
3. Restore one entry, quit, relaunch → that entry is gone and the others keep their
   order.
4. Settings → **Clear Stash...** → the stash file is gone from
   `~/Library/Application Support/org.bobs.bob-mac-capture/`.
5. Set capacity to 0 → the file is removed; set it back to 10 → the stash stays empty
   (nothing is resurrected).
6. `ls -ld` the directory and `ls -l` the file → `drwx------` and `-rw-------`.

## Out of scope

- **The retained live draft.** Escape and a failed capture retain the in-editor draft in
  `CapturePanelModel` memory only; that is still lost on restart. It is a different
  store with different semantics (one draft, not a bounded list) and the user asked
  about the stash. Worth proposing as a follow-up task bead through `/sase_new_task`
  after this lands — do not fold it in.
- **A separate "persist stash to disk" Settings toggle.** Capacity 0 already exists as
  an off switch and now also deletes the file, so a second switch would add UI surface
  and a second empty-vs-disabled state for no real gain on a single-user tool. Flagging
  it here so it can be requested at plan approval if the owner disagrees.
- **Encrypting the file or moving it to the Keychain.** A 0600 file inside a 0700
  directory matches the protection the vault's own Markdown notes already have on the
  same disk; encrypting scratch drafts while the notes they become sit in plaintext
  would be security theater.
- Any `bob-cli` change, and any change to `Package.swift`, the bundle scripts, or CI.

## Risks

- **Documented-guarantee reversal.** The README privacy language is precise and clearly
  maintained deliberately. If step 5 is skipped or done loosely, the repo ships a
  documented promise it no longer keeps. Treat the README edits as part of the change,
  not cleanup.
- **A pathological draft size.** A user who pastes a multi-megabyte note into the editor
  and cancels it makes each subsequent write proportionally slower, since the whole
  stash is rewritten per mutation. Real capture drafts are a few lines, so no cap is
  being added; if this ever bites, the fix is a per-entry size ceiling at push time, not
  an async writer.
- **Two instances writing the same file.** LaunchServices dedupes by bundle identifier
  and the relaunch helper waits for the old PID to exit, so overlap is not expected.
  Even if it happened, writes only occur on user mutation and each is atomic, so the
  worst case is a lost mutation, never a corrupt file.
