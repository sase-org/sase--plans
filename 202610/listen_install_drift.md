---
tier: tale
title: Make sase-listen multi-machine installs self-diagnosing
goal: Every sase-listen install reports exactly which build it runs, a stale install
  fails with the one command that repairs it instead of a traceback, and doctor and
  publish flag renderer/feed-host drift, so publishing from any machine (the user's
  MacBook, or another user's laptop) fails loudly and fixably rather than silently
  or confusingly.
size: medium
decisions:
  doctor_skew:
    ask: How should `sase-listen doctor` treat a renderer/feed-host build mismatch?
    choices:
      fail: '`feed:host-build` FAILs (doctor exits 3) on any known build mismatch'
      warn: '`feed:host-build` stays ok but its detail names both builds and the fix'
    default: fail
    why: Drift is the failure class we hit; doctor is diagnostic-only and nothing
      gates on it.
    answer: fail
  publish_skew_warning:
    ask: Should a successful remote publish add a warning when the feed host's build
      differs?
    default: true
    why: Users rarely run doctor; the publish/render output is where drift must surface.
    answer: true
  rollout_installs:
    ask: Refresh the apollo, athena, and mac sase-listen installs to origin/master
      as part of this work?
    default: true
    why: All three run different stale commits today; the mac is 10 commits behind.
    answer: true
proposed_by: bbugyi200.apollo.5w
decided_by: auto
status: done
---

# Plan: Make sase-listen multi-machine installs self-diagnosing

## Diagnosis (verified 2026-10-08)

**The suspicion is wrong for the mac's current state.** The MacBook (`mac`, Tailscale
name `kellys-macbook-pro`, user `bbugyi`) _can_ render and publish to the feed on
`apollo`. All of the following were run on the mac:

- SSH works: `ssh -o BatchMode=yes apollo true` and
  `ssh -o BatchMode=yes apollo-do true` exit 0, from both a plain and an interactive
  login shell. The mac's `~/.ssh/id_rsa` is unencrypted and apollo accepts it. The mac's
  `~/.ssh/agent.sock` agent holds no identities, but ssh falls back to the key file, so
  the athena-style agent failure (plan `202610/podcast_feed_ssh_agent_fix.md`) does not
  apply.
- `sase-listen doctor` passes every check, including
  `feed:host (apollo via apollo · sase-listen 0.1.0 · 43 episodes)`, which is a real
  `sase-listen feed --json` round trip against apollo's live feed.
  `pass show gemini_cli_api_key` works non-interactively.
- A real Gemini render of an article URL (writer plus TTS) succeeds on the mac in about
  90 s.
- `sase-listen publish <id>` and `render --publish` both stream the episode over SSH,
  and `feed receive` on apollo publishes it. This was tested with and without a TTY. The
  test pointed the remote side at a scratch feed via a `SASE_LISTEN_SSH` wrapper, so the
  real feed was never touched.

**What actually broke, and still bites, is install drift.** The mac's history shows:

1. **Oct 2.** Cross-machine publishing did not exist yet. It landed Oct 4 in `0e03944`,
   and chezmoi first pointed `feed.host` at apollo the same evening in `9e8d272c`. The
   mac could only render locally (`-o`).
2. **Oct 5 10:04.** `sase-listen render <url> -e full` crashed before fetching. The
   mac's tool is an editable `uv tool` install. Its venv dated from Oct 1, but the
   checkout had been pulled past `e085c63`, which added `trafilatura`, `curl_cffi`, and
   `lxml`. A `git pull` updates the code of an editable install but never its
   dependencies. The venv's dist-info timestamps show the user reinstalled at 10:05:01.
   The retry fetched the article and was abandoned within about 40 s. The same render
   was then started on apollo at 10:05:48.
