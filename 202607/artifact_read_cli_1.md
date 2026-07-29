---
tier: tale
title: Implement the sase artifact read CLI
goal:
  Humans and agents can discover, inspect, resolve, print, open, repair, and verify indexed artifacts through one
  compatible CLI group.
bead: sase-ax.3
create_time: 2026-07-29 17:41:40
status: done
---

- **PROMPT:** [202607/prompts/artifact_read_cli_1.md](prompts/artifact_read_cli_1.md)
- **PARENT:**
  [202607/artifact_read_cli.md](https://github.com/sase-org/sase--plans/blob/main/202607/artifact_read_cli.md)
- **BEAD:** [sase-ax.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ax/sase-ax.3.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ax.3--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ax.3.md#member-code)
  - [bbugyi200.athena.sase-ax.3--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ax.3.md#member-plan)
- **COMMITS:**
  - [30e2ed3](https://github.com/sase-org/sase/commit/30e2ed37ed28cc2dab894e69419d206fec79ce05) — feat(cli): add
    artifact read commands

# Implement the `sase artifact` read CLI for bead `sase-ax.3`

## Goal

Replace the create-only canonical `sase artifact-file` surface with a complete `sase artifact` command group while
retaining `artifact-file` as a compatibility alias. The command must let humans and agents discover, inspect, resolve,
print, and open indexed artifacts, and must expose the artifact-index doctor delivered by the preceding
record-enrichment phase.

The implementation consumes two already-landed prerequisites:

- `sase-core-rs` 0.12.14 provides `artifact_files_query` and `artifact_file_query_wire_schema_version`.
- The Python artifact-file domain provides enrichment fields, tolerant index reading/writing, and
  inspect/backfill/verify reports.

This phase does not change the Rust core, canonical memory, generated provider instructions, the artifact skill
template, or documentation; those skill/docs updates belong to the dependent epic phase.

## Implementation

### 1. Adopt the released Rust query contract

- Raise the `sase-core-rs` lower bound from 0.12.12 to 0.12.14 in `pyproject.toml` and refresh `uv.lock` so a normal
  install always has the artifact query bindings.
- Add a focused Python query facade under `sase.core` that:
  - checks the query wire-schema handshake before invoking the query;
  - accepts the CLI filter set (`kinds`, canonical project key, agent, since, explicit-only, substring query, and
    limit);
  - passes the resolved/default artifact index path to Rust;
  - defensively converts each complete wire row into the existing `ArtifactFile` model while preserving the query order;
  - fails clearly when the installed binding is stale or returns an incompatible schema.
- Test the binding arguments, schema mismatch, wire conversion, and parity between the Rust query facade and the
  existing Python writer-side reader for a representative v1/v2 fixture.

### 2. Register the canonical parser and compatibility alias

- Replace `parser_artifact_file.py` with `parser_artifact.py` and register a canonical `artifact` group with
  `aliases=["artifact-file"]`.
- Document in group help and examples that bare `sase artifact` defaults to `sase artifact list`; rely on the central
  exact-`list` defaulting and notice mechanism rather than implementing another default.
- Register `create`, `doctor`, `list`, `open`, `path`, and `show` alphabetically. Register every public long option
  alphabetically and give it a short alias:
  - create: kind, label, path;
  - doctor: fix, verify;
  - list: agent, explicit, json, repeatable kind, limit (default 50), project, query, and syntactically validated since;
  - show: json.
- Reuse `ARTIFACT_FILE_KINDS`, `nonnegative_int`, and `plan_date_arg` instead of duplicating domain values or date
  syntax.
- Update root parser registration and entry dispatch so both the canonical token and the alias reach one thin artifact
  handler.
- Add parser/help/defaulting tests for the canonical command, alias, alphabetical subcommands/options, bare-group
  delegation notice, all filter defaults, repeatable kinds, and malformed dates/limits.

### 3. Build thin command modules with stable output contracts

- Replace `artifact_file_handler.py` with a thin `artifact_handler.py` dispatcher and put command behavior/presentation
  in a new `sase.artifact_cli` package.
- Preserve `create` semantics and agent/environment gating, but use the canonical command spelling in diagnostics and
  print a third durable output line, `ref: file:<id>`, alongside the id and stored path.
- Implement `list` through the Rust query facade only:
  - resolve a supplied project display name, alias, or key to the canonical stored key before querying;
  - render known project keys as configured display names in human output, falling back to the key only when no name is
    known;
  - emit a cyan Rich panel/table modeled on `sase chat list`, with KIND, REF, LABEL, PROJECT, AGENT, SIZE, and
    minute-precision CREATED columns;
  - emit a stable JSON array containing every `ArtifactFile` field plus the rendered `file:<id>` reference.
- Implement `doctor` as presentation over the existing inspect, backfill, and verify library. Missing source paths
  remain informational; missing enrichment/stored objects, duplicates, malformed/unsupported rows, missing verification
  digests, and digest mismatches make the command unhealthy. `--fix` backfills first and reports changed ids; `--verify`
  re-hashes live stored files. Exit 0 only for the resulting healthy report and 1 for health problems.
- Keep handler-level failures on stderr and make usage/argument failures exit 2 where appropriate.

### 4. Share reference resolution across `show`, `path`, and `open`

- Export the existing launch-context assembly in `sase.artifact_refs` as a supported public helper and keep prompt
  preprocessing on that same helper, so CLI resolution sees the same document roles, chats, artifact index,
  repositories, projects, aliases, and workspace context as launch-time resolution.
- Add shared artifact-CLI reference helpers that:
  - accept canonical references of every registered kind;
  - normalize bare `default:<hash>` and `explicit:<hash>` ids to `file:` refs;
  - parse, canonically render, and resolve once through `sase.artifact_refs`;
  - find file-record metadata via the Rust query facade;
  - serialize parsed fragments and exact/drifted/ambiguous/missing resolution details, including candidate paths.
- Implement `show`:
  - for `file:` refs, render a Rich metadata panel with every record field, the durable ref, stored-path liveness, and
    live/missing source-path status;
  - for all other kinds, render kind, canonical ref, fragment, resolution status, locator/path, and ambiguity
    candidates;
  - make `--json` emit one stable common envelope containing the canonical reference, parsed kind/fragment, optional
    full file record, and resolution.
- Implement `path` with a script-safe contract: exactly one absolute path and no decoration on stdout for exact/drifted
  filesystem-backed refs; exit 1 with status/candidates on missing, ambiguous, unknown, or malformed refs; exit 2 with a
  `show` hint for `commit:` and `bug:` refs, which have no filesystem identity.
- Implement `open` on the same normalized/resolved result:
  - filesystem refs use the existing safe text, bounded kitten image, and bounded mpv command builders where applicable;
  - PDF and unknown binary MIME types use `xdg-open`;
  - `bug:` resolves its tracker URL with `issue_url_for_number` and opens it in the browser;
  - `commit:` exits 2 with a `show` hint;
  - unavailable viewers and failed subprocesses produce clear stderr diagnostics naming the resolved path;
  - every filesystem argument is absolute and follows a `--` option boundary, including filenames with spaces or leading
    dashes.

### 5. Lock behavior with parser, handler, and edge-case tests

- Replace and expand the former artifact-file handler tests with focused tests for each subcommand and shared
  resolver/query helpers.
- Cover create gating and `ref:` output; list filters, Rust query use, Rich display-name rendering, empty output,
  humanized sizes, and exact JSON keys; doctor healthy/unhealthy/fix/verify exits; show file/non-file JSON and pretty
  output; bare-id sugar and fragment echo; path exact/drifted, ambiguous, missing, malformed, and non-filesystem exits.
- Mock subprocess/browser boundaries for open tests and assert text fallback without `bat`, kitten/mpv/xdg-open
  selection, missing-tool diagnostics, nonzero child exits, bug URLs, commit hints, and safe construction for paths
  containing spaces or beginning with a dash.
- Preserve the existing artifact create end-to-end coverage while updating its canonical imports/namespace and output
  assertion.

## Validation

1. Run `just install` before repo checks so the workspace receives the new dependency floor and current editable
   package.
2. Run focused parser, query-facade, artifact command, artifact-reference, and existing artifact-file
   persistence/enrichment/doctor tests while iterating.
3. Run `just check` and fix every formatting, lint, type, Symvision, and test failure.
4. Re-run the focused artifact CLI tests after the full check to confirm no formatter or cross-suite adjustment
   regressed the strict output and exit contracts.
5. Smoke the installed CLI help and harmless read paths against a temporary index: canonical and alias help, bare-list
   delegation, JSON list/show, and path resolution.

## Acceptance criteria

- `sase artifact` and the `sase artifact-file` alias expose all six subcommands, and bare invocation delegates to `list`
  with the central notice.
- `list` is Rust-query-backed, filters correctly, renders project display names, and has a stable full-record JSON form.
- `show`, `path`, and `open` resolve any artifact-reference kind consistently; `path` obeys its exact stdout/exit-code
  contract, and viewer commands are safely bounded.
- `doctor` reports, fixes, and verifies the enriched index with meaningful health exits.
- `create` retains its prior behavior and prints the durable `file:` reference.
- `just check` passes with no memory, generated instruction, skill-template, or documentation changes, and bead
  `sase-ax.3` can be closed with the verified evidence.
