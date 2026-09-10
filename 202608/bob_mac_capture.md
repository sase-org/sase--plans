---
tier: epic
title: Native Bob Mac Capture app
goal:
  A signed native macOS 26 menu-bar app in bobs-org/bob-mac-capture replaces the
  Hammerspoon capture pop-up with a pre-warmed global-hotkey panel that supports
  multi-line editing, authoritative marker highlighting and completion, exact live
  preview, lossless failure handling, and reliable inline and system feedback, while
  bob-cli remains the only implementation of capture grammar and vault mutation.
phases:
  - id: grammar
    title: Authoritative capture parser endpoint
    depends_on: []
    size: medium
    description:
      "grammar: refactor bob-cli's existing capture parser into a span-aware reusable
      model and add the read-only bob capture-parse command with stable JSON,
      diagnostics, CLI/docs wiring, and Rust plus ported Hammerspoon grammar coverage."
  - id: completion
    title: Cursor-aware capture completion endpoint
    depends_on:
      - grammar
    size: medium
    description:
      "completion: add bob capture-complete as the authoritative cursor-aware completion
      service over capture targets, sections, and open tasks, returning replacement
      ranges and stable JSON while preserving the current discovery-command contracts."
  - id: foundation
    title: Signed app foundation and macOS CI
    depends_on:
      - grammar
    size: medium
    description:
      "foundation: scaffold bobs-org/bob-mac-capture as a macOS 26 SwiftPM app with a
      pure CaptureCore, direct cancellable bob process client, fixture-backed tests,
      macOS CI, deterministic bundle/install/sign scripts, LSUIElement lifecycle, Carbon
      hotkey, pre-warmed non-activating NSPanel, draft-safe multi-line editor, settings,
      and launch-at-login support."
  - id: feedback
    title: Capture execution and reliable feedback
    depends_on:
      - foundation
    size: medium
    description:
      "feedback: wire submit and explicit clipboard-resolving preview through bob
      capture, prevent duplicate writes, preserve drafts and destinations on every
      failure, add inline status, and implement signed UNUserNotificationCenter
      notifications with foreground presentation, authorization diagnostics, a test
      action, and Open Note."
  - id: intelligence
    title: Highlighting, completion, and live preview
    depends_on:
      - completion
      - foundation
    size: medium
    description:
      "intelligence: connect macOS 26 attributed editing to capture-parse spans, add an
      inline keyboard completion popover backed by capture-complete, and add debounced
      cancellable exact preview through bob capture --dry-run --no-clip with cached
      targets and explicit stale-response handling."
  - id: hardening
    title: Integrated macOS validation and release hardening
    depends_on:
      - feedback
      - intelligence
    size: medium
    description:
      "hardening: complete app accessibility, appearance, privacy, error, packaging,
      installation, and user documentation; exercise core and full-app CI; and run the
      owner-assisted signed on-Mac notification, focus, clipboard, Spaces, IME, latency,
      and smoke-test gate without changing the existing Hammerspoon hotkey."
  - id: cutover
    title: Hammerspoon cutover and migration cleanup
    depends_on:
      - hardening
    size: small
    description:
      "cutover: only after the hardening gate passes, remove the old Hammerspoon hotkey,
      WebView workflow, duplicated task_capture.lua grammar, and migrated specs from
      chezmoi; deploy the dotfiles change and document rollback to the last known-good
      Hammerspoon revision."
proposed_by: bbugyi200.athena.005
create_time: 2026-09-09 19:59:46
status: wip
---

- **PROMPT:**
  [prompts/202608/bob_mac_capture.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/bob_mac_capture.md)

# Plan: Native Bob Mac Capture app

## Outcome and scope

Build the replacement described in the research report
`202608/bob_mac_capture_replacement/bob_mac_capture_replacement.md` in the existing
public repository `bobs-org/bob-mac-capture` (default branch `master`; currently one
empty `README.md` commit). The product is a native Swift/AppKit + SwiftUI macOS 26 app,
not a feature inside `bob-cli`: `bob-cli` owns parsing, completion data, preview, and
all vault writes; the app owns presentation, input, process orchestration,
notifications, and macOS integration. The final migration removes the superseded UI and
duplicated Lua grammar from the linked `chezmoi` repository.

