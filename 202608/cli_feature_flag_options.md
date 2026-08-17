---
tier: tale
title: Enable and disable feature flags from the sase command line
goal:
  Root-level `-f/--enable-feature` and `-F/--disable-feature` options force any
  registered feature flag on or off for one `sase` invocation and every process it
  launches, outranking config layers and an inherited SASE_FEATURE_FLAGS value.
size: medium
proposed_by: bbugyi200.athena.051.f0
create_time: 2026-08-17 12:53:44
status: wip
---

# Plan: Root-level feature-flag CLI options

## Problem

Feature flags can only be flipped through config files (`feature_flags:` in user,
overlay, or project-local config) or the `SASE_FEATURE_FLAGS` JSON env var. There is no
one-shot way to try a flag for a single command. Worse, a shell that exports
`SASE_FEATURE_FLAGS` pins those keys for every `sase` process it starts, because env is
the highest-precedence layer; a user who wants to try `coder_inherits_planner_chat` once
has to edit and restore an env var.

Add two repeatable root options:

```
sase -f coder_inherits_planner_chat run "..."
sase --enable-feature coder_inherits_planner_chat --disable-feature prettier_enabled ace
```

## Design decisions

Make these decisions exactly as written; each one is load-bearing.

### 1. The options live on the root `sase` parser and must precede the subcommand

`-f` is already taken by many subcommands with unrelated meanings
(`sase bead list -f json`, `sase monitor -f`, `sase commit -f`, `sase stitch -F`,
`sase proc -f/-F`, and more). A scan that harvested `-f` from anywhere in argv would
silently steal those. So the new options are recognized **only in the leading run of
option tokens, before the first non-option token** — the `git -c x=y commit` shape.

`sase agent list -f x` therefore stays an `agent list` error, unchanged. Do not try to
make the options work after the subcommand.

### 2. Consume the options from `sys.argv` before anything else in `main()`

`src/sase/main/entry.py` dispatches three pre-parse paths that key off `sys.argv[1]`:
the `bead` fast path, the `completion candidates` fast path, and
`handle_run_special_cases` for `sase run`. `parser_only_hint(sys.argv)` likewise narrows
parser construction on `sys.argv[1]`. If a leading `-f` sat in argv, all four would be
skipped: two lose their startup fast path, `parser_only_hint` falls back to building the
full parser, and `sase -f key run .` would lose the special-case handling that makes
`.`, `#workflow`, and space-containing prompts work.

So `main()` extracts the leading feature-flag options first, applies them, and rewrites
`sys.argv` in place to the remaining tokens. Every existing path downstream then behaves
exactly as it does today. Nothing else in `entry.py` moves.

Because the tokens are removed before `parser.parse_args()`, argparse never sees them.
The root-parser registration in decision 6 exists for `--help`, `--full-help`, and the
completion spec only. Guard against the two drifting apart with a test (decision 9).

Keep startup cheap: only look for these options when `sys.argv[1]` starts with `-`, and
only import the feature-flag registry once at least one option token was actually found.
`sase -h` and every ordinary `sase <command> ...` invocation must not pay a new import.

### 3. A new `cli` flag source that outranks `env`

`FlagSource` currently ranks `default → user → overlay → local → override → env`, with
env applied last. A CLI option modeled as an `override` would lose to an inherited
`SASE_FEATURE_FLAGS`, which is precisely the case this feature exists to fix. Add `cli`
to `FlagSource` and apply it **after** env, as the new highest layer. Leave `override`
(the in-process test hook) where it is.

### 4. Mirror the CLI values into `SASE_FEATURE_FLAGS` for child processes

Only three call sites run `install_process_feature_flags()` (ACE, axe, the agent
runner), and only that function writes the resolved env transport. A command such as
`sase -f coder_inherits_planner_chat run "..."` spawns a child that resolves flags
itself, so without help it would never see the option.

When CLI values are recorded, merge them over the inherited `SASE_FEATURE_FLAGS` JSON
and write the merged object back to `os.environ`. This is the same "CLI option to env
var for downstream resolution" pattern `--vcs-provider` already uses in `ace_handler.py`
and `axe_handler.py`. Merge, never replace: unrelated inherited keys must survive.

Write only the CLI keys plus whatever was inherited — do not resolve and pin the full
snapshot here. `handle_ace_command` deliberately calls `set_include_local_config(False)`
_before_ `install_process_feature_flags()`, and eagerly resolving in `main()` would pin
a snapshot that still included project-local config.

### 5. Validate the keys up front, and fail like argparse does

Unknown keys must not be silently dropped the way the resolver drops unknown config keys
— the user typed this one deliberately. Exit 2 with `sase: error: <message>` on stderr
for:

