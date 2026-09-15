---
tier: tale
title: Copyable sase commands with native alias completion
goal:
  Add a reliable, attractive root print-command option and restore sbd as a completing
  alias in chezmoi.
size: medium
proposed_by: bbugyi200.athena.0l3
create_time: 2026-09-15 08:01:16
status: wip
---

# Plan: Print the command behind a SASE alias

## Outcome and scope

Add `sase -p` / `sase --print-command`, then replace the user's `sbd()` function with
`alias sbd='sase -p bead'` in the linked chezmoi repository. A copied terminal
transcript will identify the command and preserve its arguments, and zsh will complete
the alias through SASE's generated completion tree.

This is a medium tale: one coding agent can implement and verify the bounded CLI,
completion, documentation, and dotfiles changes together. The work is CLI argument
handling and terminal presentation; it belongs in Python under `src/sase/main/` and does
not introduce shared domain behavior requiring a Rust core change.

## Findings that determine the design

- Chezmoi's `home/dot_config/aliases.sh` currently contains
  `sbd() { echo "❯ sase bead $*" && sase bead "$@"; }`. The execution preserves argument
  boundaries, but the displayed `$*` does not. The user's zsh configuration sources this
  file after initializing completion.
- `src/sase/main/entry.py` consumes leading global options before bead dispatch,
  completion-candidate dispatch, `run` special cases, and normal argparse dispatch.
  Printing only after `parse_args()` would miss several actual execution paths.
- `src/sase/main/global_options.py` currently consumes root feature switches and removes
  them from `sys.argv`. The displayed command must be captured before those switches
  disappear, or it would omit behavior-changing arguments.
- The root parser and compact root help are separate surfaces. Completion derives its
  grammar from the actual parser, and `tests/completion/snapshots/cli_spec.json`
  enforces structural consistency.
- `-p` already belongs to subcommands, including `completion candidates --project` and
  `ace --profile`. The new option must be scoped to the leading root options.
- Chezmoi tracks generated scripts for bash, fish, and zsh, their ownership stamps, and
  a zcompile hook. Updating only the alias would leave its installed grammar stale. The
  normal `completion deploy-chezmoi` command also deploys and commits; use its existing
  Python API's source-only controls while preparing these changes.

## User-visible contract

### Invocation and output

```console
$ sbd show sase-example
❯ sase bead show sase-example
<normal command output>

$ sase --print-command bead search 'two words'
❯ sase bead search 'two words'
<normal command output>
```

The prompt marker is presentation. The text following it is the command a person can
copy and execute in bash or zsh.

1. `-p` and `--print-command` are equivalent boolean root options, defaulting to off.
   They appear before the first subcommand and can mix with the existing root
   `-f/--enable-feature` and `-F/--disable-feature` options in either order. Repeating
   the print option still produces exactly one command header.
2. Print the expanded invocation once to **stderr**, with a trailing newline and an
   immediate flush, before command output, delegation notices, or early handlers. The
   option requests printing even when stderr is redirected. Stdout remains the command's
   original data stream, including JSON and generated completion scripts.
   `sbd list -f json >beads.json` therefore keeps a valid JSON file; `2>&1` explicitly
   captures the header together with output when desired.
3. Begin with the stable executable name `sase`, regardless of the installed launcher
   path or a `python -m sase` entry. Remove only the root print option occurrences from
   the displayed invocation. Retain the other root options, all subcommand arguments,
   their order, and any `--` in the command tail. Never strip a matching string used as
   another option's value or as positional text.
4. Show the invocation supplied to SASE after shell expansion. Do not reconstruct shell
   redirections, environment assignments, original quote style, implicit `list` tokens,
   or expanded prompt/workflow internals. Those are not argv. A bare group still prints
   its existing delegation notice after the command header.
5. This option prints and then executes normally; it is not a dry run. Exit codes,
   command errors, stdin, signal handling, and command side effects retain their
   existing behavior. A syntactically accepted leading print option also reports an
   attempted invocation that subsequently fails or prints help. An invalid root option
   prefix may fail with the normal usage error before a header is available.
6. Keep print state local to one `main()` invocation. Do not store it in configuration,
   environment variables, or module state that could leak into child commands or a
   second `main()` call. Normal Tab completion runs its existing helper command without
   `-p`, so it must remain silent apart from candidate data.

### Readability and faithful quoting

