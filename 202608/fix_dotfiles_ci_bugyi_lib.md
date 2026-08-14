---
tier: tale
title: Install bugyi.sh in dotfiles CI so the tmux_ai_window bash tests pass
goal:
  The dotfiles CI test job installs the real ~/lib/bugyi.sh library before running
  bashunit, so all 41 bash tests pass and CI returns to green.
size: small
proposed_by: bbugyi200.athena.01r
create_time: 2026-08-14 15:48:06
status: done
---

# Fix dotfiles CI: make `~/lib/bugyi.sh` available to the bash test job

## Problem

Every CI run on `bbugyi200/dotfiles` (the GitHub repo behind the `chezmoi` linked repo)
has failed since 2026-08-14 13:34 UTC. Only the `test (ubuntu-latest)` job fails; the
`lint` job is green.

`actstat --repo bbugyi200/dotfiles -n 5` and `gh run list` show the first red run is
`a2f3cb1` ("feat: add grok and muse tmux AI launchers"), and every push since then is
red:

| run  | sha       | title                                                | result  |
| ---- | --------- | ---------------------------------------------------- | ------- |
| 5108 | `742692c` | fix(hammerspoon): repair Pomodoro startup bootstrap  | success |
| 5109 | `a2f3cb1` | feat: add grok and muse tmux AI launchers            | failure |
| 5110 | `7a709bd` | feat(tmux_ai_window): sort AI-agent menu rows        | failure |
| 5111 | `7360e4e` | fix: filter tmux AI menu to installed providers      | failure |
| 5112 | `0ba8f3f` | fix(tmux_ai_window): replace mapfile with while-read | failure |
| 5113 | `4bf5c92` | feat(tmux): show unavailable AI providers in menu    | failure |

Failing job log for the newest run:
<https://github.com/bbugyi200/dotfiles/actions/runs/31810976846/job/94801249979>

The failing step is `Run tests` → `just test` → `just test-bash`
(`bashunit ./tests/bash`). Result: `25 passed, 16 failed, 41 total`. **All 16 failures
are in `tests/bash/tmux_ai_window_test.sh`**; `bas_test.sh`, `codex_config_test.sh`,
`fake_test.sh`, `install_sase_github_test.sh`, and `tmp_trash_empty_test.sh` all pass,
as do the nvim, hammerspoon, and Python suites.

## Root cause

`home/bin/executable_tmux_ai_window` line 3 is:

```bash
source ~/lib/bugyi.sh
```

`bugyi.sh` is not part of the dotfiles repo. It is Bryan's personal Bash library, whose
canonical source is <https://github.com/bbugyi200/basher> (see
`xfiles/bugyi_bash_lib.txt`), installed per-user at `~/lib/bugyi.sh`. On a dev machine
that file exists, so the tests pass locally. On a GitHub runner `$HOME` is
`/home/runner` and nothing ever installs the library, so the `source` fails and the
script collapses:

```
executable_tmux_ai_window: line 3: /home/runner/lib/bugyi.sh: No such file or directory
executable_tmux_ai_window: line 10: usage: command not found
awk: fatal: cannot open file `/tmp/tmp.XXXX/tmux-calls.txt' for reading
```

The script never reaches `run "$@"` (its entry guard compares `SCRIPTNAME`, a variable
that `bugyi.sh` defines), so no tmux stub calls are ever recorded and every assertion in
the file fails.

`tmux_ai_window` is simply the **first script covered by a bashunit test that sources
`bugyi.sh`** — `bas`, `tmp_trash_empty`, and `install_sase_github` are all
self-contained, which is why CI was green before `a2f3cb1` added
`tests/bash/tmux_ai_window_test.sh`. The library surface the script actually uses is
`usage`, `die -x N`, `log::debug`, `log::warn`, and `SCRIPTNAME`; its own
`parse_cli_args` is defined locally.

Root cause confirmed by local reproduction (see Verification below): with a scratch
`HOME` the suite fails `16 of 17` — exactly the CI signature — and with `bugyi.sh`
exported into that scratch `HOME` it passes `17 of 17`.

## Fix

Install the real library in CI before tests run. `basher` is a **public** package on
PyPI (`uv tool install basher`, `uvx basher`) and ships the `export` subcommand that
writes the conventional unversioned per-user copy:

```
basher export DEST   # writes DEST/bugyi.sh; DEST defaults to ~/lib
```

The `test` job already runs `astral-sh/setup-uv@v4`, so `uvx` is on PATH and one line
does it. Using the real library (rather than a hand-written stub) keeps the tests
honest: assertions like "Launch unknown provider exits 2" depend on the real `die -x 2`
semantics.

## Repo access

This work happens in the **chezmoi/dotfiles** repo, not in the sase workspace checkout.
Open it with the `/sase_repo` skill first and use only the path it prints:

```bash
sase repo open chezmoi -r "Fix dotfiles CI test job that lacks ~/lib/bugyi.sh"
```

No files in the sase repo change, so `just check` in the sase workspace is **not**
required for this plan.

## Step 1 — Add the library-install step to the CI test job

In `.github/workflows/ci.yml`, in the `test` job, insert a step between
`- uses: astral-sh/setup-uv@v4` (currently line 118) and `- name: Run tests` (currently
line 120):

```yaml
- name: Install the bugyi.sh Bash library
  run: uvx basher export "$HOME/lib"
