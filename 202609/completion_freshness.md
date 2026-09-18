---
tier: epic
title: Keep installed shell completion fresh across SASE upgrades
goal:
  Bash, fish, and zsh completion follow the installed SASE command tree after upgrades,
  including screenshot, with the same behavior on machines with and without chezmoi and
  without rebuilding the parser on every completion.
phases:
  - id: runtime-loaders
    title: Cached grammar resolution and portable shell loaders
    depends_on: []
    size: medium
    description:
      "runtime-loaders: add a lightweight completion ensure fast path, runtime-aware
      grammar caches, and portable loaders for bash, fish, and zsh, preserving the
      existing raw script generators and candidate fast path."
  - id: install-update
    title: Install, refresh, update, and diagnostic integration
    depends_on:
      - runtime-loaders
    size: medium
    description:
      "install-update: make stamped installs use the loaders, refresh both local and
      chezmoi-owned installs, include successful update mode switches, and distinguish
      loader registration from cached grammar freshness in diagnostics."
  - id: chezmoi-integration
    title: Chezmoi migration and upgrade regression coverage
    depends_on:
      - runtime-loaders
      - install-update
    size: medium
    description:
      "chezmoi-integration: distribute portable loaders instead of frozen grammar and
      runtime stamps, migrate the linked chezmoi configuration, prove upgrade behavior
      through real shells and repeated applies, and document the lifecycle."
proposed_by: bbugyi200.apollo.0f
create_time: 2026-09-18 06:05:17
status: wip
---

# Keep installed shell completion fresh across SASE upgrades

## Diagnosis and evidence

The reported failure is reproducible on the inspected host. The generator supports
`screenshot`; the installed scripts do not. This is a distribution and refresh problem,
not a missing parser registration.

- `src/sase/main/parser_registry.py` registers `screenshot`, and
  `src/sase/completion/build.py` deliberately calls `create_parser(only=None)`. The
  structural snapshot in `tests/completion/snapshots/cli_spec.json` includes it.
- Running each of `sase completion bash`, `fish`, and `zsh` produces grammar that
  includes `screenshot`. The installed files at
  `~/.local/share/bash-completion/completions/sase`,
  `~/.config/fish/completions/sase.fish`, and `~/.zfunc/_sase` all lack it.
- `sase completion list --json` reports all three as `managed stale`, owned by
  `chezmoi`. Their stamped digest was `40701bc579c5d8c2`; the live generator digest was
  `2be82a643153abd6`. These digests are observations, not test constants.
- The stamp and generator both use the nominal version `0.17.1` despite different trees.
  `sase version` reports an editable development build. Version equality alone therefore
  cannot establish freshness.
- `sase completion refresh --dry-run --json` refuses every installed shell with "legacy
  chezmoi-managed ... is not refreshed automatically". This is an explicit branch in
  `src/sase/completion/install_refresh.py`, covered by tests that currently require
  managed installs to be skipped.
- `sase doctor -D -C completion.registration` passes. Zsh finds the stamped `_sase`, and
  its `.zwc` is fresh relative to the stale text. Registration and bytecode freshness do
  not prove that the grammar matches the installed CLI.
- `src/sase/completion/deploy_chezmoi.py` and the linked chezmoi repository distribute
  full generated scripts and version/digest stamps as static files. Its
  `run_onchange_after_zcompile_sase_completion.tmpl` recompiles those files but never
  regenerates their grammar. Reapplying those sources can undo a local refresh.
- Normal successful `sase update`, including its no-change case, already launches the
  updated executable with `completion refresh --json`. Preserve that correct
  fresh-process behavior. `update_handler_mode_switch.py` omits refresh, and external
  package upgrades cannot run this SASE-specific update hook.

The existing parser snapshot gate detects source-tree drift. It cannot keep copies in
users' home directories current. The 60-second dynamic candidate TTL also has no effect
on static commands and options.

## Chosen design and scope

Use a small, portable shell loader at each normal completion discovery path. Keep the
full generated grammar in a SASE-owned runtime cache. When a fresh shell loads
completion, the loader asks the selected SASE executable for the current cached grammar
and sources that file. A cache hit must avoid parser construction; a miss generates from
the live argparse tree. Subsequent static completions in that shell use the loaded
functions without additional SASE processes.

