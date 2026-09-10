---
tier: epic
title: Any-task @route+ completion with inline block-ID authoring
goal: "Bob Mac Capture can select any open task from @file+ completion, safely assign a
  user-authored block ID through Bob when needed, and clearly distinguish ready tasks
  from tasks that require an ID.

  "
phases:
  - id: bob_task_identity_contract
    title: Bob completion and task-ID mutation contract
    depends_on: []
    size: medium
    description:
      "bob_task_identity_contract: define opt-in all-task discovery, safe ID assignment,
      and the tested CLI wire contract."
  - id: mac_capture_task_id_prompt
    title: Beautiful stateful macOS selection and prompt
    depends_on:
      - bob_task_identity_contract
    size: medium
    description:
      "mac_capture_task_id_prompt: build grouped task presentation, inline ID authoring,
      reliable state transitions, and end-to-end app coverage."
proposed_by: bbugyi200.athena.02a
create_time: 2026-09-09 20:00:04
status: wip
---

- **PROMPT:**
  [prompts/202608/file_plus_any_task.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/file_plus_any_task.md)

# Select Any Parent Task from `@file+`

## Outcome

Make the Bob Mac Capture panel's `@route+` completion able to select every open Obsidian
task in the routed note, including tasks without block IDs. Existing IDs remain the fast
path. Selecting a task without an ID opens a focused inline prompt; submitting a valid
ID writes ` ^<id>` to that exact task through Bob, then replaces the completion range so
the draft immediately becomes `@route+<id>`. The feature must work for `@file+` /
`~/bob/file.md` and retain the route-generic contract already used by the capture
language.

This is an epic because it crosses two repositories and two ownership boundaries. The
Bob CLI phase defines discovery and mutation semantics; the dependent macOS phase builds
the interaction and presentation on that contract.

## Product and Design Contract

### Completion menu

- Keep only open tasks eligible, using the same Obsidian Tasks settings, task parsing,
  status metadata, and stale-safe refs as `bob capture-tasks`.
- In the `task` (`@route+...`) completion context, show two visually distinct groups:
  **Ready to use** first for tasks that already have block IDs, then **Needs block ID**.
  Preserve document order inside each relevance bucket and never interleave an
  unidentified task ahead of an identified task.
- Existing-ID rows use the established indigo block-ID treatment, a link/ready symbol,
  and a prominent `^existing-id` capsule. Missing-ID rows use a quieter warm/secondary
  treatment, an add-link symbol, and an outlined `Add ID` capsule. Both retain the
  literal checkbox/status, task description, section, nesting/child metadata, selection
  treatment, dark/light/high-contrast adaptability, and full VoiceOver descriptions.
- An empty query lists every eligible task. A non-empty query matches block ID (when
  present), task text, section, and status name/symbol case-insensitively; prefix
  matches precede substring matches within each of the two ID-availability groups.
- Do not change `@route:` Pomodoro completion: it still offers only tasks that already
  have block IDs because that flow has no author-ID prompt.

### Add-ID prompt

- Accepting an existing-ID row keeps today's one-step insertion behavior.
- Accepting a missing-ID row leaves the draft untouched and replaces the menu with a
  compact inline card titled **Add block ID**. The card repeats the selected task and
  status, shows a fixed `^` prefix beside a focused monospaced text field, explains
  `Letters, numbers, and hyphens`, and offers **Cancel** plus **Add & Select**.
- Validate the simple character rule locally for immediate feedback, but treat Bob's
  validation as authoritative for duplicates and vault state. Disable **Add & Select**
  for an empty or syntactically invalid value.
- While saving, show progress (`Adding to file.md…`) and prevent draft edits or a second
  submission. Return/Enter submits the prompt, Escape cancels back to the task list with
  the previous selection, and Control-C retains its explicit discard-and-close meaning.
  Cancel performs no vault mutation.
- Validate the draft snapshot and UTF-8 replacement range before starting the write. On
  success, use the canonical block ID returned by Bob, splice it into the saved
  replacement range, place the caret after it, return focus to the capture editor, and
  rerun parse/live preview without reopening completion. The task write must finish
  before the draft expands.
- On duplicate-ID, stale/ambiguous task, terminal-task, or I/O failure, keep the draft,
  selected task, and authored ID in the prompt; show the actionable server error inline
  and allow correction, retry, or return to the refreshed task chooser. Never imply the
  block ID was added until Bob confirms the write.