The work spans three repositories:

- `bob-cli` (the primary project): authoritative capture grammar, new read-only JSON
  endpoints, CLI documentation, and Linux-verifiable tests.
- `bobs-org/bob-mac-capture`: the application, Swift package, bundle resources,
  packaging/install scripts, documentation, and macOS CI. It is not registered as a
  linked repo yet, so every phase that accesses it must first run
  `sase repo open gh:bobs-org/bob-mac-capture -r "<specific reason>"` and use the path
  printed by that command.
- `chezmoi`: the source of the deployed Hammerspoon configuration. The cutover phase
  must open it with `sase repo open chezmoi -r "<specific reason>"`, and after its
  commit is created it must run `chezmoi update -a --force` as that repo's instructions
  require.

SASE agents run on Linux `athena`, which currently has no `swift`, `swiftc`,
`xcodebuild`, AppKit, SwiftUI, or UserNotifications. Do not report local compilation of
the Swift code unless a Linux Swift toolchain is deliberately installed. The required
full-app compiler/test loop is GitHub Actions on a macOS runner; the owner-assisted
interaction gate runs on the actual Mac. This constraint is why macOS CI and a UI-free
`CaptureCore` are foundation work, not later polish.

## Product contract

The shipped app must meet these behavioral requirements:

1. A resident `LSUIElement` application opens a preconstructed capture panel from a
   configurable Carbon `RegisterEventHotKey`; the production default is `control` +
   `shift` + `command` + `I`, while development uses a non-conflicting temporary
   shortcut until cutover. No Accessibility permission is required.
2. The panel uses a native multi-line editor. Return submits, Shift-Return or
   Option-Return inserts a newline, Command-Return submits and opens the target, and
   Escape first closes completion and then the panel. Closing a nonempty draft requires
   explicit discard or retains it for the next invocation. For the first release,
   embedded newlines are editing affordances and `bob capture` keeps its documented
   whitespace-normalizing single-line output; child-bullet semantics and batch capture
   remain out of scope.
3. Marker coloring and diagnostics come from `bob capture-parse`; completion candidates
   and replacement ranges come from `bob capture-complete`. The app never recognizes or
   synthesizes capture grammar independently.
4. Live preview is the exact destination and rendered Markdown reported by
   `bob capture --dry-run --no-clip --format json`. The continuously invoked path must
   never read the clipboard. An explicit user action and final submission may resolve
   `%`, `%N`, and `%header` normally so live clipboard and Clipy history behavior remain
   intact.
5. Submission runs `bob capture --format json -- <draft>` directly, never through a
   login shell or interpolated shell command. One submit can cause at most one mutation;
   failure keeps the complete draft, parsed destination, and actionable error visible.
6. Every result has visible in-panel feedback. Success and failure also use
   `UNUserNotificationCenter` when authorized. The notification center delegate is set
   before authorization is requested, `willPresent` returns banner/sound/list options,
   notification bodies omit captured text by default, Settings exposes live permission
   status and a test notification, and successful notifications can open the returned
   Obsidian target.
7. The installed `.app` has stable bundle identifier `org.bobs.bob-mac-capture`, a
   stable install location, a valid signature, a menu-bar status/preferences surface,
   and an `SMAppService` launch-at-login control. Ad-hoc signing may support
   development, but release/install documentation must prefer an available Apple
   Development identity and explain certificate expiry and notification permission
   implications.
8. Warm hotkey-to-focused-editor is p50 below 50 ms and p95 below 100 ms on the actual
   Mac. Panel construction, login-shell startup, target discovery, and other subprocess
   work are absent from the hotkey path. Completion from cached data appears within 50
   ms and typing never waits synchronously for `bob`.

Non-goals are App Store distribution/notarization, cross-platform UI, direct Markdown
mutation in Swift, a resident `bob` daemon or FFI boundary, batch multi-line capture,
Undo, and recent-capture history. Those can follow only after measurements justify them.

## Phase `grammar`: authoritative capture parser endpoint

