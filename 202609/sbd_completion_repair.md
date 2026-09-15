---
tier: tale
title: Repair zsh completion for the sbd alias
goal:
  Make the managed zsh completion load reliably and complete sbd subcommands, options,
  and partially typed bead IDs.
size: medium
proposed_by: bbugyi200.athena.0l3.f0
create_time: 2026-09-15 10:33:42
status: wip
---

# Repair zsh completion for the sbd alias

## Outcome and scope

Make `alias sbd='sase -p bead'` complete subcommands, options, and partially typed bead
IDs in a fresh shell using Bryan's actual chezmoi configuration. Repair two
independently reproduced failures, and add tests that would have caught both.

This is a medium tale: one coding agent can implement the bounded shell-helper,
dotfiles, generated-script, documentation, and regression-test changes together. These
are shell integration and presentation changes; no shared domain behavior or Rust core
change is needed.

The user's referenced command output was absent from the planning request. The diagnosis
below comes from direct inspection and real zsh reproductions, rather than an assumption
about the missing output. If that output arrives, explicitly relate it to these
findings.

## Confirmed diagnosis

### 1. An older completion file shadows the managed file

On the inspected host, a fresh interactive zsh reported:

- `aliases[sbd]` is `sase -p bead`; there is no remaining `sbd` function.
- `completealiases` is off, so ordinary alias expansion is available to completion.
- `_comps[sase]` is `_sase`, but loading that function resolves to
  `~/.oh-my-zsh/custom/completions/_sase`.
- That loaded function lacks `--print-command`. The managed `~/.zfunc/_sase` contains
  it, and its compiled `.zwc` is fresh.
- `sase doctor -D -C completion.registration` correctly warns that the oh-my-zsh copy
  precedes the stamped `~/.zfunc/_sase`.

Chezmoi's `home/dot_zshrc` prepends `~/.zfunc` before sourcing oh-my-zsh, but the
framework then puts its own directories ahead of it. The earlier chezmoi test only
checks source-line order. Being present before `compinit` establishes discovery; it does
not establish which same-named file wins when zsh later autoloads `_sase`.

Real PTY tests using the live startup configuration established:

| Input followed by Tab          | Current startup   | Managed directory given priority in the test shell |
| ------------------------------ | ----------------- | -------------------------------------------------- |
| `sase bead sh`                 | `sase bead show ` | `sase bead show `                                  |
| `sase -p bead sh`              | unchanged         | `sase -p bead show `                               |
| `sase --print-command bead sh` | unchanged         | `sase --print-command bead show `                  |
| `sbd sh`                       | unchanged         | `sbd show `                                        |
| `sbd show --for`               | unchanged         | `sbd show --format `                               |

Only process-local state was changed for these comparisons; no live configuration or
repository source was edited.

### 2. The dynamic helpers reset options required by zsh completion

Even with the managed script selected, `sbd show zzz<Tab>` and `sbd +1 zzz<Tab>` fail
against a fixture candidate named `zzz-completion-fixture`. An empty `sbd +1 <Tab>`
succeeds. The same prefix failure occurs with `sase bead show zzz<Tab>`, so it is
independent of the alias and `-p`.

Tracing the managed script in a clean zsh showed that `__sase_candidates` receives
`PREFIX=zzz`, fetches the expected candidate, and passes
`zzz-completion-fixture:Fixture` to `_describe`, which returns 1 without inserting it.
The helpers use `emulate -L zsh`, resetting `extendedglob` before calling
completion-system functions. Zsh's `_main_complete` establishes `_comp_setup`, which
includes `extendedglob`; `_describe` depends on its extended patterns while extracting
and filtering described candidates.

Two in-memory experiments independently restored prefix completion: enabling
`extendedglob` after the reset, and removing the option resets. Neither required
changing candidate data, root parsing, or `-p` output. Use the design below to preserve
the full completion environment rather than restore one option piecemeal.

A final process-local proof removed emulation from exactly the four helpers named below,
retaining it in the subprocess resolver and fragment parser. All four direct and aliased
typed-prefix cases passed, and the caller's `extendedglob` option remained off after
completion returned.

