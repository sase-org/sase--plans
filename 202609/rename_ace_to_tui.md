---
tier: tale
title: Rename sase ace to sase tui and update public terminology
size: medium
goal:
  Rename the public sase ace command to sase tui and replace maintained user-facing ACE
  terminology with sase's TUI while preserving internal identities and behavior.
proposed_by: bbugyi200.athena.0l8
create_time: 2026-09-15 11:08:42
status: wip
---

# Rename the public command and terminology for sase's TUI

## Outcome and tier

Users launch the existing application with `sase tui [QUERY] [OPTIONS]`. Maintained
help, messages, guides, snippets, and active media call it **sase's TUI**. This is a
single coordinated rename, with no changes to what the application does.

Use a **tale**, implementation size **medium**. One coding agent can implement the
bounded command change, presentation edits, and small linked-repository updates. The
broad text inventory needs careful classification, but no new backend model or
multi-phase migration. Authoring this tale is large planning work.

The canceled `sase-113` epic and its `plan:202609/schedule_tui_interfaces.md` were
reviewed through the audited interface. They supply context only. Do not reopen that
epic or inherit its scheduler, configuration, SDK, feature-flag, or multi-phase rollout
proposals.

All paths below are relative to their named repository. Open non-primary repositories
with `/sase_repo` and use the returned checkout. Do not use another agent's workspace or
directly read archived sidecar artifacts.

## Public contract and scope

### Command

- Make `tui` the sole public command name for launching the application. This plan
  proposes a direct rename: `sase ace` becomes an unknown command, including through
  `python -m sase`. Do not add an alias, migration command, or compatibility flag. State
  the breaking command change in the current release notes. Existing shell wrappers
  maintained in the linked dotfiles repository are updated in the same work.
- Preserve the optional query, saved-query fallback, validation, exit statuses, model
  controls, VCS provider override, profiling, refresh settings, `--tmux`, `--tab`,
  `--no-axe`, and `--restart-axe`, including their existing short options and legacy tab
  aliases. Preserve global CLI flags before the command.
- New help, completion, errors, examples, subprocess invocations, restart commands, and
  tmux relaunches use `sase tui`. Internal handler and registrar names may remain
  `handle_ace_command` and `register_ace_parser`.
- The canceled epic's compatibility contract is not active authorization for this work.
  If review requests an old-command compatibility branch, revise the plan under
  `sase_flags.md` rather than silently adding one during implementation.

### Language and preserved identities

Use the exact naming **sase's TUI** in prose. Replace compound labels such as `ACE TUI`
as a whole; do not produce `sase's TUI TUI`. Rewrite surrounding grammar where needed:
`ACE's Agents tab` becomes `the Agents tab in sase's TUI`, and `Quit ACE` becomes
`Quit sase's TUI`. Use `sase tui` for command examples and the application title; retain
the existing version suffix. Do not replace AXE with another name.

The boundary is what a string does, not which directory contains it:

| Surface                                                                                                                                       | Treatment                                                                                |
| --------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Runtime labels, help, onboarding, notices, errors, clipboard headings, fresh diagnostic prose                                                 | Replace the old product name; preserve surrounding behavior and structured fields.       |
| Current README, handbook, reference docs, navigation, captions, alt text, generated skill sources and operational guidance                    | Update names and executable examples.                                                    |
| Configuration descriptions and explanatory comments shipped to users                                                                          | Update prose only.                                                                       |
| `ace` config keys, schema property names, environment variables, APIs/imports, module/class/function names, widget/action IDs, test filenames | Keep their actual spelling. Document literal keys truthfully.                            |
| Storage paths, `ace-run`, profile filenames, tmux session IDs, proc kinds, registry keys and persisted data                                   | Keep existing identities and values; no migration or blanket output translation.         |
| User-authored prompts, stored chats/logs, archived plans/research/beads, released changelog entries, explicitly historical generation records | Preserve verbatim.                                                                       |
| Internal comments/docstrings and synthetic historical fixtures                                                                                | Leave alone unless they are actually surfaced as public help or executable instructions. |

Public JSON field names, machine identifiers, and echoed paths remain stable. Authored
human-readable messages inside a JSON response change at their source; arbitrary
payloads and old records are not rewritten. Source filenames such as `docs/ace.md` and
media filenames may remain to preserve links. Headings and navigation labels change;
check fragment links affected by any heading change.