Work in `bob-cli`. The current grammar is private in `src/native/capture.rs` around
`ParsedCaptureText`, `CaptureKind`, `TerminalMarkers`, and
`parse_capture_text_with_clip_control`; the new endpoint must reuse this parser instead
of copying it.

### Parser model and spans

- Refactor the grammar into a reusable module (for example
  `src/native/capture_language.rs`) consumed by both `capture.rs` and the endpoint.
  Preserve `bob capture` behavior and error text byte-for-byte unless a ported legacy
  fixture proves the interactive Hammerspoon grammar is intentionally richer.
- Stop normalizing before span discovery. Scan the original UTF-8 input, retain a
  normalized body for capture execution, and express offsets as UTF-8 byte offsets into
  the original input because Rust strings and `AttributedString` conversion can validate
  that contract unambiguously. Every span must be on a Unicode scalar boundary,
  non-overlapping, ordered, and half-open `[start, end)`.
- Model recognized tokens at least as `route`, `section`, `pomodoro_route`,
  `pomodoro_block_id`, `sub_bullet_route`, `sub_bullet_block_id`, `schedule`,
  `priority`, `clipboard`, and `interactive_placeholder`. Include a stable overall mode
  (`task`, `bullet`, `pomodoro_task`, `sub_bullet`, or `incomplete`), normalized body,
  resolved route/section/block ID where available, and whether a route, section,
  Pomodoro ID, or parent task still needs completion.
- Return diagnostics as structured entries with severity, code, message, and optional
  byte range. Incomplete interactive markers such as `@`, `@#`, `@#Ideas`, `@route#`,
  `@:`, `@route:`, `@^`, and `@route^` are valid parse states for editing, not fatal CLI
  capture requests. Invalid terminal marker components remain diagnostics and
  `bob capture` retains its existing strict execution errors.
- Generate the execution parse and the editor parse from the same tokenizer and marker
  classification functions. Do not leave a second grammar hidden behind the new command.

### Command contract

Add native, read-only:

```text
bob capture-parse [-f|--format human|json] [--] [TEXT]...
```

Use the existing one-line stdin convention when `TEXT` is omitted. JSON success has a
versioned, documented stable shape similar to:

```json
{
  "ok": true,
  "schema_version": 1,
  "input": "Call bank @Cash^",
  "body": "Call bank",
  "mode": "incomplete",
  "route": "cash",
  "needs": ["task"],
  "spans": [
    { "start": 10, "end": 15, "kind": "sub_bullet_route" },
    { "start": 15, "end": 16, "kind": "interactive_placeholder" }
  ],
  "diagnostics": []
}
```

Human output should be concise and styled only through `Styler`; piped output stays
ANSI-free. JSON failures print exactly one object to stdout and keep stderr clean,
following the other capture discovery commands. Add `capture-parse` in sorted order to
`NativeCommand`, `SUBCOMMANDS`, top-level examples, install smoke checks, README command
tables/contracts, and all help-surface tests. Follow `cli_rules.md`: alphabetized
options, excellent help, and a short alias for every public long option.

### Tests

- Unit-test Unicode and CRLF byte offsets, leading/trailing markers, terminal-marker
  precedence, duplicate/invalid markers, whitespace normalization, all marker kinds,
  incomplete states, diagnostics, and parse/execution equivalence for complete input.
- Port the behavioral cases in chezmoi's `tests/hammerspoon/task_capture_spec.lua` into
  Rust before that suite can be deleted. Where the Lua model merely staged pickers,
  represent that as `needs`; where the Lua code deliberately left unsupported forms to
  `bob capture`, preserve the authoritative current `bob` result and document any
  intentional delta in the test name.
- Add CLI integration coverage for help order, native-only dispatch, clean human/JSON
  output, error codes, UTF-8 offsets, and absence of vault or clipboard access. Assert
  the endpoint succeeds with a nonexistent `BOB_DIR` and a clipboard marker so its
  lexical nature cannot accidentally grow I/O.
- Run the focused tests, `cargo fmt --check`,
  `cargo clippy --all-targets --all-features`, `cargo test`, `just install-smoke`, and
  `git diff --check`.