- an unregistered flag key (name the key; point at `sase flag list`),
- the same key passed to both `-f` and `-F`,
- a missing option value (`sase -f` with nothing after it).

Repeating the same key on the same side is fine and idempotent.

### 6. Register both options on the root parser for help and completion

Register them in `create_parser()` with `action="append"`, `metavar="<flag>"`,
`dest="enable_feature"`/`dest="disable_feature"`, and mark both actions `ValueKind.FLAG`
via `sase.completion.kinds.set_completion_kind`, so shell completion offers registered
flag keys once a `FLAG` candidate provider exists (`ValueKind.FLAG` is already defined;
only `PROJECT` and `BEAD` have live providers today, and a kind without a provider is
inert).

Also surface them in compact root help: update the usage line in
`_format_compact_root_help` and `_format_colored_compact_root_help` to
`usage: sase [-h] [-H] [-f <flag>] [-F <flag>] <command> [args...]`, and add a short
`Global options:` block listing both with one-line summaries plus an example line such
as `sase -f coder_inherits_planner_chat run "..."`. Keep the two renderers' stripped
output identical — an existing test asserts that.

### 7. Provenance rendering

`sase flag list` and `sase flag show` both route through `source_text()` in
`src/sase/feature_flags/cli_render.py`, which marks env provenance prominently so a
long-running process cannot hide an override. Give `cli` the same treatment:
`CLI:--enable-feature` / `CLI:--disable-feature`, styled like the ENV chip.

Derive `source_detail` from the value inside the resolver — an enabled CLI key can only
have come from `--enable-feature`, a disabled one from `--disable-feature` — so no extra
plumbing is needed to carry which option was used.

In `sase flag show`, add a `cli` row to the LAYERS block exactly where `_layer_rows`
already adds the `env` row, and by the same rule: only when that source won. (The env
mirror from decision 4 means env also carries the key; showing both rows would read as a
contradiction. `_env_value_from_decision` returns `None` unless
`decision.source == "env"`, so a CLI-won decision correctly suppresses the env row.
Preserve that.)

`flag_view_json` already emits `source` and `source_detail` generically, so `--json`
needs no change.

## Implementation

### New module: `src/sase/main/global_options.py`

Owns everything about root options parsed outside argparse.

- `FEATURE_FLAG_OPTION_STRINGS` — the enable/disable option strings, the single source
  of truth shared with the parser registration.
- `register_global_feature_flag_options(parser)` — adds both options to the root parser
  and sets their completion kind (decision 6).
- `extract_leading_feature_flag_options(argv)` — pure function over an argv sequence,
  returning the requested `{key: bool}` mapping and the remaining argv. Accepts the four
  argparse spellings: `-f KEY`, `-fKEY`, `--enable-feature KEY`, `--enable-feature=KEY`
  (and the `-F`/`--disable-feature` equivalents). Stops at the first token that is not
  one of these options, and never looks past a `--` separator. Raises a module-local
  error type for a missing value or a key requested on both sides.
- `consume_global_options()` — the one function `entry.main()` calls. Extracts from
  `sys.argv`, and when the mapping is non-empty: validates every key against
  `feature_flag_definitions()`, calls `set_cli_feature_flags()`, and assigns the
  remaining tokens back into `sys.argv[1:]`. Converts the error type into
  `print("sase: error: ...", file=sys.stderr)` + `sys.exit(2)`.

### `src/sase/feature_flags/models.py`

Add `"cli"` to the `FlagSource` literal.

### `src/sase/feature_flags/resolver.py`

Add a `cli: Mapping[str, bool] | None = None` keyword to `resolve_feature_flags()`,
applied after the `env_value` block as the last layer. Apply it in two `_apply_values`
calls so `source_detail` is `--enable-feature` for the `True` keys and
`--disable-feature` for the `False` keys, both with `source="cli"`.

### `src/sase/feature_flags/env.py`

Add `merge_feature_flags_env(values, env=os.environ)`: parse the existing
`SASE_FEATURE_FLAGS` (empty when unset), update it with _values_, and write back the
same stable `json.dumps(..., sort_keys=True, separators=(",", ":"))` encoding
`_encode_feature_flags_env` already uses — factor the encoding into a shared helper
rather than duplicating the separators. A malformed inherited value keeps raising
`FeatureFlagEnvError`, matching the documented "startup error for that process"
behavior; `consume_global_options()` reports it as a `sase: error:` line, not a
traceback.

### `src/sase/feature_flags/snapshot.py`

Add module-level CLI values behind the existing `_lock`, a
`set_cli_feature_flags(values)` that records them, clears the cached `_snapshot`, and
calls `merge_feature_flags_env()`; and pass them into `resolve_feature_flags()` from
`_build_snapshot()`. Do not add accessors nothing calls — the symvision gate rejects
unused symbols.

