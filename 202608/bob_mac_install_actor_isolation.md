---
tier: tale
size: medium
title: Restore Bob Mac Capture installation under current Swift toolchains
goal:
  "`just install` builds, bundles, signs, and installs Bob Mac Capture successfully with
  the current Xcode 26 Apple Swift toolchain, while the notification and canceled-draft
  code expresses its actor boundaries without the reported isolation diagnostics."
proposed_by: bbugyi200.athena.02b
create_time: 2026-09-09 19:59:47
status: wip
---

# Restore Bob Mac Capture installation under current Swift toolchains

## Diagnosis and evidence

`just install` is only a wrapper: `justfile` line 22 invokes `Scripts/install.sh`, which
invokes `Scripts/bundle.sh`, which first runs a release `swift build` through
`Scripts/xcode-swift.sh`. The recipe-level `exit code 1` therefore does not identify the
failing stage by itself.

The supplied output is the tail of the compiler output, not the fatal diagnostic. A
matching macOS 26 GitHub Actions run for current commit `77da370` (run `31887787743`,
Xcode 26.6 and Apple Swift 6.3.3) fails in the Build step with two earlier errors:

- `CapturePanelModel.swift:65` evaluates the main-actor-isolated `CanceledDraftStash()`
  initializer as a default argument from a synchronous nonisolated context.
- `CaptureKeyCommandRouter.swift:121` calls the main-actor-isolated
  `CanceledDraftStash.acceleratorIndex` helper from the deliberately nonisolated,
  synchronous key router.

Those errors were introduced with the canceled-draft stash at `77da370`; they prevent
`bundle.sh` from reaching app assembly, signing, or installation. The subsequently
printed `NotificationService` diagnostics are warnings because every static member of
the `@MainActor` service inherits main-actor isolation while its pure builders, routing
helpers, and `UNUserNotificationCenterDelegate` callbacks are explicitly nonisolated.
They are not what returns exit 1 in the matching log, but the compiler states that they
become errors in Swift 6 language mode, so leaving them in place would retain the same
underlying boundary defect and make the notification phase less safe to land.

The immutable notification identifiers are `String`, and
`UNNotificationPresentationOptions` conforms to `Sendable`; explicit `nonisolated`
constants therefore model the real concurrency contract without an unsafe escape. Swift
SE-0449 supports explicit `nonisolated` stored declarations in the project's required
Swift 6.1-or-newer toolchain.

## Repository and in-flight epic coordination

Work only in the linked `bob-mac-capture` repository, opened through:

```bash
sase repo open bob-mac-capture -r "Implement the approved install actor-isolation fix"
```

Use the path printed by that command. The repository is changing under active epics, so
inspect its clean/dirty state, current tip, and the current versions of affected files
before editing. Preserve unrelated work and do not hard-reset or overwrite concurrent
phase changes. If the relevant phase has already landed an equivalent fix, verify it and
only implement the remaining gaps.

Coordination notes have already been added to:

- `bob-cli-t.2`, whose macOS integration scope overlaps `CapturePanelModel`, the key
  router, and the two fatal canceled-stash diagnostics;
- `bob-cli-t.3`, which owns `NotificationService.swift` and will expand the notification
  constants and nonisolated routing helpers; and
- `bob-cli-u.2`, whose completion-state scope overlaps a separate existing
  `CapturePanelModelTests.testTaskBlockIDRouteSpanUsesCachedRouteCompletion` CI failure
  that can affect full-suite validation but does not cause `just install` to fail.

Re-read those beads immediately before implementation and again before final handoff. Do
not change `bob-cli-u.1` or the closed `bob-cli-t.1`; neither owns the failing macOS
code. Do not edit SASE memory files.

## Implementation

### 1. Keep canceled-draft state isolated and make pure helpers honestly nonisolated

Keep `CanceledDraftStash` itself `@MainActor`: its `@Published` capacity and entries are
mutable UI-observed state and should not be made generally concurrent.

In `CanceledDraftStash.swift`, explicitly mark only immutable Sendable constants and
pure static helpers that must be callable without a main-actor hop as `nonisolated`.
This includes the capacity constants and the accelerator lookup data used by the key
router. The implementation should make every dependency reached by
`acceleratorIndex(for:entryCount:)` consistently nonisolated; it must not use
`nonisolated(unsafe)`, `@unchecked Sendable`, detached tasks, or a synchronous
main-actor assumption to silence the compiler.

Preserve the key router as a synchronous value-type policy object. Making the whole
router `@MainActor` would conceal that its result depends only on event values and a
value context and would unnecessarily spread isolation into tests and controller code.

### 2. Construct the default stash inside the main-actor initializer

In `CapturePanelModel`, remove the main-actor initializer call from the default-argument
expression. Retain dependency injection by accepting an optional stash (or an equivalent
overload with the same call-site ergonomics), then create the default
`CanceledDraftStash` inside `CapturePanelModel.init`, where execution is already
main-actor isolated. Bind the resolved instance consistently to the stored property and
its entries subscription so the model observes exactly the injected-or-created stash
once.