## Phase `completion`: cursor-aware capture completion endpoint

Work in `bob-cli` after `grammar`. Add a read-only completion service; the client tells
the service the original text and UTF-8 cursor byte offset, and the service decides
whether completion applies.

```text
bob capture-complete --cursor BYTE [-b|--bob-dir DIR]
                     [-f|--format human|json] [--] [TEXT]...
```

`--cursor` / `-c` is required and must be a UTF-8 boundary within the input. Keep
options alphabetized and fully documented. Add the command to all the wiring, smoke,
README, and help-surface locations listed for `capture-parse`.

### Completion behavior and JSON

- Reuse the phase-`grammar` parse model to identify the token under or immediately
  before the cursor. Do not independently parse marker prefixes in this module.
- Return `ok`, `schema_version`, original cursor, a half-open UTF-8 byte `replacement`
  range, completion `context`, and ordered `candidates`. Empty/no-applicable completion
  is a successful empty result, not an error.
- Contexts:
  - route completion for `@`, route fragments, and missing route portions of `@:...`,
    `@^...`, and `@#...`, backed by `capture-targets` scanning;
  - section completion after `@route#prefix`, backed by the same section scanner used by
    `capture-sections`;
  - Pomodoro block-ID completion after `@route:prefix`, and parent-task completion after
    `@route^prefix`, backed by the same open-task scanner used by `capture-tasks`.
- Candidate objects contain the exact replacement text plus display metadata the app
  needs without synthesis: canonical route, target label/kind/status; section title and
  level; or task ref, block ID, checkbox/status, text, section, depth, and child count.
  Candidates must replace only the active marker component, preserving the body and
  adjacent `%`, `p:`, and `s:` tokens.
- Match case-insensitively with deterministic ordering: exact prefix matches before
  substring matches while retaining each discovery source's existing stable order. Never
  silently fall back to inbox when discovery fails; return the same actionable error in
  human and JSON forms.
- `capture-targets`, `capture-sections`, and `capture-tasks` remain public and backward
  compatible. Promote their internal scanners/result models as needed rather than
  spawning `bob` recursively or reparsing their JSON.

### Tests

Cover every context, a cursor in body text, cursor at every scalar boundary in Unicode
input, invalid/out-of-range cursor, empty candidates, ranking, exact replacement ranges,
all interactive incomplete forms, terminal-marker adjacency, missing notes, discovery
errors, JSON metadata, and no mutations. Run the same complete `bob-cli` validation set
as phase `grammar`.

## Phase `foundation`: signed app foundation and macOS CI

Work in `bobs-org/bob-mac-capture` after opening it through `/sase_repo`. This phase may
run in parallel with `completion` because it needs the parse contract but not yet the
completion command.

### Repository and build layout

Create:

```text
Package.swift
Sources/CaptureCore/
Sources/BobMacCapture/
Tests/CaptureCoreTests/
Tests/BobMacCaptureTests/
Tests/Fixtures/fake-bob
Resources/Info.plist
Scripts/bundle.sh
Scripts/install.sh
.github/workflows/ci.yml
justfile
README.md
```

- Use SwiftPM without a checked-in `.xcodeproj`. Target macOS 26.0 and Swift language
  mode supported by the selected stable Xcode runner. `CaptureCore` imports only
  Foundation and contains models, JSON decoding, executable resolution, environment
  construction, subprocess orchestration, cancellation/generation control, and caches.
  `BobMacCapture` owns AppKit, SwiftUI, Carbon, ServiceManagement, and
  UserNotifications.
- CI runs `swift format lint` (or a pinned, documented equivalent), `swift build`,
  `swift test`, bundle assembly, `plutil -lint`, `codesign --verify --deep --strict`,
  and a bundle-identifier assertion on a GitHub-hosted macOS runner. Pin action major
  versions and select an Xcode version that includes the macOS 26 SDK. It must run for
  pull requests and pushes to `master`.
- `Scripts/bundle.sh` assembles `Bob Mac Capture.app`, copies resources, sets executable
  permissions, signs nested code then the bundle, and accepts an explicit identity; `-`
  is the documented development fallback. `Scripts/install.sh` replaces only the
  explicit target under `/Applications` or `~/Applications` via a staged bundle and
  verifies identifier/signature after installation. `just build`, `test`, `bundle`,
  `install`, and `all` provide the normal workflow.

