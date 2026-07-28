---
tier: tale
title: Accept multi-word and multi-line values in `sase var set`
goal: An agent can publish an output-variable value containing spaces and newlines
  through an unambiguous CLI input path, values are normalized and size-checked at
  write time, and the documented contract matches the implementation.
create_time: 2026-07-28 11:30:07
status: done
---

- **PROMPT:** [202607/prompts/var_set_multiline_values.md](prompts/var_set_multiline_values.md)

# Plan: Accept Multi-Word and Multi-Line Values in `sase var set`

## Context

### What is actually restricted today

Output variables are set with `sase var set KEY=VALUE [KEY=VALUE ...]`. The restriction is **not** in the storage or
rendering stack — it is in the argv-shaped input path.

Verified current behavior:

- `src/sase/main/parser_var.py` declares a single positional `assignments` with `nargs="+"` and no `type=`. Every argv
  token must independently be a complete `KEY=VALUE` string.
- `src/sase/core/agent_output_variables.py::parse_output_variable_assignments` splits each token on the first `=` and
  validates only the key (`_KEY_RE = [A-Za-z_][A-Za-z0-9_]*`). The value is stored verbatim with no character
  restrictions at all.
- Storage is `agent_meta.json["output_variables"]`, written as JSON, so any string round-trips.

Consequences, each confirmed by running the CLI:

| Invocation                            | Result today                                                  |
| ------------------------------------- | ------------------------------------------------------------- |
| `sase var set "summary=tests passed"` | Works. Stores `tests passed`.                                 |
| `sase var set summary=tests passed`   | Fails: `output variable assignment must be KEY=VALUE: passed` |
| `sase var set "$(printf 'b=x\ny')"`   | Works. Stores `x\ny`.                                         |
| Value from a file or a pipe           | No supported form exists.                                     |

So spaces already work _when the caller quotes correctly_, newlines already work _through awkward `$'...'` or
`$(printf ...)` gymnastics that silently strip trailing newlines_, and there is no way at all to feed a value from a
file, a heredoc, or a pipe. The failure mode for the common unquoted case is a terse error that names the stray token
and offers no remedy.

### The downstream stack is already multi-line-safe

This was checked rather than assumed, and it is why the change stays confined to the input path:

- `src/sase/ace/tui/widgets/prompt_panel/_agent_output_variables.py` branches on `"\n" in value` and indents
  continuation lines for both the flat and role-attributed layouts. Covered by
  `tests/ace/tui/widgets/test_agent_display_output_variables.py`.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_sections.py::append_variables_section` previews with
  `first_meaningful_line(...)` when folded and renders the full body when expanded.
- `src/sase/agents_sync/rendering_variables.py` routes values through `md_cell`, which maps `\n` to `<br>` and `\r` to a
  space. Covered by `tests/agents_sync/test_rendering.py`.
- The Telegram plugin's completion message (`sase-telegram`, `src/sase_telegram/formatting.py`) already renders a
  multi-line value inside a fenced code block.
- The Rust core only reads: `crates/sase_core/src/agent_scan/wire.rs` types `output_variables` as
  `BTreeMap<String, String>` and `scanner.rs` coerces it with `coerce_str_str_map`. Arbitrary strings already pass, so
  no wire or binding change is required.

### The one real correctness gap

`src/sase/agents_sync/v2_validation.py` caps a value at `MAX_OUTPUT_VARIABLE_VALUE_BYTES = 8_192` UTF-8 bytes, but
nothing enforces that at write time. `src/sase/agents_sync/inventory_io.py::_portable_output_variables` **silently
skips** any oversized value when building the sidecar payload. An agent that writes a large multi-line value today gets
a success message from `sase var set` and then finds the variable missing from the published agents sidecar, with no
error anywhere. Making multi-line values a first-class input path makes large values far more likely, so the cap must be
enforced where it can be reported — at the point of the write.

## Implementation

1. **Add normalization and a write-time size guard to `src/sase/core/agent_output_variables.py`.**
   - Define `MAX_OUTPUT_VARIABLE_VALUE_BYTES = 8_192` and `MAX_OUTPUT_VARIABLES = 256` here, so the storage layer owns
     the limits alongside `_KEY_RE`.
   - Add a module-private normalizer applied to every value on the write path. It must, in order: reject a value
     containing `\x00` with a message naming the key; convert `\r\n` and any remaining lone `\r` to `\n`; then reject a
     value whose UTF-8 length exceeds the cap, with a message naming the key, the actual byte count, and the limit.
   - Apply it in both `parse_output_variable_assignments` and `set_agent_output_variables`, so a caller that bypasses
     the parser (the new value-source flags) still gets the same contract. Keep raising `ValueError`; the handler
     already converts that into a clean `Error: ...` exit.
   - Do not otherwise alter the value: leading/inner/trailing spaces, blank lines, and the `split("=", 1)` behavior that
     lets a value contain further `=` characters all stay exactly as they are.
   - Export the two new constants from `__all__`.

2. **Make `src/sase/agents_sync/v2_validation.py` import the limits from the core module** instead of defining its own,
   and keep re-exporting them so `v2_io.py` and `inventory_io.py` importers are unaffected. There must be exactly one
   definition of each constant in the tree.

3. **Add single-key value-source options to `src/sase/main/parser_var.py`.**
   - Add a mutually exclusive group on the `set` subparser:
     - `-v, --value TEXT` — use `TEXT` as the value verbatim, never split on whitespace.
     - `-f, --value-file PATH` — read the value from `PATH`; `-` reads stdin.
   - Leave the `assignments` positional at `nargs="+"` so the existing `KEY=VALUE ...` form is untouched.
   - Follow the repo CLI rules: options listed alphabetically, every long option has a short alias, and `-h/--help` is
     genuinely useful. Give the subparser a `description` explaining the two forms and a `RawDescriptionHelpFormatter`
     `epilog` with worked examples covering a quoted multi-word value, `--value`, a heredoc into `--value-file -`, and a
     pipe. Show the `--value=...` spelling in at least one example so a value beginning with `-` is covered.

4. **Implement the two input modes in `src/sase/main/var_handler.py::_handle_var_set`.**
   - When neither option is given, behavior is unchanged.
   - When either is given, require exactly one positional and require it to be a bare key with no `=`. Otherwise fail
     with a message that states the rule (`sase var set KEY --value TEXT` sets exactly one variable).
   - `--value-file`: read `sys.stdin.read()` for `-`, else the file's text as UTF-8. Report a missing or unreadable file
     and an undecodable file as distinct, readable errors rather than a traceback.
   - Strip **at most one** trailing `\n` from a file/stdin value (after the core normalizer runs), so the near-universal
     trailing newline of a heredoc or a file does not become a trailing blank line in every renderer. Do not strip
     anything from a `--value` or `KEY=VALUE` value — those are already exactly what the caller typed.
   - Keep the existing success output (`agent:`, `keys:`, `artifacts_dir:`) unchanged. It prints key names only, so it
     stays single-line regardless of value content.

5. **Improve the malformed-assignment error in `parse_output_variable_assignments`.** When a token has no `=`, keep
   naming the offending token and add the remedy: quote the whole assignment, or use `sase var set KEY --value TEXT` for
   a value with spaces or newlines. This is the error an agent hits when it forgets to quote, so it should teach the
   fix.

6. **Extend `tests/main/test_var_handler.py`** to cover, at minimum:
   - The parser accepts `-v/--value` and `-f/--value-file` and rejects using both together.
   - `--value` stores a value containing spaces and newlines verbatim.
   - `--value-file` reads from a file and from stdin, and strips exactly one trailing newline (`"a\n\n"` stores
     `"a\n"`).
   - A value flag with zero, two, or a `KEY=VALUE`-shaped positional fails with the single-bare-key message.
   - A missing `--value-file` path fails cleanly with exit code 1.
   - `\r\n` and lone `\r` normalize to `\n`; a `\x00` value is rejected; a value one byte over the cap is rejected and
     names the limit, while a value exactly at the cap is accepted.
   - The existing unquoted-multi-word failure now surfaces the quoting/`--value` remedy.
   - The pre-existing `KEY=VALUE` tests keep passing untouched, proving backward compatibility.

7. **Add one test asserting the limits are defined once** — that `sase.agents_sync.v2_validation` and
   `sase.agents_sync.v2_io` expose the same objects the core module defines — so the two definitions cannot drift apart
   again.

8. **Update the documentation to match.**
   - `docs/configuration.md` `### sase var`: extend the form table with the `--value` and `--value-file` rows and
     replace the "Values are strings split on the first `=`" paragraph with the full contract — spaces and newlines are
     supported, how each of the three input forms handles trailing newlines, `\r\n` normalization, NUL rejection, and
     the 8 KiB per-value cap that is now reported at write time instead of silently dropping the variable at
     publication.
   - `docs/xprompt.md` "Cross-Agent Output Variables": note that a value may be multi-line and show the heredoc form,
     since the consumer-side Jinja rendering is unchanged.
   - `docs/cli.md`: refresh the one-line `sase var set` description if the new forms make it inaccurate.
   - `docs/ace.md` `**OUTPUT VARIABLES**`: it already documents indented multi-line values; adjust only if it implies
     multi-line values are unusual.
   - `docs/agents_sidecar.md`: it documents published variables; state that oversized values are now rejected at write
     time rather than dropped at publication.

