---
tier: epic
title: Safe status sections for bob task-status-hooks
goal: "Group project and area Tasks sections by final task status while preserving
  authored context and minimizing concurrent-edit risk through guarded, recoverable note
  writes.

  "
phases:
  - id: guarded_writes
    title: Protect task-status-hooks writes against concurrent vault edits
    size: medium
    depends_on: []
    description:
      "guarded_writes: route existing hook writes through snapshot validation, shared
      maintenance locking, staged replacement, recovery records, and accurate failure
      outcomes."
  - id: status_group_transform
    title: Implement a lossless Markdown status-group transformation
    size: medium
    depends_on: []
    description:
      "status_group_transform: preserve task subtrees and authored topic context while
      producing stable status headings, source-aware change records, and conservative
      skip diagnostics."
  - id: integrate_status_groups
    title: Integrate grouping, reporting, compatibility, and acceptance coverage
    size: medium
    depends_on:
      - guarded_writes
      - status_group_transform
    description:
      "integrate_status_groups: compose grouping after final status derivation, enable
      guarded structural writes, expose clear reports, and verify capture and collection
      compatibility."
proposed_by: bbugyi200.athena.0if
create_time: 2026-09-10 12:54:00
status: wip
---

- **PROMPT:**
  [prompts/202609/task_status_groups.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/task_status_groups.md)

# Safe status sections for bob task-status-hooks

## Outcome and tier

Extend the existing command to organize Tasks sections in active area and project notes
into readable, foldable status subsections after deriving task statuses. Preserve the
user's task hierarchy, authored context, capture workflow, and order within each group.
Protect every write made by this command against stale input, with recoverable originals
and explicit reporting when a run cannot safely finish.

This is an epic: safe filesystem application and lossless Markdown rearrangement are
separate substantial components with independently testable contracts. Both must land
before one integration phase enables automatic grouping. Each phase is bounded medium
implementation work; no phase needs another planning exercise. The two foundation phases
have no dependencies; integration depends on both.

This proposal authorizes implementation in bob-cli after approval. Development and
acceptance use disposable fixture vaults. Do not run a mutating command against the
personal vault as an implementation test. No linked plugin, macOS frontend,
configuration, memory, or deployment change is required.

## Relevant existing behavior

- `src/native/task_status_hooks.rs` owns `sync_task_statuses`, the active vault scan,
  `note_kind`, Tasks settings, status derivation, daily-note structural cleanup,
  `compose_outputs`, output reporting, and a local `atomic_write`.
- `FileScan` retains original contents. `compose_outputs` currently produces only
  `(path, replacement)` pairs; `apply_outputs` replaces these files without checking
  their original contents again. Its predictable PID temporary filename is deleted
  before reuse. Atomic rename prevents a partially written replacement from being
  observed but does not prevent overwriting an intervening save.
- Preserve current ledger guards: a missing current daily note, missing Pomodoros
  section, multiple open timed Pomodoros, or invalid required Blocked registry
  definition still fail before writes. The selected previous daily remains read-only.
- Area/project classification already delegates to `projects::frontmatter_is_area` and
  `frontmatter_is_project`, including supported quoted `type: "[[area]]"` and
  `type: "[[project]]"` values. Reuse these predicates, not folder-name guesses or a new
  classification policy.
- Reuse active-scan exclusions: hidden directories, `done`, `_generated`, and
  `_templates`. Read-only archive resolution must remain outside grouping.
- `markdown.rs` supplies frontmatter, fence, and ATX-heading helpers. `collect_done.rs`
  and `note_tasks.rs` demonstrate task-subtree preservation, but their indentation-based
  helpers are not complete Markdown parsers: do not assume they already handle every
  fence or lazy continuation case required below.
- `capture.rs::tasks_section` deliberately stops at the first subsequent heading.
  `insert_task_line` therefore puts ordinary captured Ready tasks directly below Tasks,
  before status subsections. Keep this behavior. `capture_task_sections.rs` concerns
  uppercase list-item sections inside tasks, a different concept.