### `CaptureCore` and direct process policy

- Define versioned `Codable` models matching `capture-parse`, existing `capture`, and
  existing `capture-targets` JSON. Unknown additive keys decode safely; schema-version
  mismatches and malformed/empty stdout produce actionable errors containing command,
  exit status, and bounded stderr without leaking captured text.
- Resolve `bob` once at launch: user-configured absolute executable first, then the
  known GUI-safe locations `~/.cargo/bin/bob`, `~/bin/bob`, `/opt/homebrew/bin/bob`, and
  `/usr/local/bin/bob`. Validate that an override is an absolute executable file.
  Surface the resolved path and a recheck action in Settings.
- Use `Process` directly with argv arrays and a minimal explicit environment containing
  required system paths plus `HOME`, locale, `BOB_DIR`, and documented Bob overrides.
  Never invoke `/bin/zsh`, `-l`, `-c`, `env`, `which`, or interpolate draft text.
- Execute asynchronously off the main actor. Cancellation terminates the previous
  process; monotonically increasing request generations prevent a late response from
  updating newer editor state. The fake-bob fixture must support delayed success,
  delayed failure, malformed JSON, signal/cancellation observation, and argv/env
  recording.
- Load and cache `capture-targets --format json` during launch, refresh asynchronously
  when the panel opens, and keep the last good cache on refresh failure while showing
  staleness. Defer FSEvents invalidation to phase `intelligence`, when the cache is a UI
  dependency.

### App shell and editor

- Set `LSUIElement=1`, bundle identifier `org.bobs.bob-mac-capture`, display name, and
  minimum OS in `Resources/Info.plist`. Use an `AppDelegate` that creates the menu-bar
  item, settings window, hotkey manager, and capture panel at launch.
- Register the configured shortcut using Carbon `RegisterEventHotKey`. Detect and
  visibly report registration conflicts. Default development builds to a documented
  temporary shortcut such as Control-Shift-Command-O; make the production I binding a
  preference/cutover choice so the still-running Hammerspoon binding cannot steal it.
- Construct one `NSPanel` exactly once with `.nonactivatingPanel`, `.titled`,
  `.fullSizeContentView`, and `.resizable` set in its initializer. Never toggle
  `.nonactivatingPanel` after initialization. Set `isFloatingPanel`, `.floating` level,
  `.canJoinAllSpaces`, `.fullScreenAuxiliary`, transparent titlebar, hidden title, and a
  hosted SwiftUI root view. The panel must accept key input immediately; if empirical
  macOS testing requires deliberate app activation, keep that as a documented fallback
  and retain the notification delegate added later.
- Bind the macOS 26 rich-text `TextEditor` to `AttributedString` and
  `AttributedTextSelection`, but render plain text until phase `intelligence`. Implement
  Return submit, Shift/Option-Return newline, Command-Return submit-and-open, and Escape
  ordering. Empty drafts dismiss directly; nonempty drafts are retained across panel
  dismissal or require a confirmation before destructive discard.
- Add Settings for bob path, vault path/environment, hotkey, launch at login through
  `SMAppService.mainApp`, and diagnostic status. Persist non-sensitive preferences with
  `UserDefaults`; never persist captured text unless the user explicitly retains the
  in-memory draft.

### Tests and verification

Test all `CaptureCore` decoding/process/error/cancellation/cache logic with the fixture,
plus macOS tests for panel style invariants, hotkey conflict reporting, key-command
routing, draft retention, bundle identity, and login-item state mapping. The phase is
complete only when macOS CI passes from a clean checkout and its produced app bundle
passes signature and plist checks; Linux-side review must also run `git diff --check`
and any non-Swift static checks available there.

## Phase `feedback`: capture execution and reliable feedback

Work in `bobs-org/bob-mac-capture` after `foundation`.

### Submission and result handling

