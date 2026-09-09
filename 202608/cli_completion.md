---
tier: epic
status: done
title: Excellent shell completion for the sase CLI
goal: "Typing `sase <TAB>` anywhere in the command tree offers the right commands,
  options, static choices, and live values — with descriptions, grouped listings, and no
  perceptible latency — in zsh, bash, and fish, from a grammar that cannot drift from
  the argparse tree.

  "
phases:
  - id: spec
    title: Completion spec model and argparse walker
    depends_on: []
    size: medium
    description: "spec: build the sase-owned CompletionSpec model, the argparse tree
      walker that produces it, help shortening, alias/hidden policy, the value-kind
      resolver, and the checked-in structural snapshot that makes grammar drift a
      reviewable diff.

      "
  - id: zsh
    title: Zsh emitter and the sase completion command group
    depends_on:
      - spec
    size: medium
    description: "zsh: emit a native `_arguments` compsys script from the spec with
      descriptions, exclusion lists, mutex groups, `-S`, and remainders, and register
      the `sase completion` command group with its `list`, `spec`, and `zsh` children.

      "
  - id: shells
    title: Bash and fish emitters
    depends_on:
      - zsh
    size: medium
    description: "shells: add bash and fish emitters over the same spec plus their `sase
      completion bash` and `sase completion fish` children, so every supported shell
      shares one grammar, one alias policy, and one description pass.

      "
  - id: candidates
    title: Pre-argparse candidates fast path
    depends_on:
      - zsh
    size: medium
    description: "candidates: add the `sase completion candidates` fast path ahead of
      argparse in entry.py, its value/description wire format, its disk cache, the
      project and bead providers, and the import-set and latency contract tests that
      keep it fast.

      "
  - id: kinds
    title: Value-kind provider catalog
    depends_on:
      - candidates
    size: medium
    description: "kinds: fill out the remaining value-kind providers behind the existing
      seam, routing each through sase_core_rs or an equally cheap catalog and rendering
      project display names rather than ProjectSpec keys.

      "
  - id: wire
    title: Dynamic values wired into every shell
    depends_on:
      - shells
      - kinds
    size: medium
    description: "wire: teach all three emitters to call the candidates fast path for
      kinded slots, with a hand-written zsh preamble that groups and tags results and
      caches them in-shell, plus the bash and fish equivalents.

      "
  - id: install
    title: Install, verification, doctor, and refresh
    depends_on:
      - shells
    size: medium
    description: "install: add `sase completion install` that picks a directory compinit
      actually scans, zcompiles, verifies registration, and stamps the install, plus the
      doctor check and the flagged `sase update` refresh hook.

      "
  - id: docs
    title: Documentation, polish, and reach
    depends_on:
      - wire
      - install
    size: small
    description:
      "docs: document the feature end to end, ship the recommended zstyle snippet, close
      the remaining polish items, and record what was deliberately deferred."
proposed_by: bbugyi200.athena.04p
bead_id: sase-oc
create_time: 2026-09-09 19:50:07
---

- **PROMPT:**
  [prompts/202608/cli_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/cli_completion.md)
- **BEAD:**
  [sase-oc](https://github.com/sase-org/sase--beads/blob/main/pages/sase-oc/README.md)

# Plan: Excellent shell completion for the sase CLI

## Background

`sase` has no shell completion today. The research report
`202608/cli_tab_completion/cli_tab_completion.md` in the `research` sidecar established
the constraints with measurements that this plan takes as given:

- Building the full argparse tree costs **250–400 ms**, and a cold `sase` process costs
  **300–640 ms**. Any design that runs `sase` to answer a keystroke is unusable.
- The tree is **331 parsers / 809 options / 140 positionals / depth 4**, edited weekly.
  Any hand-maintained grammar rots.
- The grammar is **static**: `_COMMAND_REGISTRARS` is a literal dict, plugins contribute
  no subparsers, and every `choices=` is a module-level constant. Only _values_ need to
  be live.
- zsh invokes the completion system **on every keystroke** when `zsh-autosuggestions`
  runs its `completion` strategy, so any completer that shells out must be cached
  in-shell.
- A generated zsh script answers in **0.4–12 ms**, but only when `zcompile`d; parsing an
  uncompiled 300 KB script costs **79–84 ms** on the first TAB of every shell.
- Writing a script into a directory that is added to `fpath` _after_ `compinit` has run
  is a **silent no-op**. File presence is not evidence of a working install.

The shape that follows is the one the industry converged on: a static grammar living in
the shell, plus a narrow dynamic protocol for live values.

## Design

### One spec, three emitters

Everything derives from a single sase-owned intermediate representation,
`CompletionSpec`, produced by one walk of the argparse tree:

```
create_parser(only=None)
        │
        ▼
  build_spec()          shorten helps · resolve aliases · resolve value kinds
        │                hide SUPPRESS · record mutex groups · record remainders
        ▼
  CompletionSpec  ──────┬──► emit_zsh   ──► _sase          (compsys `_arguments`)
   (JSON-serializable)  ├──► emit_bash  ──► sase           (`complete -F`)
                        ├──► emit_fish  ──► sase.fish      (`complete -c`)
                        └──► snapshot   ──► drift gate + `sase completion spec`