`sase update` still eagerly refreshes installed shells. The loader is the independent
fallback for `uv tool upgrade`, pip/pipx upgrades, reinstallations, and editable source
changes. No installer-specific post-install hook, daemon, network request, or chezmoi
executable is necessary to use or refresh completion.

The freshness guarantee is at the next completion load in a fresh shell, plus eager
on-disk refresh after `sase update`. A process cannot replace functions already loaded
in its parent shell: document opening a fresh shell after an upgrade. Live hot reload of
every existing shell on each keystroke is outside this change.

Keep `sase completion bash|fish|zsh [-o FILE]` as raw script export commands. Explicitly
hand-managed exports remain snapshots; the automatic guarantee applies to installed
loaders. Preserve `install`, `refresh`, `list`, and `deploy-chezmoi`, including their
existing public controls. This is a fix and an additive loader interface, not removal of
an existing command or a deprecation campaign.

Use an epic because the runtime protocol, existing installation/update contracts, and
the separately managed dotfiles need coordinated implementation and independent
regression coverage. Each phase is bounded direct implementation work. Dependencies are
intentionally sequential; land the completed combined implementation.

This work concerns shell frontend grammar, Python import resolution, and filesystem/
subprocess installation adapters. Keep it in `src/sase/completion` and the CLI glue; do
not rewrite shared domain behavior. The inspected `sase-core` repository has no
shell-install lifecycle API to extend. If implementation uncovers a genuinely shared
domain decision, use `sase repo open sase-core` and the required Rust binding boundary
instead of adding a competing Python domain implementation. No memory edits or new task
beads are part of this plan.

## Phase runtime-loaders

### Runtime cache and CLI contract

1. Add `sase completion ensure SHELL` as a pre-argparse fast path next to the existing
   candidates fast path, with normal parser/help registration and a thin fallback
   handler. On success its stdout is exactly one usable absolute grammar-file path;
   diagnostics go to stderr. Support optional loader-target directory and ownership
   metadata so a chezmoi-distributed loader can register itself on a fresh host. Follow
   `cli_rules.md`: sorted options/subcommands, short aliases for public long options,
   positional shell, clear help. It must work outside any SASE project.
2. Add a loader-emission interface, `sase completion loader SHELL`, with output and
   ownership controls as needed by install and deploy. Keep loader rendering separate
   from the raw grammar emitters and reuse it for both installation paths.
3. Store grammar and its manifest under a new subtree of the existing SASE completion
   state, honoring `SASE_HOME`. Keep it separate from the live-values cache. Partition
   by resolved runtime installation identity and shell so different installed SASE
   runtimes cannot consume one another's grammar.
4. Do not key the cache solely by `sase.__version__`, a launcher timestamp, or the
   `.zwc` timestamp. Include resolved package/interpreter identity, loader/cache format
   revision, distribution metadata, and a conservative source-change fingerprint. For
   editable and same-version installs, include file membership and stat signatures of
   the SASE Python sources, including parser and emitter dependencies; changes to
   core/package metadata that affect parser-provided choices must also invalidate.
   Inventory only local installation files, without git subprocesses, importing the full
   CLI, or loading project state. Include any environment inputs actually used when
   constructing the grammar. Explain the chosen inputs in code and test them.
5. On a miss, build the full parser once, emit the requested grammar, and record the
   actual structural digest and generated-content checksum in the manifest. Reuse
   existing script publication and zcompile helpers where appropriate. On a hit, verify
   cache-file integrity without building the parser. Explicit `refresh` must remain able
   to force regeneration, including same-version changes.
6. Serialize concurrent generation for the same runtime/shell with a bounded lock and
   recheck after acquiring it. Stage grammar and zsh bytecode before atomically
   publishing a completed generation and its manifest. Preserve the last valid
   generation on generation, compilation, or publication failure. Never claim stale
   fallback data is current. Bound retained generations; do not leave one directory per
   invocation or indefinitely retain failed staging files.

### Shell loaders

- Resolve a usable SASE executable using the existing emitters' PATH behavior, including
  avoiding ephemeral workspace executables when a durable one is available. Resolve at
  runtime; bake no machine home, virtualenv, or checkout path into portable loader
  bytes. Quote paths correctly, including spaces.
- Resolve/cache the grammar once when the completion file is loaded or first invoked,
  according to shell semantics. Static completion after that must launch no new SASE
  process. Keep dynamic candidates and their existing caches unchanged.
