---
tier: epic
title: Keep shell completion current with the installed SASE CLI
goal:
  Restore usage completion and reliably refresh and verify generated shell scripts when
  commands change, including on chezmoi-managed machines.
phases:
  - id: completion_freshness
    title: Detect stale scripts and make local installation and refresh reliable
    size: medium
    depends_on: []
    description:
      "completion_freshness: add content-based diagnostics, reliable installation,
      explicit refresh, and refresh from the updated executable."
  - id: managed_installation
    title: Replace frozen chezmoi snapshots with a repeatable installation recipe
    size: medium
    depends_on:
      - completion_freshness
    description:
      "managed_installation: generate an apply-time installer, migrate owned snapshots,
      and update the linked chezmoi source."
  - id: completion_acceptance
    title: Verify shell behavior, migration, and documentation end to end
    size: medium
    depends_on:
      - completion_freshness
      - managed_installation
    description:
      "completion_acceptance: add real-shell regressions for usage and future commands,
      migration and update scenarios, and accurate troubleshooting guidance."
proposed_by: bbugyi200.athena.0i9
create_time: 2026-09-10 09:46:01
status: wip
---

- **PROMPT:**
  [prompts/202609/completion_freshness.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/completion_freshness.md)

# Keep shell completion current with the installed SASE CLI

## Outcome and scope

After deployment, `sase us<TAB>` offers `usage`, and `sase usage <TAB>` offers `list`
and `refresh`. A successful `sase update` refreshes installed completions even when the
package version does not change. Installation status and doctor detect drift in the
actual command tree and generated scripts. Chezmoi application cannot restore an
obsolete command snapshot over a newer installation.

This is an epic because the fix spans the completion lifecycle in SASE, a coordinated
change to the linked chezmoi repository, and integration tests that exercise both. The
three phases are sequential where their interfaces overlap. Each is bounded direct
implementation work; no phase needs to author another plan.

Keep native static shell scripts and the existing live argparse source of truth. This
work is shell integration, argparse presentation, and installation glue; retain the
existing Rust-backed providers for domain values. Do not reimplement provider or project
logic in Python. No memory edits, remote-machine rollout, provider-usage requests, or
new shell support are needed.

## Diagnosis and evidence

Read-only investigation on September 10, 2026 established:

- The installed `sase completion spec` equals the workspace's live structural spec. Both
  contain `usage`, with `list` and `refresh` children. All three current emitters
  include it. `src/sase/main/parser_usage.py` supplies visible help, and
  `parser_full_registrars.py` registers it. The introduction commit is `3f9c7b451`,
  dated September 8.
- All three installed scripts exactly equal their corresponding chezmoi source files and
  contain no `usage` entry. Their stamps identify owner `chezmoi`, version `0.17.1`,
  timestamp `2026-09-06T14:44:12Z`, and structural digest `9aa2b88141137d0f`. The live
  digest during investigation is `298808f3fc224446`; treat these as evidence, never
  constants to embed in production.
- `list_shell_statuses()` / `_status_for_shell()` in `src/sase/completion/install.py`
  check version and `.zwc` mtimes but ignore the recorded structural digest.
  Consequently, both `sase completion list` and `sase doctor -C completion.install -v`
  report success for obsolete scripts.
- `_refresh_stamped_completions()` unconditionally skips chezmoi-owned stamps and
  reports that skip as successful. The chezmoi hook
  `home/.chezmoiscripts/run_onchange_after_zcompile_sase_completion.tmpl` recompiles
  existing bytes; it does not regenerate command definitions.
- Zsh registration resolves to `_sase`, and chezmoi's `home/dot_zshrc` puts `.zfunc` on
  `fpath` before compinit. Registration succeeds while the registered script is stale.
  This is not a dynamic-candidate TTL problem.

Related weaknesses found in the same installation path: any non-UNSET registration
string is accepted, the probe does not establish which script wins on `fpath`, and
installation advances stamps and removes an old target even after zcompile fails. The
update handler also calls refresh inside the updating interpreter. A fresh post-update
process is needed to avoid using already imported pre-update code; this last risk needs
a regression that demonstrates the old/new-process difference.

Existing protection worth preserving includes the full-parser registry parity tests,
`tests/completion/test_build.py`'s usage provider checks, the structural snapshot gate,
atomic text writes, and the isolated update-refresh test harness. They currently protect
generation better than deployment.

## Design contracts

1. The complete live argparse tree remains authoritative. Adding a command must not
   require adding a separate completion command list. Keep hidden and compatibility
   commands hidden and preserve current candidate caching.
2. Chezmoi owns a stable installation recipe and shell registration configuration. SASE
   owns the generated, host-specific scripts and their runtime stamps. Run the recipe
   during chezmoi apply, never on every shell startup or TAB. Existing direct local
   installs continue to work.
3. Update refresh runs once through the upgraded installation's executable/interpreter,
   after successful updates including unchanged-version editable updates. It does not
   commit, push, edit a dotfiles repository, or invoke another update recursively.