```

**This plan owns all three emitters rather than delegating to `shtab`.** The research
report recommended `shtab` as the emitter and that recommendation is sound for a naive
first cut, but it does not survive the quality bar stated in the request:

- `shtab` emits no exclusion lists, no `_arguments -S`, and ignores all 15 mutually
  exclusive groups — the majority of what separates "words appear" from "feels like
  `git`". Injecting those into generated zsh text afterwards is string surgery on
  generated code.
- `shtab` duplicates the four legacy top-level aliases (`changespec`, `task`, `vcs`,
  `artifact-file`) as full second subtrees.
- `shtab` has **no mechanism for dynamic values at all**: `shtab.cmd()` emits
  `($(command))`, which is bare words, uncached, and re-run on every keystroke. Using it
  would mean bash and fish never get live values, or a second post-processing pass.
- The research report itself predicts replacing the zsh emitter in its Phase 3.

Owning the emitters costs one extra phase up front, removes a runtime dependency,
removes a predicted rewrite, and gives all three shells the same alias policy,
description pass, and dynamic-value protocol. It also makes the emitters pure functions
of a JSON-shaped spec, which is far easier to test than a parser walk.

**Fallback, if the zsh emitter proves harder than budgeted:** add `shtab` as a runtime
dependency and use it for bash and fish only, keeping the sase-owned zsh emitter. The
spec boundary makes that swap contained. Do not adopt `shtab` for zsh.

### What lives where

```
src/sase/completion/
  model.py              CompletionSpec dataclasses + to_json/from_json
  kinds.py              ValueKind + the metavar/dest -> kind table + path overrides
  shorten.py            help text -> short, column-friendly description
  build.py              argparse tree -> CompletionSpec
  emit_zsh.py           spec -> compsys script body
  emit_zsh_preamble.py  hand-written zsh helper functions (literal text)
  emit_bash.py          spec -> bash completion function
  emit_fish.py          spec -> fish complete directives
  install.py            target resolution, write, zcompile, verify, stamp
  candidates/
    protocol.py         value/description wire format, prefix filter, limits
    cache.py            ~/.sase/completion/cache/<kind>.tsv + invalidation
    providers.py        kind -> provider dispatch, lazily imported
