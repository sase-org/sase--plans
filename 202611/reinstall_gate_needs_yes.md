---
tier: tale
title:
  Reinstall gates pass -y to just install-dev, and apollo's F11 reinstall is retried
goal:
  Agent-authored global-reinstall gates stop failing on the installer's non-TTY refusal,
  and apollo gets the sase-1s5 F11 install.
size: small
decisions:
  regate:
    ask:
      Also stage a corrected apollo reinstall gate (with -y) that carries the original
      F11 follow-up?
    default: true
    why: The failed gate is hash-pinned, so retrying it would be refused again
    answer: true
proposed_by: bbugyi200.athena.1ab
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 08:54:19
status: wip
---

# Reinstall gates fail because `just install-dev` refuses to run without a terminal

## Diagnosis

**What failed.** Agent `1a8--code` created gate `custom-92df387c-…` ("Reinstall apollo
for sase-1s5 F11"). Its `approve` command runs a heredoc over
`ssh -o BatchMode=yes apollo /bin/bash <<'REMOTE'`, which fetches both repos, checks
that the sase-1s5 commits are on origin, fast-forwards apollo's `sase-core` and `sase`
checkouts to `origin/master`, and then runs:

```sh
SASE_GLOBAL_INSTALL_BYPASS='Human approved sase-1s5 F11 rollout' just install-dev >&2
```

The installer engine (`tools/sase_install`, around the `_stdin_is_tty` check) refuses
every non-dry-run without a TTY unless it gets `-y`. It prints the plan panel and
`✗ just install-dev needs -y without a terminal`, then returns `EXIT_REFUSED` (2). Gate
commands never have a terminal: stdin carries the gate input JSON, the executor here was
a proc (`decision_receipt.json` → `execution_owner.kind = proc`), and the remote bash
reads its stdin from the heredoc. So the command was refused before it changed anything.
The installer worked exactly as designed. The bug is in the command the gate author
wrote.

**Root cause: every place that teaches agents the gate path leaves out `-y`.**

- `tools/_sase_install_env.py::agent_refusal_text`, the message an agent sees when it
  tries `just install-dev`, says "the user asked for a reinstall → propose it with
  /sase_gate and SASE_GLOBAL_INSTALL_BYPASS='<reason>'". Following it literally produces
  a command that is certain to fail inside a gate.
- `docs/contribute/development.md` ("`just install` and `just install-dev` are
  human-only…") describes the same recipe without `-y`.
- The `/sase_gate` skill (`src/sase/macros/skills/sase_gate.md`, the "Option and
  operation commands run with the gate bundle as their working directory…" paragraph)
  covers the command's environment but never says the command has no terminal.
- The decision record `global-install-is-human-only` mentions `-y` only under Cost, and
  only for "a human running a global install non-interactively".

**This has happened before.** On 2026-11-08, gate `custom-196bbcc8-…` (from
`sase-1s5.2`, "Reinstall apollo sase-core and restart gateway") failed with the same
non-TTY refusal. The earlier `sase-1jx-post-landing-reinstall` gate succeeded because
its command was `just install-dev --sync -y`.

**Side effect on apollo right now.** The failed `approve` command had already
fast-forwarded apollo's checkouts (sase `8f4e6503d3`, sase-core `5c0e49cf`) before the
installer refused. `sase` is an editable install, so apollo's Python is now the new
code. Its compiled `sase_core_rs.abi3.so` dates from 2026-11-09 10:27 UTC, before the
sase-core identity-filter commit, so the new `agent_names` / `source_run_ids` filters
are not active on apollo. The mixed state lasts until a successful `just install-dev -y`
rebuilds the extension.

## Changes

### 1. Agent refusal hint names the full gate command, `-y` included

In `tools/_sase_install_env.py`, rewrite `agent_refusal_text(prog)` so the reinstall
line gives the exact command a gate must run and says why `-y` is there. Keep the
existing two-column layout and the `just install-venv` line unchanged. Target text, for
`prog = "just install-dev"`:

```text
✗ just install-dev won't run inside a SASE agent — it replaces the global `sase` that every agent and the scheduler run.
  repair this workspace's .venv      just install-venv
  the user asked for a reinstall     propose a /sase_gate whose command runs
                                     SASE_GLOBAL_INSTALL_BYPASS='<reason>' just install-dev -y
                                     (gate commands have no terminal, so -y is required)
```

Build the command from `prog` and `BYPASS_ENV_VAR` (that is,
`f"{BYPASS_ENV_VAR}='<reason>' {prog} -y"`) so `just install` gets `just install -y`.
Leave `non_tty_text` and the `-y` / TTY semantics unchanged (see Out of scope).

### 2. Tests pin the hint

In `tests/sase_install/test_guards.py`:

- In `test_agent_env_refuses_with_placeholder_text` (dev mode), also assert
  `"SASE_GLOBAL_INSTALL_BYPASS='<reason>' just install-dev -y" in err`.
- In `test_monitor_id_refuses_for_pypi_mode`, also assert
  `"SASE_GLOBAL_INSTALL_BYPASS='<reason>' just install -y" in err`.

### 3. The `/sase_gate` skill says commands have no terminal

In `src/sase/macros/skills/sase_gate.md`, add to the end of the paragraph that begins
"Option and operation commands run with the gate bundle as their working directory…":

> Commands also run without a terminal. Stdin carries the gate input JSON, and a remote
> `ssh host sh <<'EOF'` reads its own heredoc, so anything that would prompt for
> confirmation must get its non-interactive flag (for example
> `SASE_GLOBAL_INSTALL_BYPASS='<reason>' just install-dev -y`). Otherwise it refuses or
> hangs after the reviewer has already approved.

Keep the wording close to this and do not restructure the skill. Do not run
`sase skill init --force` or `chezmoi apply` from this workspace: generated skills are
deployed only from a landed, clean tree (memory note `generated_skills.md`). The deploy
happens after landing, and the final report should say so.

### 4. Contributor docs and INSTALL.md

- `docs/contribute/development.md`: in the paragraph that begins "`just install` and
  `just install-dev` are human-only…", replace the gate sentence so it states the full
  command: `SASE_GLOBAL_INSTALL_BYPASS='<reason>' just install-dev -y`. Add that gate
  commands run without a terminal and the installer refuses any non-dry-run without one
  unless it gets `-y`. Keep the sentence "Agents never set the bypass on their own
  authority."
- `INSTALL.md`: after "Preview any global install first with `-n`
  (`just install-dev -n`)…", add one sentence: without a terminal (a script, cron job,
  or gate command), every run except `-n` needs `-y`. This is true even when the run
  would be a no-op, because the refusal comes before the no-op shortcut.

### 5. (`regate` = yes) Stage a corrected apollo reinstall gate

> [!decision] regate Do this last, after `sase tool run check` passes, because creating
> the gate is a turn-ending handoff. Use the `/sase_gate` skill and follow it exactly.

Base the request on the failed gate's `request.json`
(`~/.sase/interaction_requests/custom/custom-92df387c-63ce-48c1-bd63-75f7f437853b/request.json`
and its `commands/`). Read them; do not edit that bundle. Then make these changes:

- **`approve` command**: the same guarded flow (fetch both repos; `git grep` presence
  guards for `source_run_ids` in sase-core `crates/sase_core/src/agent_archive/wire.rs`
  and `agent_names` in sase `src/sase/ace/dismissed_bundle_index/_api.py`;
  `checkout master && pull --ff-only` for both repos; then the install and the gateway
  restart). The install line becomes
  `SASE_GLOBAL_INSTALL_BYPASS='Human approved sase-1s5 F11 rollout (replacement gate)' just install-dev -y >&2`.
  Keep `sase service proc restart gateway >&2` and the final
  `printf '{"status":"installed"}\n'`.
- **New operation `preview_install`** (`kind: run_command`, `display: text`): over the
  same ssh, run `cd "$SASE" && just install-dev -n 2>&1`. Wrap the captured text in one
  JSON object `{"summary": "...", "body": "<plan text>"}`, using `python3 -c` with
  `json.dumps` for safe escaping. Then the reviewer can see exactly what the approval
  will replace. Keep the existing `show_source_presence` operation.
- **`presentation`**: change the title to "Reinstall apollo for sase-1s5 F11 (retry with
  -y)", keep the `deployments` panel, icon, and chip, and set `origin_agent` to
  `1a8--code`. Notes must say:
  - this gate replaces `custom-92df387c-…`, which failed because `just install-dev` ran
    without `-y` in a terminal-less gate command;
  - apollo's checkouts were already fast-forwarded to `8f4e6503d3` / `5c0e49cf`, so
    apollo runs new Python against a stale `sase_core_rs` until this gate is approved;
  - in the old gate's "Gate execution failed" dialog, cancel it with `c`. Do not restart
    it with `R`: it would run the same pinned command and be refused again.
- **`turn`**: copy the failed gate's `turn.next.prompt` (the F11 convergence
  measurement) and the `turn.branches.reject.prompt` text word for word, with the same
  `pending_status` / `settled_status` / `workspace: inherit` / `fork: session`. There is
  no `next.model` override.