The prior coding transcript explicitly weakened its dynamic alias test from a typed
prefix to an empty slot. Keep the empty-slot coverage and add the missing prefix
coverage; do not weaken a failing behavior assertion again.

### What the status warning does and does not establish

`sase completion list --json` also reports differing script/digest content and
`legacy chezmoi-managed install is not refreshed automatically`. Regenerating the
managed script alone cannot fix a different file winning on `fpath`. Furthermore,
`src/sase/completion/install_status.py` currently always adds the legacy-owner reason
for a chezmoi stamp, even if script content matches. Report content drift, registration,
and that ownership advisory separately. Do not promise an entirely green install report
merely because this alias works.

## Implementation

### 1. Preserve the zsh completion environment

In `src/sase/completion/emit_zsh_preamble.py`, remove the blanket `emulate -L zsh` reset
from helpers that call completion-system functions:

- `__sase_candidate_lines`, which calls `_retrieve_cache` and `_store_cache`;
- `__sase_candidates`, which calls the candidate loader and `_describe`;
- `__sase_run_prompt_embedded`, which calls the loader and `compadd`;
- `__sase_run_prompt`, which calls embedded completion and `_alternative`.

These helpers should inherit the environment established by zsh's completion entry
point. Retain local variables and the existing candidate/reply protocol. Add a short
comment explaining why resetting shell options around compsys calls is incorrect. The
pure subprocess resolver `__sase_run` and pure fragment parser can retain their local
emulation; they return before their callers invoke compsys. Do not turn on shell options
globally or change the user's completion styles.

Keep candidate fetching, shell quoting, cache keys, descriptions, helper names, and the
narrow subprocess command unchanged. Generate scripts from the emitter; never hand-edit
the generated helper text as the source of the repair.

### 2. Make the managed directory authoritative after framework startup

Use `/sase_repo` to open `chezmoi` with a specific repair reason and read its current
instructions. All paths in this step are relative to that returned checkout.

In `home/dot_zshrc`, preserve `~/.zfunc` visibility before the framework's first
`compinit`, then reassert its priority immediately after `source $ZSH/oh-my-zsh.sh`,
before aliases and the later explicit `compinit` block. Avoid accumulating duplicate
entries. Explain that the framework prepends its own directories, so the second step
controls lazy function lookup as well as discovery.

Keep `sbd` as the alias already shipped. The fresh-shell path must load the managed
function naturally. Do not add per-alias completion wrappers, evaluate generated Python
completion on startup, delete framework trees, or require routine clearing of the user's
completion cache. Leave other completion directories available in their relative order.
The older `_sase` can remain lower-priority; a destructive cleanup or a new
completion-ownership migration is unnecessary for this repair.

### 3. Add behavioral regression coverage

Extend `tests/completion/test_zsh_smoke.py` using its existing real-zsh PTY and
logged-candidate fixtures. Cover:

- `sbd sh` and `sbd show --for`, plus direct `sase -p` and `sase --print-command`
  equivalents;
- `sbd show zzz` and `sbd +1 zzz`, with a unique matching fixture bead ID, plus direct
  `sase bead show zzz` and `sase -p bead show zzz`;
- both the empty dynamic slot and a typed prefix;
- a nonmatching prefix, multiple matching candidates/common-prefix insertion, and
  descriptions where the existing harness supports them;
- cached and uncached candidate loading, proving repeated completion uses the existing
  cache contract and calls only `completion candidates bead`;
- existing run-prompt/embedded completion tests, since those helpers also call compsys,
  and no leakage of helper option changes into the interactive shell.

Assert the resulting ZLE command buffer, not just the presence of a candidate in a
terminal listing or the echoed setup commands. A small capture widget invoked after Tab
can expose `BUFFER` deterministically without executing the completed command. Use
unique output markers and handle redraws correctly. Never press Enter on a bead command.
A genuine completion failure must not be relabeled as harness timing.

Update `tests/completion/test_emit_zsh_preamble.py` as needed, with behavior tests
remaining the proof of correctness.