## Implementation

### 1. Rename the command across parsing, dispatch, and launch paths

In the primary `sase` repository:

- Change the parser token in `src/sase/main/parser_ace.py`, the lazy registry key in
  `parser_registry.py`, dispatch in `entry.py`, and the compact help command/example in
  `parser_root_help.py`. Keep registrar imports and handler implementation names stable.
  Check full and narrow parser construction and the registrar mapping in
  `parser_full_registrars.py` together.
- Provide a clear command description for the whole application, not just its Patch
  query. Preserve root-help conventions and alphabetically sorted full command and
  option listings. Follow `cli_rules.md`.
- Update `_build_ace_restart_argv()` in `src/sase/main/ace_handler.py` and
  `_build_relaunch_cmd()` in `src/sase/main/ace_tmux.py` to recognize and emit `tui`.
  These currently prepend `ace` when they do not recognize the first forwarded token.
  Handle a root command preceded by global options using the existing root-argument
  utilities; do not introduce a duplicate command token.
- Preserve query argument boundaries and shell quoting. Preserve the `--` separator and
  literal query tokens after it while removing only launch-time `--tmux`/`-T` flags and
  applying the existing restart-AXE policy. Keep Python interpreter, environment
  forwarding, tmux targets, window names, and lifecycle behavior intact.
- Sweep command emitters beyond literal `sase ace` text: argv lists, module-form
  invocations, shell strings, demo tapes, test setup, and helpers may separate `sase`
  and `ace` into different tokens. Change executable command tokens wherever owned code
  launches the TUI, even when the enclosing implementation is internal.
- Regenerate the parser-derived completion snapshot with `just sync-completion-spec`.
  Check `src/sase/completion/build.py`, the Bash/Zsh/Fish emitters, and
  `tests/completion/snapshots/cli_spec.json`; do not hand-edit generated completions in
  the user's home directory.

### 2. Update displayed application text and diagnostics

Start with the confirmed public string owners, then audit remaining matches:

- `src/sase/ace/tui/app.py` and `util/app_version.py`: base and versioned title, crash
  messages shown in logs, and title-dependent snapshots.
- `modals/help_modal/`: modal title and copy-snapshot descriptions in the Agents,
  Artifacts/Patches, and Axe help sections. `actions/clipboard/_core.py` and
  `_patch.py`: copied snapshot headings.
- `widgets/agent_onboarding.py` and `axe_onboarding.py`: welcome text, command
  instructions, and links to the guide.
- Quit, tmux Agent, plugin install/update/uninstall, feature-flag, and notification
  modals; `update_restart.py`; `actions/base.py`; and configuration error messages.
  Audit displayed descriptions and fresh session labels outside these examples too.
- CLI help in `src/sase/main/parser_*.py`, query-launch guidance in
  `main/query_handler/_launch.py`, `agents/cli_list.py`, `agents/_restart_render.py`,
  `agent/_restart_planning.py`, `agent/_restart_recovery.py`,
  `bead/cli_work_cleanup_types.py`, `plan_shell/create.py`, `axe/runner_reporting.py`,
  and `procs/runner.py`.
- Doctor titles, summaries, remediation instructions, and details in `src/sase/doctor/`;
  retention-preview headings in `artifact_cli/prune_runs.py`; displayed feature-flag
  descriptions in `feature_flags/registry.py` and restart notices in
  `feature_flags/cli_set.py`. Keep flag identities and states unchanged.
- `src/sase/config/sase.schema.json` and `src/sase/default_config.yml`: descriptions and
  comments only, including descriptions displayed in the Admin Center. Keep `ace`,
  `ace.*`, keymaps, `axe`, and all setting values unchanged.

Review longer replacements in compact dialogs, buttons, help boxes, and header areas.
Respect the help popup's fixed box width and binding-description constraints from
`src/sase/ace/AGENTS.md`; wrap prose or adjust presentation without changing bindings.
Do not shorten the requested product name back to ACE to make it fit. Keep the startup
and refresh paths free of new I/O or runtime text-replacement passes.

