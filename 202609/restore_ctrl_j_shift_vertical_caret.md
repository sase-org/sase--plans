---
tier: tale
title: Restore Ctrl-J bullet insertion and shift vertical caret movement
goal:
  Ctrl-J inserts bullet rows again while exact Ctrl-Shift-J/K provide the existing safe
  sticky-column vertical movement.
size: small
proposed_by: bbugyi200.athena.02t.f0
create_time: 2026-09-09 20:00:41
status: wip
---

# Restore Ctrl-J bullet insertion and move vertical caret navigation to Ctrl-Shift-J/K

## 1. Scope and repository

Implement this entirely in the `bob-mac-capture` linked repository. Open it with the
`sase_repo` skill and work only in the path that `sase repo open bob-mac-capture ...`
prints. Do not change `bob-cli`: the key routing, native text-view behavior, tests, and
user documentation all live in the macOS app, and no subprocess or JSON contract is
involved.

The plan is based on clean `bob-mac-capture` master at `976a835` and on the approved
predecessor plan `plan:202609/capture_vertical_caret_movement.md`. The current tree
already contains the sticky-column vertical movement introduced by `78315d9`; this work
reassigns its shortcuts without redesigning it.

Expected files:

| File                                                  | Purpose                                                                                                             |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` | Restore Ctrl-J to bullet insertion, remove the Ctrl-I route, and require Ctrl-Shift-J/K for vertical movement.      |
| `Sources/BobMacCapture/CapturePanelController.swift`  | Update shortcut-specific comments while leaving movement and edit behavior unchanged.                               |
| `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`   | Rewrite router expectations and shortcut-specific comments; retain all resolver/live-text-view regression coverage. |
| `README.md`                                           | Document the restored and shifted shortcuts, including Ctrl-K's restored native behavior.                           |

Do not add or remove `CaptureKeyCommand` cases and do not change the vertical movement
resolver, sticky goal-column state, or `perform(_:)` dispatch. Those pieces are already
behavior-named and remain valid after the keymap change.

## 2. Verified current behavior

- `CaptureKeyCommandRouter.KeyCode` currently declares `i = 34`, `j = 38`, and `k = 40`.
  Its editor switch maps exact Ctrl-I to `.insertBulletNewline`, exact Ctrl-J to
  `.moveToNextLineKeepingColumn`, and exact Ctrl-K to
  `.moveToPreviousLineKeepingColumn`.
- Routing compares the device-independent modifier flags by equality. This is the
  correct mechanism for distinguishing Ctrl-J from Ctrl-Shift-J and for rejecting
  additional Option or Command modifiers.
- `command(for:context:)` exits into the task-ID prompt, Pomodoro-name prompt, or stash
  picker before the editor switch. The editor shortcuts therefore do not apply in those
  modal contexts, and this isolation must remain intact.
- `.insertBulletNewline` already performs the established indentation-aware native
  `NSTextView` edit and dismisses completion. Only the key that selects the command must
  change.
- `.moveToNextLineKeepingColumn` and `.moveToPreviousLineKeepingColumn` already dispatch
  to `moveVertically`, which moves by physical lines, preserves a sticky UTF-16 goal
  column across ragged lines, clamps safely, handles composed characters, scrolls the
  caret into view, leaves completion open, and consumes an edge-of-document move. None
  of this logic must change.
- At present the app claims Ctrl-K, replacing AppKit's native `deleteToEndOfParagraph:`
  behavior. Once only Ctrl-Shift-K is claimed, exact Ctrl-K will fall through to AppKit
  again. That is an intentional consequence of the requested remap and must be stated
  accurately in the README.
- Ctrl-I will likewise become unrouted in the editor and return to native AppKit
  handling. The global Control-Shift-Command-I capture hotkey is a separate Carbon
  hotkey and remains outside this editor route.

## 3. Design decisions

1. **Use exact modifier sets.** Ctrl-J means exactly `.control`; vertical down/up mean
   exactly `[.control, .shift]` on physical key codes J/K. Extra Option or Command
   modifiers must return `nil`. This avoids stealing related system or AppKit shortcuts
   and matches the router's existing Ctrl-family policy. The pre-existing Caps Lock
   interaction remains out of scope.

2. **Route by virtual key code, not produced characters.** Continue using key codes 38
   and 40, so the Shift modifier changes the command without depending on whether macOS
   reports a control character or uppercase character for the event.

3. **Give Ctrl-J priority over Ctrl-Shift-J within the same key-code case.** The J case
   must return `.insertBulletNewline` for exact Ctrl and `.moveToNextLineKeepingColumn`
   for exact Ctrl-Shift. Do not use a broad `modifiers.contains(.control)` check, which
   would make the two bindings collide.

4. **Keep vertical movement behavior identical.** Ctrl-Shift-J/K retain physical-line
   semantics, sticky column restoration after a short line, selection collapsing,
   composed-character safety, no wrap, and completion re-anchoring. At the first/last
   line they remain consumed by `moveVertically`; this is symmetric and prevents a
   declined Ctrl-Shift-K from reaching an unexpected native command.

5. **Restore native exact Ctrl-K and Ctrl-I behavior.** Exact Ctrl-K no longer invokes
   custom upward movement and may once again perform AppKit's `deleteToEndOfParagraph:`
   action. Exact Ctrl-I is no longer claimed. Do not invent replacement commands for
   either native behavior.

6. **Keep modal contexts native/isolated.** Do not add J/K handling to
   `isolatedPromptCommand`, `stashPickerCommand`, or `printableModalCommand`. The Add
   block ID and Name Pomodoro fields keep native behavior, while the stash picker keeps
   its existing arrow and Ctrl-N/P navigation.

7. **Preserve completion semantics.** Ctrl-J still dismisses completion because bullet
   insertion edits text. Ctrl-Shift-J/K still leave completion visible and let the
   existing selection-change bridge re-anchor it.

## 4. Implementation

### 4.1 Router

In `CaptureKeyCommandRouter.swift`:

- Remove `KeyCode.i` if it becomes unused, and remove the `case KeyCode.i` editor route.
- Keep `KeyCode.j` and `KeyCode.k`.
- Make the J case distinguish the two exact modifier sets:

  ```swift
  case KeyCode.j:
      if modifiers == .control {
          return .insertBulletNewline
      }
      return modifiers == [.control, .shift] ? .moveToNextLineKeepingColumn : nil
  case KeyCode.k:
      return modifiers == [.control, .shift] ? .moveToPreviousLineKeepingColumn : nil
  ```

- Leave the early modal-context returns, all behavior-named command cases, and all
  controller dispatch unchanged.

This makes the routing matrix explicit:

| Input                                   | Editor command                     |
| --------------------------------------- | ---------------------------------- |
| Ctrl-I                                  | `nil` (native fallthrough)         |
| Ctrl-J                                  | `.insertBulletNewline`             |
| Ctrl-K                                  | `nil` (native fallthrough)         |
| Ctrl-Shift-J                            | `.moveToNextLineKeepingColumn`     |
| Ctrl-Shift-K                            | `.moveToPreviousLineKeepingColumn` |
| Any of the above plus Option or Command | `nil`                              |

### 4.2 Controller comments

Do not alter executable code in `CapturePanelController.swift`. Update every
shortcut-specific comment so behavior and key labels agree:

- Bullet-edit references currently naming Ctrl-I become Ctrl-J, including the helper,
  physical-line sibling lists, native-text-view summary, and placeholder deletion note.
- Vertical movement references currently naming Ctrl-J/Ctrl-K become
  Ctrl-Shift-J/Ctrl-Shift-K, including the direction/result types, sticky goal state,
  resolver, apply helper, and edge-consumption explanation.
- In the edge-consumption comment, explain that Ctrl-Shift-K is consumed at the first
  line. Do not continue claiming that exact Ctrl-K belongs to the custom editor keymap.

Use a final search rather than relying on old line numbers because master has advanced
since the predecessor plan.

## 5. Tests

Update `Tests/BobMacCaptureTests/BobMacCaptureTests.swift` without weakening the
existing movement algorithm tests.

### 5.1 General shortcut smoke coverage

In `testKeyRouterMatchesCaptureShortcuts()`:

- Change the bullet-insertion assertions (normal and completion-visible) from key code
  34/Ctrl to key code 38/Ctrl.
- Replace the old Ctrl-Shift-I negative assertion with an explicit Ctrl-I/Ctrl negative
  assertion, demonstrating that the removed binding falls through.
- Keep Ctrl-Shift-J/K mapping assertions in the dedicated vertical-routing test rather
  than overloading this general smoke test.

### 5.2 Bullet shortcut coverage

Rename `testKeyRouterMatchesControlIAsBulletNewlineOnlyWithControlModifier()` back to a
Ctrl-J name and update it to prove:

- key code 38 with exact Ctrl returns `.insertBulletNewline`, both with and without
  completion visible;
- J with no modifiers, Shift alone, and modifiers containing Ctrl plus Command/Option do
  not insert a bullet;
- Ctrl-Shift-J is specifically _not_ bullet insertion (it is the vertical-down command);
- key code 34 with Ctrl returns `nil`, proving Ctrl-I is no longer captured.

### 5.3 Vertical shortcut coverage

Rename `testKeyRouterMapsControlVerticalCaretMovement()` to identify Ctrl-Shift and
rewrite it to prove:

- key code 38 with exact Ctrl-Shift maps to `.moveToNextLineKeepingColumn` and key code
  40 with exact Ctrl-Shift maps to `.moveToPreviousLineKeepingColumn`;
- both mappings remain the same while completion is visible;
- exact Ctrl-J is bullet insertion, while exact Ctrl-K is `nil` and therefore available
  to AppKit;
- unmodified, Shift-only, Command-only, Option-only, Ctrl-Option, Ctrl-Command,
  Ctrl-Shift-Option, and Ctrl-Shift-Command J/K variants do not map to either vertical
  command.

Keep regression assertions for task-ID, Pomodoro-name, and stash-picker contexts, but
exercise the new combinations: Ctrl-J for bullet insertion and Ctrl-Shift-J/K for
vertical movement must all remain unrouted in the two prompt contexts and the stash
picker. For stash events, continue passing realistic control characters (`\u{0A}` and
`\u{0B}`) so `printableModalCommand`'s control-character guard is actually exercised.
Also retain a Ctrl-I modal assertion if useful to show the retired editor binding does
not acquire modal behavior.

### 5.4 Preserve movement behavior tests

Do not rewrite or delete:

- `testMoveVerticallyKeepsColumnInEditableTextView()`;
- `testVerticalMovementTargetKeepsColumnAcrossPhysicalLines()`;
- `testVerticalMovementTargetHandlesEdgeCaseDrafts()`;
- any tests ensuring edits invalidate the carried vertical goal.

They test behavior-named controller APIs and remain the regression suite for the new
Ctrl-Shift-J/K bindings. Only update comments that name the old shortcuts, including the
empty-bullet-row comment that currently says Ctrl-I inserted the row.

## 6. Documentation

In `README.md`:

1. Change the current `Ctrl-I` bullet-insertion row back to `Ctrl-J`, preserving its
   four behavior cells.
2. Rename the vertical movement rows from `Ctrl-J`/`Ctrl-K` to
   `Ctrl-Shift-J`/`Ctrl-Shift-K`; keep the descriptions of physical-line movement,
   clamping, completion, and edge stopping.
3. In the bullet-edit prose, change each Ctrl-I reference back to Ctrl-J. Re-read the
   nearby shortcut count after editing; the set of behaviors is unchanged.
4. Rewrite the vertical-movement paragraph to name Ctrl-Shift-J/K throughout while
   retaining the sticky-column, invalidation, no-wrap, undo, and IME guarantees.
5. Replace the now-false claim that Ctrl-K belongs to the editor and no longer deletes
   with an accurate note that exact Ctrl-K is again left to AppKit's native
   delete-to-end-of-paragraph binding, while Ctrl-Shift-K is the custom upward move and
   Ctrl-U remains Bob's custom line-deleting shortcut.
6. Do not add these editor bindings to the stash-picker table or imply that they apply
   in prompt fields.

## 7. Validation

From the opened `bob-mac-capture` checkout:

1. Search for shortcut labels and inspect every match:

   ```bash
   rg -n 'Ctrl-I|Ctrl-J|Ctrl-K|Control-I|Control-J|Control-K' Sources Tests README.md
   ```

   All bullet references must name Ctrl-J; all custom vertical references must name
   Ctrl-Shift-J/K; any exact Ctrl-K mention must describe native fallthrough rather than
   custom movement.

2. Parse all macOS sources and tests on the Linux host:

   ```bash
   export PATH="$HOME/.local/share/swiftly/bin:$PATH"
   swiftc -parse Sources/BobMacCapture/*.swift Tests/BobMacCaptureTests/*.swift
   ```

3. Run the cross-platform suite:

   ```bash
   swift build
   swift test
   ```

   On Linux, `Package.swift` excludes the AppKit target and its tests, so this protects
   `CaptureCore` but does not validate the changed router. Report any pre-existing test
   flake separately and rerun it on an unchanged tree before attributing it to this
   work.

4. Type-check the macOS app with an Apple toolchain where available:

   ```bash
   ./Scripts/xcode-swift.sh build
   ```

   If using a separate linked Mac checkout, transfer or check out the implementation
   safely, run the build, and restore that checkout afterward. Do not claim its XCTest
   suite passed if the machine only has Command Line Tools and lacks XCTest.

5. Treat the `macOS 26 SwiftPM` GitHub Actions job as the authoritative gate. It runs
   formatting lint, app build, XCTest (including `BobMacCaptureTests`), bundling,
   signature/plist checks, a launch smoke test, and install/reinstall checks.

6. If the app is available for manual verification, use `alpha\nhi\nbravo`: Ctrl-J
   should insert the canonical bullet row; Ctrl-Shift-J twice from column four should
   clamp on `hi` and restore column four on `bravo`; Ctrl-Shift-K twice should return;
   holding either vertical binding at an edge should not wrap or edit text; and vertical
   movement with completion visible should leave it open and re-anchor it. Also confirm
   Ctrl-I is no longer captured and exact Ctrl-K once again performs native AppKit
   behavior. If no suitable Mac is available, state that the manual check was skipped.

## 8. Out of scope

- No changes to physical-line or sticky-column semantics.
- No visual-line/soft-wrap navigation, wrap-around, or shift-selection behavior.
- No user-configurable keymap.
- No change to prompt or stash-picker navigation.
- No replacement for native Ctrl-I or Ctrl-K behavior.
- No global-hotkey change.
- No general Caps Lock modifier normalization.

## 9. Done when

1. Exact Ctrl-J invokes the existing indentation-aware bullet newline command and
   dismisses completion exactly as before the first migration.
2. Exact Ctrl-Shift-J/K invoke the existing next/previous physical-line movement,
   preserving the goal column, clamping safely, leaving completion open, and stopping
   without edits at document edges.
3. Ctrl-I and exact Ctrl-K are not claimed by the editor router; extra Option/Command
   variants are also not claimed.
4. Editor shortcuts remain isolated from the Add block ID prompt, Name Pomodoro prompt,
   and stash picker.
5. Existing resolver and live-`NSTextView` movement tests remain intact, and the updated
   router tests cover the complete modifier matrix and completion-visible behavior.
6. Source comments and README consistently distinguish Ctrl-J bullet insertion,
   Ctrl-Shift-J/K vertical movement, and exact Ctrl-K's restored native behavior.
7. The syntax gate and applicable local build/tests pass, the macOS app type-checks with
   an Apple toolchain, and the macOS CI job is green.