### `src/sase/main/entry.py`

At the top of `main()`, before the `bead` fast-path check:

```python
if len(sys.argv) > 1 and sys.argv[1].startswith("-"):
    from .global_options import consume_global_options

    consume_global_options()
```

### `src/sase/main/parser.py`

Call `register_global_feature_flag_options(parser)` in `create_parser()` after the `-H`
registration, and update both compact-help renderers per decision 6.

### `src/sase/feature_flags/cli_render.py` and `cli_show.py`

Per decision 7.

### Docs

- `docs/configuration.md`, `### feature_flags` (the resolution-order paragraph): add the
  CLI layer as the new highest-precedence source and describe both options, including
  that they merge into `SASE_FEATURE_FLAGS` so launched agents inherit them.
- `docs/configuration.md`, `## CLI Flags`: add a `### sase (global)` table above
  `### sase ace`, in the same column shape the surrounding tables use.
- `docs/configuration.md`, environment-variable table: note that the root options merge
  into `SASE_FEATURE_FLAGS`.
- `docs/cli.md`: one short note after the intro paragraphs that `sase` accepts
  `-f/--enable-feature` and `-F/--disable-feature` before the command, linking to the
  configuration reference.

### Regenerate

`just sync-completion-spec` rewrites `tests/completion/snapshots/cli_spec.json`, whose
drift test fails until then. The feature-flag config schema is unchanged (no registry
entry is added), so `just sync-feature-flags-schema` is not needed.

## Tests

- `tests/main/test_global_options.py` (new): all four option spellings; repeated
  options; mixed `-f`/`-F`; stops at the first non-option token; ignores tokens after
  `--`; argv untouched when no feature-flag option is present; unknown key, both-sides
  conflict, and missing value each exit 2 with a `sase: error:` line;
  `consume_global_options()` rewrites `sys.argv` and records the values.
- Drift guard (same file): the option strings registered on the root parser equal
  `FEATURE_FLAG_OPTION_STRINGS`.
- `tests/feature_flags/test_resolver.py`: `cli` beats `env`, beats `override`, and beats
  every config layer; a project-scoped flag is settable from the CLI; `source` is `cli`
  and `source_detail` is the matching option string.
- `tests/feature_flags/test_env.py`: `merge_feature_flags_env` creates the var when
  unset, merges over inherited keys, overwrites a conflicting inherited key, leaves
  unrelated keys alone, and raises on a malformed inherited value.
- `tests/feature_flags/test_snapshot.py`: `set_cli_feature_flags` invalidates the cached
  snapshot and exports the merged env transport.
- `tests/feature_flags/test_cli.py`: `flag list` and `flag show` render the CLI chip,
  and `flag show` adds the `cli` LAYERS row without also printing an `env` row for that
  key.
- `tests/main/test_parser_root_help.py`: the updated usage line, the `Global options:`
  block, and both options in `--full-help`.
- An entry-level test that `sase -f <key> bead list` still reaches the `bead` fast path
  / narrow parser — i.e. `sys.argv` is `["sase", "bead", "list"]` after
  `consume_global_options()`, so `parser_only_hint` still returns `"bead"`.

## Acceptance criteria

With `SASE_FEATURE_FLAGS='{"coder_inherits_planner_chat":false}'` exported:

1. `sase -f coder_inherits_planner_chat flag show coder_inherits_planner_chat` reports
   `effective: on` with a CLI source chip and a `cli` LAYERS row.
2. `sase -F prettier_enabled flag list` shows `prettier_enabled` off with the CLI chip.
3. A process launched by a `sase -f <key> ...` command sees the key in its inherited
   `SASE_FEATURE_FLAGS`.
4. `sase -f bogus_key ace` prints `sase: error: ...` and exits 2 without starting ACE.
5. `sase bead list -f json`, `sase run .`, and `sase run "#workflow ..."` behave exactly
   as before.
6. `sase --help` and `sase --full-help` both document the options.

## Out of scope

- No persistence: these options affect one invocation and its children. Writing flags to
  config (a `sase flag set`) is not part of this work.
- No acceptance of the options after the subcommand token (decision 1).
- **Do not edit `sase/memory/sase_flags.md`, `AGENTS.md`, or any generated provider
  instruction shim.** That note documents how flags are enabled and will want a line
  about these options, but memory edits require the user's explicit in-conversation
  permission, which a plan file cannot grant. Surface the proposed memory update to the
  user at handoff instead.

## Verification

Run `just install` first — workspace checkouts are ephemeral and may have stale
dependencies. Then `just check`. Because this change touches the root parser,
`entry.py`, and the checked-in completion spec, also run `just check-full` before
landing, through `/sase_monitor` rather than inline.