The `sase-core` audit found ACE in internal comments and test fixtures, not production
public messages needing a change. No Rust changes are planned. If a missed shared
message is discovered, edit it in its core owner and test the binding; do not add a
Python fallback or change the Rust wire schema for this rename.

### 3. Update maintained documentation, guidance, and linked consumers

Primary documentation includes `README.md`, `INSTALL.md`, `docs/index.md`,
`docs/getting_started.md`, `docs/cli.md`, `docs/ace.md`, `docs/configuration.md`, the
rest of the maintained `docs/` guides, `mkdocs.yml`, the PDF cover template
`docs/templates/pdf/front.html.j2`, and public tool instructions. Update the current
unreleased changelog section without rewriting release history. Check textual branding
variants (`Ace`, full acronym expansions, and `ACE-TUI`) as well as `ACE`.

Treat dated blog posts that record historical behavior as history. Where a post is still
a current linked tutorial, update its executable instructions and relevant terminology
or clearly identify its historical version. Do not silently classify all blog content as
exempt. Do not change file URLs merely to remove `ace` from a path. Update PDF
validation expectations in `tools/validate_docs_pdf` if they assert a renamed displayed
heading.

Update these generated skill **sources**, not installed provider copies:

- `src/sase/xprompts/skills/sase_patches.md`
- `src/sase/xprompts/skills/sase_notify.md`
- `src/sase/xprompts/skills/sase_var.md`
- `src/sase/xprompts/skills/sase_agents_status.md`
- `src/sase/xprompts/skills/sase_questions.md`
- `src/sase/xprompts/skills/sase_run.md`

Apply `/sase_memory_write` and audited `/sase_memory_read` before memory edits. The
user's requested public terminology update covers these narrowly scoped source edits:

- `sase/memory/tui_perf.md`: command examples become `sase tui`; name the public
  application as sase's TUI while preserving actual trace names and API identifiers.
- `sase/memory/lint_and_test.md`: update the visual-suite prose, keeping test paths.
- `sase/memory/glossary/current-project.md`, `proc.md`, and `sase-gate.md`: update
  references to the public application, without changing definitions or semantics.

Regenerate memory outputs with `sase memory init`. Never hand-edit generated root
`AGENTS.md` or provider shims. Immutable decision records, including the canceled or
retired interfaces they describe, remain history. Inspect other current guidance via
audited reads if a remaining executable example is discovered. Preview skill output with
`sase skill init --diff` or `--dry-run`; deployment happens only from the clean, landed
source under `generated_skills.md`.

Linked-repository edits are limited to confirmed public consumers:

| Repository                        | Files and changes                                                                                                                                                                                                                                                                                      |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `chezmoi`                         | `home/dot_config/aliases.sh`: preserve `ace`, `acei`, and `aceii` wrapper names, but execute `sase tui`, retaining existing arguments.                                                                                                                                                                 |
| `chezmoi`                         | `home/dot_config/sase/sase.yml` and `home/dot_config/nvim/lua/bb_utils/_snip_utils.lua`: update emitted commands and descriptions together, preserving snippet triggers. Update the notification-store description's ACE reference. The local instruction requires matching SASE and LuaSnip snippets. |
| `sase-telegram` and `sase-github` | `README.md` installation instructions launch `sase tui`.                                                                                                                                                                                                                                               |
| `sase-nvim`                       | `README.md` public command and prompt-widget prose use the new names. Leave Lua implementation comments alone.                                                                                                                                                                                         |

The `sase-research-artifacts` audit found no matching maintained public text. Do not
modify it or research/agent/plan/bead sidecars just to make searches empty. Use each
changed repository's required checks and host finalization. Apply dotfiles only after
the new SASE command has landed and is available; honor chezmoi's post-landing apply
instruction without deploying an unlanded skill tree.

### 4. Refresh active visual material

The current README and docs homepage visibly embed `ACE TUI` in
`docs/images/sase_overview.png` and `docs/images/sase-component-communication.png`.
Update the affected labels to `sase's TUI` with the image-editing skill/tool appropriate
to the asset, retaining composition, dimensions, and other terminology. Inspect the
result at its documentation display size. Update maintained generation instructions and
any content hashes that describe the edited asset. Leave the explicitly retired
`sase_tui_tabs_infographic` generation record and unembedded retired PNG alone.