- `ob.rs` owns the maintenance lock shared by vault-sync and nightly maintenance. `fs2`,
  `sha2`, and `hex` already exist as dependencies. Follow the state directory convention
  used by `vault_sync.rs`: `${XDG_STATE_HOME:-$HOME/.local/state}/bob-cli`.
- Relevant contracts and checks: `docs/task-status-hooks.md`, `docs/capture.md`,
  README's task-status-hooks section, `tests/cli.rs`, unit tests in the native modules,
  and `justfile` (`cargo fmt --check`, clippy, and cargo test).

## User-facing layout

Use exactly these three generated group titles, in this order:

1. **Next & In Progress**: canonical Next `[*]` and WIP `[/]` tasks together.
2. **Blocked**: canonical Blocked `[?]` tasks.
3. **Done & Canceled**: recognized completed or canceled tasks together.

Ready `[ ]` tasks remain in the unheaded intake directly beneath Tasks, along with
introductory prose. This is an intentional treatment of the status the request did not
assign a group: it retains existing capture placement and avoids a fourth generated
subsection. Other recognized open statuses and unknown statuses are not invented into
one of the three groups. They also belong in the unheaded intake when relocating them
out of a previously managed group; otherwise leave them there.

For a conventional note, the rendered Markdown reads as follows. The example is the
result after existing status reconciliation, not a new source of statuses.

```markdown
## Tasks

Short context for this project.

- [ ] #task An idea to pick up later ^idea

### Next & In Progress

- [/] #task Finish the design ^design
  - Keep the keyboard interaction simple.
- [*] #task Review the implementation ^review

### Blocked

- [?] #task Ship when the dependency is ready ^ship

### Done & Canceled

- [x] #task Agree on the scope ^scope
- [-] #task Superseded experiment ^experiment
```

Keep headings plain: no emoji, volatile counts, extra task tags, artificial task IDs,
timestamps, callouts, or plugin-specific rendering. Native Obsidian heading folding lets
the user collapse Blocked and Done & Canceled. Do not edit Obsidian fold state. Preserve
source order inside each group; do not sort WIP ahead of Next or sort by priority/date.
Moving a task between groups should be the smallest necessary structural change.

Create the three headings together when a container first has a groupable task. Do not
decorate empty or Ready-only Tasks sections. Once created, keep all three headings even
when one empties, preserving heading links and fold targets. Subsequent runs with
unchanged semantic input must produce byte-identical notes and no writes.

## Markdown transformation contract

### Discovery and preservation of context

Discover every real heading whose normalized plain title is exactly `Tasks` (ASCII
case-insensitive), at every location in every eligible note. Support ATX levels 1–5 and
Setext Tasks headings at their Markdown levels. A section extends to the next heading of
equal or shallower depth. Never treat YAML, fenced or indented code, HTML comments, or
blockquoted examples as document headings or task roots. Do not create a Tasks section
where none exists. H6 cannot have a valid Markdown child heading: leave that container
unchanged and report the limitation.

Preserve existing authored topic headings. Treat each Tasks section and each of its
authored descendant heading sections as a container of direct task blocks. Group direct
blocks locally instead of extracting them from their topic. This means a pre-existing
`### Backend` receives `#### Next & In Progress`, etc.; it does not lose its context or
move under a different topic. A nested Tasks heading is processed once through the same
heading tree, not again through an overlapping outer-section rewrite. H6 leaf containers
are explicitly reported as skipped.

Generated groups are immediate children of their container. Insert them after its
authored child sections, in canonical group order, so even authored headings that skip
levels retain their original ancestry. The unheaded intake remains before the first
child heading. Preserve authored child ordering, titles, and heading levels. In ordinary
sections with no authored children this gives the compact layout above.

### Task blocks and classification