- Zsh must remain a `#compdef sase` autoload file. Load compiled grammar without
  recursively calling `_sase`, and preserve the compsys option context required by
  `_arguments`, `_describe`, prefix filtering, and aliases such as `sbd='sase -p bead'`.
  Distinguish the small installed loader's bytecode from the large grammar's bytecode.
- Bash must retain `complete -F` registration and reinvoke the loaded completer for the
  initial completion where needed. Fish must source its native completion definitions
  once and avoid spawning the ensure command from every candidate expression. Guard
  against recursive loader invocation.
- If SASE is missing or cache generation fails, completion must fail quietly without
  breaking shell startup. Preserve recoverability; a later load can retry. Errors from
  explicit ensure/refresh commands remain diagnosable.

### Phase verification

Add focused cache, import-boundary, concurrency, and shell smoke tests. Prove cold
generation, unchanged cache reuse, same-version parser and emitter edits, missing or
corrupt cache files, separate runtime identities, paths with spaces, and failure
preservation. Instrument process calls and parser builds: warm ensure must not import
the full parser/TUI, repeated static Tab presses must not invoke ensure again, and
candidate import-budget tests must still pass. Measure cold/warm CPU time and retain a
generous CPU budget comparable to the existing 250 ms candidates bound; do not use tight
wall-clock assertions on a shared host.

## Phase install-update

1. Make `install_completion` warm and validate its grammar cache before replacing a
   working install with the loader, then record both loader identity and grammar
   freshness. Publish stamps only after the referenced files are usable; installation
   failure must leave the previous working installation recoverable. Extend stamp/result
   schemas additively; continue reading existing local and chezmoi stamps. A raw
   snapshot and a runtime loader are distinct representations, not interchangeable bytes
   in status checks.
2. Refresh an existing stamped raw install into the loader representation at its stamped
   target. Both local and chezmoi-owned generated installs participate; preserve
   ownership rather than relabeling a managed file as local. Automatic refresh touches
   installed completion state only, never commits/applies a dotfiles repository. Keep
   the explicit force requirement when a caller intentionally takes over an unrelated
   file or changes an install's owner.
3. Protect the existing behavior for absent stamps, explicitly selected shells, portable
   `~/` targets, target relocation, and first-install fpath verification. Do not install
   every shell just because its generator exists. Refresh still skips the interactive
   registration probe and reports each shell independently. A dry run must not warm the
   cache or mutate scripts, stamps, locks, or bytecode.
4. Keep the existing post-update child-process refresh and integrate it into successful
   `sase update --to ...` paths, including successful no-change cases. Resolve the
   post-switch executable correctly. Dry runs, cancelled switches, and failed upgrades
   must not refresh. Preserve JSON output, quiet mode, the existing timeout, and
   nonfatal per-shell refresh reporting.
5. Update completion list/doctor to assess the registered loader and its runtime grammar
   separately. A current managed loader is healthy; `owner: chezmoi` alone must no
   longer make it stale. Detect missing/corrupt grammar, structural and emitter drift,
   and stale/missing grammar bytecode. Keep the deep fpath-shadowing diagnostic. Listing
   and doctor remain read-only; neither repairs as a side effect.
6. Provide an actionable migration message for old chezmoi sources: refreshing the
   applied target fixes the current installation, but the source must be migrated to
   loaders to prevent a later apply from restoring a snapshot. Do not claim to have
   migrated a source repository merely because its applied stamp was updated.

### Phase verification

Extend `tests/completion/test_install*.py`, `test_update_refresh_soak.py`,
`tests/main/test_update_command_completion.py`, mode-switch tests, and doctor tests.
Replace the managed-skip expectation with successful managed refresh and preserved
ownership. Cover ordinary and editable updates, unchanged nominal versions, mode
switches, child process selection, absent installs, mixed shell failures, JSON, and
read-only dry runs. Retain the existing test that a failed zcompile preserves prior
working state and add concurrent readers during publication.

## Phase chezmoi-integration

1. Change `deploy_chezmoi_completion` to distribute the portable loaders at its existing
   shell target paths. Stop deploying machine/version/digest/timestamp snapshots under
   `dot_sase/completion/stamp/`; runtime metadata belongs to SASE. Include removal of
   precisely those old generated stamp source files in the deploy plan, dry-run output,
   and commit/apply path so apply cannot restore them. Keep the command's existing
   no-commit/no-push/no-apply/source controls.