9. **Update the skill source `src/sase/xprompts/skills/sase_var.md`.** Replace the "Quote assignments when your shell
   would otherwise split" bullet with the real guidance: use `KEY=VALUE` for simple tokens, `--value` for anything with
   spaces, and a heredoc into `--value-file -` for multi-line bodies. State the 8 KiB cap and that output variables are
   for small values, not report bodies. Edit only the source template — **do not** run `sase skill init --force`, since
   deploying requires a committed, landed source tree.

10. **Run `just check`** and fix everything it reports, including Symvision findings on any new module-level symbol.

## Validation

1. `just install`, then `just lint` and `just test`. The `sase var` tests are in `tests/main/test_var_handler.py`; the
   multi-line rendering tests that must keep passing are in
   `tests/ace/tui/widgets/test_agent_display_output_variables.py` and `tests/agents_sync/test_rendering.py`.
2. `just check` must pass before the work is reported complete.
3. Exercise the CLI by hand against a scratch artifacts directory (create a temp dir containing a minimal
   `agent_meta.json`, then run with `SASE_AGENT=1` and `SASE_ARTIFACTS_DIR` pointing at it). Confirm each of these, and
   read back the resulting `agent_meta.json` to check the stored strings byte for byte:
   - `sase var set "summary=tests passed"` still stores `tests passed`.
   - `sase var set summary --value "tests passed"` stores the same string.
   - A heredoc into `sase var set body --value-file -` stores the body with no trailing newline.
   - `printf 'a\r\nb\r\n' | sase var set body --value-file -` stores `a\nb`.
   - `sase var set a=1 b=2` still stores two variables.
   - `sase var set summary=tests passed` fails and the error names the quoting and `--value` remedies.
   - Setting both `--value` and `--value-file` fails as mutually exclusive.
   - A value of 8193 UTF-8 bytes fails and names the limit.
4. Review `sase var set --help` and confirm it reads well standalone: both forms visible, options alphabetical, examples
   that can be copied verbatim.
5. Confirm a multi-line value renders correctly in ACE's Agents-tab `OUTPUT VARIABLES` section for a real agent, or note
   explicitly that the existing widget tests are the evidence if a live run is impractical.

## Non-goals

- **Do not auto-join unquoted trailing tokens into the previous value.** `sase var set summary=tests passed` staying an
  error is deliberate: joining is ambiguous against the multi-assignment form (`sase var set a=1 b=2`), and any rule
  that resolves the ambiguity is harder to document than it is worth. The explicit `--value` flag is the supported
  answer, and the improved error points at it.
- **Do not raise `MAX_OUTPUT_VARIABLE_VALUE_BYTES` above 8 KiB.** Raising it changes the agents-sidecar wire contract
  and what older readers accept. Output variables are small handoff values by design; a report body belongs in an
  artifact file with the variable holding its path.
- **Do not port the output-variable write path to the Rust core.** All of it — `_KEY_RE`, the limits, and the new
  normalizer — already lives in `sase/core/agent_output_variables.py`, and `sase_core` stores the field as
  `BTreeMap<String, String>` with no validation to mirror. This change adds no behavior that crosses the boundary. If
  output-variable writes are ever moved to Rust, that module is the whole surface to port.
- **Do not fix the Telegram agent-detail row in this change.** `sase-telegram`'s `src/sase_telegram/agent_format.py`
  builds an "Outputs" row with `" · ".join(f"{key}={value}" ...)`, which a multi-line value breaks. This is pre-existing
  — newlines are already storable today — and it lives in a separate repo with its own commit flow. The follow-up is to
  give that row a first-meaningful-line preview like the ACE clan section already uses.
- **Do not add an `--append` mode, escape-sequence decoding, or a `KEY=@file` shorthand.** `--value` plus `--value-file`
  covers the requested capability without adding parsing ambiguity.
- **Do not deploy skills or commit/push.** Skill deployment happens only from a clean, landed tree, and committing is a
  separate, explicitly requested step.
