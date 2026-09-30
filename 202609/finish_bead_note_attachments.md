---
tier: tale
size: medium
title: Finish and land the bead note attachments epic
goal:
  The seven remaining defects of epic sase-1ck are fixed, just check gates it broke pass
  again, and the epic is closed with its plan marked done.
proposed_by: bbugyi200.athena.sase-1ck.land
bead: sase-1ck
create_time: 2026-09-30 00:00:02
status: wip
---

- **PARENT:**
  [202609/bead_note_attachments.md](https://github.com/sase-org/sase--plans/blob/main/202609/bead_note_attachments.md)
- **BEAD:**
  [sase-1ck](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ck/README.md)

# Plan: Finish and land the bead note attachments epic (sase-1ck)

## Context

Epic `sase-1ck` (bead note attachments) has all ten phases and both nested epics
(`sase-1ck.4.1`, `sase-1ck.5.1`) closed. The shipped feature works: the focused
attachment suites pass (160 tests). The land audit reproduced seven remaining defects
caused by this epic, and this tale fixes all of them and then closes the epic. The land
agent has already triaged every follow-up and recorded the outcomes on `sase-1ck`. Task
beads `sase-1cy`, `sase-1cz`, `sase-1d0`, `sase-1d1`, `sase-1d2`, `sase-1d3`, and
`sase-1d4` are filed, so do **not** file or re-triage follow-ups. Reference: epic plan
`plan:202609/bead_note_attachments.md`. Read it with `sase artifact read` only if you
need a detail that is not written here.

Rules:

- Rust core rule (`rust_core_backend_boundary`): shared rules change in the linked
  `sase-core` checkout. Open it with `sase repo open sase-core -r "<why>"` and follow
  its `AGENTS.md`. Do **not** hand-edit `sase-core-revision.txt`. `sase/sase.yml`
  declares `revision_pin` for sase-core, so the host commits the sase-core change first
  and records its pushed SHA in the pin file on the same landing.
- Run `just fix` before verification. Verify with `sase tool run check`, never raw
  `just check`, and never run `just check-full`. If a command may run long, hand it to
  `/sase_monitor`.
- Keep modules under the `toobig` limit.

## Work items

### 1. Terminology audit: classify the sase-core corpus fixture

`tools/audit_patch_stitch_terminology --repo-root . --allow-missing-linked-repos` exits
1 with 14 `defect (unclassified)` hits. They are all in sase-core
`crates/sase_core/tests/fixtures/note_attachment/at_bearing_notes.jsonl`, the corpus
golden added by `sase-1ck.1` (sase-core `39324ac`). The file holds verbatim historical
bead note texts, the same kind of data as `.beads/`, so it is immutable history.

- In `src/sase/patch_stitch_audit.py`, make `_is_immutable_history` (which currently
  ignores `repo`) also return true for `repo == "sase-core"` and that fixture path.
  Declare the path in a small named frozenset next to
  `_SASE_CORE_MIGRATION_COMPATIBILITY_PATHS`, with a comment saying it is an exported
  bead note corpus. Do not classify all of `tests/fixtures/`, and do not edit the
  fixture: the corpus golden pins its contents.
- Add a classifier test to `tests/test_patch_stitch_terminology_audit.py`, mirroring
  `test_classifier_accepts_sase_core_patch_record_migration_headers`. It should assert
  that a `ChangeSpec` line in that fixture classifies as `immutable-history`, and that
  the same line in another sase-core test fixture is still a defect.
- Done when the audit command above exits 0.
- Task `sase-1cv` tracks exactly this failure. Close it in the closeout (item 8).

### 2. Symvision: make the kitty detection helper public

`just symvision` fails: `src/sase/bead/show_images.py::kitty_graphics_supported` imports
the private `_kitty_graphics_support` from `src/sase/doctor/checks_deep_terminal.py`.
The epic plan says to "reuse or expose the doctor's kitty detection". Rename it to
`kitty_graphics_support` and update the doctor call site (`check_kitty_graphics`), the
`show_images.py` import, and any tests. Done when `just symvision` passes.

### 3. Classify the purge reason option

`tests/test_bead/test_cli_at_path_values.py::test_every_bead_free_text_option_is_classified`
fails on `('attachment', 'purge', 'reason')`, added by the lifecycle phase.
`src/sase/bead/cli_attachment_lifecycle.py` records the reason verbatim in tombstones
and never expands `@file` in it. Add the triple to `_DELIBERATELY_LITERAL_FREE_TEXT`
under the existing "Audit reasons are recorded verbatim, never expanded." group.

### 4. Styled `sase bead show` crash: compute chip spans on the plain text

In `src/sase/bead/cli_show_batch.py`, the section builder (around the
`attachment_targets_for_body(issue.id, body, names)` call) passes the raw styled body,
which contains SGR escapes. `PagerSection.__post_init__` (`src/sase/pager/document.py`)
validates targets against `_body_to_text(body).plain`, so any SGR before a chip shifts
the spans. The result is either `ValueError: attached target span … exceeds body length`
or a silently wrong link. The `try/except` around the target computation does not cover
the `PagerSection` constructor. This fires on a color TTY whenever images resolve to
`never` (for example `SASE_AGENT=1`, `-i never`, or `bead.show.images: never`) and also
in kitty mode. It was reproduced with a 400-char SGR prefix.

- Search the same plain text the pager validates against. Either expose a small public
  helper from `sase.pager.document` that returns `_body_to_text(body).plain`, or use
  `rich.text.Text.from_ansi(body).plain` when `body` is a `str`, keeping the two
  consistent. Pass that plain text to `attachment_targets_for_body`.
- Regression tests: a styled (`style=True`) show-batch section for a bead with
  attachments and images `never` builds without error, and its targets cover exactly the
  `[name]` chip and descriptor-name text in `section.plain_text`. Add a direct
  `attachment_targets_for_body` plus `PagerSection` test with an SGR-prefixed body.
  Existing `tests/test_bead/test_show_images.py` helpers may be reused.

### 5. Same-text `@attachment` reuse: validate the token set, not the multiset (sase-core)

The grammar allows `@attachment:<name>` to reuse an attachment "attached earlier in the
same text". The Python authoring service (`_build_manifest` in
`src/sase/bead/attachments/authoring.py`) correctly emits one descriptor per name even
when the stored text repeats the token. Two examples of such notes:

- `"shot @./shot.png again @attachment:shot.png"`
- the same `@./shot.png` written twice

sase-core `validate_note_attachment_manifest`
(`crates/sase_core/src/note_attachment/manifest.rs`) compares the sorted token
_multiset_ against the manifest names. It therefore rejects these notes with
`tokens [shot.png, shot.png] vs manifest [shot.png]` (reported by `sase-1ck.10`).

- In the linked sase-core checkout, deduplicate token names before comparing, so every
  token must name a descriptor and every descriptor must have at least one token. Keep
  the duplicate-_manifest_-name rejection and the existing "must match one-to-one" error
  text, because existing tests and callers match on it. Update the doc comment.
- Rust tests: a repeated token with one descriptor validates, both in
  `note_attachment/tests/manifest.rs` and through the note-appended event validation in
  `bead/events/tests/attachments.rs`. The existing missing-token, orphan-descriptor, and
  +1 mismatch tests stay green. Run the targeted sase-core tests (`note_attachment`,
  `bead` events, mutation and +1 attachment tests) plus fmt and clippy per its
  `AGENTS.md`. Use its wrapped check if time allows.
- In sase, run `just install` so the local `sase_core_rs` builds from the edited
  checkout. Then add CLI regression tests in
  `tests/test_bead/test_cli_note_attachments.py`: a note with `@./shot.png` plus
  `@attachment:shot.png` in the same text, and one with the same `@./shot.png` twice,
  each writes one descriptor, stores two tokens, and renders both chips.

### 6. Explicit `attachment open` and `attachment:` resolution fetch

The epic plan says: "Explicit `path`/`open`/`-d/--download` always fetch." Only `path`
does. `show_images` landed before the shared store, and no later phase added fetching.

- `src/sase/bead/cli_attachment.py`:
  - `handle_bead_attachment` currently wraps `open` in `fetch_context(mode="never")`.
    Run `open` without that wrapper, like `path`.
  - In `_handle_bead_attachment_open`, resolve the chosen attachment under
    `fetch_context(mode="force")`. Call
    `attachment_availability(sha256, size_bytes=…, origin=…, name=…)` exactly as
    `_handle_bead_attachment_path` does.
  - When the result is not `cached`, print the same badge-bearing error as `path`.
  - With no name, choose among every roster attachment that is not purged, not only
    cached ones. Keep the existing TTY picker and non-TTY "specify a name" behavior,
    then force-fetch the chosen one.
  - The n/p set from `viewable_media_specs` can stay cached-only.
- `src/sase/bead/attachment_resolve.py::materialize_attachment_view`: its docstring
  promises "fetching if needed" but says "This phase has no shared-store fetching". Make
  it fetch the same way (force mode, full availability kwargs) and fix the docstring.
  Its callers are explicit user actions: `sase artifact path|open|read attachment:…` and
  pager link activation.
- Leave the TUI `beads_open_attachments` key cached-only. Its help text already says
  "Open cached attachments".
- Tests: reuse the two-home, one-local-bare-remote fixtures in
  `tests/test_bead/test_attachment_fetch.py`. A note written on home A can be opened on
  home B through `attachment open <id> <name>`: stub `view_artifact_files` and assert
  that it receives a digest-verified, extension-preserving path. The same holds for
  `materialize_attachment_view`. An unreachable store still fails with the badge error.