Only tasks accepted by the command's existing Obsidian Tasks globalFilter rules
participate; honor an explicitly empty filter. A block ID is not required.
Classification uses the final composed task symbols and existing completion/
cancellation semantics, including configured single-character DONE and CANCELLED symbols
and conventional `[x]`/`[X]` completion. Explicit Next/WIP/Blocked symbols must not be
grouped by status type alone: Next may be ON_HOLD too. Reuse the existing status
registry's precedence; do not reinterpret NON_TASK, EMPTY, or an unrecognized symbol as
completed or blocked.

The indivisible move unit is a root task's complete Markdown list-item subtree. All
descendants, nested checkboxes, notes, links, IDs, inline metadata, log bullets,
continuations, blank lines internal to the item, and fenced snippets travel with their
parent. The parent's final status chooses the group; children are never lifted out or
separately regrouped. A task nested under an ordinary authored list item also stays with
that item: report it as structurally ineligible rather than destroying its hierarchy.
Never split or renumber an ordered list to make it fit a status group; ordered-list
roots with context-sensitive numbering should be preserved and reported as unsupported
in v1.

Use source byte ranges, not Markdown serialization. Preserve task payload bytes,
indentation, list markers, IDs, Unicode, links, and metadata exactly except for the
existing intentional checkbox edits that precede this transform. Preserve each existing
line ending. Newly inserted lines use the local section's newline style; preserve the
file's original final-newline convention. Changing group separator whitespace must not
normalize the remainder of the file. Do not attach standalone prose, a query block, or
an unrelated sibling bullet to a preceding task. Ambiguous lazy continuations or other
ambiguous boundaries are a diagnostic and a skip for that container, not a guessed move.

### Ownership and manual edits

Identify generated headings with an unobtrusive standalone HTML comment immediately
below each heading, e.g. `<!-- bob:task-status-group:v1:active -->`, with `blocked` and
`closed` IDs for the other groups. These comments are hidden in rendered notes and are
excluded from the visual example. The marker owns the heading and group slot, not
arbitrary neighboring user text. This is source ownership metadata; it must not change
the heading title/anchor or become an Obsidian block ID.

Existing unmarked headings with these exact titles may be adopted only when the whole
candidate body consists of unambiguous eligible task blocks and whitespace, there are no
child headings, and there is at most one candidate of each title in the container. Keep
their task order. If an unmarked matching heading also contains authored context, report
a collision and skip that container; do not hijack it or create a second heading with
the same title. A partial set of safely adoptable headings can be completed to three.

Malformed/duplicate markers, a renamed marked heading, or authored child headings
inserted inside a managed group must fail closed for that container and identify the
affected note and heading in the warning. Do not silently repair ownership by deleting
user text. Valid plain prose inside an established managed group stays with that group
while tasks move in or out. Unknown/unfiltered content remains preserved. Direct Ready
tasks recovered from managed groups append to the existing intake task sequence, leaving
introduction/context intact. For each destination, stable-partition source task blocks
in their current document order. Re-running must neither duplicate tasks nor churn group
markers and blank lines.

The pure transform returns changed bytes plus structured grouping and skip records. It
does not read the filesystem, mutate task statuses, resolve links, or write files.

## Write safety contract

### A guarded plan, not a stale replacement

Replace the hooks writer with a small dedicated module, scoped to this command for now.
A planned write carries original bytes, proposed bytes, the actual/canonical path, and
the captured regular-file identity and metadata. Capture inputs with
metadata/read/metadata bookends and reject an unstable read. Maintain a read-set of all
actual planning inputs: scanned notes, current and selected previous daily, explicitly
loaded archive notes, Tasks settings including missing/present state, and the scan
manifest used for reference resolution and earlier-day selection.