4. Automatic refresh touches only recorded local installation targets. Migrating an old
   chezmoi-owned snapshot is a separate, explicit operation. Unstamped foreign files and
   custom destinations must not be overwritten by a recurring hook.
5. Reports distinguish command/spec drift, script drift, missing files, compilation
   trouble, and registration trouble. A skip or failed probe must not imply verified
   success. Explicit refresh fails when a requested refresh fails; update reports the
   failure without failing an otherwise successful package update.

## Completion freshness

Primary surfaces:
`src/sase/completion/{install,install_stamp,install_targets,model}.py`, the shell
emitters, `src/sase/main/{completion_handler,parser_completion,update_handler}.py`, and
`src/sase/doctor/checks_completion.py`. Extract focused helpers as needed rather than
expanding the already large installer indefinitely.

- Introduce one coherent freshness assessment shared by list, doctor, and refresh. Build
  the full spec once per operation when there are installed scripts. Compare its
  structural digest with each stamp and compare the actual generated file with current
  expected output. Cover emitter-only changes as well as grammar changes. If recording a
  script checksum, version the stamp schema and retain an honest legacy-stamp read path;
  absence of new metadata is not proof of freshness. Do not run the full parser or hash
  the CLI tree on every completion request.
- Add `sase completion refresh [SHELL]` for existing stamped installations, with
  `-d/--dry-run` and `-j/--json`. Omitted SHELL means all stamped supported shells; no
  stamps means a clearly reported no-op. Use the same refresh implementation from
  update. Preserve custom stamped target directories and per-shell failure isolation.
  Keep parser help, choices, and options sorted and refresh the structural snapshot
  through `just sync-completion-spec` when CLI changes are final.
- Invoke production update refresh in a bounded child of the upgraded tool installation,
  with structured results. Select the executable from the actual tool installation
  rather than an arbitrary PATH shadow or the agent's workspace venv. Preserve the
  existing JSON `completion_refresh` envelope and truthful per-shell outcomes, including
  timeouts, nonzero exit, malformed output, and unavailable child executable. Retain
  dependency injection for isolated tests.
- Stage and syntax-check generated scripts; for zsh successfully compile before
  publishing a fresh stamp or removing a superseded target. Preserve the previous usable
  installation on emission, write, or compilation failure. Keep paired script and
  bytecode publication recoverable, with the stamp as a conservative completion record,
  and test interruption/failure boundaries. Avoid rewriting identical outputs
  unnecessarily. Dry runs do not create scripts, directories, stamps, or bytecode.
- Tighten the bounded zsh probe with framed output and exit-status checking. Verify the
  expected completion registration and effective `_sase` file, and report an earlier
  conflicting `fpath` entry. A custom directory requiring an rc change should receive an
  actionable hint. Do not edit rc files or delete unrelated completion files or global
  compinit caches.
- Extend list/doctor human and JSON output with actionable drift reasons. Legacy
  chezmoi-owned installs should report the managed-migration remedy, not silently
  recommend an unconditional forced takeover of arbitrary files.

Phase verification: targeted installer, stamp, target, doctor, handler, parser, and
update tests; same-version spec mutation; emitter-only mutation; damaged script;
zcompile failure; malformed/failed probe; shadowed target; and a child-process test
where the parent has old parser code loaded and the new installation adds a command. Run
`just check` after changing SASE files.

## Managed installation

Primary surfaces: `src/sase/completion/deploy_chezmoi.py`, the completion CLI and its
tests, plus the linked chezmoi repository. Open chezmoi with `sase repo open chezmoi`
and a specific reason, and read its AGENTS.md before editing. Use the returned path; do
not depend on a particular agent workspace location.

- Change `completion deploy-chezmoi`'s new-format output to a repeatable apply-time
  installation recipe, with no baked command tree, version, timestamp, or host-absolute
  home path. The recipe uses the installed SASE CLI to generate the three native scripts
  at the established user locations. Compile zsh when zsh is available; clearly skip
  unavailable prerequisites and allow a later apply to recover. It must work on initial
  apply, subsequent apply, and hosts running different SASE versions. It must not run
  Git operations from the apply hook. Exercise the actual chezmoi hook scheduling rather
  than merely executing the rendered script: the recipe must rerun on subsequent applies
  when the installed CLI has changed, even though the recipe's own source bytes are
  identical.
- Supply an explicit `-m/--migrate-managed` mode on `completion refresh` for adoption of
  legacy chezmoi-owned snapshots. Require a matching supported-shell stamp, expected
  target, and SASE-generated file identity; do not equate all writable files with owned
  files. Honor unusual existing target paths conservatively, report conflicts without
  replacing them, and leave recoverable old bytes on migration failure. Successful
  migration generates from the current host CLI and changes the runtime stamp to
  `local`. This mode is the recipe's explicit migration step; normal update refresh
  never opts into taking over managed files.