2. Through `sase repo open chezmoi`, migrate the linked source files
   `home/dot_zfunc/_sase`, `home/dot_local/share/bash-completion/completions/sase`, and
   `home/dot_config/fish/completions/sase.fish` to those loaders. Remove the generated
   completion stamp sources only. Update the existing zcompile apply hook so it is
   loader-aware and no longer treats recompiling a frozen grammar as a refresh. Keep the
   before-compinit fpath setup and post-oh-my-zsh priority fix in `home/dot_zshrc`; do
   not add a full SASE parser build to shell startup.
3. A chezmoi apply on a host with no SASE installed must still succeed; the loader
   becomes functional once SASE is installed. A loader's first successful ensure creates
   host-local runtime metadata, without chezmoi present. Applying the same source on two
   machines with different loader-capable SASE versions must use each local version.
   Land and install the SASE loader protocol before applying the dotfiles migration.
   Document its minimum supported release: an older pinned SASE without
   `completion ensure` cannot supply this new protocol. Handle that case quietly in
   shell startup and explicitly in diagnostics; do not claim cross-version support for a
   protocol the older package does not implement.
4. Extend the linked `tests/bash/sase_completion_test.sh` and relevant startup tests.
   Repeated `chezmoi apply` after a runtime upgrade must leave loader bytes stable and
   completion current. No source regeneration, git commit, push, or network operation is
   part of normal package upgrades or tab completion.
5. Update `docs/completion.md` and `INSTALL.md`: explain the loader/cache distinction,
   new-shell guarantee, external-upgrade behavior, static export limitations, one-time
   managed-source migration, diagnostics, and recovery commands. Remove the misleading
   implication that emitting a current grammar makes installed static copies perpetually
   current. Refresh the checked-in CLI spec with `just sync-completion-spec` after the
   new completion command surface settles.

### End-to-end acceptance

Use disposable home, SASE state, PATH, and chezmoi source directories; never upgrade the
operator's actual environment as a test.

- Install older grammar without `screenshot`, change the effective runtime, and verify
  real completion offers `sase scr<TAB>` and screenshot options in fresh bash, fish, and
  zsh shells. Test both a version bump and a source-tree change with the same nominal
  version. Include an emitter-only change and a synthetic newly added command so the
  regression is not tied solely to screenshot's presence.
- Exercise local installs with no chezmoi executable, loader-based managed installs, and
  legacy chezmoi-stamped snapshots. Prove `sase update` refreshes the legacy installed
  target and the source migration prevents later rollback by apply.
- Simulate an external installer replacing the runtime without calling SASE's update
  hook. A fresh shell's loader must select the new grammar without manual refresh. Test
  an upgrade and a downgrade between loader-capable builds, including changed
  executable/virtualenv locations. Separately document the one-time migration needed for
  a raw export or pre-loader installation updated entirely outside `sase update`.
- Execute repeated managed apply/upgrade cycles, mixed installed and absent shells, two
  runtime identities sharing one user state directory, and simultaneous cold starts.
  Check script/bytecode/manifest consistency and bounded cache retention.
- Preserve zsh alias and prefix completion tests, fpath conflict tests, shell syntax
  checks, dynamic candidate cache tests, and the existing no-heavy-import contract. Run
  real fish completion tests in a lane with fish installed; an all-skipped fish result
  is not evidence of fish support.

Each implementation phase runs focused tests and `just check` in the SASE repo after
reading the current `lint_and_test.md`. The linked repo runs its focused bashunit
completion/startup tests and applicable repository checks. Format before verification.
The combined epic requires `just check-full` through `/sase_monitor` before landing;
report any unresolved failures accurately. After the linked chezmoi changes are landed,
its AGENTS.md requires `chezmoi update -a --force`; the landing workflow must arrange
that application and then check `sase completion list --json` plus fresh-shell
completion on the host. Do not apply unlanded dotfiles during implementation testing.

## Completion criteria

The installed CLI and a fresh shell agree about screenshot and future commands;
successful SASE updates refresh every stamped supported shell regardless of owner;
external upgrades are detected when completion next loads; chezmoi cannot restore a
version-specific grammar or runtime stamp after its one-time source migration; and warm
static completion has no new per-keystroke process or full-parser cost. The
implementation report must distinguish tested upgrade paths from the documented
requirement to reopen shells that already loaded the old functions.