For a live run, acquire the existing shared vault-maintenance lock before planning and
hold it through application, excluding simultaneous hooks/vault-sync/nightly runs on
this machine. Refactor only the lock outcome plumbing needed for hooks to report
contention in its own human/JSON output; preserve other commands' behavior and
`BOB_VAULT_SYNC_LOCK_FILE`. Obsidian and other independent writers do not honor this
advisory lock, so lock acquisition never substitutes for input validation.

Before the first replacement, re-read and compare every planning input and verify the
manifest, settings state, file identity, and path destination still match. Compare
complete bytes, not only mtime, length, or an abbreviated digest. If anything changed,
write no note and report that the vault changed and the command should be rerun. Do not
rebase stale line offsets or automatically retry a partially applied plan. Inputs
already replaced by this run are compared with their planned outputs where relevant to
subsequent checks, not their old snapshots.

For outputs that structurally regroup tasks, add a bounded quiet-period check: require
two seconds since their observed on-disk modification time. Wait only the remaining
portion of that interval, once for the whole set (maximum two seconds), then revalidate.
If an editor saved during that interval, defer the run without writing any note. Do not
loop until the user stops typing. Use real filesystem/ monotonic time, never BOB_NOW;
uncertain/future timestamps fail conservatively. Stable grouping candidates incur no
delay. Status-only changes keep their existing timing, but still use snapshot checks and
guarded application.

Immediately before each individual replacement, repeat byte/identity/path checks for
that target and validate that no previously written output has since diverged. If a
later target changes or I/O fails after earlier replacements, stop further writes and
report the exact applied paths and remaining paths. Never roll back an earlier note
automatically: a newer editor save may already be there. A multi-file run is not a
transaction, and output must never claim it is.

### Staging, recovery, and honest guarantees

Stage all changed outputs first in unique, exclusively created temporary files in their
destination directories; never unlink a predictable filename owned by someone else.
Preserve source permissions, flush/sync staged contents, and use same-directory atomic
rename. Clean up only this run's own unused temporary files on every error path.
Revalidate after staging. Reject non-regular files, symlink substitutions, unexpected
destination changes, and unsupported multiply linked output files instead of replacing
an unexpected object. Preserve supported file metadata or explicitly fail where safe
preservation is not available on a supported platform. Preserve relevant access
permissions/ACLs and extended attributes; do not restore the old modification time onto
newly changed contents.

Before any note replacement, durably retain exact original and proposed bytes for all
outputs in a unique private recovery directory under
`${XDG_STATE_HOME:-$HOME/.local/state}/bob-cli/task-status-hooks/<vault-hash>/<run-id>/`.
Use a canonical-root hash to isolate vaults. Include a manifest of note paths,
before/after full hashes, planned/applied state, and run outcome; sync recovery data
before replacing notes. State directory/files should be private (0700/0600 on Unix).
Recovery failure prevents note writes. Never store backups as Markdown siblings inside
the vault. Keep incomplete/conflicted recovery records until user removal; retain
completed runs for 30 days and prune only this tool's positively identified completed
records older than that window during a later successful live write run. Dry-run and
no-op do not create or prune recovery records. A live no-op may create the ordinary
runtime maintenance-lock file, but no note staging files. Cleanup failure warns without
misreporting a completed note write as rolled back.

Document the practical limit precisely: byte checks and atomic replacement minimize
lost-update risk but cannot provide a true compare-and-swap against an editor that does
not coordinate with the command. An unsaved Obsidian buffer is not observable, and a
save can race the final check/rename or follow the command. A quiet interval is a
heuristic, not a guarantee. A post-write divergence check must not overwrite a newer
save. Recovery copies protect the observed originals and intended outputs, not every
unobserved editor version. Recovery instructions must compare/merge into the current
note, not blindly restore an entire vault. Cross-machine sync locking and an Obsidian
IPC editing protocol are outside this feature.

## Integration and output contract