In chezmoi's `tests/bash/sase_completion_test.sh`, replace or supplement the
line-order-only check with an isolated zsh startup test using two `_sase` fixtures: a
managed current script and a framework copy without root `-p`. A lightweight framework
fixture must actually prepend its directory and run `compinit`; exercise the relevant
startup code from the real managed zsh configuration. Avoid merely copying the intended
fixed order into the test.

After startup, assert both `_comps[sase] == _sase` and that autoload resolves to the
managed script. Keep the conflicting copy present so the test proves precedence, not
absence of a competitor. Exercise startup with and without the competing file, and
preserve the existing apply/zcompile tests. Add the real Tab assertion here if the SASE
harness cannot cover the same configuration behavior without coupling the primary tests
to a developer's linked checkout.

### 4. Regenerate managed sources and explain troubleshooting

Once the helper and startup tests pass, call
`sase.completion.deploy_chezmoi.deploy_chezmoi_completion` from the modified SASE
environment, with the opened chezmoi checkout's `home` as `source_root` and
`no_apply=True`, `no_commit=True`, `no_push=True`. Review the generated scripts and
stamps together. Preserve the existing zcompile hook. Regeneration must include the
fixed helper body even when the argparse structural digest is unchanged.

Update `docs/completion.md` to explain:

- registration under `_sase` does not alone prove the correct file is loaded;
- `sase doctor -D -C completion.registration` detects conflicting paths;
- oh-my-zsh may reorder `fpath` after an earlier user prepend;
- static failures after `-p` can indicate an older loaded grammar, while successful
  empty-slot completion does not prove typed-prefix matching works;
- the managed source/deployment path and a fresh shell are required to pick up repaired
  files and previously loaded functions.

Keep the existing stderr-only print-command contract. Tab must emit neither a command
header nor a real bead invocation. Do not change the CLI parser or candidate backend to
compensate for these shell-side defects.

## Verification and rollout

1. Show that the new prefix and conflicting-directory tests fail on the pre-fix behavior
   and pass with their respective fixes. Use fixture candidates; do not mutate user
   beads or launch agents for test data.
2. Run the focused zsh smoke/preamble tests and relevant deployment, install,
   registration, and print-command regression tests. Check generated zsh syntax and use
   the existing bash/fish checks if regeneration changes their output.
3. In the opened chezmoi checkout, run `zsh -n home/dot_zshrc`, alias/script syntax
   checks, and `bashunit tests/bash/sase_completion_test.sh`. Review both diffs and run
   `git diff --check` in each repository.
4. Read `lint_and_test.md` via `/sase_memory_read` and run `just check` for SASE
   changes. Follow the current note's broader-check rules. Use `/sase_monitor` for a
   long check and always for `just check-full`; do not repeat long inline waits or
   silently substitute an old core wheel from the earlier transcript. Match verification
   to the current checkout's pinned dependency contract.
5. Submit both repositories through the host-owned finalizer. The linked chezmoi
   instructions require `chezmoi update -a --force` after its commit lands; ensure the
   authorized apply is part of the completion handoff. Use `/sase_monitor` if that
   operation requires a long-running continuation. Do not apply unreviewed source during
   implementation preparation.
6. Once the changed SASE version and chezmoi source have landed and been applied, open a
   fresh interactive zsh and verify the alias, loaded function source, subcommands, leaf
   options, and prefixed dynamic IDs. Re-run `sase doctor -D -C completion.registration`
   and inspect the zsh script's `.zwc` freshness. Registration must resolve to the
   stamped managed path. Compare script/digest drift against the actual deployed
   executable, and report any remaining legacy ownership advisory accurately.

If host-owned landing/application is still pending when the coding turn ends, report the
repair as prepared and verified, state the pending rollout explicitly, and include the
exact live checks in the handoff. Do not claim Bryan's shell has been updated based
solely on a passing isolated fixture.

## Acceptance

The repair is complete when the managed file wins despite the older framework copy, both
direct and aliased commands complete after `-p`, and fixture bead IDs complete from both
empty and partially typed arguments. Regression tests must exercise those distinctions,
generated sources must contain the helper repair, and the final report must distinguish
tested source changes from applied live behavior and any remaining ownership advisory.