- Submit the exact draft via `bob capture --format json -- <draft>`. Disable submission
  while one mutation is in flight and assign a unique request ID so repeated Return, a
  late callback, cancellation, or view recreation cannot execute a second mutation.
- Decode the existing result fields including route label, relative/absolute target,
  rendered task line, kind, placement, clipboard output, schedule log, block link, and
  parent details. On success, show an in-panel confirmation, clear the draft only after
  decoding a confirmed `ok: true`, and optionally open the target for Command-Return.
- On launch, discovery, exit, signal, JSON, or `ok: false` failure, keep the complete
  draft and last parsed destination in the panel, move focus to an accessible inline
  error, and offer retry and Copy Diagnostic. Never silently route to inbox after a
  failed target lookup.
- Add an explicit full preview command/button for clipboard-bearing drafts. It runs
  `bob capture --dry-run --format json -- <draft>` only on user request and labels that
  it reads the current clipboard/history. It never writes. The continuously running
  preview added later remains separate and always passes `--no-clip`.
- Build Obsidian open URLs from the successful returned target using URLComponents and
  percent encoding; do not reconstruct a target from the typed grammar. If Obsidian
  cannot open, the capture remains successful and the UI exposes the target path.

### Notifications

- Make a long-lived `NotificationService` the
  `UNUserNotificationCenter.current().delegate` before requesting authorization.
  Implement `userNotificationCenter(_:willPresent:withCompletionHandler:)` with
  `.banner`, `.sound`, and `.list` so foreground notifications are presented.
- Request alert/sound authorization from a clear Settings or onboarding action rather
  than treating a denied prompt as capture failure. Display the live authorization
  status (`notDetermined`, `denied`, `authorized`, `provisional`, or `ephemeral` as
  applicable), refresh it whenever Settings opens, link to system notification settings,
  and provide a test-notification button with visible success/failure.
- Post success and failure notifications under the app's stable bundle identity, but
  never make correctness depend on them. Omit draft/captured text from bodies by
  default. Register an Open Note category/action whose response uses the target returned
  by `bob`; handle default clicks the same way. Do not implement Undo in this epic.
- Detect an unsigned/ad-hoc/development bundle as far as public signing APIs and the
  install script allow and make signing state part of diagnostics. Documentation must
  say notification verification requires the installed stable signed bundle, not
  `swift run`.

Test double-submit suppression, submit-and-open, success clearing, every failure's draft
retention, diagnostic redaction, explicit clipboard preview argv, inline feedback,
notification payload privacy, action routing, foreground presentation options, and all
authorization states. CI must rebuild/retest/reverify the signed bundle.

## Phase `intelligence`: highlighting, completion, and live preview

Work in `bobs-org/bob-mac-capture` after both `completion` and `foundation`.

### Parse and preview pipeline

- On each edit or selection change, debounce approximately 40–60 ms, cancel superseded
  work, and asynchronously run `bob capture-parse --format json -- <draft>`. Convert
  validated UTF-8 byte ranges to `AttributedString.Index` ranges; reject the entire
  malformed span set rather than applying corrupt partial highlighting. Apply semantic
  foreground/background attributes that adapt to light/dark mode and system accent color
  without destroying the user's selection or marked-text/IME composition.
- Separately run live preview after parse with
  `bob capture --dry-run --no-clip --format json -- <draft>`. Assert `--no-clip` in the
  process-layer API and its tests, not only at a call site. Render exact `task_line`,
  destination label/path, placement, kind, scheduled date, and diagnostics. A `%` marker
  stays literal in this preview and the UI explicitly says clipboard content is resolved
  only by explicit preview or submit.
- Prevent unstable `p:<N>` previews: give each draft revision a fixed
  `BOB_PRIORITY_ROLL_SEED` in the preview environment and reuse that seed for the final
  submit so the displayed randomized scheduled date is the date written. Reset it only
  after successful submission or deliberate draft discard.
- Parse and preview failures are distinct. A missing/incomplete capture may show a
  neutral prompt without erasing the last successful destination; a real diagnostic is
  shown inline. Generation checks must make out-of-order responses impossible.

### Completion UI and target caching

