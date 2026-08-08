---
tier: tale
title: Use source basenames as Highlights research IDs
goal:
  Hook-created research PDFs embed a basename-derived id while legacy research-path
  metadata remains safely synchronizable.
proposed_by: bbugyi200.athena.vq.f0
create_time: 2026-08-08 12:36:03
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.vq.f0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.vq.f0.md)
- **COMMITS:**
  - [3a615f3](https://github.com/bobs-org/bob-cli/commit/3a615f329d2c6e90a0a3d1bd152b57a5be75df35)
    — feat(highlights)\!: derive marker ids from markdown basenames

# Plan: Use source basenames as Highlights research IDs

## Goal

Make PDFs created by the `research-highlights` SASE file hook embed an `id` property
whose value is the source Markdown filename without its final `.md` extension. For
example, rendering `202608/xprompt_role_binding/xprompt_role_binding.md` must produce a
page-one Highlights marker containing `id: xprompt_role_binding`, not the current
`research: 202608/xprompt_role_binding/xprompt_role_binding.md` property.

Keep unrelated/manual `bob highlights create` calls opt-in, retain the existing
`status`, `parent`, and `title` marker properties, and preserve safe synchronization of
PDFs and reference notes that were created during the brief repository-relative
`research`-property rollout.

## Current behavior and design decision

- In `bob-cli`, `bob highlights create -R/--research-root PATH` canonicalizes the root
  and source, requires the source to be contained by the root, and embeds their
  normalized relative path as the marker's optional `research` value. The create plan,
  report output, tests, and documentation all expose that behavior.
- `research` is currently a standard Highlights user field, so sync copies it between
  PDF markers and generated Obsidian reference-note frontmatter without requiring a
  `highlights_marker_fields` opt-in.
- In the chezmoi-managed global SASE configuration, the `research-highlights` hook runs
  `bob highlights create --research-root .`; SASE appends the matched absolute Markdown
  path and executes from the matched research-sidecar root.
- Bob already uses the Markdown file stem to name the target PDF. The desired ID can
  therefore be derived inside Bob from the validated source path without a repository
  root, SASE command templating, or shell path manipulation.

Replace `-R/--research-root PATH` with the boolean public option `-i/--include-id`. When
selected, Bob will derive the marker `id` from the canonical Markdown source's UTF-8,
nonempty file stem. This keeps automatic ID generation explicit to the hook, makes the
command independent of its working directory, and removes root containment logic that no
longer contributes to the emitted value.

Treat `id` as a standard synced user field. Keep `research` in the standard-field set as
legacy compatibility so existing markers and notes continue to round-trip without
metadata loss or a surprise `highlights_marker_fields` change, but stop generating
`research` in new PDFs and describe it as legacy. Do not automatically rewrite existing
PDFs or reference notes from `research` to `id`: a repository-relative path does not
always provide enough context to prove that an automated migration chose the intended
identifier, and this request is forward-looking (“start using”).

## Implementation

1. In the `bob-cli` repository, update `src/native/highlights_ref/create.rs`:
   - Remove `-R/--research-root`, its `PathBuf` option state, repository containment
     helper, relative-path plan field, and `research` report lines.
   - Add the alphabetically placed public flag `-i/--include-id` with help text that
     states it embeds the Markdown filename without `.md` as marker `id`. Follow Bob's
     CLI rule that every public long option has a short alias.
   - Carry the boolean through `CreateOptions` and the optional derived ID through
     `CreatePlan`, so dry-run and real execution consume the same preflighted plan.
   - Derive the ID only when opted in, after validating and canonicalizing the Markdown
     source. Use the source file stem exactly (remove only the final `.md` extension),
     require a nonempty valid-UTF-8 value, and return a source-identifying error before
     pandoc invocation or output-directory writes if it cannot be represented in the
     marker.
   - Compose `id`, never `research`, into newly created markers when the flag is set;
     continue omitting both fields when it is absent. Include `id` in dry-run and
     success reporting, and retain marker parse/render round-trip validation.
2. In `src/native/highlights_ref/mod.rs`:
   - Introduce a shared `id` field constant and include it in the standard synced user
     fields so marker-to-frontmatter and opted-in frontmatter-to-marker synchronization
     treat it as ordinary metadata.
   - Retain `research` as a legacy standard synced field for existing PDFs/notes, with
     no create-path generation or automatic conversion between the two properties.
3. Update Bob's automated coverage:
   - Change the create help/order test to require `-i, --include-id` in the correct
     position and to ensure the obsolete `--research-root` option is gone.
   - Replace relative-path planning and dry-run expectations with a nested source such
     as `202608/xprompt_role_binding/xprompt_role_binding.md`; assert the plan, report,
     and rendered marker contain exactly `id: xprompt_role_binding`, contain no
     `research`, and still write nothing during dry-run.
   - Preserve explicit coverage that omitting `--include-id` leaves the default marker
     unchanged. Replace the obsolete outside-root test with preflight coverage for an
     opted-in source whose stem cannot be represented as a nonempty UTF-8 marker value
     where the platform permits constructing one; assert Bob neither invokes pandoc nor
     creates vault output.
   - Update the optional pandoc/xelatex render integration test to use `--include-id`
     and verify `bob highlights marker` reads the embedded basename ID.
   - Update marker/reference-note sync coverage to prove `id` is first-class
     frontmatter, survives a second idempotent sync, and does not need
     `highlights_marker_fields`. Retain or add focused compatibility coverage showing an
     existing `research` property still round-trips as a standard field without being
     renamed automatically.
4. Update `README.md` and `docs/highlights-ref-sync.md` in `bob-cli`:
   - Replace the old option syntax, repository-root examples, containment semantics,
     marker examples, and reporting text with `-i/--include-id` and basename-derived
     `id` behavior.
   - Document the exact derivation rule (only the final `.md` extension is removed),
     opt-in behavior for manual calls, standard sync semantics for `id`, and legacy
     round-trip support/no automatic migration for existing `research` metadata.
5. In the chezmoi repository, change only the `research-highlights` command in
   `home/dot_config/sase/sase.yml` from `bob highlights create --research-root .` to
   `bob highlights create --include-id`. Keep the hook description, sidecar filter, path
   globs, agent-name globs, operation filter, and timeout unchanged.

## Validation and rollout

- In `bob-cli`, run the focused create planning/CLI and Highlights sync tests while
  iterating, then run `just all` for formatting, Clippy, and the full Rust test suite.
- Inspect `bob highlights create --help` and confirm `-i, --include-id` is documented
  and ordered correctly, while `--research-root` is no longer accepted or advertised.
- Exercise a hook-equivalent dry run with a nested source path and confirm the report
  and marker preview contain `id: xprompt_role_binding`, contain no `research` line, and
  report `writes: none`. Also exercise create without the flag to confirm the default
  marker remains unchanged.
- Where pandoc and xelatex are available, create a real PDF and inspect it with
  `bob highlights marker`; otherwise rely on the existing optional integration test and
  the deterministic dry-run marker projection.
- Validate the chezmoi YAML, install or otherwise make the newly validated Bob binary
  available before activating the new hook command, and then apply the normal chezmoi
  workflow. Inspect `sase file-hook list --json` to confirm the effective command is
  exactly `bob highlights create --include-id` and every pre-existing filter is intact.
  After any eventual chezmoi commit, follow that repository's required
  `chezmoi update -a --force` workflow.
- Run final diffs/status checks in both repositories and verify no SASE source, schema,
  or generic file-hook changes were introduced.

## Acceptance criteria

- A hook-created PDF sourced from `202608/xprompt_role_binding/xprompt_role_binding.md`
  contains `id: xprompt_role_binding` alongside `status`, `parent`, and `title`, and
  does not contain `research`.
- The generated reference note contains the same `id` as ordinary frontmatter;
  bidirectional/repeat sync remains idempotent and does not add `id` to
  `highlights_marker_fields`.
- `bob highlights create` without `--include-id` preserves its pre-provenance marker
  shape, while the research hook opts in without relying on a repository-root working
  directory.
- Existing PDFs and notes carrying `research` remain readable and synchronizable as
  legacy standard metadata, with no implicit rename or bulk mutation.
- ID derivation and validation complete before pandoc or output writes, and invalid
  filename stems produce a clear diagnostic naming the source.