3. **Today.** No machine runs the same code, and nothing says so:

   | machine       | checkout  | behind origin/master | `--version` | missing deps |
   | ------------- | --------- | -------------------- | ----------- | ------------ |
   | apollo (host) | `355e649` | 6                    | 0.1.0       | –            |
   | athena        | `f64bc6c` | 2                    | 0.1.0       | –            |
   | mac           | `9da5586` | 10                   | 0.1.0       | pdfminer.six |
   | origin/master | `3ae7310` | 0 (v0.1.1, on PyPI)  | –           | –            |
   - The mac lacks PDF and arXiv sources (`355e649`, `eb05e7c`) and Gemini
     `content_blocked` split-and-retry (`ff7a90f`). So some renders that work on athena
     and apollo fail on the mac.
   - Apollo, the feed host, lacks same-title supersede (`f8154ad`).
   - A plain `git pull` on the mac would crash any PDF render with
     `ModuleNotFoundError: pdfminer` until a reinstall.
   - `__version__` comes from install-time dist metadata, so every commit between
     releases, and every editable install, reports the same `0.1.0`. `doctor` prints the
     host's version but never compares it with the local one.
   - The research-audio xprompt already works around this ad hoc. It sniffs
     `sase-listen render --help` for `--generated-cover` before trusting an install.

The user's general-purpose concern is real, but it is not about SSH or where the feed
lives. Multi-machine publishing means two or more independently installed copies of
sase-listen must cooperate, and today that drift is invisible and fails with tracebacks.
Anyone who installs on a laptop and a server hits the same thing. That is true whether
they use PyPI releases (different releases per machine), `uv tool install git+…` (same
version string on every commit, as `docs/multi-machine.md` recommends), or editable
checkouts (deps never refresh).

The fix has three parts:

- Give every install a precise build identity.
- Turn a stale environment into a clear exit-3 error naming the repair command.
- Compare renderer and host builds wherever they meet.

We are **not** redesigning the transport. SSH publish works from the mac.

## Ground rules for the implementer

- Open the repo with `sase repo open sase-listen -r "<reason>"` and work only in the
  printed path. Read its `AGENTS.md` first. Run verification as `sase tool run check`
  there, never bare `just check`.
- The no-sase-import rule applies. All new code is stdlib-only plus existing sase-listen
  modules. Do not edit `pyproject.toml`, `uv.lock`, or the `mkdocs.yml` nav. None of
  them are needed.
- Never print secrets: API keys, pass output, the feed token, or the token path segment
  of the feed URL.
- Do not publish test episodes to the real feed. For end-to-end checks, use
  `--no-publish` with scratch `XDG_DATA_HOME`/`XDG_CACHE_HOME`/`XDG_STATE_HOME` dirs. Or
  reuse the diagnosis trick: a `SASE_LISTEN_SSH` wrapper that prefixes the remote
  command with
  `export SASE_LISTEN_CONFIG=<scratch>/config.yml XDG_DATA_HOME=… XDG_STATE_HOME=…;`, so
  `feed receive` publishes into a scratch feed. Delete scratch dirs afterwards.

## Changes (sase-listen repo)

### 1. New stdlib-only module `src/sase_listen/buildinfo.py`

This module computes, once per process (lazily, `functools.cache`), a frozen
`BuildInfo`:

- `install`: one of `index`, `vcs`, `editable`, `local`, or `unknown`. It comes from PEP
  610 `direct_url.json` via
  `importlib.metadata.distribution("sase-listen").read_text("direct_url.json")`:
  - absent: `index`
  - `dir_info.editable: true`: `editable`
  - `vcs_info`: `vcs`
  - other `dir_info` or `archive_info`: `local`
  - `PackageNotFoundError`: `unknown`

  This was verified: a uv editable tool install writes
  `{"url":"file:///…","dir_info":{"editable":true}}`.

- `source`: the editable checkout path (from the `file://` URL) or the VCS URL;
  otherwise empty.
- `version`: for editable installs, `[project].version` read with `tomllib` from
  `<source>/pyproject.toml`, which is the truth. Otherwise the dist metadata version.
  `metadata_version` always holds the dist metadata version.
- `commit` and `dirty`:
  - `vcs`: `vcs_info.commit_id`.
  - `editable`: `git -C <source> rev-parse HEAD` and
    `git -C <source> status --porcelain --untracked-files=no`, each with about a 2 s
    timeout. Any failure yields `""` or `False`.
- `missing_dependencies` (editable only): names from `<source>/pyproject.toml`
  `[project].dependencies` whose distribution is not installed.
  - Parse the leading name with a regex and normalize per PEP 503.
  - Look each one up with `importlib.metadata.distribution`.
  - **Skip requirements with environment markers**, since evaluating them needs
    `packaging`.
  - Never report a false positive.