### Ownership, compatibility, and safety

- Bob remains the only vault writer. The Swift app must not open or rewrite `file.md`
  directly.
- Keep the current `capture-complete` behavior for callers that do not opt in. Add an
  explicit `-a, --all-tasks` completion option; it includes missing-ID tasks only for
  the `task` / plus context. This prevents older Bob Mac Capture builds from receiving
  action candidates they do not understand.
- Missing-ID candidates carry their route, stale-safe task ref, nullable block ID, and
  an explicit action/requirement field (for example `requires_block_id: true`). Their
  placeholder `replacement` must never be inserted by the updated client. Existing
  candidates keep their normal block-ID replacement. Keep the change additive within
  completion schema version 1 and document the new fields and opt-in behavior.
- Add a dedicated versioned Bob command for the write, named `bob capture-task-id`:
  `--route/-r`, `--task-ref/-t`, `--block-id/-i`, `--bob-dir/-b`, `--format/-f`, and
  `--dry-run/-d`, plus excellent sorted help and examples. JSON success returns schema
  version 1, route/target details, the canonical ID, updated line/ref, and task
  metadata; JSON failure preserves a machine-readable, actionable error.
- The command validates the route and block-ID grammar with Bob's shared validators,
  resolves the selected task by stale-safe ref (including line-shift recovery), confirms
  it is still open and lacks an ID, rejects IDs already used anywhere in the routed
  note, appends the ID only to the selected task line while preserving its line ending
  and all unrelated bytes, and replaces the note with the repository's same-directory
  temporary file/rename discipline. A stale, edited, ambiguous, terminal,
  already-identified, or missing task fails without writing.

## End-to-End State Flow

```text
@file+ completion
        |
        +-- task has ^id ------> splice existing id ------> @file+id
        |
        +-- task needs id -----> inline Add block ID card
                                      |
                                      +-- Cancel/Error --> draft unchanged
                                      |
                                      +-- Submit valid id
                                             |
                                  bob capture-task-id
                                             |
                                      atomic task write
                                             |
                                  splice returned id ------> @file+id
```

## Phase 1: Bob Completion and Task-ID Mutation Contract

Work in the primary `bob-cli` repository.

1. Extend `src/native/capture_complete.rs` with the opt-in all-task request flag and an
   additive task-candidate representation that can carry `route`, nullable `block_id`,
   stale-safe `ref`, and an explicit requires-ID action. Partition identified tasks
   before unidentified tasks, implement the multi-field prefix/substring search
   contract, and preserve the identified-only behavior for the Pomodoro context and for
   callers that omit `--all-tasks`.
2. Add `src/native/capture_task_id.rs` and register `capture-task-id` in the sorted
   native command and top-level help tables (`src/native.rs`, `src/runner.rs`). Reuse or
   extract task-ref parsing, block-ID validation, task scan/range data, duplicate
   detection, and atomic single-note replacement helpers instead of creating a second
   interpretation of Bob task syntax.
3. Implement dry-run, human, JSON-success, and JSON-failure output. The mutation plan
   must minimally append ` ^<id>` to the resolved physical task line and calculate the
   post-write ref from the updated line. Real execution performs one atomic replacement;
   every validation/error path is write-free.
4. Update `README.md` for the new command, option, candidate fields, ordering/search
   behavior, and the fact that missing-ID discovery is opt-in and plus-context-only.
5. Add focused unit and CLI integration coverage in the native modules and
   `tests/cli.rs` for:
   - default identified-only compatibility versus `--all-tasks`;
   - identified-first stable ordering and search across ID/text/section/status;
   - nullable-ID candidate JSON, route/ref/action metadata, and unchanged Pomodoro
     results;
   - successful and dry-run insertion with LF and CRLF notes;
   - shifted-line ref recovery and the returned updated ref;
   - invalid/duplicate IDs, non-task duplicate anchors, stale/ambiguous refs, tasks that
     became terminal or gained an ID, missing/unreadable notes, and proof that every
     failure leaves bytes unchanged;
   - sorted/help/native-dispatch/package-list invariants for the new command.

Phase acceptance:

- `bob capture-complete --all-tasks ... '@file+'` returns every open task with all
  identified tasks before all unidentified tasks; the same call without the option and
  every `@file:` call remain identified-only.
- `bob capture-task-id` is the sole operation needed to turn a returned missing-ID
  candidate into an identified task, and its success is observable only after the atomic
  note replacement completes.