- Validate with `sase gate create --turn < request.json`, run in the foreground. Confirm
  the descriptor reports `"status": "staged"` before finishing through `/sase_final`.

Before staging, record
`sase bead note sase-1s5.3 "F11 apollo reinstall gate custom-92df387c failed: just install-dev ran without -y in a terminal-less gate command (installer non-TTY refusal, exit 2). Replacement gate staged with -y; installer/skill/docs guidance fixed to name -y."`

> [!decision] regate = no Skip section 5 and the bead note. The final report must tell
> the user that apollo still runs new Python against a stale `sase_core_rs` and that
> they need to run `just install-dev -y` (or `-n` first) on apollo themselves.

## Out of scope (deliberately)

- **Making the bypass imply `-y`.** `SASE_GLOBAL_INSTALL_BYPASS` answers "may an agent
  context run this?". `-y` answers "is a terminal-less run intended?". The decision
  record keeps them separate on purpose, and because the bypass is "deliberately
  easier", folding consent into it would weaken the stray-reinstall guard. The fix is
  guidance, not semantics.
- **Memory changes.** `decisions:global-install-is-human-only` stays true: its Cost
  already says non-interactive runs need `-y`. Decision records are immutable, and no
  memory note becomes false. No `/sase_memory_write` work.
- **Linting `just install*` commands in `sase gate create`.** The skill note and the
  refusal hint cover the general mechanism; a command-specific validator would be
  brittle.
- **Skill deployment** (`sase skill init --force`), which happens after landing.

## Verification

- `sase tool run check` (the agent default; do not run `check-full`). For a fast loop
  first, run
  `.venv/bin/pytest tests/sase_install/test_guards.py tests/sase_install/test_consequential.py tests/main/test_init_skills_source_content.py tests/main/test_init_skills_sources.py -q`
  (run `just install-venv` first if the workspace venv is stale).
- Manual check of the hint: inside the agent, `just install-dev -n` must refuse (exit 2)
  and print the new reinstall line, including
  `SASE_GLOBAL_INSTALL_BYPASS='<reason>' just install-dev -y`.
- `just fmt` before the check, so the Markdown edits are formatted.
- With `regate` = yes, the staged gate descriptor shows `"status": "staged"`. Do not run
  any of its bundle commands by hand.