- Call `bob capture-complete --cursor <UTF8_BYTE> --format json -- <draft>` only when
  phase-`grammar` parse state says completion is applicable. The app sends the cursor
  offset but never decides the replacement range or candidate syntax.
- Display a native completion popover anchored to the active editor range. Include route
  kind/status, section level, and task checkbox/status/context from JSON. Tab or Return
  accepts, Control-N/Control-P and arrows navigate, Escape dismisses completion before
  it dismisses the panel, and ordinary typing/filtering remains responsive. Apply the
  server-provided byte replacement range exactly and restore the caret after inserted
  text.
- Serve route completion immediately from the last good launch/show cache where
  possible, while authoritative endpoint responses update it. Add an FSEvents watcher
  for the vault root that coalesces events and refreshes targets asynchronously; watcher
  loss or permission errors mark the cache stale but do not clear it or block typing.
  Section/task candidates stay request-scoped because they depend on a selected note.
- Keep `CaptureCore` independent of AppKit so JSON/range/ranking orchestration stays
  fixture-tested. If macOS 26 `TextEditor` cannot preserve selection, IME, and
  completion interaction reliably at the owner gate, the pre-registered fallback is a
  custom AppKit `NSTextView`/TextKit adapter inside the same view boundary—not a grammar
  copy or a change to `CaptureCore`. A WebView/CodeMirror rewrite is outside this epic
  and requires a new approved plan.

Test byte-to-string range conversion with emoji and combining characters, malformed
spans, selection preservation, debounce and cancellation, stale generations, exact
preview rendering, fixed priority seed parity, hard assertion of `--no-clip`, every
completion context, replacement, navigation/key precedence, IME marked text, cache
refresh/staleness, and watcher coalescing. CI must pass all core and app tests.

## Phase `hardening`: integrated macOS validation and release hardening

Work in `bobs-org/bob-mac-capture`. Finish application-quality behavior and produce a
repeatable owner-assisted test checklist. This phase does not alter the Hammerspoon
binding.

### Product hardening

- Finish native materials, resizing, light/dark/high-contrast appearance, Dynamic Type,
  VoiceOver labels/order/announcements, Reduce Motion, empty/loading/error states,
  menu-bar state, settings layout, and keyboard-only operation. Do not log or include
  capture text in notification bodies, CI artifacts, diagnostics, or signposts.
- Add signposts around hotkey receipt, panel ordering, editor focus, parse, completion,
  preview, submit, and notification scheduling. Keep rolling diagnostic summaries
  metadata-only and bounded.
- Make installation and upgrade atomic and documented. README covers requirements,
  installing/configuring `bob`, bundle/signing identity, notifications, hotkey conflict,
  login item, update/reinstall, certificate expiry, troubleshooting/test notification,
  privacy, keyboard map, multi-line normalization, clipboard-preview semantics, and
  complete uninstall/rollback.
- Ensure every process invocation has bounded output and timeout behavior, the panel
  never blocks on process completion, terminated helpers are reaped, and app quit
  cancels outstanding work cleanly.

### Automated and owner-assisted release gate

From a clean revision, require green macOS CI for formatting, build, all tests, bundle,
plist, and signing verification. Then install that exact signed bundle on the target Mac
and record the following results in the phase handoff:

1. `codesign -dv --verbose=4` and `codesign --verify --deep --strict` report the stable
   identifier and a valid signature; reinstalling the same identity/path retains the
   notification authorization state.
2. Settings reports authorization accurately, its test notification presents, and both
   success and failure notifications present while the capture panel is visible. Open
   Note reaches the returned Obsidian target. Denied authorization still leaves complete
   inline feedback.
3. The temporary hotkey opens the already-created panel and focuses the editor over a
   normal app, every Space, and a full-screen app. Measure at least 30 invocations after
   login; signposts show hotkey-to-focused-editor p50 < 50 ms and p95 < 100 ms, with no
   process or panel construction on that path.
4. Return/newline/Command-Return/Escape/completion bindings, Unicode, emoji, IME marked
   text, VoiceOver, light/dark appearance, multiple displays, secure-input recovery, and
   hotkey collision messaging behave as documented.