- `stale`: true when `missing_dependencies` is non-empty, or when an editable install's
  `metadata_version != version` (the install predates a version bump).
- `display()`: `0.1.1` for index installs, `0.1.1 (git 3ae7310)` for vcs installs, and
  `0.1.1 (editable @ 3ae7310)` for editable installs, with a `, dirty` suffix when
  dirty.
- `to_json()`: a dict with the fields above.
- `upgrade_command()`: one copy-pasteable command.
  - When `Path(sys.prefix, "uv-receipt.toml")` exists (a uv tool env), use the absolute
    `uv` path from `shutil.which("uv")`, falling back to `uv`, so the command works
    verbatim over non-interactive SSH:
    - editable:
      `git -C <source> pull --ff-only && <uv> tool upgrade --reinstall sase-listen`.
      This was verified: `uv tool upgrade --reinstall` on a stale editable install
      installed the newly added `pdfminer.six`, kept the receipt's editable path,
      extras, and Python, and refreshed metadata to 0.1.1.
    - index: `<uv> tool upgrade sase-listen`
    - vcs: verify in a scratch `UV_TOOL_DIR`/`UV_TOOL_BIN_DIR` whether
      `uv tool upgrade --reinstall sase-listen` moves a branch-tracking git install to
      the new tip. If it does, use that. If not, use
      `<uv> tool install --force <source url>`, which matches `_too_old_error`.
  - Otherwise, use a generic sentence: reinstall with the installer you used, for
    example `pipx reinstall sase-listen`.
- `compare_builds(local: dict, remote: dict | None)`: returns a small result naming one
  of `same`, `differ`, `remote_older`, `local_older`, or `remote_unknown`, plus a human
  sentence.
  - Release versions are compared as numeric tuples. If parsing fails, the result is
    "differ, order unknown".
  - Equal versions with both commits known and different are `differ`.
  - Equal versions with a commit missing on either side are `same`.
  - A remote with no build info is `remote_unknown`. Every such host predates this
    change, so it is older.

`buildinfo` must import nothing outside the stdlib and `sase_listen.errors`, so it still
works when third-party dependencies are missing. It must never raise.

### 2. Stale-environment guard at the console-script entry

`src/sase_listen/cli/__init__.py` currently imports `cli.app` eagerly, and `app` imports
every command module (and so `lxml` and friends) at import time. Change it so
`main(argv=None)` imports `sase_listen.cli.app` **inside** a `try`, then runs it:

- Catch `ImportError` (which includes `ModuleNotFoundError`) whose `exc.name` root is
  not `sase_listen`, whether it is raised at import or during the command, such as a
  lazy `pdfminer` import mid-render.
- Convert it into an exit-3 (`ExitCode.CONFIG`) error:
  - Message:
    `sase-listen cannot import '<module>': this installation's Python environment is out of date with its code`
    (adapt the wording for "cannot import name X from Y").
  - Hint: `Reinstall: <BuildInfo.upgrade_command()>`.
- Re-raise ImportErrors from `sase_listen.*` modules, since those are real bugs.
- With `--json` in argv (use `sys.argv[1:]` when `argv` is None), print
  `{"ok": false, "error": {"code": 3, "message": …, "hint": …}}` to stdout. This is the
  shape `feedhost._raise_remote_error` already parses, so a stale **host** comes back to
  the client as `on apollo: sase-listen cannot import …` plus the host's own repair
  command. Otherwise, print to stderr in the same style other command errors use.
- Inline the JSON dict. Do not import `pipeline.error_to_json`, which is heavy.

The installed console-script shims call `sase_listen.cli:main`, so existing installs
pick this up with no reinstall. Leave the shadowed `src/sase_listen/cli.py` alone (see
Out of scope).

### 3. `--version` shows the build

In `cli/app.py`, replace the static `action="version"` string with a small custom
argparse `Action` that computes `BuildInfo` only when `--version` is passed. It prints
`sase-listen <display()>`, for example `sase-listen 0.1.1 (editable @ 3ae7310)`. The
first two tokens stay `sase-listen <release>`. Never compute build info while merely
building the parser.