- After migration, bootstrap missing conventional installations with non-forced
  `completion install` calls and refresh existing local stamps. A repeat apply should be
  idempotent. Keep existing stamp targets rather than moving local installations to
  conventional paths on every apply. An unrelated failure in one shell should produce a
  useful report while allowing the other supported shells to refresh. If SASE itself is
  unavailable during an initial apply, report a deferred install and ensure the next
  apply retries; do not create a success stamp for that skip.
- In chezmoi, replace the three frozen source scripts, three source stamp JSON files,
  and the compile-only hook with the generated installation recipe. Preserve the correct
  `.zfunc` ordering in `home/dot_zshrc`. Verify removal of tracked generated sources
  does not delete the working home targets during migration and that a later apply
  cannot resurrect old snapshots. Update `tests/bash/sase_completion_test.sh` to
  exercise this lifecycle.
- Follow the project's sunset-flag rule for the temporary old/new `deploy-chezmoi`
  output branch. Create `completion_managed_install_recipe` through `sase flag new`
  during implementation, with the new recipe enabled by default and the existing
  snapshot exporter available when disabled. The removal gate is successful migration of
  supported managed deployments and removal of reliance on the snapshot exporter; the
  normal date/release thresholds also apply. Cover both states. Do not hand-create a
  flag registry entry or task bead during planning.
- Generate linked-source changes for review with the explicit `--source` pointing into
  the opened repository and no commit/push/apply side effects (the existing no-commit
  control supports staging source output). Host finalizers own commits. After landing
  the linked repository, follow its required chezmoi apply workflow and verify the local
  installation. Remote fleet rollout is outside this plan.

Phase verification: disposable home/source directories, repeated chezmoi applies, legacy
ownership conversion, a foreign target, custom local targets, absent SASE, absent zsh,
failure followed by recovery, and two installed CLI versions producing different
commands from identical recipe source. Run the relevant bashunit suite in chezmoi and
`just check` for SASE changes.

## Completion acceptance

- Extend the existing shell harnesses to test actual candidates, not just script
  substrings or `_comps[sase]`. In disposable shell environments, verify the live
  `usage` command, its `list` and `refresh` children, and its options in bash, zsh, and
  fish. Do not call usage providers to do this. Use real zsh TAB/PTY coverage where
  necessary, bash COMPREPLY, and fish `complete -C`.
- Add a synthetic visible command and nested option to an argparse tree without changing
  its version. Prove the spec and all emitters discover it automatically, old installed
  scripts are diagnosed as stale, and refresh makes it available. Exercise this through
  both direct local installation and the managed recipe path. Keep explicit coverage for
  hidden commands and compatibility aliases.
- Cover an unchanged-version update, a release-version update, a failed upgrade that
  must not refresh, and a refresh failure that must not falsify package-update success.
  Verify a later chezmoi apply preserves or regenerates the newer command definitions
  and does not replace local stamps with old managed stamps.
- Test warm completion with a counting executable shim to ensure static completion does
  not spawn SASE repeatedly. Preserve current dynamic-value cache behavior and native
  shell loading performance. Regeneration belongs in install/apply/update.
- Rewrite `docs/completion.md` around the actual lifecycle and ownership model. Explain
  how to repair old managed installations, manually refresh after upgrades performed
  outside `sase update`, detect a shadowing completion, and open a fresh shell after
  replacing a script already loaded in memory. Distinguish static grammar drift from
  dynamic-candidate cache TTLs. Remove the claim that matching versions guarantee
  current completion.
- Record the final local verification: `sase completion list --json`, the completion
  doctor checks including the deep zsh registration check, and a new shell offering
  `usage`. Report any uninstalled shell as such. Isolated integration tests should cover
  all three even if a particular host lacks one shell.

The planning turn changes only this scratch plan. At implementation time, read the
current lint/test memory, prepare the workspace environment with `just install` if
needed, and run `just check` for each SASE phase. The planning workspace's venv could
build the completion spec but lacked a usable Rust extension; the installed `sase` CLI
worked, so do not mistake that environment issue for the completion root cause. Use the
monitor skill for long commands and for the required combined-tree `just check-full`
before epic landing. Run appropriate chezmoi checks for its changes. Do not broaden test
runs repeatedly without a new failure or concern.

## Acceptance criteria

- The reported missing `usage` completion is fixed on the deployed local shell.
- Equal package versions with unequal command trees are visibly stale.
- Adding a visible parser command requires no manual completion registry change.
- Successful updates regenerate installed local scripts using the upgraded code,
  including installations created by the chezmoi recipe.
- Repeat chezmoi application cannot restore the September 6 command snapshot.
- Failed compilation or refresh preserves recoverable prior scripts and does not publish
  misleading freshness; dry runs and foreign-file conflicts are safe.
- Real-shell regression tests, SASE verification, and applicable chezmoi checks pass.
