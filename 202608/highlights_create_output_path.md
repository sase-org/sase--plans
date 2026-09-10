---
tier: tale
title: Add exact PDF output paths to Highlights creation
goal:
  Callers can select the complete generated PDF path without weakening the default
  intake workflow or its safety guarantees.
size: medium
proposed_by: bbugyi200.athena.0fv
create_time: 2026-09-09 20:00:08
status: wip
---

# Add an exact PDF output path to `bob highlights create`

## Goal

Add a public `-o, --output <PDF>` option to `bob highlights create` so callers can
choose the complete path and filename of the generated PDF. Preserve the current
`<xlib-dir>/<ref-type>/<markdown-stem>.pdf` behavior, intake workflow, and safety checks
when the option is omitted.

## Command contract

- Accept the output value as an OS-native path, expand a leading `~` in the same way as
  other Bob path options, and interpret a relative value relative to the caller's
  current working directory. Do not replace its filename or extension: require a
  nonempty filename with a case-insensitive `.pdf` extension and report a preflight
  error for any other value.
- Keep `-o, --output` in alphabetical help order, with clear help and after-help text
  explaining that it selects the complete PDF path. Treat an explicitly supplied
  `--ref-type` as conflicting with `--output`, because `--ref-type` only participates in
  default target derivation and silently ignoring it would be misleading. Continue
  accepting the configured Bob, library, reference, and intake directories because they
  determine whether an exact path participates in the managed scan workflow.
- When `--output` is absent, derive the target exactly as today and retain all current
  behavior and output.
- When the exact target is inside the configured intake directory, preserve the target's
  full intake-relative path when deriving the mirrored library destination. Retain the
  existing refusal for an occupied library PDF or its Markdown/TextBundle sidecar, even
  with `--force`, because the next scan could not ingest the new PDF safely.
- When the exact target is already inside the configured library, treat the selected PDF
  itself as the library destination: an existing target requires `--force`, but there is
  no second mirrored-destination collision check. When the target is outside both
  managed directories, do not invent or check an unrelated library destination.
- Apply the existing same-stem Markdown sidecar guard and `--force` overwrite contract
  to the exact target. Keep directory creation, adjacent temporary rendering, marker
  embedding, cleanup, and atomic PDF installation unchanged, so `--dry-run` remains
  strictly write-free and failures never install a partial target.
- Make dry-run and success output describe the resolved exact PDF and its workflow
  accurately: show a mirrored library destination only for intake targets; direct
  library targets should still recommend `bob highlights scan`; out-of-library targets
  should state that recursive scan will not discover them and recommend a targeted
  `bob highlights sync <PDF>` instead.

## Implementation

1. In `src/native/highlights_ref/create.rs`, define the sorted Clap option, parse it
   into `CreateOptions`, and add small path-resolution/validation helpers. Refactor
   `CreatePlan` to represent whether the selected target is intake-managed, directly
   library-managed, or outside the managed trees, including an optional mirrored library
   destination.
2. Update `plan_create` to choose either the exact target or the unchanged derived
   default, classify it against normalized configured intake/library paths with
   component-aware path comparisons, and run only the collision checks that apply to
   that workflow. Update plan/success reporting to consume this classification without
   changing PDF rendering or marker contents.
3. Extend unit tests beside `create.rs` for unchanged default derivation, exact nested
   paths and filenames, relative/tilde resolution as practical, `.pdf` validation,
   managed-path classification, mirrored-library collisions, direct-library force
   behavior, out-of-library behavior, and the existing sidecar guard.
4. Extend `tests/cli.rs` so help proves `-o, --output` is documented in alphabetical
   order and remains covered by the repository-wide short-alias invariant. Add dry-run
   coverage proving the exact path is reported with no writes or pandoc invocation,
   conflict/error coverage for invalid option combinations and non-PDF values, and
   render coverage (using the existing prerequisite-gated test) proving the installed
   PDF is at the requested path and still contains its outline and marker.
5. Update the Highlights command synopsis and overview in `README.md` and the full
   contract in `docs/highlights-ref-sync.md`. Document default versus exact-path
   behavior, relative and tilde resolution, overwrite/sidecar safeguards, the
   `--output`/`--ref-type` conflict, and the different next step for intake, library,
   and out-of-library targets.

## Validation

Run focused Highlights create unit and CLI tests while iterating, then run the
repository's complete checks:

```bash
cargo test native::highlights_ref::create::tests
cargo test --test cli highlights_create
cargo fmt --check
cargo clippy --all-targets --all-features
cargo test
```

Acceptance requires unchanged behavior without `--output`, exact placement with both
`-o` and `--output`, no dry-run writes, retained collision and atomic write guarantees,
accurate workflow guidance, excellent sorted help with a short alias, updated public
documentation, and a clean full check suite.