### 4. Report the build across the SSH boundary

- `feed.py` feed status payload (the dict with `sase_listen_version` and
  `receive_protocol`, around line 863): add
  `"sase_listen_build": buildinfo.current().to_json()`. Keep `sase_listen_version`
  unchanged for compatibility.
- `feedhost.receive_episode` result: add the same `sase_listen_build` and
  `sase_listen_version`.

### 5. `doctor`

- Replace the trailing `version` check's detail with `display()`, keeping the check
  name.
- Add an `install` check, ok when not `stale`:
  - Detail: install kind and source.
  - When stale, the detail names the missing dependencies or the metadata/source version
    mismatch, plus `upgrade_command()`.
- Remote role (`_remote_host_check`):
  - Show the host's `display` in the `feed:host` detail.
  - Add a `feed:host-build` check driven by `compare_builds`:
    - It must FAIL when the host's `receive_protocol` is lower than the local
      `RECEIVE_PROTOCOL`.
    - It must FAIL when the host reports `stale`. The detail includes the host's own
      `upgrade_command`, prefixed with `ssh <dest> '…'`.
    - The detail always names both builds and, when known, which side is older and how
      to upgrade it.
    - Hosts without build info get a generic "upgrade sase-listen on <host>" hint that
      points at the new docs section.

> [!decision] doctor_skew = fail `feed:host-build` is **not ok** (doctor exits 3) for
> `differ`, `remote_older`, `local_older`, or `remote_unknown`. It is ok only for
> `same`.

> [!decision] doctor_skew = warn `feed:host-build` stays ok for version or commit
> mismatches, and its detail starts with `builds differ:`. It fails only for a protocol
> mismatch or a stale host.

### 6. Surface drift on normal publishes

> [!decision] publish_skew_warning In `feedhost.publish_any` (remote role, after a
> successful receive), run `compare_builds` against the payload's `sase_listen_build`.
> If they are not `same`, attach one warning, for example
> `feed host apollo runs sase-listen 0.1.0 (editable @ 355e649); this machine runs 0.1.1 (editable @ 3ae7310) — run sase-listen doctor`.
>
> - Do the same in `flush_pending`'s pushes only if it falls out naturally.
> - The render pipeline's auto-publish path appends it to `RenderResult.warnings`.
> - The `publish` command shows it: on stderr in human mode, and in a `warnings` list in
>   its JSON.
> - Publishing still succeeds with exit 0.

### 7. Docs

- `docs/multi-machine.md`: add a section **Keep machines in step**:
  - Every machine runs its own copy, and the host runs `feed receive`.
  - `--version` and `doctor` show the exact build.
  - `doctor`'s `feed:host-build` check, and the publish warning if enabled, flag drift.
  - Upgrade commands per install kind.
  - The warning that a plain `git pull` updates an editable install's code but not its
    dependencies, so always follow it with `uv tool upgrade --reinstall sase-listen`.

  Also update the per-machine checklist so its final step is "run `sase-listen doctor`
  on the renderer; `feed:host` and `feed:host-build` must be ok". Upgrade-order guidance
  (host first) stays.

- `docs/troubleshooting.md`: add entries for:
  - `cannot import '<module>' … out of date with its code` (exit 3, and the remote
    `on <host>:` variant).
  - `feed:host-build` / "feed host runs sase-listen X; this machine runs Y".
- `docs/cli.md`: update it wherever it documents `--version` or the doctor checks.
- Add a changelog entry only if `docs/changelog.md` has an unreleased section by
  convention. `CHANGELOG.md` is release-please-owned, so do not edit it.

### 8. Tests

- New `tests/test_buildinfo.py`, with distributions, `direct_url.json`, `sys.prefix`,
  and git faked or monkeypatched:
  - Each install kind is detected.
  - Editable version and dependencies are read from a tmp `pyproject.toml`.
  - Missing dependencies are detected, with markers skipped.
  - The metadata/source mismatch sets `stale`.
  - `display()` strings.
  - `upgrade_command()` per kind, with and without `uv-receipt.toml`.
  - Every `compare_builds` outcome.
  - Git failures and timeouts degrade to empty values.