- Docs:
  - In `docs/beads.md`, replace "never fetches" for open with the explicit-fetch rule in
    the viewing section and the `attachment list/open/path/push` section. List stays
    never-fetch.
  - In `docs/cli.md`, make the `sase bead attachment open` row say it fetches when
    needed. Give that row and the `attachment path` row the missing third (docs link)
    column, `[Beads](beads.md#cli-commands)`, like the neighboring rows.

### 7. `sase validate` must not fail when the optional attachments-private store is absent

`sase validate`, and therefore `sase tool run check`'s "SASE validation" stage, fails on
any machine without the hidden attachments-private clone. The cause is that
`sase init repo --check` plans
`create ~/.sase/projects/<key>/repos/attachments-private`. The store is optional:

- Creation is a default-no consent prompt (`sase-1ck.5.1.1`).
- With no store, attachments are local by design and the echo says so.
- `tools/ci_bootstrap_sidecars` documents that `init repo --check` "merely warns" about
  hidden sidecars.

Changes:

- In `src/sase/main/_repo_init_sidecars.py::plan_sidecar_actions`, when an
  attachments-private clone is missing and its role is not recorded, append a warning
  instead of a `create` action. Example wording: "optional private attachment store
  attachments-private is not set up; attachments stay local-only — run `sase repo init`
  to create it". Do not set `requires_tty` for it.
- Leave the agents-sidecar and other roles unchanged.
- Confirm that interactive `sase repo init` (the apply path in
  `_run_configured_sidecars` and `_run_materialized_sidecars`) still reaches the
  default-no attachments-private consent prompt when the plan's only attachments-private
  entry is that warning. If apply short-circuits on an action-free plan, keep the prompt
  reachable for this role.
- Tests: the `init repo --check` planner yields a warning, not an action, for a missing
  attachments-private clone, and the check exits 0 when nothing else needs attention.
  The existing preflight, consent, and bare-clone tests (for example
  `tests/test_linked_repo_sidecar_hidden_agents.py` and the repo-init sidecar tests)
  stay green.
- Done when `.venv/bin/sase validate </dev/null` passes on a machine without that clone
  (this host has none).

### 8. Verify, then close out epic `sase-1ck` (final step, same turn)

1. `just fix`. Run the focused suites:
   - `tests/test_bead/test_attachment_*.py`
   - `tests/test_bead/test_cli_attach_verbs.py`
   - `tests/test_bead/test_cli_note_attachments.py`
   - `tests/test_bead/test_show_images.py`
   - `tests/test_bead/test_git_attachment_store.py`
   - `tests/test_bead/test_cli_at_path_values.py`
   - `tests/ace/tui/test_beads_attachment_views.py`
   - `tests/test_patch_stitch_terminology_audit.py`
   - the repo-init sidecar tests

   Then run `sase tool run check`, handed to `/sase_monitor` if long. Any remaining
   failure that reproduces identically on a clean tree and is unrelated to attachments
   goes in the close note. It does not block the close.

2. Run `sase bead epic-symbols sase-1ck`. The land audit found no entries. If any
   appear, resolve each one (wire it up, privatize it, add a non-test pragma, or delete
   it) per the Symvision epic-whitelist policy. Re-key a Justfile line only to a
   still-open bead that still needs it.
3. Close the duplicate task:
   `sase bead close sase-1cv --note "Fixed by the sase-1ck landing: the sase-core note-attachment corpus fixture is now classified as immutable history in src/sase/patch_stitch_audit.py; the terminology audit exits 0."`
4. Close the epic with a concrete verification note. The note must cover:
   - that items 1-7 are fixed, with the tests that prove each one
   - the check result
   - that the follow-up triage is already recorded on the epic (tasks `sase-1cy` through
     `sase-1d4`, and the declined proposals)

   Command: `sase bead close sase-1ck --note "<that verification>"`. Never use `--force`
   to make the close succeed. If the close is rejected for leftover `--epic-symbol`
   entries or unfinished phases, fix the cause and close again.

5. Run `just symvision` and confirm it is clean.
6. Set `status: done` (it is currently `status: wip`) in the frontmatter of the epic's
   plan file, the PLAN path that `sase bead read sase-1ck -r "<why>"` prints
   (`plan:202609/bead_note_attachments.md`).
7. `sase-1ck` has no `parent_bead`, so nothing further up needs closing.

## Acceptance

- The terminology audit and `just symvision` both exit 0.
- `test_every_bead_free_text_option_is_classified` passes.
- A styled `sase bead show` of a bead with attachments renders with images `never` and
  in kitty mode without a span error, and its chip links point at the chip text.
- Same-text `@attachment:` reuse and a repeated `@path` both write successfully, with
  one descriptor per name.
- `sase bead attachment open` and `attachment:` artifact resolution fetch from the
  shared store on a second home, and the docs describe that.
- `sase validate` passes without the optional attachments-private clone, while
  `sase repo init` still offers to create it.
- Epic `sase-1ck` and task `sase-1cv` are closed, and the epic plan file says
  `status: done`.
