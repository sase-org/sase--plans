---
create_time: 2026-07-10 15:47:03
status: done
prompt: 202607/prompts/changespec_project_name_query.md
tier: tale
---
# Make ChangeSpec project queries respect `PROJECT_NAME`

## Problem and root cause

`project:<project>` currently filters on the ProjectSpec storage directory derived from each ChangeSpec's `file_path`.
That is why `project:gh_sase-org__sase` matches ChangeSpecs stored under `~/.sase/projects/gh_sase-org__sase/`, even
though the active ProjectSpec declares `PROJECT_NAME: sase` and ACE groups those ChangeSpecs under `sase`.

The inconsistency spans the full query pipeline:

- Project lifecycle discovery already parses `PROJECT_NAME` into `ProjectRecordWire.display_name`, but ChangeSpec file
  discovery discards that metadata and returns only active/archive paths.
- The Python `ChangeSpec.project_name` property and reference `project:` matcher derive identity from the parent
  directory.
- The ChangeSpec wire shape carries `project_basename` and `file_path`, but no configured project display name.
- The Rust query corpus therefore derives its cached project name from `file_path` again. The ACE PRs tab uses this Rust
  batch evaluator, while its grouping labels independently call the display-name resolver, producing the screenshot's
  contradictory label and filter behavior.
- Archive ProjectSpecs generally do not repeat project-level metadata, so parsing `PROJECT_NAME` only from an individual
  file would still leave archived ChangeSpecs with the wrong query identity.

The intended contract is:

- A valid configured `PROJECT_NAME` is the exact, case-insensitive identity matched by `project:` and its `+` shorthand.
- When `PROJECT_NAME` is absent or invalid, matching falls back to the canonical directory key for backward
  compatibility.
- When a display name is configured, the underlying directory key is not a second query alias. `PROJECT_ALIASES` also
  remain outside this filter unless separately specified in a future change.
- The same identity applies to active and archived ChangeSpecs, while storage paths, workspace lookup, VCS operations,
  and `project_basename` continue using canonical storage identity.

## Implementation plan

1. **Extend the shared ChangeSpec/query contract in `sase-core`.**
   - Add an optional configured project display-name field to `ChangeSpecWire`, bump the ChangeSpec wire schema, and
     retain explicit backward deserialization behavior for older records that lack the field.
   - Teach the Rust ProjectSpec parser to read a valid `PROJECT_NAME` once from the metadata header and stamp it onto
     every parsed ChangeSpec wire record. Keep the field absent when metadata is missing or invalid.
   - Change both the one-shot matcher and persistent `QueryCorpus` construction to cache and compare the effective
     project name (`PROJECT_NAME` when present, otherwise the parent-directory key). Do not reparse metadata during
     evaluation.
   - Update PyO3 wire/binding fixtures and Rust parser, wire, one-shot query, and persistent-corpus parity tests. Cover
     configured-name-only matching, directory-key fallback, case insensitivity, and `+project` shorthand parity.

2. **Carry project metadata through SASE's lifecycle-aware ChangeSpec loading.**
   - Add an optional display-name/effective-query-name value to the Python ChangeSpec model without changing the
     existing canonical directory-key and file-basename APIs used by storage and VCS code.
   - Preserve the `ProjectRecordWire` association during ChangeSpec discovery so the active ProjectSpec's effective name
     is attached to ChangeSpecs loaded from both its active and archive files.
   - Include the effective project name in snapshot-cache validity (or equivalently reapply it deterministically on
     every discovery pass) so changing `PROJECT_NAME` invalidates archived ChangeSpec query metadata even when the
     archive file itself is unchanged.
   - Update Python-to-wire and wire-from-dict conversion for the new field and make the Python reference matcher use the
     same effective-name fallback as Rust. Keep corpus construction in the existing background load path so filtering
     remains in-memory and performs no filesystem work per keystroke.

3. **Add end-to-end regression coverage at the SASE boundary.**
   - Build temporary lifecycle-aware projects whose directory key differs from `PROJECT_NAME`, with ChangeSpecs in both
     active and archive files. Assert that `project:<PROJECT_NAME>` and `+<PROJECT_NAME>` match all applicable rows,
     while `project:<directory-key>` matches none.
   - Cover projects without `PROJECT_NAME` to pin the legacy directory-key fallback, and cover a display-name change to
     prove the snapshot/query corpus refreshes rather than serving stale archive metadata.
   - Extend wire schema/conversion tests and Python-vs-Rust query parity tests so the reference and production batch
     paths cannot diverge again.
   - Exercise the CLI/ACE filtering integration rather than only unit-testing the matcher, since the reported defect is
     specifically on the persistent Rust corpus route.

4. **Document and validate the user-facing contract.**
   - Update the query-language and ProjectSpec documentation to say that the `project:` property filter uses
     `PROJECT_NAME` when configured and otherwise the directory key, while clarifying that aliases and storage identity
     are unchanged.
   - Run the focused Rust and Python query/parser/cache tests first, then the full `sase-core` checks. In the SASE repo,
     run `just install` followed by the required `just check`.
   - Reproduce the original case with `sase changespec search`: `project:sase` must return the ChangeSpecs from the
     screenshot (and their archived peers when not hidden by status), while `project:gh_sase-org__sase` must return no
     ChangeSpecs.

## Design and risk notes

- The display name must be resolved once during lifecycle discovery/corpus compilation. Calling the global display-name
  resolver from the row matcher would introduce filesystem/cache work into ACE's hot filtering path and could also make
  archive behavior depend on an unrelated cache signature.
- Active and archive files must share project-level metadata from the same lifecycle record; archive files intentionally
  contain ChangeSpec blocks without duplicating the active file's metadata header.
- The wire schema and installed Rust binding need to move together. Compatibility tests should make a stale binding fail
  clearly rather than silently accepting the new Python record while continuing to filter on the directory key.