- CLI guard:
  - An `ImportError(name="lxml")` raised from the app import, and one raised from a
    command, each exit 3, with the JSON shape under `--json` and a hint containing the
    upgrade command.
  - A `sase_listen.*` ImportError propagates.
- `--version` output starts with `sase-listen <version>`.
- `doctor`, extending its existing tests with `run_remote` monkeypatched:
  - Same build passes.
  - Differing builds, a host with no build info, a stale host, and a lower protocol
    behave per the `doctor_skew` decision.
  - A local stale install fails the `install` check.
- `tests/test_feedhost.py`:
  - The receive result and feed status include `sase_listen_build`.
  - A `publish_any` skew warning appears when builds differ and is absent when they
    match (per `publish_skew_warning`).
  - `_raise_remote_error` surfaces a stale host's JSON error with its hint.
- Run `sase tool run check` until green.

## Rollout of the current master (independent of the code change)

> [!decision] rollout_installs Bring each machine's editable install to the current
> `origin/master` (`3ae7310`, v0.1.1) and repair its environment. This fixes today's
> drift right away. The mac gains PDF/arXiv/content_blocked handling and `pdfminer.six`;
> apollo, the feed host, gains same-title supersede.
>
> Do the host first: apollo, then athena, then the mac. The mac is best-effort, since it
> is offline unless its lid is open. On each machine:
>
> 1. Inspect `~/projects/github/sase-org/sase-listen`. Proceed only if it is on `master`
>    with a clean tree and `git pull --ff-only` succeeds. Otherwise, skip and report.
> 2. Run `uv tool upgrade --reinstall sase-listen`. Use absolute paths over
>    non-interactive SSH, where `~/.local/bin` and Homebrew are not on `PATH`:
>    `~/.local/bin/uv`, or wrap the command in `zsh -lic '…'` on the mac.
> 3. Run `sase-listen --version` and `sase-listen doctor`. All checks must be ok. On the
>    mac and athena, `feed:host` must reach apollo.
> 4. On the mac only, prove a render with
>    `sase-listen render <installed package>/data/demo.md -n tone --no-publish` in
>    scratch XDG dirs. Delete them afterwards.
>
> A publish that races the host reinstall lands in the publisher's outbox. Check each
> machine's `sase-listen feed` output for `Outbox: N pending` and run
> `sase-listen publish --pending` if needed.
>
> This turn's own commit reaches `origin/master` only after the turn ends. The final
> response must therefore tell the user to repeat steps 1–2 on each machine, host first,
> once it lands. After that, the new `doctor` confirms the machines match.

## Verification

- `sase tool run check` is green in the sase-listen repo.
- In the repo's dev environment, run `uv run sase-listen --version` and
  `uv run sase-listen doctor`. Confirm the build display and the new checks. With
  `rollout_installs`, apollo's host still runs the pre-change build, so the
  `feed:host-build` result must match the `doctor_skew` decision.
- Simulate a stale env in a scratch `UV_TOOL_DIR`/`UV_TOOL_BIN_DIR`:
  1. Make an editable tool install from a temporary clone of an older commit (for
     example `9da5586`).
  2. Check the clone out to the working tree's HEAD.
  3. Run `sase-listen render <a PDF> --no-publish --json`, and run an import of the
     `trafilatura`-dependent path with that dependency uninstalled. Each must exit 3
     with the `uv tool upgrade --reinstall sase-listen` hint, not a traceback.
  4. The hint command repairs it.

  Clean up afterwards.

## Out of scope (possible follow-up task beads via `/sase_new_task`)

- Redesigning the feed transport (object storage or an HTTP upload endpoint). SSH
  publish is verified working from the mac.
- Auto-refreshing installs from dotfiles or `sase update` on each machine.
- `feed_role` hostname matching (`socket.gethostname()`) on macOS hosts, whose hostnames
  can differ from Tailscale and SSH names. It only matters if a Mac becomes the feed
  host.
- The hardcoded `$HOME/.local/bin` remote `PATH` in `feedhost.ssh_argv`.
- The dead `src/sase_listen/cli.py`, which the `cli/` package shadows.
- Replacing the research-audio xprompt's `--help` feature sniffing with build info.