Update `demos/tapes/sase_ace_*.tape`, affected captions, visible seeded prose in
`demos/scripts/seed_sase_ace_demo`, and `demos/README.md`. Keep tape/media filenames and
fixture identities stable. Regenerate the changed GIF/MP4 outputs and their active
README/blog derivatives so recordings show the renamed command, title, and notices. Use
the isolated seeded demo environment; do not launch real user work.

`demos/tapes/AGENTS.md` requires regeneration after tape edits, naming `just demos -y`.
The current recipe's `-y` branch directly invokes `git commit`, which conflicts with the
host-owned completion rule. Fulfill regeneration through the same render, postprocess,
check, and timestamp steps in noncommitting mode (`just demos` with noninteractive
stdin), and leave all changes for `/sase_final`. Do not invoke `-y` or independently
commit. Use `/sase_monitor` for long render or verification commands. Verify that
postprocessing updates every actively embedded derivative; a new tape source with old
displayed media is incomplete.

## Verification and acceptance

Use existing meaningful behavioral suites, updating assertions that exercise the renamed
public surface. Avoid mass-changing internal fixture names or adding one test per
replacement string.

1. **Command contract:** cover full/narrow parser parity and dispatch; root `-h` and
   `-H`; `sase tui -h`; module-form invocation; rejection of `ace`; optional query
   fallback; representative flags and errors; and global options before `tui`. Relevant
   suites include `tests/main/test_ace_handler.py`, `test_parser_narrowing.py`,
   `test_parser_root_help.py`, and `test_global_options.py`. Stub application launch
   rather than starting the daemon.
2. **Relaunch behavior:** in `tests/main/test_ace_handler.py` and `test_ace_tmux.py`,
   assert captured argv has exactly one `tui` command, preserves global flags and a
   query with spaces/quotes, and handles `--` followed by literal flag-like query data.
   Verify TUI-only and TUI-plus-AXE restarts, tmux launch flag removal, interpreter
   reuse, and error prefixes without calling real `execv` or changing a live session.
3. **Completion:** regenerate and verify the structural snapshot. Check Bash, Zsh, and
   Fish output offers `tui` and its options, with no root `ace` candidate. Exercise the
   existing completion build/emitter and shell-smoke tests using isolated targets.
4. **Presentation:** update the existing title/version, clipboard, onboarding, help,
   quit, plugin-restart, feature-flag, and doctor tests as applicable. Review the longer
   wording at narrow widths. Run affected PNG snapshot cases, inspect diffs, and accept
   only intentional text/layout changes; do not blanket-refresh unrelated goldens.
5. **Documentation and media:** run `just docs-check` and `just docs-pdf-check` for
   changed navigation/cover text. Verify page and fragment links, active image labels,
   demo frames, and `just --justfile demos/Justfile check`. Preserve truthful literal
   `ace.*` configuration examples and stable media URLs.
6. **Integration/generation:** check shell syntax and changed wrapper command bodies;
   verify SASE/LuaSnip emitted-text parity and valid YAML/Lua. Preview skill generation,
   regenerate memory output, and run applicable checks in each changed linked repo. No
   core check is needed if core is untouched; if changed, its required `just check` must
   include PyO3 tests, not only `cargo test -p sase_core`.
7. **Required repository gate:** read `lint_and_test.md` and run `just check` after
   tracked changes. If selection escalates, touches its broadening set, or is unusual,
   run `just check-full` through `/sase_monitor`, as required. Long visual/media jobs
   also use that handoff. Complete required verification before declaring the rename
   finished.
8. **Final residual audit:** search tracked text case-sensitively and case-insensitively
   for whole-word ACE/Ace, `sase ace`, module-form commands, and separated argv tokens.
   Review matches semantically rather than requiring zero occurrences. Every remaining
   match must be an unchanged identifier/literal path, internal implementation detail,
   user-authored data, historical record, or explicit old-command rejection assertion.
   Separately inspect active binary media because text searches cannot cover it.

Deliver the verified changes and a concise residual-scope summary. The accepted result
has one working public launch command, consistent requested terminology on maintained
user surfaces, unchanged internal identities/configuration and runtime behavior, and no
accidental scheduler rename. Host-owned finalizers handle all changed repositories; the
implementation agent does not create commits, branches, or PRs directly.
