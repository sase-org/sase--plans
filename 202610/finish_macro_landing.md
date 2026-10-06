---
tier: tale
title: Finish macro rename repairs and close the nested rename epics
goal:
  Canonical macro contracts and readable docs pass verification, then sase-1eq.12 and
  its ready parent sase-1eq close normally.
size: medium
proposed_by: bbugyi200.athena.sase-1eq.12.land
bead: sase-1eq.12
create_time: 2026-10-06 08:39:00
status: wip
---

- **PARENT:**
  [202610/land_xprompts_to_macros.md](https://github.com/sase-org/sase--plans/blob/main/202610/land_xprompts_to_macros.md)
- **BEAD:**
  [sase-1eq.12](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1eq/sase-1eq.12.md)

# Finish the remaining macro rename repairs and close sase-1eq.12 and sase-1eq

## Goal and scope

Finish the bounded gaps found by the land agent for epic `sase-1eq.12`, then perform its
closeout and the readiness review of its parent plan bead `sase-1eq` in this same coding
turn. This is one direct implementation tale; it has no separate land agent. Do not
defer closeout until this turn's commit, push, release, or CI result exists.

Read these audited records before implementation:

- `sase bead read sase-1eq.12 --no-links -r "Need remaining landing evidence and triage"`
- `sase bead read sase-1eq --no-links -r "Need previous parent landing review"`
- `sase artifact read plan:202610/land_xprompts_to_macros.md "Need child epic exit criteria"`
- `sase artifact read plan:202610/xprompts_to_macros.md "Need parent compatibility and exit criteria"`

The current landing review used sase `312f17dc3f` and core `b19690e3`. All three phases
of `sase-1eq.12` are closed. Their commits are core `d65f7246` (core-green), core
`b19690e3` plus sase `312f17dc3f` (key-flip), and sase `b6114d4f95` (infographic). The
source pin already names `b19690e3`; the host moved it.

Already verified: local-helper canonical/retired-key precedence was repaired;
canonical-output assertions replaced stale expected output while stored legacy proc
inputs remain; both catalog environment-mutating tests share `ENV_SERIAL`; content
layout emits schema 7 with canonical macro keys and compatible legacy paths; Python
launch/snippet/proc mirrors use the new keys. Do not redo these changes.

## Follow-up triage already completed

The land agent read every child note and recorded every outcome in a
`LANDING TRIAGE AND VERIFICATION` note on `sase-1eq.12`. Preserve these outcomes in the
final close note:

- `sase-1eq.12.2` note #1, the Rich-truncated long-path snippet-delete assertion:
  duplicate `sase-18v`, with independent source/impact +1 evidence recorded. The
  lander's local rerun hit the unbuilt editable Rust extension before exercising the
  renderer. Do not claim it independently reproduced the renderer failure.
- `.2` note #2, the infographic prompt terminology failure: retained as epic-caused work
  below, not filed as a task.
- `.2` note #3: the source-pin part is resolved. The published-floor probe still says
  `blocked_unpublished` (floor 0.35.0, 95 missing capabilities;
  `load_macro_input_type_registry` at `af5df61` is unreleased). This is the existing
  release-floor gap, corroborated with +1 on `sase-10d`; an integration note was also
  added to `sase-1c1`, whose release phase `.14` still describes the too-old 0.36.0
  floor. The check recipe invokes this probe with `--advisory`, so it is not a failing
  local/dev product gate and must not order this landing after a release.
- `sase-1eq.12.3` note #1, generated input-schema drift: declined as resolved by the
  intervening named-input-type work. The workflow and config generated blocks match the
  installed builtin catalog. Plugin-qualified catalog rows must be excluded when
  comparing the builtin generation contract.

No new task beads were created. Unrelated follow-ups from the original parent were
already triaged in `sase-1eq` note #6; retain that accounting rather than filing them
again.

## 1. Finish canonical core terminology and the snippet schema contract

Open the core with `sase repo open sase-core -r "Finish macro landing repairs"` and read
its `AGENTS.md`. All `crates/...` paths below are relative to that printed checkout; all
`src/...`, `tests/...`, and `docs/...` paths are relative to sase.

Audit `git grep -n -i xprompt -- crates` and fix canonical identifiers, comments, and
messages that the earlier sweep missed. Concrete confirmed examples:

- `crates/sase_core/src/macro_catalog/types.rs`: `Read` and `LayoutCollision` errors
  still say "xprompt catalog"; these are ordinary canonical errors, not retirement
  diagnostics. Keep messages that name a genuinely retired authored key.
- `macro_catalog/loader.rs`: canonical local variables `xprompt` / `xprompts`, and
  canonical memory/catalog prose; Rust's `macro` is reserved, so use `macro_entry`,
  `macro_def`, or `macros` as appropriate.
- `editor/hover.rs`: `xprompt_markdown` should be `macro_markdown`.
- `editor/completion/trigger_context.rs`: rename
  `detect_xprompt_arg_completion_at_position` and its callers.
- `editor/argument_syntax_edit.rs`, `editor/mod.rs`, and `lib.rs`: rename the shared
  `normalize_xprompt_spacer_transition` function and the root re-export
  `editor_normalize_xprompt_spacer_transition`; update the LSP caller in
  `crates/sase_macro_lsp/src/server/spacer.rs` and all tests.
- `host_bridge.rs` and `lib.rs`: retire/rename `EditorXpromptInputWire`, an alias of
  `MobileMacroInputWire`, after checking actual consumers.
- `crates/sase_macro_lsp/src/server/completion_items.rs`: remove the unused temporary
  wrappers `xprompt_snippet_items` and `xprompt_completion_skeleton`, and update tests
  to call the canonical functions. A `// legacy xprompt spelling` comment is not
  sufficient justification for keeping an in-process alias with no durable reader.
- The LSP's `server/spacer.rs` emits "Accept xprompt completion", and `server/mod.rs`
  logs "sase xprompt LSP initialized". Emit macro wording. Update canonical prose in
  `project_tags.rs`, `server/jinja.rs`, and host-bridge docs.
- Review gateway contract prose, such as the description of macro memories. Preserve the
  deprecated `/api/v1/xprompts/catalog` route required by mobile API v1.

Classify every surviving retired literal as a durable legacy reader, flag-gated sunset
input, a realistic legacy-input fixture, or accepted history/deferred plugin packaging.
Preserve stored data readers and their legacy-input fixtures. Give Rust legacy literals
their explicit marker. Do not label ordinary new output, canonical locals, or dead
aliases as legacy just to skip a rename.

The key-flip changed `EditorSnippetEntryWire.xprompt_name` to `macro_name` and emitted
kind/source to `macro`, but `EditorSnippetCatalogRequestWire` /
`EditorSnippetCatalogResponseWire` and their producers still use schema 1. Complete the
original plans' requirement to version every changed emitted wire:

- Add an owning, operation-specific snippet-catalog schema version of 2 in the core,
  expose its getter through PyO3, and use it in the Rust catalog producer and LSP
  request construction. Avoid bumping unrelated mobile/fleet response families.
- Update the thin Python mirror and `src/sase/integrations/_editor_helper_snippets.py`,
  which currently takes the generic version-1 `GATEWAY_WIRE_SCHEMA_VERSION`. Add the
  required binding/schema probe to `tools/validate_sase_core_rs` and its contract tests.
  Use the common core constant/getter rather than scattering new literals through Python
  callers.
- Update wire fixtures and the meaningful Rust/Python helper-catalog tests together;
  assert schema 2, canonical `macro_name`, and absence of the retired output key. Check
  request-version handling and preserve any demonstrated persisted old input. Regenerate
  gateway snapshots only where the actual described wire changes.
- Inspect editor consumers with
  `sase repo open sase-nvim -r "Check snippet schema-2 consumer readiness"`; read its
  instructions. Ensure it accepts the newly emitted snippet response version and
  canonical keys, updating a genuinely incompatible consumer if found. Do not change
  unrelated contracts or drop its published-floor compatibility readers.
- In `src/sase/completion/candidates/catalog_snippets.py`, the live Rust result still
  has a raw `xprompt_name` fallback described as durable without a stored fixture. Check
  that claim. Drop request/result-only aliases; if persistence is demonstrated, use the
  centralized legacy-name module and a realistic input test. Apply the same
  centralization rule to remaining raw legacy-kind branches in snippet adapters.

## 2. Complete the promised shared stdio wait bound

`crates/sase_macro_lsp/tests/jsonrpc_stdio.rs` now bounds only the loop in
`stdio_jsonrpc_frontmatter_diagnostics`. Its shared `read_message` near line 1756 still
awaits header/body bytes indefinitely, and response/shutdown loops elsewhere call it
unbounded.

Put a generous, deterministic timeout around the shared message read, with useful
partial-header/body and received-message context on failure. Retain bounded response
selection and the correct macro diagnostic codes. Test the stalled/no-message path and
normal JSON-RPC path without lowering time allowances to a load-sensitive value. Do not
weaken diagnostics assertions or remove realistic legacy frontmatter fixtures.

## 3. Repair the infographic raster and its terminology-guarded record

The current image hash matches the recorded
`7529a15d0c2a77f0144d3804cef156b315949de0bf0ac59872ad5ccdd63f314e`, but the relabel
overlay left double-drawn/clipped discovery text in rows 1, 2, 3, 9, 10, and 11. Some
labels start left of their discovery card. The pre-relabel raster at
`b6114d4f95^:docs/images/macro-resolution-infographic.png` has clean glyph placement.

Follow the approved child's explicit deterministic ImageMagick rendering approach:

- Export that pre-relabel raster to a scratch file as a clean base, check for any
  subsequently landed changes before using it, and replace all 13 retired labels with
  exact canonical text from current `docs/macros.md`.
- Cover the entire old ink area with opaque local card background, then render each full
  label once. Fit and position the path labels inside their own cards; do not use a
  partial per-column fill that leaves old glyph fragments. A deterministic SVG overlay
  rendered by ImageMagick is suitable. Preserve the 1672x941 sRGB canvas, arrows,
  borders, numbering, and all other image content; strip metadata.
- Inspect the whole PNG at full resolution and every repaired discovery card. OCR alone
  is insufficient: old and new glyphs can overlap while OCR finds no retired word.
  Require clean, single-drawn, legible text inside each card.
- Rewrite the relabel section of `docs/images/macro-resolution-infographic.prompt.md` to
  describe only the final canonical labels. It currently has 11 unallowlisted
  retired-label lines, which independently fail
  `test_macro_docs_and_memory_avoid_xprompt_terms`. Preserve the truthful
  method/provenance, add reproducible commands/label positions, update its final SHA,
  and update `macro-resolution-infographic.critique.md` to the actual final result. Do
  not broaden the terminology allowlist to hide these residuals.

## 4. Recheck post-start integration and verify

Review drift since the landing review and the earlier phase commits in both repos. The
lander reviewed primary master/origin-master commits from `b6114d4f95^` onward and core
commits from `d65f7246` onward. In particular, named enum/model/effort and plugin-shared
input types landed concurrently (sase `8fc4b4ccd6`, `1a2dc5e4dd`, `55c46c3633`,
`abcfedc31d`, `81eaae59ea`; core `16095fcf`, `af5df614`). The new
`SASE_MACRO_PLUGIN_INPUT_TYPES_JSON` export survived the key-flip. Preserve these
runtime/LSP/TUI contracts and test their compatibility with the repaired wires.

Before verification, read `lint_and_test.md` and `symvision.md` through
`sase memory read`. This fresh workspace's editable core extension was unbuilt;
`sase core health` through the global install was green, but local pytest and schema
tools could not import its extension. Build the edited local core with `just install` in
sase before judging those checks; do not substitute a Python fallback.

Run `sase tool run check` (the wrapped `just check`) in every code repo changed,
including core and sase. Use a sufficient duration budget; core's check is several
minutes. Use the monitor skill if a gate must hand off, with a concrete continuation
that preserves this tale's closeout. Do not end a turn while a command is running. Run
`just fix` before the final sase gate. Targeted diagnostic, legacy-input,
snippet-helper/catalog, LSP, and named-input-type checks should cover the changed
boundaries; they do not replace the repository checks. Run `sase core health` after the
local build. Never run `just check-full` for this tale.

Treat new/unknown failures as yours. Proven unrelated Rich/known Symvision issues retain
their recorded owners; do not weaken tests. The advisory published-floor probe stays
with the existing release work. Run both terminology sweeps again and ensure all
surviving exceptions have a concrete compatibility or historical reason.

## 5. Finish this epic's closeout and the parent readiness review in this turn

After implementation and verification, before any final declaration:

1. Reread `sase-1eq.12` and each child `.1`, `.2`, `.3`, including every note, through
   audited `sase bead read ... --no-links -r "Need final landing readiness"`. Confirm
   the scopes, descendant statuses, linked plan's exit criteria, and all newly added
   notes are addressed. Fold the follow-up outcomes above into the verification note.
2. Run `sase bead epic-symbols sase-1eq.12`. There were no entries during review, but
   recheck. Resolve every listed symbol by wiring it up, privatizing it, adding a
   justified non-test pragma, or deleting it under the Symvision policy. Re-key a
   Justfile exemption only if a concretely named, still-open later bead still needs it.
   Retire phase-keyed entries too.
3. Close normally with
   `sase bead close sase-1eq.12 --note "<verified phase scopes, code/commit and post-start integration review, repairs and check results, image inspection, and every follow-up outcome>"`.
   If rejected for symbols, clean them and retry. If named phases are incomplete, finish
   or reopen them deliberately. Never force a successful nested landing. `--force` is
   permitted only for a real canceled/superseded outcome with its reason, not to make a
   command succeed.
4. Run `just symvision` to confirm the whitelist is clean. Use the supported wrapped
   form if the recipe is guarded; no `check-full` is authorized. Open the plans repo
   with `sase repo open plans -r "Record completed rename epic plans"` before editing
   it, use only the printed path, and set `status: done` in the frontmatter of
   `202610/land_xprompts_to_macros.md` (artifact
   `plan:202610/land_xprompts_to_macros.md`). Read sidecar artifacts through
   `sase artifact read`, not direct file reads.
5. Inspect `sase bead read sase-1eq.12 --no-links -r "Need the parent link"`. The
   concrete parent is plan bead `sase-1eq`, not a phase. Review its previous landing
   note #6 and later notes, every descendant and its notes (recursively), and the
   audited linked plan `plan:202610/xprompts_to_macros.md`. Rerun descendant and
   linked-plan readiness checks, inspect post-child drift and needed integration, and
   verify this tale has resolved parent notes #5 and #7 as well as the child gaps. Do
   not rely only on all immediate phases showing closed. The previous parent accounting
   already distinguishes completed work from triaged follow-ups.
6. If the parent is fully ready, run `sase bead epic-symbols sase-1eq`, retire every
   remaining entry with the same policy, and close normally with
   `sase bead close sase-1eq --note "<what was rechecked in the descendants, linked plan, post-child drift, checks and legacy policy; cite prior and new triage>"`.
   Confirm with `just symvision`, set `status: done` in `202610/xprompts_to_macros.md`
   in the opened plans checkout, and inspect its own parent link. There was no ancestor
   above `sase-1eq` during this review; if one now exists, repeat for directly parented
   plan ancestors while fully ready. For a phase parent, close only that phase and leave
   its containing epic to its lander.
7. Stop at the first incomplete or ambiguous parent and append a concrete blocker note
   on that parent; report it rather than force-closing it. Any further epic-caused work
   remains its scope and needs the normal remaining-work process.

Only then use `/sase_final` for the host-owned commits. Declare every edited repo,
including the plans sidecar and any consumer repo. The core declaration must use a
breaking Conventional Commit with a `BREAKING CHANGE:` footer naming the renamed public
surfaces and snippet schema change. Declare sase and core together so the host commits
core first and moves `sase-core-revision.txt`; do not hand-edit the pin, versions, or
changelogs. Bead closure and plan completion happen before those future commits, never
after a wait for this work's own SHA or release.