Compose in one direction: original scan -> existing final checkbox changes -> existing
daily structural edits -> status-group transformation of eligible notes -> one guarded
write per changed path. Grouping must also run on eligible notes with no checkbox
changes. Exclude all canonical daily notes, the exact selected daily (including an
override outside the vault), and the selected previous daily from grouping even if they
have area/project frontmatter. Other task status and archive-resolution behavior remains
intact. Grouping uses composed bytes, never old offsets after a structural move.
Preserve original source line numbers in existing status-change reports and derive new
grouping locations from original source mappings.

Keep the command name, hidden aliases, options, and existing JSON fields. Add no new
public flags or configuration knobs. Dry-run computes the same transformation and
describes proposed changes without locks/state/tempfiles/backups/note writes; it may
report a changing snapshot rather than give a false stable preview. It does not wait out
the live quiet interval or pretend its preview reserves later writes.

Add `grouped_task_sections` (only changed containers), `grouping_warnings`,
`applied_files`, `deferred_files`, and nullable `recovery_directory` fields. A changed
container record includes path, original one-based heading line, heading ancestry, group
counts of root task blocks, and moved-block count. Count nested children neither as
extra moves nor duplicate grouping records. Heading-only first setup is still a change.
Preserve all existing field types and original report ordering.

On a successful live run, records describe applied edits; dry-run records describe
proposed edits and `applied_files` is empty. Extend the existing failure envelope
(`ok: false`, `error`) with a stable reason code and applied/deferred paths, including
the recovery location where one exists. Lock contention or a changing/actively saved
input is a retryable deferred result with exit 1 and no fabricated success records. A
partial apply also exits 1. Container-level unsupported Markdown or ownership warnings
do not fail otherwise safe work, but are structured in JSON and visible on stderr. Do
not print `already in sync` when grouping changed, grouping was skipped, or application
was deferred.

Human output adds a compact “grouped task sections”/“would group task sections” section
using the existing Styler, with note/heading context and the three counts. Summarize
preserved/skipped containers clearly. On a live changed run, include a short
recovery-path line. Explain quiet-period deferral and partial application in plain
language. Preserve stdout's single-object JSON contract and the canonical help/alias
behavior. Update long help and docs with layout, Ready behavior, custom topics, empty
headings, safety limits, recovery, and dry-run examples.

## Phase guarded_writes

Implement the snapshot/read-set, structured planned-write and apply-outcome types,
shared-lock integration, staging, recovery, and failure reporting in a dedicated native
module plus narrow hooks/ob wiring. This phase must actually route existing hooks writes
through the guard; no dead safety abstraction left for integration. Expose a plan
annotation for structural regrouping so the integration phase can activate the
quiet-period policy without duplicating application logic. Register the new module in
`src/native.rs`. Preserve existing hook semantics and reports.

Tests must use temporary fixture vaults and injectable filesystem-boundary callbacks or
barriers/clock values, not probabilistic threads or sleeping races. Cover:

- Changed contents of equal length and restored mtime, replacement inode, deletion,
  symlink substitution, changed Tasks settings, and changed reference/previous-daily
  inputs all prevent the first note replacement.
- A new/deleted scan candidate invalidates a plan; an unchanged read-set applies.
- Two cooperating runs contend on the maintenance lock without interleaved writes.
- An edit between scan and preflight, between staging and preflight, or before a later
  replacement survives. Later failure reports partial application accurately.
- Exclusive temp creation, original mode preservation, failure during staging, recovery
  persistence or rename, and cleanup preserve existing notes/foreign temps.
- Original/proposed recovery copies and manifest match actual bytes; interrupted runs
  stay recoverable; retention never removes incomplete records or other state.
- Quiet-period logic is bounded and based on real file timestamps; stable files avoid
  waits, changed files defer, BOB_NOW cannot bypass it. No-op creates no recovery or
  note staging artifacts; dry-run additionally creates no lock file.

Run focused writer and existing task-status-hooks tests. Establish deterministic helpers
the integration phase can reuse. Do not migrate other CLI writers.