- Run `just all`, `just check-scripts`, and `just package-list` successfully from
  `bob-cli`.

## Phase 2: Beautiful, Stateful macOS Selection and Prompt

Before reading or editing the linked repository, open it with
`sase repo open bob-mac-capture -r "Implement the approved @file+ any-task design"` and
use the printed checkout path.

1. Extend `CaptureCore` models to decode the additive completion action/nullable-ID
   metadata and the versioned `capture-task-id` result. Update `BobProcessClient` to opt
   `capture-complete` into `--all-tasks` and add an assignment method on its own process
   lane. Decode structured JSON errors even on nonzero exit so duplicate/stale failures
   reach the UI intact.
2. Introduce an explicit pending-ID state in `CapturePanelModel` containing the selected
   candidate, route/ref, immutable draft snapshot and UTF-8 replacement range, authored
   ID, progress, and error. Branch completion acceptance by the explicit candidate
   action; validate before mutation, call Bob once, and only apply the returned ID after
   success. Cancel/escape restores the chooser and selection without mutation; retry
   retains input; unrelated late analysis or process responses cannot overwrite current
   state.
3. Expand key routing/controller interaction modes so Return, Escape, Tab/arrows, and
   Command-Return cannot accidentally submit the capture while the Add-ID card is
   active. Preserve current editor, completion, discard, close, and plain-text paste
   behavior in every other state, including predictable first-responder/focus
   restoration.
4. Refine `CompletionRowContent`, `CaptureEditorPalette`, and `CapturePanelView` to
   render the two labeled completion groups, the ready-versus-needs-ID icons/capsules,
   and the inline prompt specified above. Keep rows scannable at the existing five-row
   viewport, let the measured panel sizing absorb the prompt and validation message, and
   provide explicit VoiceOver labels/hints plus adaptive light/dark/increased-contrast
   styling.
5. Update the fake Bob fixture and tests across `CaptureModelTests`,
   `BobProcessClientTests`, `CompletionRowContentTests`, `CapturePanelModelTests`, key
   router/controller tests, and layout tests. Cover:
   - decoding both task kinds and invoking the exact opt-in/assignment argv;
   - identified rows taking the existing immediate path;
   - missing rows opening the prompt without changing the draft or vault;
   - local validation, submit progress, server error retention/retry, cancel/back, stale
     response suppression, and one successful mutation followed by exact `@file+new-id`
     expansion/caret placement;
   - keyboard isolation, focus restoration, group order, visual-content semantics,
     accessibility strings, and panel height changes;
   - regression coverage for route/section/Pomodoro/wikilink completions and normal
     capture submission.
6. Update the macOS `README.md` completion, keyboard, privacy, troubleshooting, and Bob
   compatibility sections. State plainly that the app sends task metadata and the
   authored ID only to the local `bob` subprocess and that Bob performs the vault write.

Phase acceptance:

- In a fixture matching `~/bob/file.md`, existing-ID tasks appear first and insert in
  one action; unidentified tasks are unmistakable, open the inline prompt, and mutate no
  file until **Add & Select** succeeds.
- A confirmed ID appears on the selected Obsidian task before the panel expands the
  draft to `@file+<id>`; errors and cancellation never create a partial UI or file
  state.
- The entire flow is keyboard-complete, VoiceOver-descriptive, adaptive in light/dark
  and increased-contrast appearances, and does not regress existing completion contexts.
- Run `just format-lint`, `just build`, `just test`, and `just bundle` successfully from
  `bob-mac-capture`.

## Final Cross-Repository Verification

After both phases, build Bob from the primary repository and configure the macOS app to
use that executable against a temporary vault containing a `file.md` with mixed open,
terminal, identified, unidentified, nested, and duplicate-description tasks. Exercise
the mouse and keyboard paths for existing selection, add-ID success, cancel, invalid ID,
duplicate ID, and a task edited between completion and confirmation. Confirm exact note
bytes outside the chosen line remain unchanged and capture a final dark/light plus
VoiceOver inspection before declaring the epic complete.

## Non-Goals

- Do not auto-generate or silently choose block IDs; the user authors the identifier.
- Do not add missing-ID support to Pomodoro (`:`) completion in this epic.
- Do not capture the draft when **Add & Select** succeeds; that action only identifies
  the parent and completes `@route+<id>`.
- Do not let the macOS app edit Markdown directly or add a second task parser.
- Do not broaden task discovery to done/canceled tasks.
