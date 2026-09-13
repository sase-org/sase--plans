---
tier: tale
title: Fix
goal:
  "#research_swarm resolves to the fixed sase-research-artifacts plugin definition on
  this Mac again, and `sase doctor` flags any xprompt definition that still uses retired
  directive syntax (such as %wait(priority=...)) with its source file before a launch
  can fail."
size: medium
proposed_by: bbugyi200.kellys_mbp.0b
status: done
---

- **AGENTS:**
  - [bbugyi200.kellys_mbp.0b](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.kellys_mbp.0b.md)
- **COMMITS:**
  - [2170f42](https://github.com/sase-org/sase/commit/2170f422e938d1e15cba5aa2002d4adca8cbb505)
    — feat(xprompt): flag retired directive syntax in doctor

# Plan: Fix the `#research_swarm` launch failure caused by a stale home xprompt override

## Diagnosis (already established — do not re-investigate)

On the user's Mac, launching
`#gh:sase #research_swarm:: The test suite for this project is incredibly slow ...`
failed at 2026-09-12 15:07 (launch-failure log entry `err_260912_150725_111b26`, proc
`qjffcn7swtnh`) with:

```text
DirectiveError: %wait(priority=...) has moved to %queue. Use %queue(priority=N) or %q(p=N), and keep dependencies on %wait.
```

It was raised from `collect_prompt_directive_matches` in
`src/sase/xprompt/_directive_collect.py` during the clan prepass
(`prepare_clan_launches` -> `extract_static_clan_directive`).

Root cause chain:

1. Commit `3f23a5374` (2026-09-08, bead `sase-yj.3`) made
   `%wait(runners=/capacity=/priority=/p=)` a hard migration error; queue controls now
   live only on `%queue` / `%q`.
2. The `research_swarm` xprompt that actually resolves on this Mac is
   `~/sase/xprompts/research_swarm.md` (`sase xprompt show research_swarm` reports
   `source: config`, `definition: /Users/bbugyi/sase/xprompts/research_swarm.md`). Its
   four swarm segments each carry `%wait(priority=20)` and use the retired model aliases
   `@research_a`, `@research_b`, `@research_lead`, which no longer exist in `sase.yml`.
3. That file is an orphan. Dotfiles commit `9f51c7b8` ("chore: migrate research config
   to plugin providers", 2026-08-12) deleted `home/sase/xprompts/research_swarm.md` and
   `home/sase/xprompts/old_research_swarm.md` from the chezmoi source and moved the
   swarm to the `sase-research-artifacts` plugin. chezmoi does not delete targets that
   disappear from source, so both files stayed on disk. `chezmoi managed` now lists only
   `sase/xprompts/pick_plan.md` and `sase/xprompts/sshot.yml`. Both orphans are
   byte-identical to their pre-`9f51c7b8` versions
   (`git show 9f51c7b8^:home/sase/xprompts/<file>` in the `bbugyi200/dotfiles` repo), so
   they are recoverable from git history.
4. Home `~/sase/xprompts/` definitions outrank plugin xprompts in discovery order, so
   the orphan silently shadows the plugin's `research_swarm`. The installed (editable)
   `sase_research_artifacts` plugin on this Mac already carries the fixed body, which
   uses `%q(w=0.25, capacity=..., priority=...)`. Upstream fixed it in plugin commit
   `5aaa244` ("fix(research): use queue capacity directive"), and loading plugin
   xprompts through the sase runtime confirms this machine serves that version.
5. Nothing warned about the problem ahead of time. `sase doctor -C config` reports
   `config.xprompt_definitions` OK for this file because it loads cleanly, and the
   launch error does not name the xprompt that contributed the retired directive.
   `sase xprompt list` shows no other visible xprompt that uses a retired `%wait` queue
   keyword.

The fix has two parts:

- **Part A:** repair this machine's state by removing the orphans, so the plugin
  definition wins again.
- **Part B:** add a read-only doctor guard, so the next directive migration that strands
  a user, project, or plugin xprompt is reported with its file path before a launch
  fails.

Part B does not change launch semantics.

## Part B — sase code change (implement first)

### B1. Share the retired `%wait` keyword diagnostics

In `src/sase/xprompt/_directive_collect.py`, the `name == "wait"` branch hard-codes four
migration messages, checked in this order: `runners`, `capacity`, `priority`, `p`.
Extract that logic into one helper that takes the parsed named args and returns the
first matching message or `None`. The collector must call the helper and raise
`DirectiveError` with the exact same text and precedence as today. Existing tests such
as `tests/test_directives_wait.py` and `tests/test_queue_directive.py` must pass
unchanged.

This stays on the Python side because the `%wait` keyword rejection already lives there.
Do not reimplement anything that the Rust `queue_directive` parser owns.

### B2. Add a public retired-directive scanner for definition bodies

Add a public function, for example `find_retired_directive_usages(content: str)`. Export
it through a non-underscore module so doctor code does not import private names; read
`symvision.md` memory before choosing placement, and keep modules under the toobig
limit. It returns a list of small frozen dataclass rows with these fields:

- 1-based `line`
- the directive `source` snippet
- the migration `message`

Behavior:

- Protect fenced code blocks and `%xprompts_enabled:false` regions first, reusing
  `protect_fenced_blocks` and `protect_disabled_regions`.
  `extract_static_clan_directive` in
  `src/sase/agent/multi_prompt_reference_directives.py` shows the pattern. Directives
  inside fences or `%alt` interiors must never be reported.
- For each directive match outside ignored regions, handle two cases:
  - **Deprecated directive names:** names in `_DEPRECATED_DIRECTIVES` (after alias
    canonicalization) report `_DEPRECATED_DIRECTIVE_MESSAGES[name]`.
  - **Parenthesized `%wait` / `%w`:** find the matching paren, parse args with the same
    `parse_args` the collector uses, and report the B1 helper's message when it returns
    one. If arg parsing raises `ValueError`, skip the directive rather than reporting
    it. Definition bodies contain Jinja (`{{ wait }}`, `{% if ... %}`), and this guard
    must not produce false positives on templated text.
- Map each hit's offset in the protected text back to a line number in the original
  content. Placeholder substitution must preserve line counts, or the mapping must
  account for it.
- Never raise. The function is diagnostic only.

### B3. New doctor check `config.xprompt_directives`

In `src/sase/doctor/checks_config_xprompts.py`, add
`check_config_xprompt_directives(context)`:

- Iterate `get_all_xprompts(context.project)` in sorted order and scan each
  `xprompt.content` with B2. Also scan prompt-part content of workflows from
  `get_all_workflows` if that is straightforward; otherwise leave workflows out and note
  it in the docstring.
- Report each hit as `"<name> (<source>):<line>: <source snippet> — <message>"`. For
  `<source>`, prefer `source_path_display(classify(...))` when cheap; otherwise use
  `xprompt.source_path`. An absolute home path or a `plugin:` identifier is what makes
  stale overrides obvious.
- Status `WARN` when there are hits, `OK` otherwise. `config.model_xprompts` also
  reports launch-blocking `DirectiveError`s as `WARN`. Cap `details` at
  `MAX_DETAIL_ROWS` and put all rows in `data`. The summary gives the count, e.g.
  `1 xprompt definition(s) use retired directive syntax`.
- `next_steps`: update the listed definition to the suggested replacement, or delete it
  if it is a stale personal or project copy shadowing a plugin or package xprompt of the
  same name. `sase xprompt show <name>` shows which definition wins. Then rerun
  `sase doctor -C config.xprompt_directives`.

Register it in `src/sase/doctor/checks_config.py`:

- Add a `CheckSpec` directly after `config.xprompt_definitions`, with id
  `config.xprompt_directives`, group `config`, and title `Retired xprompt directives`.
  Runner: `lambda: check_config_xprompt_directives(context)`.
- Add the matching `_check_config_xprompt_directives = ...` module alias alongside the
  others.

### B4. Tests

Add `tests/doctor/test_checks_config_xprompt_directives.py`, following the
monkeypatching style of `tests/doctor/test_checks_config_model_xprompts.py`. Add a
scanner unit test module under `tests/` near the other directive tests.

Cases that must be reported, each with its line number:

- `%wait(priority=20)`
- `%w(runners=2)`
- `%wait(capacity=1)`
- `%wait(p=3)`
- A deprecated directive name such as `%name:foo`
- A multi-segment swarm body (segments separated by `---`) where only the third segment
  carries `%wait(priority=20)`; the reported line must be correct.

Cases that must stay clean:

- `%q(p=20)`, `%queue(priority=20)`, and `%wait:agent %q:1`
- `%w(builder, time=5m) %q(1, p=20)`
- A fenced code block containing `%wait(priority=1)`
- The plugin's real templated form:
  `%wait:{{ wait }} {% endif %}%q(w=0.25{% if runners is not none %}, capacity={{ runners }}{% endif %}{% if priority is not none %}, priority={{ priority }}{% endif %})`
- `%wait(time=5m)`

Doctor check tests:

- A fake xprompt set containing one bad definition yields `WARN` with its name and
  source path in `details`.
- A clean set yields `OK`.
- Registration appears in the `config` group (`sase doctor -L` lists the id).

### B5. Docs

- `docs/xprompt.md`, `## Troubleshooting`: add a short paragraph on what to do when a
  launch fails with a directive migration error such as
  `%wait(priority=...) has moved to %queue`. Run
  `sase doctor -C config.xprompt_directives` to locate the definition file. Remember
  that a personal `~/sase/xprompts/<name>.md` (or project `sase/xprompts/`) copy shadows
  the plugin or package xprompt of the same name, and `sase xprompt show <name>` reveals
  which definition wins.
- `grep -rn "config.model_xprompts\|config.xprompt_definitions" docs` and add
  `config.xprompt_directives` next to them wherever doctor checks are enumerated.

### B6. Verify the code change

- `just install` if the workspace venv is stale, then `just check` (per
  `lint_and_test.md`).
- Before touching any file under `~/sase/xprompts/`, run
  `sase doctor -C config.xprompt_directives -v`. It must `WARN` and name
  `research_swarm (/Users/bbugyi/sase/xprompts/research_swarm.md)` with the
  `%wait(priority=20)` lines. This is the real-world acceptance check for the guard;
  record the output in your final summary.

## Part A — repair this Mac's xprompt state (after B6 passes)

These two files are outside any repository. Deleting them is the approved user-facing
action of this plan. Do not touch any other file in `~/sase/xprompts/`: `pick_plan.md`
and `sshot.yml` are live chezmoi-managed files.

1. Re-confirm that both files are still unmanaged orphans:
   - `chezmoi managed | grep '^sase/xprompts/'` must not list them.
   - `sase xprompt show research_swarm` must still report
     `definition /Users/bbugyi/sase/xprompts/research_swarm.md`.

   If either condition changed, stop and report instead of deleting.

2. Delete `~/sase/xprompts/research_swarm.md` (the shadowing, broken override) and
   `~/sase/xprompts/old_research_swarm.md` (the legacy swarm, also removed from dotfiles
   in `9f51c7b8`). Both are recoverable with
   `git show 9f51c7b8^:home/sase/xprompts/<file>` from the dotfiles repo.
3. Verify:
   - `sase xprompt show research_swarm` now reports the plugin definition
     (`plugin:sase_research_artifacts/research_swarm.md`) with `%q(w=0.25 ...)` queue
     lines and no `%wait(priority=...)`.
   - `sase doctor -C config.xprompt_directives` reports `OK`.
   - Dry-run the failing path without launching agents. Expand the user's prompt
     (`sase xprompt expand '#research_swarm:: smoke test'`, or `sase xprompt explain` if
     `expand` does not accept it). Pass each resulting swarm segment through
     `extract_static_clan_directive` / `extract_prompt_directives` in a short
     `python -c` snippet under the installed sase runtime, and confirm that no
     `DirectiveError` is raised.
   - Do **not** launch a real `#research_swarm`; it would start four paid agents.

## Out of scope and notes

- Do not edit the `sase-research-artifacts` plugin; upstream `5aaa244` is already
  correct and installed here.
- Do not edit dotfiles or chezmoi config. The orphan exists only because a deleted
  source file was never removed from the target. The new doctor check surfaces this
  class on any machine.
- Existing bead `sase-zo` (`sase repo open` rejects configured linked and sidecar repos
  when inventory uses the project display name) currently blocks
  `sase repo open chezmoi` and `sase repo open sase-research-artifacts` from numbered
  workspaces. It is not needed for this plan and must not be fixed here. If you need
  those repos read-only, open them as external refs (`gh:bbugyi200/dotfiles`,
  `gh:sase-org/sase-research-artifacts`).
- Stale bead follow-ups `sase-zp.1` and `sase-zl.12` asked for the plugin swarm to adopt
  `%queue`. That already shipped in plugin commit `5aaa244`; mention it in the final
  summary so the user can close them, but do not modify beads.
- Attributing launch-time `DirectiveError`s to their contributing xprompt source would
  be a deeper expansion-pipeline change and is not part of this plan.

## Final summary to the user

Explain the root cause chain above in a few sentences. Include the before and after
`config.xprompt_directives` doctor output. Confirm the two orphan files were removed and
`research_swarm` now resolves to the plugin. Tell the user they can relaunch the failed
prompt, which is saved in the ACE prompt stash with source `failed_launch`.