Do not change canceled-draft capacity, ordering, restore behavior, picker shortcuts,
settings propagation, or lifecycle. This is an isolation-boundary correction, not a
redesign of the new feature.

### 3. Make notification metadata available to nonisolated notification code

In `NotificationService.swift`, explicitly declare every immutable Sendable static value
consumed by a nonisolated builder, routing helper, or delegate callback as
`nonisolated`. At the current baseline that includes the open-action identifier,
capture-category identifier, target-path user-info key, and foreground-presentation
options. If `bob-cli-t.3` has already added singular/plural categories, action
identifiers, or an array user-info key, apply the same rule to all equivalent immutable
metadata rather than fixing only the four old names.

Keep the service instance `@MainActor`, because authorization display state, the opener,
and UI-facing lifecycle belong there. Keep the UserNotifications delegate callbacks
nonisolated as required by the imported protocol, and keep the pure content/category/URL
builders synchronously usable by their existing tests. Do not move those callbacks onto
the main actor and do not duplicate string literals at use sites.

### 4. Preserve and sharpen regression coverage

Use the existing canceled-stash, key-router, capture-model, and notification tests as
the behavioral baseline. Add or adjust focused coverage only where needed to pin the
corrected construction and dependency-injection behavior:

- default `CapturePanelModel` construction creates one usable stash, while an injected
  stash remains the exact observed instance;
- key routing can use numeric/alphabetic stash accelerators through the synchronous pure
  router without changing the existing 36-slot mapping; and
- notification builders and target routing retain their current content, category,
  user-info, foreground presentation, and open-action behavior from nonisolated test
  contexts.

Compilation itself is the primary regression gate for these annotations. Do not add a
runtime actor hop solely to make a test appear concurrency-aware, and do not weaken or
delete the tests added by either active epic.

## Validation

Run formatting first, then the same build path used by installation:

```bash
just format-lint
just build
just test
just bundle
./Scripts/install.sh --target '~/Applications' --identity '-'
```

On a macOS 26 host with Xcode 26 selected, capture the `just build` output and verify
all of the following, not merely the final exit status:

- no error remains at `CapturePanelModel` default stash construction;
- no error remains at the key router's accelerator lookup;
- no `main actor-isolated static property` warning remains in
  `NotificationService.swift` for notification metadata or foreground options;
- bundle signing verification succeeds; and
- install prints the installed app path and the resulting app passes
  `codesign --verify --deep --strict` with bundle identifier `org.bobs.bob-mac-capture`.

Run the install a second time to exercise the existing replacement/rollback path, as CI
does. Do not change `install.sh`, `bundle.sh`, the just recipe, codesigning behavior, or
the deployment target unless new evidence shows a separate failure after the compiler
errors are gone.

The full `just test` command currently has independent evidence of a cached-route
completion assertion failure on pre-epic commit `593398a`. Re-evaluate it against the
implementation-time tip because `bob-cli-u.2` is actively changing that path. If it
still fails while all changed-surface tests pass, do not silently weaken the assertion
or fold an unrelated completion behavior change into this fix: preserve the failure
output and append the new evidence to `bob-cli-u.2`. The installation fix is not
complete, however, until `just build`, `just bundle`, and the two install passes succeed
on macOS.

If the implementation environment is not macOS, perform every available static review
and non-AppKit check, then state exactly which macOS commands remain unexecuted. Do not
claim installation success from a Linux-only run; use the existing macOS 26 workflow or
the user's MacBook for the final build/bundle/install evidence when authorized.

## Bead handoff

After validation, append concise outcome notes (do not replace existing notes):

- to `bob-cli-t.2`, record the exact canceled-stash/model/router fix and build/install
  evidence;
- to `bob-cli-t.3`, record the notification metadata isolation contract and whether its
  in-flight expanded notification implementation already incorporated it; and
- to `bob-cli-u.2`, only if the known completion test was re-run, record whether it
  still fails or has been resolved by the current phase tip.

## Non-goals

- Migrating the package from `.swiftLanguageMode(.v5)` to Swift 6 language mode.
- Clearing unrelated selector, deprecation, redundant-`await`, FileManager Sendable, or
  cache clock warnings.
- Changing capture completion behavior, canceled-draft product behavior, notification
  product copy, installation destinations, or signing policy.
- Committing, pushing, or otherwise publishing changes without an explicit user request
  or the normal SASE completion workflow.

## Done when

- The fatal actor-isolation errors that stop release compilation are eliminated without
  weakening main-actor ownership of mutable UI state.
- The reported notification isolation warnings are eliminated with safe explicit
  nonisolated immutable metadata.
- Focused behavior remains covered, the available test suite has been run with any
  unrelated failure clearly separated, and macOS `just build`, `just bundle`, plus two
  install passes succeed with a valid signed app at the requested destination.
- The active epic phase beads carry final coordination and validation evidence.