Use the existing `❯` marker, a dim cyan marker and bold `sase` on a capable stderr
terminal, and neutral arguments. Add no box, timestamp, explanatory label, blank line,
automatic wrapping, or ellipsis. Let the terminal naturally wrap long commands without
altering the text.

Emit styling only when stderr is a TTY, `NO_COLOR` is absent, and `TERM` is not `dumb`,
matching the root help's policy. Redirected output must contain no ANSI styling. Use `>`
if the stream encoding cannot represent `❯`. Keep the rendering lightweight; importing
the full parser or Rich merely to print the header would regress fast paths.

Quote printable arguments using `shlex.quote` semantics, including empty strings,
spaces, apostrophes, dollar signs, backticks, glob characters, and shell punctuation.
For arguments containing control characters, use bash/zsh ANSI-C quoting (`$'…'`) with
escaped quotes, backslashes, and controls. For example, an embedded newline is displayed
as `\n` inside that quoting, keeping one physical header line while preserving the
actual argument on replay. Raw escape, carriage-return, tab, and other control
characters must not reach the terminal. Preserve printable Unicode; escape unencodable
characters instead of failing before the requested command. Document bash/zsh as the
replay syntax rather than claiming universal shell syntax.

The formatted command is display-only. Execute the original argument vector through the
existing handlers; never execute, evaluate, or reparse the rendered string.

## Implementation

### 1. Root parsing and early dispatch

- Extend the existing leading-option consumer in `src/sase/main/global_options.py`.
  Separate extraction from feature-flag application so it can supply both the remaining
  execution arguments and the display arguments with only the print switches removed.
  Preserve the original input before mutating `sys.argv`.
- Use one understanding of root-option arity. A feature option consumes its value even
  when that value resembles `-p`; scanning stops at a subcommand, unknown root token, or
  `--`. Do not search-and-delete matching tokens across the whole command. Preserve all
  existing feature-option forms, conflict diagnostics, and exit codes.
- Keep the consumer and argparse registration consistent for boolean short clusters that
  argparse accepts, such as `-pp` and `-pfKEY`; `-fp` remains feature key `p`. Reject an
  attached value such as `--print-command=yes`. Do not allow an accepted argparse
  spelling to silently bypass printing through a different entry path.
- Register the boolean option on the root parser in `src/sase/main/parser.py`, with help
  such as `Print the shell-quoted command to stderr before running it`. Keep parser
  construction free of output and argument mutation so help/completion generation is
  safe. Generalize the feature-only registration name if appropriate.
- Place the single header emission in `entry.py` before all early dispatch and
  feature-flag application diagnostics once prefix extraction succeeds. Continue
  dispatch using the consumed argv so parser narrowing and the existing fast paths still
  work. The no-print path must not import the new renderer.
- Update root-option awareness in `src/sase/main/parser_root_args.py`, particularly
  `root_command_index`, so direct parser use still normalizes `bead note` correctly with
  `-p`. Retain a lightweight approach to any shared parsing helper; do not build the
  complete command tree to locate a root command.
- Add a small CLI presentation module for quoting and writing the header. Keep
  extraction, rendering, and output independently testable without a live store.

### 2. Help, documentation, and completion grammar

- Update `_COMPACT_ROOT_USAGE` and `_COMPACT_GLOBAL_OPTIONS` in
  `src/sase/main/parser_root_help.py`, the full root help, the root-options section in
  `docs/configuration.md`, and the introduction in `docs/cli.md`. Sort listed global
  options alphabetically by their long names and give the new option a concise alias
  example. Preserve the curated command inventory.
- Add an alias example and refresh guidance in `docs/completion.md`. Explain leading
  placement, stderr/redirection behavior, omission of the print switch from the header,
  and argument quoting. Do not imply bash/fish automatically inherit zsh's
  alias-completion behavior; the requested alias is verified in zsh.
- Confirm the generated specification contains one boolean root option with both
  spellings and that the three emitters handle it as taking no value. Regenerate the
  checked-in snapshot with `just sync-completion-spec`; do not hand-edit generated
  grammar. Change emitters only if a behavior test identifies a necessary fix.

### 3. Convert the alias and refresh the linked source

Only after the CLI behavior and focused tests pass:

1. Use `/sase_repo` and `sase repo open chezmoi` with a specific reason. Read the
   returned checkout's instructions and use only its printed path. Repository paths in
   this section are relative to that checkout.