5. Route/section/task completion, marker highlighting, and exact preview match `bob`.
   Cached route completions appear within 50 ms. A simulated slow and failed `bob`
   process never blocks typing, applies stale output, falls back silently, or loses the
   draft.
6. `%`, `%N`, and `%header` submit correctly with live clipboard and Clipy history from
   the non-activating panel. Continuous preview never touches either source; explicit
   clipboard preview does. `p:<N>` live preview and final submission write the same
   scheduled date.
7. Rapid repeated submission mutates the vault at most once. Missing routes, missing or
   ambiguous Pomodoro entries, stale parent tasks, malformed JSON, missing `bob`, and
   target scan failures preserve the complete draft and destination with actionable
   inline errors.
8. Launch at login survives logout/login, the app remains an `LSUIElement`, and no
   Accessibility prompt appears. If correct editor input requires activation, record
   that deliberate fallback and re-run foreground notification tests.

Any failure blocks `cutover`; fix it in the owning phase's code, rerun affected focused
tests, rerun full CI, reinstall the newly signed bundle, and repeat the relevant manual
checks. If the latency budget fails because direct `bob` work is actually slow, gather
profiles and propose a separate daemon/FFI plan rather than adding one here.

## Phase `cutover`: Hammerspoon cutover and migration cleanup

Only begin after the phase-`hardening` handoff records a passing installed-app gate.
Work in `chezmoi` after opening it through `/sase_repo`.

- Change the app's configured shortcut from the temporary development binding to
  Control-Shift-Command-I and verify `RegisterEventHotKey` succeeds after the following
  Hammerspoon removal.
- Remove only the superseded capture feature from `home/dot_hammerspoon/init.lua`:
  `TaskCapture` import/state, embedded HTML/CSS/JS, WebView lifecycle, picker
  commands/choosers, capture subprocess commands, capture notification helpers, and the
  `cmd` + `shift` + `ctrl` + `I` binding. Preserve unrelated Hammerspoon functions,
  including the Pomodoro menu-bar's own subprocess path and any shared helpers still
  referenced elsewhere.
- Delete `home/dot_hammerspoon/task_capture.lua` and
  `tests/hammerspoon/task_capture_spec.lua` only after confirming every behavioral
  fixture was ported in phase `grammar` and the new app gate covers picker interaction.
  Remove obsolete test-runner references if the chezmoi `justfile` enumerates the file.
- Update any chezmoi documentation that still names the Hammerspoon capture panel. Add a
  concise rollback note identifying the last known-good pre-cutover chezmoi revision and
  the app preference needed to return to the temporary shortcut; do not maintain two
  active production hotkeys.
- Run the repo's Lua formatting/lint/tests and `git diff --check`, then use the required
  commit workflow when requested and run `chezmoi update -a --force` after the commit.
  On the Mac, reload Hammerspoon, launch/restart the installed app, confirm only the app
  owns Control-Shift-Command-I, and rerun capture, foreground notification, clipboard,
  and draft-retention smoke tests.

## Cross-phase invariants

- Preserve unrelated changes in every checkout. Open external/linked repositories with
  `/sase_repo` immediately before use and follow their own `AGENTS.md` instructions.
- All source edits use `apply_patch`. Never commit unless explicitly requested or a
  post-completion finalizer invokes the required `/sase_git_commit` workflow.
- Do not add a Swift grammar, direct vault writer, login-shell invocation, Accessibility
  event monitor, hand-written LaunchAgent, unsigned loose-executable notification path,
  or runtime toggling of `.nonactivatingPanel`.
- JSON contracts are versioned and documented. Unknown additive fields are tolerated by
  the app, but unsupported schema versions fail visibly. Draft text and clipboard data
  are treated as private throughout.
- A phase that discovers unrelated objective work must use `/sase_new_task` before
  filing a bead, as project instructions require; phase workers that are explicitly
  forbidden to create beads instead record `PROPOSED FOLLOW-UP:` on their own bead.
- Every phase ends with focused tests, its repository's complete validation suite where
  available, `git diff --check`, and a review of the final diff against this plan. The
  macOS app phases additionally require green macOS CI; `hardening` and `cutover` also
  require the stated owner-assisted Mac checks.
