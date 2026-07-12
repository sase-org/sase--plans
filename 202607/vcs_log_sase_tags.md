---
create_time: 2026-07-09 13:20:16
status: done
prompt: 202607/prompts/vcs_log_sase_tags.md
tier: tale
---
# Plan: Opt-in SASE Commit Tags in `sase vcs log`

## Goal

Add an opt-in CLI display mode for `sase vcs log` that enriches commits with the trailing `SASE_*` metadata footer tags
SASE writes into commit messages, while stripping the `SASE_` prefix in user-facing output.

The first pass should keep default output byte-for-byte compatible unless the new option is supplied.

## Current Context

- `sase vcs log` is wired through:
  - `src/sase/main/parser_vcs.py` for CLI arguments.
  - `src/sase/main/vcs_handler.py` for dispatch.
  - `src/sase/vcs_log/collect.py` for repo resolution, provider calls, remote comparison, and aggregation.
  - `src/sase/vcs_log/render.py` for `pretty`, `full`, `oneline`, and `json` output.
- Commit records already include `subject` and `body` through `VcsCommitWire`.
- The git provider already captures the body with the pinned `VCS_LOG_GIT_FORMAT`.
- SASE commit footers are written by `src/sase/workflows/commit/runtime_tags.py`; new footer keys render as
  `SASE_<KEY>=<value>`.
- `runtime_tags.parse_trailing_commit_tags()` already knows the trailing footer-block semantics, but it also accepts
  legacy unprefixed keys. The requested UI should focus on actual `SASE_*` keys and display them without the prefix.
- The Rust-backed `VcsCommitWire` schema currently mirrors Python schema version 2. Adding a first-class `tags` field
  there would require coordinated `sase-core` and Python schema changes.

## User-Facing Contract

Add a public log option:

```text
-t, --tags    Show trailing SASE_* commit tags without the SASE_ prefix
```

Default behavior:

- `sase vcs log`
- `sase vcs log --format pretty`
- `sase vcs log --format full`
- `sase vcs log --format oneline`
- `sase vcs log --format json`

remain unchanged when `-t/--tags` is absent.

With `-t/--tags`:

- `pretty`: append compact dim tag metadata to each row that has tags, after the subject and before the author suffix.
  - Example: `subject text  · TYPE=sdd · PLAN=sdd/foo.md  · Bryan`
- `oneline`: append a plain compact bracketed suffix.
  - Example: `abc1234 sase subject text [TYPE=sdd PLAN=sdd/foo.md]`
- `full`: show parsed tags on a dedicated dim metadata line, and avoid duplicating the raw trailing `SASE_*` footer in
  the printed body.
  - Example: `tags: TYPE=sdd · AGENT=worker-1 · MACHINE=host`
- `json`: add a per-commit `sase_tags` object only when the option is enabled.
  - Keys are stripped, for example `"TYPE": "sdd"`.
  - Commits without SASE tags get `"sase_tags": {}` for predictable opt-in JSON shape.

Only trailing footer tags whose rendered key starts with `SASE_` should be displayed. Legacy unprefixed footer keys can
continue to be parsed by existing commit-workflow code, but this new CLI enrichment should match the requested `SASE_*`
surface.

## Technical Approach

1. Add CLI plumbing.
   - Add `-t/--tags` to `_add_log_options()` in `src/sase/main/parser_vcs.py`.
   - Respect CLI rules: keep help text clear, give the long option a short alias, and place the option consistently in
     the existing option order.
   - Thread `args.tags` through `_handle_log()` in `src/sase/main/vcs_handler.py` into
     `sase.vcs_log.render.render(show_tags=...)`.

2. Add a small tag view helper.
   - Create a focused helper near the VCS log renderer, for example `src/sase/vcs_log/tags.py`.
   - Input: `VcsCommitWire`.
   - Reconstruct the commit message from `subject` and `body`.
   - Parse only the contiguous trailing footer block.
   - Return:
     - ordered stripped SASE tags for display/JSON, for example `(("TYPE", "sdd"), ("PLAN", "sdd/foo.md"))`;
     - body text with the trailing SASE footer removed for `full` mode when tags are shown.
   - Reuse the semantics from `runtime_tags` where practical, but avoid depending on private helpers unless they are
     promoted to a public API.

3. Keep the Rust wire schema unchanged in the first pass.
   - Since `body` already contains the footer, the CLI can derive the opt-in display view without changing
     `VcsCommitWire` or `AggregatedCommitWire`.
   - This avoids a schema v3 migration across Python and `sase-core` before the UI contract is proven.
   - If another frontend later needs parsed tags as domain data, promote the helper into the Rust-backed VCS log model
     in a follow-up:
     - add `sase_tags` to Python and Rust `VcsCommitWire`;
     - bump the VCS log wire schema;
     - update parser parity tests in both repos.

4. Update renderers.
   - `render()` accepts `show_tags: bool = False`.
   - `_render_json()` conditionally includes `sase_tags`.
   - `_render_oneline()` conditionally appends the bracketed suffix.
   - `_commit_line()` conditionally appends the dim tag suffix.
   - `_print_full_commit()` conditionally prints the tag line and uses the de-tagged body.
   - Keep warning behavior, remote state summaries, date grouping, reverse ordering, and empty-state output unchanged.

5. Formatting decisions.
   - Display keys without `SASE_`.
   - Render values exactly as parsed from the footer after normal line trimming.
   - Use stable ordering from the footer block after duplicate-key resolution; if duplicate handling needs a tie-break,
     later footer lines win to match the existing parser behavior.
   - Do not truncate values in the first pass; Rich soft wrapping already handles `pretty` and `full`, and `oneline` is
     intentionally plain.

## Tests

Add or update focused tests:

- `tests/main/test_vcs_parser.py`
  - default `ns.tags is False`;
  - `-t` and `--tags` set `ns.tags is True`;
  - handler test namespaces include `tags=False`.
- `tests/test_vcs_log_render.py`
  - default `pretty`, `full`, `oneline`, and `json` goldens remain unchanged;
  - `pretty --tags` shows stripped tag suffixes;
  - `oneline --tags` shows stripped bracketed suffixes;
  - `full --tags` shows a tag line and does not duplicate the raw trailing SASE footer in the body;
  - `json --tags` includes `sase_tags`, including `{}` for commits without tags.
- New helper tests, either in `tests/test_vcs_log_render.py` or `tests/test_vcs_log_tags.py`
  - parses a trailing `SASE_*` block;
  - strips only the prefix from keys;
  - ignores non-trailing `SASE_*` text;
  - ignores legacy unprefixed keys for this display mode;
  - handles duplicate SASE keys with later values winning.

Optional integration coverage:

- Add one `tests/test_vcs_provider_vcs_log.py` case showing that a real git commit with `SASE_TYPE=...` in the footer
  still arrives in `VcsCommitWire.body`, so the renderer helper has the raw data it needs.

## Verification

After implementation:

```bash
just install
pytest tests/main/test_vcs_parser.py tests/test_vcs_log_render.py tests/test_vcs_log_tags.py
pytest tests/test_commit_runtime_tags.py tests/test_vcs_provider_vcs_log.py
just check
```

If the optional provider integration test is not added, skip its targeted pytest command but still run `just check`.

## Risks And Follow-Ups

- Full-mode body cleanup needs careful handling so ordinary body text is not removed unless it is part of the trailing
  `SASE_*` footer block.
- JSON consumers should be protected by opt-in shape changes only; default JSON must remain unchanged.
- If parsed tags become useful outside this CLI renderer, move them into the Rust-backed VCS log wire model in a
  separate schema bump instead of spreading renderer-local parsing across frontends.