src/sase/main/parser_completion.py      the `sase completion` parser
src/sase/main/completion_handler.py     the `sase completion` handler
src/sase/main/completion_fast_path.py   the pre-argparse door
src/sase/doctor/checks_completion.py    the doctor check
tests/completion/…                      one test module per source module
tests/completion/snapshots/cli_spec.json  the checked-in drift snapshot
```

Keep every source file under the 700-line `toobig` soft limit; split before growing past
it rather than after. Every phase must leave `just check` green, and symvision requires
that nothing added is unreachable — wire each new symbol to a caller in the same phase.

### CLI surface

```
sase completion [list]                     # default child; what exists and what is installed
sase completion zsh    [-o/--output FILE]
sase completion bash   [-o/--output FILE]
sase completion fish   [-o/--output FILE]
sase completion spec   [-j/--json] [-o/--output FILE]
sase completion install [SHELL] [-d/--dry-run] [-f/--force] [-t/--target DIR]
sase completion candidates <KIND> [PREFIX] [-l/--limit N] [-p/--project NAME]
```

Per `cli_rules.md`: the group has an exact `list` child, so a bare `sase completion`
delegates to it through `_default_list_subcommands()` — do not re-implement that. `KIND`
is required for `candidates` to execute, so it is a positional, never an option; `SHELL`
for `install` is optional (auto-detected) and so may stay positional too. Every long
option gets a short alias. Subcommands and options stay alphabetically sorted, and each
subparser carries a `description` and an `epilog` with worked examples, matching
`parser_flag.py`.

### The two contracts that keep this honest

**Drift contract.** `tests/completion/snapshots/cli_spec.json` holds the _structural_
spec: for every command path, its aliases, hidden flag, option strings, per-option value
kind, choices, repeatability, mutex group id, and per-positional metavar/nargs/kind.
Descriptions are excluded — they churn and are already covered by the existing `--help`
tests — but each command carries a digest of its descriptions so a wording change is
still a visible one-line diff. A test regenerates the spec from the live parser and
compares. `just sync-completion-spec` rewrites it, mirroring
`sync-feature-flags-schema`.

This is deliberately a pytest snapshot rather than a new `_lint-*` recipe: the test
transitively imports `create_parser()`, so `tools/select_tests`'s import-graph closure
already selects it whenever any `parser_*.py` module changes, and `just check` stays
fast. A new command that silently completes as a bare file becomes a `"kind": null` line
in the diff.

**Latency contract.** The `candidates` fast path is guarded by two tests. The durable
one asserts the **import set**: after the fast path runs, `sys.modules` must contain no
`sase.main.parser`, no `sase.ace.*`, no `textual`, and no `rich`. The soft one asserts
wall-clock under 40 ms for a warm process with a generous CI multiplier. Timing tests
are flaky; the import assertion is the one that actually prevents a future agent from
reaching for `sase agent list` (6.4–6.9 s) or `sase file list` (157 ms, and it drags ACE
widgets into the process).

### Non-goals

- **No `eval "$(sase completion zsh)"` anywhere.** That would add 300–640 ms to every
  new shell. Print to a file; the docs say so explicitly.
- **No path completion through `candidates`.** File and directory slots use the shell's
  native `_files` / `-f` / `-F`. Never `sase file list`.
- **No `agent` value kind** unless a catalog lookup measures inside the 40 ms budget.
  Ship the kind or do not ship it; do not ship a slow one.
- **No carapace, Elvish, Nushell, or PowerShell** in this epic. The spec makes a
  carapace emitter cheap later; SASE targets POSIX hosts today.
- **No memory-file edits.** If `cli_rules.md` should gain a completion-annotation rule,
  the land agent raises that with the project owner; agents must not edit
  `sase/memory/*.md` without explicit user permission.
- **No hand-editing of `~/.zshrc`.** Diagnose and print the exact line; never rewrite.

## Completion spec model and argparse walker

Build the library that every later phase consumes. No CLI in this phase.

**`model.py`** — frozen dataclasses, JSON round-trippable:

- `CompletionSpec`: `prog`, `version`, `root: CommandSpec`.
- `CommandSpec`: `name`, `path: tuple[str, ...]`, `aliases: tuple[str, ...]`, `hidden`,
  `summary`, `options: tuple[OptionSpec, ...]`,
  `positionals: tuple[PositionalSpec, ...]`, `subcommands: tuple[CommandSpec, ...]`,
  `default_child: str | None`, `mutex_groups: tuple[tuple[str, ...], ...]` (each a tuple
  of option dests).
- `OptionSpec`: `strings` (all forms, e.g. `("-j", "--json")`), `dest`, `summary`,
  `takes_value`, `repeatable`, `choices`, `kind: ValueKind | None`, `hidden`.
- `PositionalSpec`: `metavar`, `dest`, `summary`, `nargs`, `choices`,
  `kind: ValueKind | None`, `is_remainder`.

Provide `to_json()` / `from_json()` and a `structural_digest()` used by the snapshot.

**`build.py`** — `build_spec(parser=None) -> CompletionSpec`. Walk from
`create_parser(only=None)`; never mutate the live parser (deep-copy or read only).
Rules:

- **Aliases collapse.** argparse `aliases=` puts several names in `action.choices`
  pointing at one parser object. Group by `id(parser)`, keep the name that appears in
  `_choices_actions` as `name`, and record the rest as `aliases`. Emit the subtree once.
  Aliases stay completable but are not listed. This covers `patch`/`changespec`,
  `proc`/`task`, `stitch`/`vcs`, and `artifact`/`artifact-file` at the top level, and
  any nested aliases the walk finds.
- **Hidden means hidden.** Skip actions with `help=argparse.SUPPRESS` (17 today) and
  parsers whose `_choices_actions` entry is suppressed. `sase editor helper-bridge` must
  not appear. Assert its absence in a test.
- **Mutex groups.** Read `parser._mutually_exclusive_groups` and record each as a tuple
  of member dests on the owning `CommandSpec`.
- **Remainders.** Mark `nargs=argparse.REMAINDER` and `nargs="..."` positionals with
  `is_remainder=True` so emitters can stop offering options after `--`
  (`sase proc run -- CMD`).
- **Default `list` child.** Record `default_child="list"` where
  `_default_list_subcommands()` applied. `sase bead <TAB>` must offer the full child
  list, not flatten into `list`'s flags.
- **The root's custom help actions.** `-h/--help` and `-H/--full-help` are custom
  `argparse.Action` subclasses; treat them as valueless options, and include the
  auto-`--help` of every subparser.

**`shorten.py`** — `short_summary(help_text, limit=60) -> str`. zsh renders descriptions
in a column beside the candidate and the tree currently contains a **271-character**
help string, with a quarter to a third of all helps over 60 chars. Policy:

1. Take the first sentence (split on `. ` / `.\n`, respecting common abbreviations).
2. Collapse whitespace; strip a trailing period.
3. If still over `limit`, truncate at the last word boundary at or before `limit - 1`
   and append `…`. **Never truncate mid-word.**
4. Strip backticks and newlines; escape nothing here — escaping is each emitter's job.

Add an optional explicit override: `set_completion_summary(action, text)` stores a short
summary on the action, and `build.py` prefers it. Use it only where derivation reads
badly. Do not sweep the parser modules in this phase.

**`kinds.py`** — `ValueKind` string enum plus resolution. Kinds: `project`, `bead`,
`repo`, `workspace`, `plugin`, `flag`, `plan`, `patch`, `artifact`, `xprompt`, `skill`,
`memory`, `proc`, `monitor`, `agent`, `model`, `tag`, `path`, `dir`.

Resolution order for an action, first match wins:

1. An explicit per-action override recorded by `set_completion_kind(action, kind)`.
2. A `(command_path, dest)` entry in the `PATH_OVERRIDES` mapping.
3. A `dest`/`metavar` entry in the `NAME_TABLE` mapping (`project` → `project`,
   `bead_id` → `bead`, `workspace_num` → `workspace`, `PLAN_FILE` → `path`, …).
4. `None`.

The measured metavar frequencies make this tractable: `name` (17), `id` (17), `project`
(13), `query` (5), `reference`/`refs` (4 each), and among options `--json` (84),
`--project` (33), `--format` (23), `--status` (12), `--kind` (11), `--agent` (9). Ten to
fifteen kinds cover the great majority of the surface. Ambiguous bare names such as
`name` and `id` **must** be resolved through `PATH_OVERRIDES`, never guessed from the
name alone — a wrong kind is worse than no kind.

Actions with a static `choices=` (94 today) need no kind; the choices _are_ the
candidates.

**Snapshot and regeneration.** Write `tests/completion/snapshots/cli_spec.json`, the
test that compares it against a fresh walk, and a `just sync-completion-spec` recipe.
The failure message must name the regeneration command.

**Tests.** Round-trip `to_json`/`from_json`; alias collapsing (assert `changespec` is an
alias of `patch` and appears once); `helper-bridge` absent; mutex groups found (expect
15); shortening cases including the 271-char string, an abbreviation, and an exact-limit
boundary; kind resolution precedence; and the snapshot gate.

## Zsh emitter and the sase completion command group

The craft phase. Target the compsys experience of `git` and `gh`.

**`emit_zsh.py`** — `emit_zsh(spec) -> str`. Structure the output the way well-written
compsys scripts are structured: a `#compdef sase` header, the preamble, one function per
command node, and a `_arguments -C -s -S` state machine dispatching on `$state`/`$line`.

Requirements, each of which is a stated gap in the alternatives considered:

- `_arguments -C -s -S`: `-s` for option stacking and `--opt=value`, `-S` so `--` stops
  option completion, `-C` for the `curcontext` the cache helpers need.
- **Descriptions on options**: `'(-j --json)'{-j,--json}'[Emit machine-readable JSON]'`.
- **Exclusion lists**: every option carries its own alternate forms in its exclusion
  list, so an already-typed `-j` is not offered again. Options in a mutex group carry
  every sibling's forms as well.
- **Repeatable options** (`action="append"`) get the `*` prefix and no exclusion list.
- **Static choices** emit as `:metavar:(a b c)`.
- **Kinded slots** emit as `:metavar:->kind` today and become
  `:metavar:_sase_candidates kind` in the `wire` phase. Emitting the placeholder now
  keeps that phase a small, local edit.
- **`path` / `dir` kinds** emit `_files` and `_files -/`. Never a subprocess.
- **Remainders** emit `'*::command:_normal'` after the remainder position.
- **Subcommand listings** use `_describe -t commands 'sase commands' <array>` so
  descriptions and grouping work; a group with a `default_child` still lists all
  children.
- **Aliases** are completable but excluded from the `_describe` array.
- **Quoting.** Single-quote every description; escape embedded `'`, `[`, `]`, and `:`.
  Command names may contain `+` (`sase bead +1`) and `-`; both must survive. Test both.

Split the literal helper functions into `emit_zsh_preamble.py` so `emit_zsh.py` stays
well under the size limit. In this phase the preamble holds only `_sase_run` — a helper
that resolves the `sase` on `PATH`, never a numbered workspace's `.venv/bin/sase`, which
may vanish. `_sase_candidates` arrives in the `wire` phase.

**The command group.** Add `src/sase/main/parser_completion.py` with `list`, `spec`, and
`zsh`; register `"completion"` in `_COMMAND_REGISTRARS` and in
`parser_full_registrars.py`; add the dispatch branch in `entry.py` alphabetically. Add
`src/sase/main/completion_handler.py`.

- `sase completion list` renders a colored table of shells with a generator and their
  status. The `install` phase extends it with installed path, `.zwc` freshness, and
  stamp version; leave the columns easy to extend.
- `sase completion spec` prints the structural JSON — the same artifact the snapshot
  gate uses, and a stable integration point for other tools.
- `sase completion zsh` prints the script to stdout, or to `-o/--output`.

**Tests.** Emitter unit tests over small hand-built specs (a group with a default child,
a mutex pair, a repeatable option, a remainder, a `+`-prefixed command, a description
containing `'` and `[`). Plus assertions on the real emitted script: `#compdef sase`
present, `_arguments -C -s -S` present, `helper-bridge` absent, `changespec` present as
a completable word but absent from the commands `_describe` array, no description longer
than 60 characters.

**Real-shell smoke test.** Add a `zsh`/`zpty` harness, skipped when `zsh` is not on
`PATH`: write the script to a temp `fpath` directory, run `compinit -u` against a
throwaway dump, assert `_comps[sase]` resolves, then drive a pty through
`sase bead +<TAB>` and assert the line becomes `sase bead +1`. Keep it to a handful of
probes; it is a smoke test, not the main suite. Mark it so it does not run in the fast
lane if it proves slow.

## Bash and fish emitters

Same spec, same alias policy, same shortened descriptions; the bar here is _good_, not
compsys-grade.

**`emit_bash.py`** — one `_sase()` function registered with
`complete -o default -F _sase sase`. Walk `COMP_WORDS` to locate the current command
node, then offer that node's subcommands, options, or choices via `compgen -W`. Aliases
are matched when walking but not offered. Fall back to `-o default` file completion for
`path`/`dir` slots and for unknown slots. Kinded slots get a placeholder call site that
the `wire` phase fills in.

**`emit_fish.py`** — a sequence of `complete -c sase` directives. Fish's
`-n '__fish_seen_subcommand_from …'` conditions get unwieldy at depth 4, so emit a
single `__sase_cmd` helper that echoes the resolved command path from
`commandline -opc`, and condition each directive on `__sase_cmd` equalling a path. Fish
shows descriptions natively, so pass `-d` on every command and option — this is where
fish looks best.

Add the `bash` and `fish` children to `parser_completion.py` alongside `zsh`, keeping
the subcommand list alphabetical.

**Tests.** Structural assertions per emitter (registration line present, a nested
command path reachable, choices present, aliases not offered, `helper-bridge` absent,
quoting of `+1` and of descriptions containing `'` and `$`). Add a `bash --norc` smoke
test that sources the script and calls the completion function directly with a synthetic
`COMP_WORDS`/`COMP_CWORD`, skipped when `bash` is absent — the research left bash and
fish latency unmeasured, so record the measured numbers in the phase notes.

## Pre-argparse candidates fast path

The door that makes live values possible without paying for argparse.

**`src/sase/main/completion_fast_path.py`** — `try_handle_completion_candidates(argv)`,
called from `main()` in `entry.py` **before** the `sase run` branch and before
`from .parser import …`, guarded exactly as:

```python
if sys.argv[1:3] == ["completion", "candidates"]:
    from .completion_fast_path import try_handle_completion_candidates
    exit_code = try_handle_completion_candidates(sys.argv[3:])
    if exit_code is not None:
        sys.exit(exit_code)
```

Mirror the existing bead fast path exactly: return `None` to fall through to argparse
for anything the fast path does not fully understand, including `-h`/`--help` and any
malformed argv. It parses its own arguments with a few `str` comparisons — no
`argparse`, no `sase.config`, no `rich`.

**`candidates/protocol.py`** — the wire format is `value\tdescription` lines on stdout,
one per candidate, description optional. This maps directly onto zsh `_describe`,
degrades to plain words in bash and fish, and matches both argcomplete's
`{value: description}` and Cobra's `__complete`. Escape or strip embedded tabs and
newlines in both fields. Apply the prefix filter and `-l/--limit` (default 200) here so
every provider gets identical semantics. Unknown kind: exit 0 with no output — a shell
must never see a traceback.

**`candidates/cache.py`** — a disk cache under `~/.sase/completion/cache/<kind>.tsv`
(`sase_subdir("completion")`), written atomically, invalidated by a per-kind
`(source mtime, ttl)` policy with a short default TTL. `SASE_COMPLETION_NO_CACHE=1`
bypasses it. This is the second of two cache layers; the in-shell layer arrives in the
`wire` phase and is the one that absorbs per-keystroke pressure.

**`candidates/providers.py`** — `candidates_for(kind, prefix, *, project, limit)`
dispatching to per-kind callables that are imported **inside** the callable, never at
module scope, so an unused provider costs nothing.

Ship two providers as proof of the seam:

- `project` — through the `list_project_records` Rust binding. **Render the configured
  `PROJECT_NAME`, never the ProjectSpec key**: project through
  `sase.project_display_names` or an already resolved `display_name`, falling back to
  the key only when no name is known. Description is the project's path or status.
- `bead` — through the `bead_list` Rust binding. Value is the bead id, description is
  the title. `sase bead show <TAB>` showing titles rather than bare ids is the single
  most visible win in this epic.

Going through `sase_core_rs` rather than re-reading `issues.jsonl` costs ~1.6 ms on top
of the ~20 ms interpreter floor and satisfies the `rust_core_backend_boundary` rule: any
frontend would need these values, so they are core.

Add the `candidates` child to `parser_completion.py` (positional `KIND`, optional
positional `PREFIX`, `-l/--limit`, `-p/--project`) so `--help` and error messages work
through the normal parser even though the happy path never reaches it.

**Tests.** The import-set contract described in the Design section, run as a subprocess
so it observes a real cold start. A wall-clock check under 40 ms with a CI multiplier.
Fall-through tests proving that `sase completion candidates --help`, an unknown kind,
and a malformed argv all reach argparse or exit cleanly, and — critically — that **no
other argv shape** takes the fast-path branch. Provider tests with a fixture store.

## Value-kind provider catalog

Fill out the catalog behind the seam the previous phase built. Each provider is small
and independent; the phase is a breadth pass, not a design pass.

Priority order, by metavar frequency: `repo`, `workspace`, `flag`, `plugin`, `plan`,
`patch`, `memory`, `xprompt`, `skill`, `proc`, `monitor`, `artifact`, `tag`.

Rules every provider follows:

- Route through `sase_core_rs` or an in-process code-owned registry. `flag` reads the
  feature-flag registry directly (it is code-owned and already in-process); `repo` uses
  the repo inventory; `memory` lists the memory notes; `xprompt`/`skill` use the
  existing catalogs — but only where they measure inside budget.
- **Measure before shipping.** Add each kind to the latency test's parametrization. A
  kind that cannot hold 40 ms does not ship; record why in the phase notes rather than
  shipping a slow one.
- Reuse `sase.xprompt.model_completion`, `vcs_*_completion`, and `ace/query/completion`
  **only** where they measure fast enough. They import the config and LLM registry
  layers, so treat them as a fallback, not the fast path. Never import
  `sase.ace.tui.widgets.*`.
- Project display names, never ProjectSpec keys, in every value and description.
- Every kind that ships must be reachable from at least one slot in `kinds.py`; a
  provider with no call site is a symvision failure and, more importantly, dead weight.

Extend `PATH_OVERRIDES` in `kinds.py` as each provider lands, and regenerate the spec
snapshot — the diff is the record of which slots gained live values.

Explicitly evaluate `agent` and `model` here: ship them only behind a cheap catalog.
`sase agent list` takes 6.4–6.9 s and must never be a completion source.

## Dynamic values wired into every shell

Turn the placeholders from the emitter phases into real calls, with the in-shell caching
that the per-keystroke reality makes mandatory.

**Zsh.** Extend `emit_zsh_preamble.py` with the hand-written helpers — this is the part
that is deliberately hand-written rather than generated, because it is where the craft
is:

```zsh
_sase_cache_policy() {
  local -a stamp
  stamp=( "$1"(Nms+${SASE_COMPLETION_CACHE_TTL:-60}) )
  (( $#stamp ))
}

_sase_candidates() {
  local kind=$1
  local -a lines
  local policy
  zstyle -s ":completion:${curcontext}:" cache-policy policy ||
    zstyle ":completion:${curcontext}:" cache-policy _sase_cache_policy
  if ! _retrieve_cache "sase-$kind"; then
    lines=( ${(f)"$(_sase_run completion candidates $kind 2>/dev/null)"} )
    _store_cache "sase-$kind" lines
  fi
  _describe -t "sase-$kind" "$kind" lines
}
```

Notes that matter:

- `_retrieve_cache`/`_store_cache` honor the user's own
  `zstyle ':completion:*' use-cache on`, so anyone who wants live data can opt out. With
  `zsh-autosuggestions`' `completion` strategy driving the engine per keystroke, typing
  `sase bead show a4f2` would otherwise spawn roughly fourteen subprocesses.
- Pass the full candidate list and let zsh filter, rather than passing the prefix — that
  is what makes one cached fetch serve a whole word.
- `-t "sase-$kind"` gives each kind its own tag, so `zstyle` group-name separates beads
  from files from options in the listing.

Change `emit_zsh.py` to emit `:metavar:_sase_candidates <kind>` for kinded slots.

**Bash.** Add a `__sase_candidates` helper that calls the fast path, caches into a shell
associative array keyed by kind with a timestamp, and feeds `compgen -W` with the value
column only (bash cannot show descriptions).

**Fish.** Emit `-a '(__sase_candidates <kind>)'` with a helper that emits
`value\tdescription`, which fish renders natively as a described completion. Fish caches
poorly across invocations; rely on the disk cache from the `candidates` phase and keep
the TTL short.

**Tests.** Emitter assertions that kinded slots produce the helper call in each shell.
Extend the zsh pty smoke test with one dynamic slot against a fixture store, asserting
both that candidates appear and that a second TAB does not re-invoke the fast path.
Regenerate the spec snapshot.

## Install, verification, doctor, and refresh

Writing a file is not installing. This phase is where reliability is won or lost.

**`install.py`** — `sase completion install [SHELL]`:

1. **Detect the shell** when `SHELL` is omitted, from `$SHELL` and the parent process
   name. Report what was detected; never guess silently.
2. **Choose a target.** In order: `-t/--target`, then `SASE_COMPLETION_DIR`, then the
   first _writable_ directory that the shell actually scans, then a conventional user
   directory. For zsh, obtain the scanned `fpath` by probing the user's real shell
   (`zsh -ic 'print -rl -- $fpath'`) with a bounded timeout, rather than assuming.
   Preferring an already-scanned writable directory means the common case needs no
   instructions at all. Conventional fallbacks: `~/.zfunc` (zsh),
   `~/.local/share/bash-completion/completions` (bash), `~/.config/fish/completions`
   (fish).
3. **Write** the script (`_sase`, `sase`, `sase.fish`), atomically.
4. **`zcompile`** the zsh script. This is mandatory, not an optimization: the first
   `sase<TAB>` in each new shell costs 79–84 ms parsing an uncompiled script versus 0.4
   ms from `_sase.zwc`.
5. **Verify registration.** For zsh, probe
   `zsh -ic 'print -r -- ${_comps[sase]:-UNSET}'` with a bounded timeout. If it reports
   `UNSET`, the install is broken even though the file exists — the documented
   `~/.zfunc` recipe is a silent no-op on any host that adds `fpath` entries after
   `compinit`. Report the failure and print the exact line to add **before** `compinit`:
   ```zsh
   fpath=(~/.zfunc $fpath)   # must appear BEFORE compinit
   ```
   Never edit the user's rc file. Diagnose precisely and hand over a copy-pasteable
   line.
6. **Stamp** `~/.sase/completion/stamp/<shell>.json` with the sase version, the spec
   structural digest, the target path, and the timestamp.
7. **Report** a colored summary of every step and its outcome, plus the recommended
   `zstyle` snippet for grouped, described, menu-selected listings.

`-d/--dry-run` prints the whole plan without touching the filesystem. `-f/--force`
overwrites a file sase did not write (detected via the stamp).

Extend `sase completion list` with the installed path, `.zwc` freshness, stamp version,
and a resolved status per shell.

**`src/sase/doctor/checks_completion.py`** — a non-blocking advisory check registered
alongside the other check groups: script present; `.zwc` present and newer than the
script; stamp version equal to the running version; and, as a deep check, `_comps[sase]`
resolving. File presence alone is explicitly not accepted as evidence.

**Refresh on update.** `sase update` delegates to `uv tool upgrade sase`, which is the
natural regeneration hook — the drift that actually reaches users is an installed script
going stale, and no repo-side gate can catch it. After a successful update, for each
shell with a stamp, regenerate, `zcompile`, and re-stamp; failures are reported, never
fatal.

Gate that hook behind a `beta` feature flag created with `sase flag new` (which also
files its removal bead), default off. Rewriting files on every user's machine during an
unrelated command is exactly the "early landed path" the flag rule exists for; the flag
lets the hook soak while the generator earns trust. The `sase completion` commands
themselves need no flag — they are new, opt-in, and change nothing pre-existing.

**Tests.** Target resolution across the ordered cases; a fake unwritable scanned dir
falling through to the conventional dir; dry-run touching nothing; the stamp round-trip;
`--force` semantics; the doctor check in each state; the update hook both flag states.
Real-`zsh` tests skip cleanly when `zsh` is absent.

## Documentation, polish, and reach

**Docs.** Add `docs/completion.md` under **Integrations** in `mkdocs.yml`
(`strict: true` — a nav entry without a page fails the build). Cover: quick start per
shell, what `install` does and why the `fpath`-before-`compinit` ordering matters, the
recommended `zstyle` snippet, the value kinds and how caching behaves,
`SASE_COMPLETION_*` environment variables, and troubleshooting keyed on `sase doctor`.
Add the `sase completion` rows to `docs/cli.md`, and a short pointer in
`docs/getting_started.md` next to the install steps. Do not hand-edit `CHANGELOG.md` —
release-please owns it, and `_lint-changelog` enforces that.

**Polish.** Close the remaining items:

- `sase run` bypasses argparse for prompts containing spaces. Complete its first
  argument as `_files` plus the stored xprompt names; leave `#xprompt`, `%directive`,
  and `@ref` completion documented as deferred.
- Sweep the worst derived descriptions and give them explicit
  `set_completion_summary(...)` overrides — the ones where first-sentence truncation
  reads badly. `sase agent-cli`'s paragraph-length command help is the known worst case.
- Confirm the emitted zsh contains no description over 60 characters and no unquoted
  metacharacter, as a test rather than a review note.
- Record measured bash and fish end-to-end TAB latency in the docs; the research left
  both unmeasured.

**Deferred, recorded rather than done.** Emitting a `carapace-spec` from the same model
for nushell/elvish/powershell; chezmoi deployment via
`src/sase/main/_init_chezmoi_deploy.py` on SASE-managed machines; and `#`/`%`/`@`
completion inside `sase run`. Each is cheap once the spec exists, and none belongs in
the critical path. File them per the discovered-work rule.

**Land criteria for the epic.** `just check-full` green through `/sase_monitor`; the
spec snapshot regenerated and reviewed; `sase completion install` verified end to end on
zsh including the `_comps[sase]` probe; and `sase bead show <TAB>` offering ids with
titles, grouped, in a shell that has `zsh-autosuggestions` active.