2. Replace the single `sbd()` definition in `home/dot_config/aliases.sh` with
   `alias sbd='sase -p bead'`. Other aliases do not need a behavior change for this
   request.
3. From the modified SASE source environment, call
   `sase.completion.deploy_chezmoi.deploy_chezmoi_completion` with `source_root` set to
   that checkout's `home` directory and `no_apply=True`, `no_commit=True`, and
   `no_push=True`. This prepares the existing three scripts and stamps together without
   prematurely publishing them. Do not use the default CLI deployment command or add new
   deployment options as part of this feature.
4. Review the generated diffs and retain the existing zcompile hook and ownership
   scheme. Verify the scripts being tested are generated from the changed CLI.
5. Use the host-owned completion workflow for both repositories. Coordinate rollout so
   the runnable SASE version supports `-p` before the alias is applied. Chezmoi's
   instructions require `chezmoi update -a --force` after its commit lands; include that
   required application in the completion handoff. Verify the resulting completion
   registration and current bytecode. If landing is pending, report the source changes
   as prepared and do not claim the live shell is updated.
6. Explain that a new shell loads the replacement alias and refreshed completion. Merely
   sourcing aliases into an old shell can leave old function state behind; use a fresh
   shell for the final smoke check.

## Verification and acceptance

Use parametrized behavior tests and existing fixtures; no new live agents, user bead
mutations, or interactive `run` launches are needed.

### Parsing and output

- Both option spellings, repetitions, boolean clusters, and both orderings with all
  existing `-f`/`-F` forms produce one header and preserve feature application.
- Subcommand `-p`, a positional `--print-command`, a root feature value resembling the
  print switch, and tokens after `--` retain their original meanings. Exercise
  `completion candidates -p PROJECT` and direct parser `bead note` normalization.
- Printable quoting round-trips to the exact original argv with `shlex.split`.
  Round-trip control-character cases through isolated bash and zsh processes using an
  argv-reporting stub, without executing SASE or interpolating raw user arguments into
  shell source. Include empty strings, whitespace, both quotes, backslashes, shell
  metacharacters, Unicode, newline, tab, carriage return, and escape.
- Test plain versus TTY styling, `NO_COLOR`, `TERM=dumb`, a narrow terminal, long
  arguments, encoding fallback, absence of raw controls, and an explicit flush.
- Stub handlers at the entry boundary to prove the header precedes stdout/stderr from
  normal dispatch, bead fast dispatch, completion fast dispatch, and `run` special
  cases. Cover a fast-path fallthrough so it cannot print twice, a bare group's
  delegation notice, help, and invalid command usage with the usual status.
- Verify stdout-only capture remains valid JSON/candidate/script data and combined
  capture begins with the header. A second `main()` call without `-p` prints no header.
  Existing behavior without the option remains covered by regression tests.

### Completion and rollout

- Extend `tests/main/test_global_options.py`, relevant parser/root-help tests, and
  completion build/snapshot tests. Use the existing fast-path import contracts to prove
  ordinary completion, and completion explicitly invoked with `-p`, do not acquire
  imports of the full parser, ACE, Textual, or Rich.
- Extend `tests/completion/test_zsh_smoke.py` with a real PTY case that initializes
  completion, defines `alias sbd='sase -p bead'`, and drives Tab. Verify subcommands, a
  leaf option, and a fixture dynamic bead ID. Record fixture helper calls to prove Tab
  neither runs a bead command nor prints a command header. Also cover the direct
  `sase -p` and `sase --print-command` forms.
- Check generated bash and fish recognition of the no-value root option using their
  existing harnesses; run shell syntax checks for available shells. Avoid claiming alias
  support in those shells beyond the behavior actually tested.
- Validate chezmoi's alias with `bash -n` and `zsh -n`, then source it in an isolated
  fresh shell with a stub `sase` to verify exact `-p bead` forwarding and exit-status
  preservation. Run `bashunit tests/bash/sase_completion_test.sh` against the changed
  generated source and its existing apply/zcompile fixtures.
- Read `lint_and_test.md` through `/sase_memory_read` before completing the coding turn
  and run `just check`. Run `just check-full` through `/sase_monitor` when the selection
  rules require broader checks and as the landing gate; do not launch the exhaustive
  suite inline. Inspect the final diffs in both repositories.

The feature is complete when the tests establish faithful command display on every entry
path, machine-readable stdout stays intact, the real zsh alias completes, and the alias
plus generated grammar are prepared together with an explicit live-apply status.
