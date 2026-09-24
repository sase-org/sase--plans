---
tier: tale
title: sase-core CommandLineGrammar resolver and sase adapter
goal:
  A frozen sase_core_rs.CommandLineGrammar parses the Command Line spec once and, per
  keystroke, returns tokens, slot, diagnostics, signature, run policy and fuzzy-ranked
  candidates; sase loads it through a typed adapter proven by a whole-spec contract
  test.
size: medium
proposed_by: bbugyi200.athena.sase-17x.5
bead: sase-17x.5
status: done
---

- **PARENT:**
  [202609/command_line_panel.md](https://github.com/sase-org/sase--plans/blob/main/202609/command_line_panel.md)
- **BEAD:**
  [sase-17x.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17x/sase-17x.5.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-17x.5](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-17x.5.md)
- **COMMITS:**
  - [ede63ea](https://github.com/sase-org/sase/commit/ede63ea9dcec4672873412d9f5d4d3f65f72ffcf)
    — feat(command-line): CommandLineGrammar resolver adapter and contract test

# Plan: sase-core `CommandLineGrammar` resolver + sase adapter (phase `line-resolver`, bead sase-17x.5)

## Context

Epic sase-17x adds a `:` Command Line panel to the TUI. Users type `sase` commands there
and get grammar-aware completion. This phase builds the completion engine: a frozen Rust
handle, `CommandLineGrammar`, in sase-core. It parses the Command Line spec JSON
(`sase completion spec -d -j`, produced by the already-landed `spec-contract` phase)
once. On each keystroke it returns tokens, the cursor slot, diagnostics, a live
signature, the run policy, and fuzzy-ranked candidates. This phase also adds a thin
Python adapter in sase.

The epic plan (`plan:202609/command_line_panel.md`) is the source of truth for the UX
contract. Its sections "Completion behavior", "Architecture and shared contracts →
Resolver API", and "`line-resolver`" define what this phase must deliver. This tale pins
down the details those sections leave open. Nothing in the panel UI is in scope;
completion-popup (sase-17x.9) is the first real consumer.

Boundary rule (core memory `rust_core_backend_boundary`): all parsing, ranking,
diagnostics and policy logic goes in `sase_core`. The Python side is only a loader plus
typed views.

Facts verified against current code while planning:

- The spec root is `{prog, version, root}`. Each command node has
  `name, path, aliases, hidden, summary, options[], positionals[], subcommands[], default_child, mutex_groups[[dest…]], run_policy[], writes, stdin`.
  - Options have
    `strings, dest, summary, takes_value, repeatable, choices, kind, hidden, required, metavar, default, value_hint`.
  - Positionals have
    `metavar, dest, summary, nargs, choices, kind, is_remainder, required, value_hint`.
  - `nargs` is `null | "?" | "*" | "+" | "..." | int`. `"..."` is argparse REMAINDER.
  - See `src/sase/completion/model.py` and `build.py` in sase.
- The live spec has 367 leaves, 51 `default_child: "list"` groups, and 13 mutex groups.
  It has 7 REMAINDER positionals (`proc run`, `tool run`, `monitor start`,
  `agent hold run`, `service proc run`, `lsp`, `screenshot`).
  - No node has both positionals and subcommands.
  - No `*`, `+` or `...` positional is followed by another positional.
  - No command currently has aliases. The model supports them, so the resolver must too.
  - Option strings only start with `-` or `--`. Some options have several long strings,
    for example `-b/--branch/--ref`.
- The run-policy `when` forms that occur today are `null`, `{"absent": [dest…]}` and
  `{"equals": {dest: value}}`. Examples: `run` is `foreground` when `prompt` is absent
  or equals `.`, and `tui` is `deny` with a note.
- sase never sets `allow_abbrev=False`, so argparse's unique long-option prefix
  abbreviations apply everywhere.
- The compact JSON (`separators=(",",":")`, sorted keys) of the full `-d` spec is about
  560 KB.
- sase-core's `crates/sase_core/src/editor/fuzzy.rs` already provides `fuzzy_match`,
  `compare_fuzzy`, and the crate-private prepared forms `FuzzyQuery`, `FuzzyText`,
  `fuzzy_match_prepared` and `compare_fuzzy_prepared`.
  - Tiers: 0 = whole-text prefix, 1 = basename prefix, 2 = substring, 3 = subsequence.
  - Runs are half-open character ranges. An empty query matches everything at tier 0
    with no runs.
- The frozen pyclass pattern is `PyGlossaryCatalogHandle` / `PyAtReferenceInventory` in
  `crates/sase_core_py/src/editor_completion/mod.rs`. That file is already 917 lines, so
  this phase adds a new binding domain instead of growing it.
- sase looks up bindings with `require_rust_binding("<name>")` from `sase.core.rust`.
  That also works for a class (a module attribute). `tools/check_sase_core_rs_bindings`
  collects those names statically, so the literal `"CommandLineGrammar"` must appear
  directly in the call.
- The sase CI pin `sase-core-revision.txt` currently points at `9956773`. sase-core
  `origin/master` is `6d0d0e6`, which already includes the proc-retention commit
  `f405c44`.

## Offsets and units (applies everywhere)

Every `start`, `end`, `cursor`, `replace_start`, `replace_end` and `match_runs` value is
a **Unicode scalar (char) offset**. That is exactly a Python `str` index. Never use byte
offsets on the wire. Clamp `cursor` to `[0, line.chars().count()]`.

## Part A — sase-core (work in the linked checkout)

Open it with `sase repo open sase-core -r "<why>"` and work only in the printed path.
Read its `AGENTS.md` first. Its rules apply here:

- never run bare `cargo`
- `mod.rs` is a facade (`mod` / `pub use` lines only)
- each file stays at or under 1,500 lines
- no `macro_rules!`
- `thiserror` errors
- import by module path, adding nothing to the root `pub use` list or the py
  `prelude.rs`
- Conventional Commit subjects (`feat(command-line): …`)
- never edit versions or CHANGELOGs

### A1. New top-level domain module `crates/sase_core/src/command_line/`

Register it in `lib.rs` as `pub mod command_line;` with a one-line `//!` summary so that
`just modules` lists it. Suggested files (each under 1,500 lines, with tests beside the
code in `tests/`):

- `mod.rs`: the facade.
- `wire.rs`: all serde types.
  - **Input spec wires** (`Deserialize`; unknown fields ignored; every field added after
    the first spec version uses `#[serde(default)]`):
    - `CommandLineSpecWire { prog, version, root }`
    - `CommandSpecWire`
    - `OptionSpecWire`
    - `PositionalSpecWire`
    - `RunPolicyRuleWire { policy, when: Option<RunPolicyWhenWire>, note }`
    - `RunPolicyWhenWire`: an untagged enum or struct with optional `absent` and
      `equals`. `equals` values are JSON scalars, compared by their string form.
    - `NargsWire`: untagged, `Int(u32) | Str(String)`, inside an `Option`.
  - **Output wires** (`Serialize`, snake_case keys exactly as listed in A3–A6).
  - `pub const COMMAND_LINE_WIRE_SCHEMA_VERSION: u32 = 1;`
- `grammar.rs`: `pub struct CommandLineGrammar`, an immutable arena of command nodes.
  - Each node records:
    - its parent index
    - a `name|alias → child index` map
    - a `strings → option index` map
    - its long-option list, for abbreviation lookup
    - its own `mutex_groups`
    - `confirms_option: Option<usize>` (an option whose strings include `-y` or `--yes`)
  - `CommandLineGrammar::from_json(&str) -> Result<Self, CommandLineGrammarError>`.
  - A path lookup that accepts names or aliases.
  - `command_count()`.
- `tokenizer.rs`: the POSIX-shlex lexer (A2).
- `resolve.rs`: the parse walk, slot classification, parsed values and `used_dests`
  (A3).
- `diagnostics.rs` (A4).
- `signature.rs`: the signature segments plus the shared usage formatter used by
  `command_help` (A5).
- `run_policy.rs`: rule evaluation (A3, last bullet).
- `complete.rs`: candidates, suppression, ranking and quoting (A6).
- `help.rs`: `command_help` (A7).
- `tests/`: the unit and golden tests (A8).

Public entry points, all taking `&self` on `CommandLineGrammar`:

- `resolve(line: &str, cursor: usize) -> LineContextWire`
- `complete(line, cursor, dynamic: &[DynamicCandidateWire], selected: &[String], limit: usize) -> CommandLineCompletionWire`
- `command_help(path: &[String]) -> Option<CommandHelpWire>`

### A2. Tokenizer

Match Python `shlex.split(line, posix=True)`, which does not treat `#` as a comment:

- Whitespace separates tokens.
- `'…'` is literal.
- In `"…"`, a backslash escapes only `\ " $` and a backtick, plus newline.
- Outside quotes, a backslash escapes the next char.
- Adjacent quoted and unquoted parts join into one token.

Each token records:

- `start` and `end`: raw span, including quotes
- `text`: the unquoted value
- `quoted`: any quote char appeared
- `unterminated`: the line ended inside a quote. The token then runs to end of line, and
  its `text` is the content so far.

The resolver needs "the unquoted text of the raw slice `[a, b)`" (for the prefix up to
the cursor), so expose a helper that re-lexes a raw slice tolerating an open quote.

### A3. `resolve` → `LineContextWire`

Output keys (the epic's Resolver API, with these exact names):

```
tokens[{text,start,end,role,quoted,unterminated}]
argv[]
path[]
node_kind            # "root" | "group" | "leaf" | "unknown"
slot{kind,dest,value_kind,choices,value_hint,prefix,replace_start,replace_end}
used_dests[]
diagnostics[{start,end,severity,code,message}]
signature{segments[{text,role,active,required}],summary}
run_policy{policy,note}
writes
confirms
confirm_flag_present
stdin
schema_version
```

- **Token roles:** `prog` (a leading `sase` at index 0), `command`, `option`,
  `option_value`, `positional`, `remainder`, `separator` (`--`), `unknown`.
- **argv:** the texts of every token, with a leading `sase` token removed. This is what
  the panel submits, so what you see is what runs.
- **Walk (argparse semantics):** start at the root. For each token in order:
  - **After an active REMAINDER:** role `remainder`, and the dest is the REMAINDER
    positional's.
  - **After `--` (and not in remainder):** role `positional`. A `--` token itself
    becomes role `separator`.
  - **Long option (`--x`, len > 2):** split at the first `=`. Resolve by exact string
    first, then by unique prefix over the node's long strings (argparse `allow_abbrev`).
    Several prefix hits that share one dest are still unique.
    - Unique → role `option`, with the `=value` part as its value.
    - Ambiguous → an `ambiguous_option` diagnostic.
    - None → `unknown_option`.
    - If the option `takes_value` and has no `=value`, the next token is its value (role
      `option_value`), unless that token looks like an option. That means it starts with
      `-`, has length > 1, and is not a negative number (`^-\d+$|^-\d*\.\d+$`).
  - **Short option (`-x…`, not a negative number):** walk the stacked chars. A flag char
    consumes one char. The first value-taking char takes the rest of the token as its
    value (`-n5`, `-p=foo` → `foo`), or the next token if nothing remains. An unknown
    char → `unknown_option`, and the rest of the token is skipped.
  - **Otherwise, when the node has subcommands:**
    - An exact name or alias match descends. The node changes, and parent options stop
      applying (argparse subparser semantics). Role `command`.
    - No match → `unknown_subcommand`. `node_kind` becomes `unknown`, and every later
      token gets role `unknown`.
  - **Otherwise it is a positional**, assigned in order:
    - `null` → 1 token
    - `"?"` → 0 or 1
    - int `n` → n
    - `"*"` / `"+"` → every later positional token. The spec has no positional after a
      greedy one, and a fixture test (A8) asserts that.
    - `"..."` → starts REMAINDER, so this token and every later token get role
      `remainder`. Option-looking tokens are included.
    - A positional token beyond every positional's capacity → role `unknown` plus an
      `extra_argument` diagnostic.
- **path / node_kind:** `path` holds the canonical command names walked (aliases map to
  primary names). `node_kind` is:
  - `root` for the empty path
  - `group` for a node with subcommands
  - `leaf` for a node with none
  - `unknown` after an unknown subcommand
- **Parsed values:** keep `dest → Vec<String>` over **all** tokens, including the one
  under the cursor. `used_dests` is its key set, in first-seen order. Flags record
  `"true"`.
- **Cursor slot.** The cursor token is the token with `start <= cursor <= end`. If none
  exists, a virtual empty token sits at `cursor`. The slot is classified from the parse
  state **before** that token:
  - The token starts with `-`, is not a negative number, and comes before any
    `--`/remainder → `option_name`. A bare `--` under the cursor with no trailing space
    is still `option_name` with prefix `--`.
  - The token is a pending option's value → `option_value`, with `dest`, `value_kind`
    (the spec `kind`), `choices` and `value_hint` from that option.
    - `--opt=val` and attached short values (`-pval`) → `replace_start` is just after
      the `=` or after the option char.
  - The node has subcommands → `subcommand`.
  - The next positional has capacity → `positional`, with that positional's fields.
    REMAINDER → `remainder`.
  - After an unknown subcommand, or past positional capacity → `none`.
  - `replace_start` / `replace_end` are the cursor token's span (adjusted as above).
    `prefix` is the unquoted text of `[replace_start, cursor)` (A2 helper).
- **Bare group with `default_child`:** `run_policy`, `writes`, `stdin` and `confirms`
  come from the default child. `signature.summary` notes
  `runs '<path> <child>' by default`. A group **without** `default_child` and with no
  subcommand typed gets a `missing_subcommand` diagnostic at severity `info`.
- **Run policy:** take the leaf's rules, or the default child's for a bare group. The
  first matching rule wins:
  - `when: null` always matches.
  - `absent: [d…]` matches when none of those dests has parsed values.
  - `equals: {d: v}` matches when the last parsed value of `d` equals `v`'s string form.
  - No match, no rules, or an unknown/root path → `{policy: "proc", note: null}`.
- **Other flags:** `confirms` means the resolved leaf has an option with `-y` or
  `--yes`. `confirm_flag_present` means that option's dest has a parsed value.
  `schema_version` is `COMMAND_LINE_WIRE_SCHEMA_VERSION`.

### A4. Diagnostics (advisory only — never block)

Codes and severities:

| Code                 | Severity  | Notes                                                                                                                                                 |
| -------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `unterminated_quote` | `error`   | Span runs from the opening quote to the end of the line                                                                                               |
| `unknown_subcommand` | `warning` | Warning, not error, because the spec omits hidden commands and compat aliases that argparse still accepts                                             |
| `unknown_option`     | `warning` | Same reason as `unknown_subcommand`                                                                                                                   |
| `ambiguous_option`   | `warning` | The message names the candidates                                                                                                                      |
| `invalid_choice`     | `warning` | The value is not among `choices`                                                                                                                      |
| `missing_value`      | `warning` | A value-taking option followed by another option or `--`, or by the end of the line when the cursor is **not** at the end of the line                 |
| `extra_argument`     | `warning` |                                                                                                                                                       |
| `missing_required`   | `info`    | Leaf only: required options with no parsed value, and required positionals with no tokens. Span is `start=end=` line length; one row per missing item |
| `missing_subcommand` | `info`    | See A3                                                                                                                                                |
| `asks_to_confirm`    | `info`    | `confirms && !confirm_flag_present`. Message `asks to confirm; add -y to skip`. Empty span at the end of the line                                     |

**Typing-in-progress rule:** a diagnostic anchored on the cursor token is dropped when
its text is still a prefix of at least one static candidate for that token's slot (a
subcommand name, an option string, or a choice). `unterminated_quote` is never dropped.

Order diagnostics by `start`, then by code.

### A5. Signature

Segments come from one shared formatter, which `command_help`'s `usage` also uses:

- `sase` and each path word (role `command`).
- For a group: `<subcommand>` (role `subcommand`).
- For a leaf, each visible positional (role `positional`) by nargs:
  - `null` → `METAVAR`
  - `?` → `[METAVAR]`
  - `*` → `[METAVAR ...]`
  - `+` → `METAVAR [METAVAR ...]`
  - `...` → `METAVAR ...`
  - int `n` → METAVAR repeated n times
- Each required option as `--long METAVAR` (role `option`, `required: true`). Use the
  first long string, or the first string if there is none. Use the option's `metavar`,
  or `dest.upper()`.
- If the active slot is a non-required option, one extra `option` segment for it.
- `[options]` (role `options`) when other visible options remain.

`active` marks the segment for the current slot:

- the positional being filled
- the option whose name or value is being typed
- `<subcommand>` for a subcommand slot

`summary` is the resolved node's summary. For an unknown path it is empty.

### A6. `complete` → `CommandLineCompletionWire`

Output:
`{replace_start, replace_end, items[{insert_text, display, description, badge, source, match_runs, selected}], total, kind, schema_version}`.
`kind` is the slot kind.

**Dynamic input:**
`DynamicCandidateWire { value, display?, description?, badge?, source?, partial? }`.
`partial: true` means no trailing space, for example a directory path ending in `/`.

**Candidate sets by slot:**

- `subcommand`: the node's children.
  - Each child is one item (`badge` `group` or `cmd`, with `description` = summary).
  - Each alias is its own item. Its `display` is the alias, and its `description` is
    `alias of <name> — <summary>`.
  - If the prefix starts with `-`, the slot is really `option_name`; the resolver
    already classifies it that way.
- `option_name`: the node's visible (non-hidden) options, after suppression.
  - One item per option. Its `display` is the option string that best matches the
    prefix: a long string when the prefix is empty or starts with `--`, otherwise the
    best fuzzy score across its strings.
  - `badge` is the metavar for value-taking options, or `flag`.
  - `description` is the summary. When other strings exist, append ` (-x, --alt)`.
- `option_value` / `positional`: static `choices` (`source: "spec"`, badge `choice`)
  plus `dynamic` items. `remainder`: `dynamic` only. `none`: no items.
- Dynamic items are ignored for the `subcommand`, `option_name` and `none` slots.

**Suppression**, computed from the parse **excluding** the cursor token so that editing
an option does not hide itself:

- Drop non-repeatable options whose dest is already used.
- Drop every other member of a mutex group that already has a used member.

**Ranking.** Match with `FuzzyQuery::new(prefix)`, built once per call, against each
item's `display` via `fuzzy_match_prepared`. Drop non-matches.

- Sort key, with a non-empty prefix:
  `(!selected, tier, -score, source_rank, char length, lowercase, input order)`.
- Sort key, with an empty prefix: `(!selected, source_rank, input order)`.
- `source_rank` is `dynamic` items with `source == "memory"` = 0, `spec` = 1, other
  dynamic sources in caller order = 2, `path` = 3.
- `selected` is true when the item's value equals any string in `selected`.
- Dedupe by value after sorting. The first occurrence wins but takes the OR of
  `selected`.
- `total` is the count after dedupe. Truncate the items to `limit`.

**match_runs** index into `display`. For items whose `display` differs from their value
(dynamic `display`), still match against `display`.

**insert_text:** the value, quoted exactly like Python `shlex.quote`:

- empty → `''`
- all chars in ASCII `[A-Za-z0-9_@%+=:,./-]` → unchanged
- otherwise → `'…'`, with each `'` written as `'"'"'`

Then add one trailing space unless `partial`. Every static item is complete, so it gets
the trailing space. An option name inserts `--opt ` whether or not it takes a value.

### A7. `command_help(path)` → `Option<CommandHelpWire>`

Returns `None` for an unknown path. Otherwise it returns
`{usage, summary, positionals[{metavar,dest,summary,nargs,required,choices,value_kind,value_hint}], options[{strings,dest,summary,metavar,takes_value,required,repeatable,default,choices,value_kind,value_hint}], children[{name,aliases,summary}], default_child, run_policy[...rules as in the spec], writes, stdin, confirms}`.

- `usage` is `usage: ` followed by the A5 segments joined with spaces.
- Hidden options are omitted.

### A8. Tests in sase-core

- **Fixture:**
  - Generate `crates/sase_core/tests/fixtures/command_line/sase_spec.json` from sase
    with `sase completion spec -d -j -o <tmp>`.
  - Re-serialize it compact with sorted keys
    (`json.dumps(obj, separators=(",",":"), sort_keys=True)`).
  - Add a short `README.md` beside it with the regeneration command.
  - Load it with
    `include_str!(concat!(env!("CARGO_MANIFEST_DIR"), "/tests/fixtures/command_line/sase_spec.json"))`.
  - Also add a small hand-written `mini_spec.json` fixture for shapes the real spec
    lacks:
    - a subcommand alias
    - an int `nargs`
    - a required option
    - a group without `default_child`
    - an option with `-y`
    - three stacked short flags
    - two long options sharing a prefix
- **Invariant tests over the real fixture**, protecting the A3 simplifications:
  - No node has both positionals and subcommands.
  - No `*`, `+` or `...` positional is followed by another positional.
  - No option string looks like a negative number.
- **Tokenizer unit tests:**
  - plain words
  - single and double quotes
  - adjacent concatenation (`a"b c"d` → `ab cd`)
  - backslash escapes inside and outside double quotes
  - unterminated `'` and `"`
  - multi-byte chars. Spans are char offsets: use `é` and an emoji.
  - whitespace-only lines
- **Golden resolve/complete cases** (table-driven; each asserts the relevant fields).
  Cover every slot kind and:
  - `bead cl|` → `subcommand` slot. The first completion is `close ` (tier 0).
  - `bead close sase-1 --reason |` → `option_value`, dest `reason`, `value_hint` `text`,
    `writes: true`.
  - `bead close --res|` → unique abbreviation. There is no `unknown_option`, and the
    completion is `--resolution `.
  - `bead close --resolution=d|` → `replace_start` is after `=`, and the choices are
    filtered to `done`.
  - `bead list -s |` → choices include `open`. `bead list -s bogus` → `invalid_choice`.
  - `bead note sase-1 --edit x --|` → `--remove` is suppressed (mutex). `-h` stays.
  - `bead close -f --force|`-style repeat: a non-repeatable flag already used is
    suppressed when completing elsewhere.
  - Stacked shorts on the mini spec (`-abc`, and `-n5` attached value).
  - `proc run -c /tmp -- ls -la|` → roles `separator` then `remainder`. The slot is
    `remainder`, and `argv` equals the unquoted tokens without `sase`.
  - `sase proc run ls -la` → `prog` role stripped from `argv`. `ls` and `-la` are
    `remainder`.
  - `agent|` / `agent |` → group with `default_child`. `run_policy` and the signature
    summary come from `agent list`, and there is no `missing_subcommand`.
  - `run` → `foreground` (absent). `run .` → `foreground` (equals). `run "fix it"` →
    `proc`. `tui` → `deny` with note `You're already in the TUI`.
  - `bead close "sase-1|` → an `unterminated_quote` error, and the prefix is `sase-1`.
  - `nosuch cmd` → `unknown_subcommand`, `node_kind: unknown`, the rest role `unknown`,
    and slot `none`.
  - `bead close` (cursor at end, after a space) → `missing_required` for `ids`, as
    `info`. `bead close sase-1 extra`: `ids` is `+`, so `extra` is still an id and there
    is no `extra_argument`. Use a mini-spec case for `extra_argument`.
  - A `-y` command without `-y` → an `asks_to_confirm` row and `confirms: true`. With
    `-y` → `confirm_flag_present: true` and no row.
  - Typing-in-progress: `bead clo|` has no `unknown_subcommand`, but `bead zzz x` does.
  - The signature `active` segment moves between the positional and option slots.
    `usage` for `bead close` matches an exact expected string.
  - `complete` with dynamic items:
    - selected-first ordering
    - `memory` before `spec`, before other sources
    - dedupe that ORs `selected`
    - `partial` means no trailing space
    - `shlex.quote` parity for `a b`, `it's`, `''` and `ok-1.2`
    - the `limit` truncates while `total` keeps the full count
- **Perf test:** `crates/sase_core/tests/command_line_perf.rs`, an integration test
  using the real fixture.
  - Run `resolve` over ~200 representative lines × 5 iterations and compute p95.
  - Run `complete` on a value slot with 1,000 synthetic dynamic candidates.
  - Under `cfg(not(debug_assertions))`, assert `resolve` p95 < 1 ms and `complete` < 2
    ms.
  - Under debug builds (what `just check` runs), assert a generous ceiling (`resolve`
    p95 < 25 ms, `complete` < 50 ms) so a debug build under load stays stable while
    quadratic regressions still fail.
  - Print the measured numbers (`eprintln!`).

### A9. Python binding domain `crates/sase_core_py/src/command_line/`

Create `mod.rs` and `tests.rs`, and register them in `lib.rs` (`mod command_line;` plus
`command_line::register_command_line(m)?;`). Model the code on
`PyGlossaryCatalogHandle`.

```rust
#[pyclass(name = "CommandLineGrammar", module = "sase_core_rs", frozen)]
struct PyCommandLineGrammar { grammar: sase_core::command_line::CommandLineGrammar }
```

- `#[new] fn new(py, spec_json: &str)` copies the string, parses it inside
  `py.allow_threads`, and maps errors to `PyValueError`.
- `#[classattr] SCHEMA_VERSION: u32`, equal to the core const.
- `fn resolve(&self, py, line: &str, cursor: usize) -> PyResult<PyObject>`.
- `#[pyo3(signature = (line, cursor, dynamic = None, selected = None, limit = 100))] fn complete(...)`.
  - `dynamic` is an optional list of dicts, parsed via `py_to_json_value` + serde into
    `Vec<DynamicCandidateWire>`. A bad shape raises `ValueError`.
  - `selected` is an optional list of str.
- `fn command_help(&self, py, path: Vec<String>) -> PyResult<PyObject>`. It returns
  `None` for an unknown path.
- `fn __len__(&self) -> usize`, the command node count.
- Return values through `serialize_to_py`.
- `tests.rs`:
  - Register the module the way `procs/tests.rs` does. Assert that `CommandLineGrammar`
    exists.
  - Construct it from the mini spec, and round-trip `resolve`, `complete` (with a
    dynamic dict) and `command_help` into Python dicts, checking a few keys.
  - Check that a bad spec raises `ValueError`, and that a bad dynamic dict raises
    `ValueError`.

### A10. Verify sase-core

Iterate with `just fast` and `just test -p sase_core command_line` /
`just test -p sase_core_py command_line`. Run `just fmt`. Then run the gate
`sase tool run check`, with a tool timeout of at least 15 minutes, until it is green. Do
not commit; the host owns completion.

## Part B — sase adapter and docs

### B1. `src/sase/completion/command_line_grammar.py`

- Module docstring: it is a thin adapter over `sase_core_rs.CommandLineGrammar`, and all
  logic lives in sase-core.
- `COMMAND_LINE_GRAMMAR_SCHEMA_VERSION: Final = 1`. This mirrors the core const; add it
  to the AGENTS "wire schema version" search terms by using that exact name.
- `TypedDict` views over the returned dicts. They are zero-copy, which matters on the
  per-keystroke path:
  - `LineToken`, `LineSlot`, `LineDiagnostic`, `SignatureSegment`, `LineSignature`,
    `RunPolicyOutcome`, `LineContext`
  - `DynamicCandidate` (`value` is required; the rest are `NotRequired`)
  - `CompletionItem`, `CommandLineCompletion`
  - `HelpPositional`, `HelpOption`, `HelpChild`, `CommandHelp`
- `class CommandLineGrammar`: a frozen, slotted wrapper holding the Rust handle.
  - `from_spec_json(text: str) -> CommandLineGrammar`. It calls
    `require_rust_binding("CommandLineGrammar")(text)` with the literal name, and raises
    `RuntimeError` when the handle's `SCHEMA_VERSION` differs from the mirror const.
  - `resolve(line, cursor) -> LineContext`
  - `complete(line, cursor, *, dynamic: Sequence[DynamicCandidate] = (), selected: Sequence[str] = (), limit: int = 100) -> CommandLineCompletion`
  - `command_help(path: Sequence[str]) -> CommandHelp | None`
  - `__len__`
- `load_command_line_grammar(path: Path) -> CommandLineGrammar`. It reads UTF-8 text and
  constructs the wrapper. Its docstring says to call it only from a worker thread (it
  does file I/O and a ~0.5 MB parse).

### B2. Contract test `tests/completion/test_command_line_grammar.py`

This test proves that the Python spec and the Rust wire agree.

- A module-scoped fixture builds the real spec in-process with
  `json.dumps(build_spec().to_json())` and wraps it with
  `CommandLineGrammar.from_spec_json`.
- A load test writes the JSON to `tmp_path` and calls `load_command_line_grammar`.
- A whole-tree agreement test walks `build_spec()` and checks every non-hidden leaf:
  - `command_help(list(path))` is not `None`, and its option dests equal the spec's
    visible option dests.
  - `resolve(" ".join(path) + " ", len(...))` has `node_kind == "leaf"` and `path` equal
    to the spec path.
  - `run_policy` and `writes` agree with the spec for leaves whose rules have
    `when: None`.
- A few end-to-end lines:
  - `bead close sase-1 --reason ` → an `option_value` slot
  - `sase proc run -- ls -la` → argv `["proc", "run", "--", "ls", "-la"]`
  - `tui` → `deny`
  - completing `bead cl` → the first `insert_text` is `"close "`
  - `bead list -s ` → the completion values include `open`
- A schema-mismatch test: monkeypatch the mirror const, then expect `RuntimeError`.

### B3. Symvision

The adapter's public symbols have no non-test consumer until completion-popup. Run
`just _lint-symvision` (via `sase tool run` if guarded). For every reported adapter
symbol that completion-popup will consume, add an `--epic-symbol` entry to the
`_lint-symvision` recipe in the `Justfile`, keyed to the **later** phase
`sase-17x.9(<symbol>)`. Never key an entry to `sase-17x.5`. Quote the argument for the
shell as needed. Make purely internal helpers `_private` instead. Afterwards,
`sase bead epic-symbols sase-17x.5` must print no entries.

### B4. Docs and pin

- `docs/rust_backend.md`: after the Architecture module table, add a short "Command Line
  grammar handle" subsection. It covers:
  - the frozen `CommandLineGrammar` pyclass, and that it is built once from the cached
    `sase completion spec -d -j` file off the event loop
  - `resolve` / `complete` / `command_help`
  - the char-offset rule
  - the `SCHEMA_VERSION` mirror
  - that sase-core owns the fixture regeneration step
- Pin: run `just ratchet-core-revision` so `sase-core-revision.txt` moves to sase-core's
  current remote HEAD (at least `f405c44`, which the proc-retention phase could not
  pin).
  - An agent cannot commit, so the new sase-core commit from Part A does not exist when
    this turn ends. Its pin move therefore must happen after the host lands it.
  - Record that with
    `sase bead note sase-17x.5 'PROPOSED FOLLOW-UP: move sase-core-revision.txt past the sase-core CommandLineGrammar commit — sase now calls require_rust_binding("CommandLineGrammar"); until the pin moves, CI "Check pinned core bindings" and the contract test fail on the pinned core'`.
  - Do not create beads.

### B5. Verify sase

Read the `lint_and_test` reference memory
(`sase memory read lint_and_test.md -r "<why>"`) and follow it:

1. Run `just install` (it rebuilds `sase_core_rs` from the linked checkout).
2. Run `just fix`.
3. Run `sase tool run check`, with a tool timeout of at least 15 minutes, until it is
   green.
4. Run the new test file alone:
   `just test tests/completion/test_command_line_grammar.py`.

### B6. Close-out

1. `sase bead epic-symbols sase-17x.5` shows nothing.
2. Record any other discovered work as `PROPOSED FOLLOW-UP:` notes.
3. Close only this bead:
   `sase bead close sase-17x.5 --note "<what was verified: sase-core check green incl. goldens + perf numbers; sase check green; contract test; pin state>"`.
4. Do not close the epic or any ancestor.
5. Finish with `/sase_final`.

## Out of scope

- Any TUI code (the popup, signature widget, sources and grammar loading at idle are
  completion-popup's).
- Changing the spec builder, run-policy tables or value kinds (earlier phases own them).
- Option `nargs` other than one value. The spec does not carry option `nargs`; treat
  every value-taking option as taking one value, and note this in the module doc.