```

Notes:

- Do **not** add this to the `lint` job. `just lint` runs ruff/mypy/llscheck/luacheck/
  prettier/keep-sorted and never sources the library; that job is green.
- Leave the version unpinned. `bugyi.sh` is Bryan's own library and dev machines track
  the latest release (local `basher --version` is `0.2.0`, matching the packaged
  `BUGYI_VERSION`); testing against the same library the deployed scripts source is the
  point, and a genuinely breaking library release should turn CI red rather than hide.
- `ci.yml` is covered by `just lint-keep-sorted`
  (`git ls-files '*.yml' | keep-sorted --mode lint`), but it has no `keep-sorted` block
  directives, so step ordering is unconstrained.

## Step 2 — Fail fast with an actionable message when the library is missing

The missing library produced 16 unrelated-looking assertion failures plus dozens of
`awk: fatal:` lines. Add a preflight to `tests/bash/tmux_ai_window_test.sh` so a machine
without the library gets one clear instruction instead. bashunit runs
`set_up_before_script` once per file:

```bash
function set_up_before_script() {
  if [[ ! -f "${HOME}/lib/bugyi.sh" ]]; then
    printf '%s\n' \
      "error: tmux_ai_window sources ~/lib/bugyi.sh, which is not installed." \
      "Install it with: uvx basher export ~/lib" >&2
    exit 1
  fi
}
```

Verify bashunit (0.47.0 in CI) actually surfaces a non-zero `set_up_before_script` as a
failing run — confirm it by running the scratch-`HOME` command in Verification and
checking the process exit status is non-zero. If bashunit swallows it and reports
success, drop the `exit 1` approach and use a dedicated first test case instead, e.g.:

```bash
function test_bugyi_library_is_installed() {
  assert_file_exists "${HOME}/lib/bugyi.sh"
}
```

Check the exact assertion name with `bashunit doc file` before using it. Either shape is
acceptable; the requirement is a single, named, actionable failure.

## Verification

Run from the chezmoi checkout root printed by `sase repo open`.

1. Baseline (dev machine already has the library) — expect `17 passed, 17 total`:

   ```bash
   bashunit ./tests/bash/tmux_ai_window_test.sh
   ```

2. Reproduce the CI environment, then prove the fix — the second run is exactly what the
   new CI step does:

   ```bash
   FAKE_HOME="$(mktemp -d)"
   HOME="$FAKE_HOME" bashunit ./tests/bash/tmux_ai_window_test.sh; echo "exit=$?"
   # expect: the Step 2 preflight message and a non-zero exit
   uvx basher export "$FAKE_HOME/lib"
   HOME="$FAKE_HOME" bashunit ./tests/bash/tmux_ai_window_test.sh
   # expect: 17 passed, 17 total
   rm -rf "$FAKE_HOME"
   ```

   (Both halves of this were already run during diagnosis: `16 of 17` failed with an
   empty scratch `HOME`, `17 of 17` passed after `uvx basher export`.)

3. Whole bash suite — expect `41 passed, 41 total`:

   ```bash
   just test-bash
   ```

4. Lint gates that cover the edited YAML (and any Markdown, if the implementer adds
   docs):

   ```bash
   just lint
   ```

   `just test` in full additionally needs busted/nlua/neovim; run it if those are
   available locally, otherwise rely on CI for the nvim/hammerspoon/Python lanes, which
   are already green.

5. After the change lands on `master`, confirm the run goes green:

   ```bash
   actstat --repo bbugyi200/dotfiles -n 1
   ```

   Use `/sase_monitor` to wait for the run to settle rather than blocking a turn.

## Landing

Committing and pushing is the user's call. Do not commit unless the user asks; when they
do, use the `/sase_git_commit` skill (`sase stitch create`) from the chezmoi checkout.
CI on `master` stays red until the fix is pushed.

## Rejected alternatives

- **Stub `bugyi.sh` inside the test.** Writing a fake `~/lib/bugyi.sh` (defining
  `usage`, `die`, `log::debug`, `log::warn`, `SCRIPTNAME`) into a test-controlled `HOME`
  would remove the network dependency, but the suite asserts real library behavior —
  `die -x 2` exit codes for unknown providers, `die -x 1`-driven exits for missing ones.
  A stub encodes those contracts in the test itself, so a real library change that broke
  the deployed script would still pass CI.
- **Vendor `bugyi.sh` into the dotfiles repo** (`basher lib` / `basher vendor`, which
  copies the library to `lib/bugyi-<version>.sh` and rewrites the `source` line to a
  repo-relative path). Rejected: `home/bin/*` is deployed by chezmoi to `~/bin`, where a
  repo-relative source path does not exist. The runtime contract is
  `source ~/lib/bugyi.sh` and must stay that way.
- **Skip the test file when the library is absent.** That would make CI green while
  silently covering nothing — the failure is real, and CI is the only place the library
  is missing.
- **Change `executable_tmux_ai_window` to degrade without the library.** Out of scope
  and wrong: the script is correct as written; the CI environment is what is incomplete.

## Out of scope

- Any change to `home/bin/executable_tmux_ai_window` behavior.
- Any change to the `lint` job or to other bash tests.
- Caching the exported library between CI runs (the export is a sub-second PyPI fetch).