## Phase status_group_transform

Implement the pure scanner/transform in `src/native/task_status_groups.rs`, with only
narrow reusable Markdown helpers in `markdown.rs` and module registration. Define
explicit input task classification supplied by hooks or a small neutral type; do not
make this module depend on filesystem scanning or hook CLI execution. This phase does
not change `sync_task_statuses`, `ob.rs`, or CLI output. Parallel foundation phases
should reconcile their small `src/native.rs` registration edits without deleting each
other's modules.

Build golden input/output fixtures and targeted unit tests for the full transform
contract: every status bucket and Ready intake; blockless and duplicate-ID tasks; custom
terminal statuses; final-status input; globalFilter behavior; complete nested task
blocks and mixed-status children; indentation/tabs, internal blanks and fences; multiple
Tasks headings, heading levels/closing hashes/Setext, authored topics, nested Tasks,
excluded Markdown contexts, and H6 diagnostics. Include CRLF, mixed endings, Unicode,
and missing final newline. Cover safe legacy adoption, marker collisions, manual prose,
malformed ownership, empty retained groups, reopening a task to Ready, and unsupported
hierarchy/ordered-list diagnostics.

Assert byte-for-byte idempotence, stable relative ordering, unchanged non-task payloads
and out-of-scope spans, and conservation of each task block exactly once. Return
source-aware records so integration need not reconstruct original line locations after
movement. Golden rendered examples must match the intended layout.

## Phase integrate_status_groups

After both foundations pass, wire the transformer after status/daily composition and
before guarded application. Carry original snapshots through every final note output;
set the structural-regrouping annotation only for actual regrouping. Integrate
changed/skipped records, quiet/deferred outcomes, recovery locations, human output,
JSON, canonical help, and docs. Keep all existing ledger guards and read-only
archive/previous-daily rules authoritative for the whole invocation.

Add CLI fixture coverage for:

- Several area/project notes in arbitrary active directories, several Tasks sections, an
  unlinked/blockless task, and an unchanged-status task that still needs grouping.
  Verify ordinary/reference/daily/archive/template/generated notes and tasks outside
  Tasks receive no new grouping edits.
- Promotion, stale Next/WIP clearing to Ready, dependency/future-date blocking, unblock,
  completion, cancellation, and reopening all land in the correct group in the same run;
  a second run is a true byte/mtime no-op.
- Existing daily structural edits and grouping elsewhere compose once per file.
  Missing/invalid current ledgers and required Blocked-registry failure still write
  nothing, including group headings, recovery payloads, or partially staged notes.
- Preview leaves every file and state directory unchanged; JSON/human reports agree with
  written bytes, grouping-only output is not called a no-op, and old fields and
  compatibility aliases remain usable.
- A saved edit at deterministic preflight/commit boundaries survives, produces the
  promised deferred/partial envelope, and has no false applied-grouping records.
- `bob capture` adds a new Ready task to intake in a grouped note. A subsequent hooks
  run groups it only when its final status warrants it. Existing block-ID task lookups,
  task-reference stale handling, nested task-section capture, and move-done-tasks still
  find/move the correct complete blocks after rearrangement. Do not broaden capture's
  section-selection grammar merely to implement grouping.

Run focused new tests and all existing task-status-hooks/capture/collection CLI
coverage, then `just all` once on the final tree. Investigate failures and repeat
affected checks after fixes. All live demos use disposable vaults, including the manual
before/after visual read-through and one dry-run/apply/no-op sequence.

## Acceptance and boundaries

The feature is complete when standard project/area Tasks lists acquire the three
specified readable groups, authored task/topic context survives, final derived statuses
and grouping agree in one run, and the safety/error/recovery behavior is covered by
deterministic tests. Unsupported structures must be visible and intact. No
implementation may claim all concurrent editor saves are race-free. No source or vault
changes are part of authoring this proposal; only the scratch plan is created before
validation and submission.
