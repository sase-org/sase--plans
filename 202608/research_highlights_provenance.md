---
tier: tale
title: Add research provenance to Highlights PDFs
goal:
  Hook-created Highlights PDFs retain the repository-relative Markdown source path as
  synchronized research metadata.
size: medium
proposed_by: bbugyi200.athena.vq
create_time: 2026-08-08 11:22:43
status: done
---

- **PROMPT:**
  [prompts/202608/research_highlights_provenance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/research_highlights_provenance.md)
- **AGENTS:**
  - [bbugyi200.athena.vq](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.vq.md)
- **COMMITS:**
  - [3c5674a](https://github.com/bbugyi200/dotfiles/commit/3c5674a39e4b74aceb7e0724dde591462bb1d34b)
    — chore(sase): pass research root to highlights hook

# Plan: Add research-source provenance to hook-created Highlights PDFs

## Goal

Make every PDF produced by the `research-highlights` SASE file hook carry a `research`
property in its page-one Highlights marker note. The value must be the source Markdown
file's normalized, repository-relative path within the matched `research` sidecar, for
example `202608/artifact_reference_rendering.md`.

Preserve the existing marker properties (`status`, `parent`, and `title`) and the
existing behavior of manual `bob highlights create` calls that do not opt into research
provenance.

## Current behavior and design decision

- The chezmoi-managed global configuration declares `research-highlights` in
  `home/dot_config/sase/sase.yml` as `bob highlights create`, filtered to added Markdown
  files in `research` sidecars.
- SASE appends the matched absolute file path to the configured command and runs the
  command with the matched repository root as its working directory. The batch also
  records the already-normalized relative path, but that value is not exposed to the
  command and does not need new plumbing for this feature.
- Bob's `highlights create` implementation in `src/native/highlights_ref/create.rs`
  currently composes the marker from only `status`, `parent`, and `title`.

Add an optional Bob CLI option `-R, --research-root PATH`. When supplied, Bob will
canonicalize the root and the Markdown source, require the source to be contained by
that root, and derive a UTF-8, slash-separated relative path for the marker's `research`
value. Configure the hook as `bob highlights create --research-root .`; for this hook,
`.` is guaranteed by the existing SASE execution contract to be the research sidecar
root.

This keeps provenance derivation and validation in Bob, makes the hook's opt-in
explicit, avoids brittle shell path manipulation, and avoids expanding SASE's generic
file-hook environment or command-template surface. It also leaves unrelated/manual PDF
creation unchanged when `--research-root` is absent.

## Implementation

1. In the `bob-cli` repository, extend `bob highlights create` in
   `src/native/highlights_ref/create.rs`:
   - Add the public `-R, --research-root PATH` option with clear help text and place it
     consistently with the command's alphabetized short-option help contract.
   - Carry the optional root through `CreateOptions` and the resolved relative research
     path through `CreatePlan`, so dry-run and write execution use the same validated
     plan.
   - After canonicalizing the Markdown source, canonicalize and validate the supplied
     research root as a directory, require the source to be below it, reject an empty,
     non-UTF-8, or otherwise non-normal relative result, and render components with `/`
     separators. Error messages must identify both the source and root and must occur
     before pandoc or output-directory writes.
   - Add `research` to the marker projection only when the option is present. Keep the
     existing required-key and marker round-trip validation, and include the derived
     value in dry-run/success reporting so provenance is auditable.
2. In `src/native/highlights_ref/mod.rs`, make `research` a standard synced user field,
   alongside `title`, `topics`, and the other marker/frontmatter properties. This lets
   the generated Obsidian reference note retain the property and synchronize later edits
   without classifying it as an unknown `highlights_marker_fields` opt-in.
3. Update Bob's tests:
   - Extend the create option-order/help test to cover `-R, --research-root`.
   - Add focused create planning/dry-run coverage using a nested source such as
     `202608/artifact_reference_rendering.md`; assert that the marker contains exactly
     that relative `research` value and that omitting the option omits the property.
   - Cover invalid provenance inputs (at minimum a source outside the supplied root) and
     assert failure happens without creating the vault/PDF or invoking pandoc.
   - Extend marker-to-reference-note sync coverage to assert `research` is rendered as
     ordinary frontmatter, survives a second idempotent sync, and does not require a
     `highlights_marker_fields` entry.
   - Where the optional pandoc/xelatex integration test runs, pass a research root and
     verify `bob highlights marker` reads the embedded `research` value from the
     installed PDF.
4. Update `README.md` and `docs/highlights-ref-sync.md` in `bob-cli` with the new CLI
   syntax, opt-in behavior, marker example, containment/error semantics, and the fact
   that `research` is a standard synced property. Document the hook-oriented example
   using a repository-root working directory rather than any machine-specific path.
5. In the chezmoi repository, change only the `research-highlights` command in
   `home/dot_config/sase/sase.yml` to `bob highlights create --research-root .`. Keep
   all existing sidecar, path, agent-name, operation, and timeout filters unchanged.

## Validation

- In `bob-cli`, run the focused create and Highlights sync tests while iterating, then
  run `just all` to enforce formatting, Clippy, and the complete Rust test suite.
- Inspect `bob highlights create --help` to confirm the new public short/long option is
  clear and correctly ordered.
- Exercise a dry run from a temporary repository root with a nested Markdown source and
  confirm the marker preview contains
  `research: 202608/artifact_reference_rendering.md`, while an outside-root source fails
  before writes.
- Validate the edited chezmoi YAML parses, inspect the effective hook after applying the
  normal chezmoi workflow with `sase file-hook list --json`, and confirm its command is
  exactly `bob highlights create --research-root .` with all prior filters intact.
- Run an end-to-end hook-equivalent smoke test from a temporary research repository root
  (using Bob dry-run if pandoc/xelatex are unavailable) to prove the appended absolute
  Markdown argument and `--research-root .` produce the required relative property.

## Acceptance criteria

- A hook-created PDF sourced from
  `<research-root>/202608/artifact_reference_rendering.md` has a page-one marker that
  includes `- research: 202608/artifact_reference_rendering.md` in addition to the
  existing status, parent, and title entries.
- The subsequent Highlights sync writes the same value to the generated reference note
  as a first-class frontmatter property and remains idempotent.
- A source cannot silently receive a misleading absolute path or a path outside the
  declared research root.
- `bob highlights create` without `--research-root` remains backward compatible and does
  not add a `research` property.
- No SASE source, schema, or documentation change is required; the feature relies only
  on the already-documented repository-root working-directory contract.
